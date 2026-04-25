<script>
  import { createEventDispatcher, onDestroy, onMount } from 'svelte';

  export let api;
  export let index = 0;
  export let baseUrl;

  const dispatch = createEventDispatcher();

  let cardEl;
  let bodyEl;
  let revealed = false;
  let isOpen = false;
  let isSending = false;
  let response = { visible: false };
  let responseImageUrl = '';
  let formValues = {};
  let paramsSignature = '';

  $: method = (api?.method || 'GET').toUpperCase();
  $: isActive = String(api?.isAktif ?? true);
  $: params = Array.isArray(api?.parameters) ? api.parameters : [];
  $: {
    const nextSignature = params
      .map((param) => `${param.name}:${param.defaultValue ?? ''}`)
      .join('|');
    if (nextSignature !== paramsSignature) {
      paramsSignature = nextSignature;
      const nextValues = {};
      for (const param of params) {
        nextValues[param.name] = param.defaultValue ?? '';
      }
      formValues = nextValues;
    }
  }

  function updateParam(name, value) {
    formValues = { ...formValues, [name]: value };
  }

  function toggleCard() {
    if (!bodyEl) return;

    if (isOpen) {
      bodyEl.style.height = `${bodyEl.scrollHeight}px`;
      requestAnimationFrame(() => {
        bodyEl.style.height = '0px';
      });
      isOpen = false;
      return;
    }

    bodyEl.style.height = '0px';
    isOpen = true;
    requestAnimationFrame(() => {
      bodyEl.style.height = `${bodyEl.scrollHeight}px`;
    });

    const handleEnd = () => {
      if (isOpen) bodyEl.style.height = 'auto';
      bodyEl.removeEventListener('transitionend', handleEnd);
    };

    bodyEl.addEventListener('transitionend', handleEnd);
  }

  function escHtml(input) {
    return String(input)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  }

  function syntaxHighlight(jsonText) {
    const safe = escHtml(jsonText);
    return safe.replace(
      /("(\\u[a-zA-Z0-9]{4}|\\[^u]|[^\\\"])*"(\s*:)?|\b(true|false|null)\b|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?)/g,
      (match) => {
        let style = 'color:#98D8C8';
        if (/^"/.test(match)) {
          style = /:$/.test(match) ? 'color:var(--accent-2)' : 'color:#A8D8A8';
        } else if (/true|false/.test(match)) {
          style = 'color:var(--post)';
        } else if (/null/.test(match)) {
          style = 'color:var(--text-muted)';
        } else {
          style = 'color:#D8A8FF';
        }

        return `<span style="${style}">${match}</span>`;
      }
    );
  }

  async function sendRequest() {
    const currentParams = {};
    for (const [name, value] of Object.entries(formValues)) {
      currentParams[name] = String(value).trim();
    }

    const apikey = currentParams.apikey || '';
    delete currentParams.apikey;

    let url = `${baseUrl}${api.endpointPath}`;
    const options = { method };

    response = {
      visible: true,
      badge: '',
      badgeClass: '',
      type: 'html',
      html: '<span style="color:var(--text-dim)">Waiting for response...</span>'
    };

    isSending = true;

    if (responseImageUrl) {
      URL.revokeObjectURL(responseImageUrl);
      responseImageUrl = '';
    }

    try {
      if (method === 'GET') {
        const query = new URLSearchParams({ ...currentParams, apikey });
        url += `?${query.toString()}`;
      } else {
        options.headers = {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apikey}`
        };
        options.body = JSON.stringify(currentParams);
      }

      const res = await fetch(url, options);
      const contentType = res.headers.get('content-type') || '';

      if (contentType.includes('image')) {
        const blob = await res.blob();
        responseImageUrl = URL.createObjectURL(blob);
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'image',
          imageUrl: responseImageUrl
        };
      }  else if (contentType.includes('json')) {
        const json = await res.json();
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'html',
          html: syntaxHighlight(JSON.stringify(json, null, 2))
        };
      }  else if (contentType.includes('video')) {
  const blob = await res.blob();
  const videoUrl = URL.createObjectURL(blob);

  response = {
    visible: true,
    badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
    badgeClass: res.ok ? 'status-ok' : 'status-err',
    type: 'video',
    videoUrl: videoUrl
  };

  } else {
        const text = await res.text();
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'html',
          html: escHtml(text)
        };
      }
    } catch (error) {
      response = {
        visible: true,
        badge: 'Error',
        badgeClass: 'status-err',
        type: 'html',
        html: escHtml(error?.message || 'Request failed')
      };
    } finally {
      isSending = false;
      if (bodyEl && bodyEl.style.height !== 'auto') {
        bodyEl.style.height = `${bodyEl.scrollHeight}px`;
      }
    }
  }

  async function copyEndpoint() {
    const queryParams = {};
    for (const [name, value] of Object.entries(formValues)) {
      const trimmed = String(value).trim();
      if (trimmed) queryParams[name] = trimmed;
    }

    const query = new URLSearchParams(queryParams);
    const url = `${baseUrl}${api.endpointPath}${query.toString() ? `?${query.toString()}` : ''}`;

    try {
      await navigator.clipboard.writeText(url);
      dispatch('toast', { message: 'URL copied to clipboard' });
    } catch {
      dispatch('toast', { message: 'Failed to copy URL' });
    }
  }

  onMount(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            revealed = true;
            observer.unobserve(entry.target);
          }
        }
      },
      { threshold: 0.05, rootMargin: '0px 0px -40px 0px' }
    );

    if (cardEl) observer.observe(cardEl);

    return () => observer.disconnect();
  });

  onDestroy(() => {
    if (responseImageUrl) URL.revokeObjectURL(responseImageUrl);
  });
</script>

<div
  bind:this={cardEl}
  class="api-card"
  class:revealed
  class:open={isOpen}
  style={`transition-delay:${index * 40}ms`}
>
  <div
    class="card-head"
    role="button"
    tabindex="0"
    on:click={toggleCard}
    on:keydown={(event) => (event.key === 'Enter' || event.key === ' ') && toggleCard()}
  >
    <span class={`card-method method-${method}`}>{method}</span>
    <div class="card-info">
      <div class="card-title"><span class={`card-status-dot-${isActive}`}></span>{api.title}</div>
      <div class="card-endpoint">{baseUrl}{api.endpointPath}</div>
      <div class="card-desc">{api.description || ''}</div>
    </div>
    <button class="card-toggle" type="button" on:click|stopPropagation={toggleCard}>
      <i class="fa-solid fa-plus"></i>
    </button>
  </div>

  <div bind:this={bodyEl} class="card-body">
    <div class="card-body-inner">
      {#if params.length > 0}
        <div class="params-section">
          <div class="params-title">Parameters</div>
          <table class="params-table">
            <thead>
              <tr><th>Name</th><th>Type</th><th>Description</th></tr>
            </thead>
            <tbody>
              {#each params as param}
                <tr>
                  <td>
                    <span class="param-name">{param.name}</span>
                    <span class={param.required ? 'param-required' : 'param-optional'}>
                      {param.required ? 'required' : 'optional'}
                    </span>
                  </td>
                  <td><span class="param-type">{param.type || 'string'}</span></td>
                  <td class="param-desc">{param.description || ''}</td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}

      <div class="try-section">
        <div class="try-title"><i class="fa-solid fa-bolt"></i>Try It Out</div>
        <div class="try-form">
          {#each params as param}
            {@const inputId = `try-${api.id}-${param.name}`.replace(/[^a-zA-Z0-9_-]/g, '-')}
            <div class="try-field">
              <label class="try-label" for={inputId}>{param.name}{param.required ? ' *' : ''}</label>
              <input
                id={inputId}
                class="try-input"
                type="text"
                placeholder={param.defaultValue || param.description || param.name}
                value={formValues[param.name] ?? ''}
                on:input={(event) => updateParam(param.name, event.currentTarget.value)}
              />
            </div>
          {/each}
        </div>

        <div class="try-actions">
          <button class="btn btn-primary" type="button" class:loading={isSending} on:click={sendRequest}>
            {#if isSending}
              <i class="fa-solid fa-circle-notch spin"></i>Sending...
            {:else}
              <i class="fa-solid fa-paper-plane"></i>Send
            {/if}
          </button>
          <button class="btn btn-ghost" type="button" on:click={copyEndpoint}>
            <i class="fa-regular fa-copy"></i>Copy URL
          </button>
        </div>

        <div class="try-response" style:display={response.visible ? 'block' : 'none'}>
          {#if response.visible}
            <div class="response-label">
              <span>Response</span>
              {#if response.badge}
                <span class={`response-status ${response.badgeClass}`}>{response.badge}</span>
              {/if}
            </div>
            {#if response.type === 'image' || response.type === 'gif'}
              <img
                src={response.imageUrl}
                class="response-img"
                alt="Response"
              />
            
            {:else if response.type === 'video'}
              <video
                class="response-video"
                controls
                autoplay
                loop
                muted
                playsinline
                aria-label="Animated preview"
              >
                <source src={response.videoUrl} />
                Browser not supported.
              </video>
            {:else}
              <div class="response-body">
                {@html response.html}
              </div>
            {/if}
          {/if}
        </div>
      </div>
    </div>
  </div>
</div>
