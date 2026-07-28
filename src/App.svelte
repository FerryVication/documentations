<script>
  import { afterUpdate, onDestroy, onMount, tick } from 'svelte';
  import ApiCard from './components/ApiCard.svelte';
  import Sidebar from './components/Sidebar.svelte';

  const CONFIG_URL = import.meta.env.VITE_CONFIG_URL;
  const STATS_URL  = import.meta.env.VITE_STATS_URL;
  const BASE_URL   = import.meta.env.VITE_BASE_URL;
  const MIN_SPLASH_MS = 1800;

  let status = 'loading';
  let errorMessage = '';
  let categories = [];
  let searchTerm = '';
  let activeCat = 'hero';
  let sidebarOpen = false;
  let sidebarCollapsed = false;
  let showSplash = true;
  let splashHiding = false;
  let showBackTop = false;
  let toastMessage = '';
  let toastVisible = false;
  let themeWipeActive = false;

  // Remote stats dari STATS_URL
  let totalRequestsAllTime = null;
  let averagePerDay = null;
  let chartHistory = [];
  let totalRequests30Days = null;
  let busiestDay = null;
  let monthStats = null;

  let toastTimer;
  let splashTimer;
  let splashRemoveTimer;
  let scrollSpyObserver;
  let scrollSpyTimer;

  const slugify = (input) => input.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '');

  const isValidConfig = (payload) =>
    Array.isArray(payload) &&
    payload.every(
      (category) =>
        category &&
        typeof category.name === 'string' &&
        Array.isArray(category.apis)
    );

  const isLightPreferred = () => window.matchMedia('(prefers-color-scheme: light)').matches;
  const isMobileViewport = () => window.innerWidth <= 768;

  // Format angka besar: 3506083 → 3.5M, 18304 → 18.3K, dst.
  function fmt(n) {
    if (n === null || n === undefined) return '—';
    if (typeof n !== 'number') return n;
    if (n >= 1_000_000) return (n / 1_000_000).toFixed(1) + 'M';
    if (n >= 1_000) return (n / 1_000).toFixed(1) + 'K';
    return n.toLocaleString('id-ID');
  }

  function applyInitialTheme() {
    const savedTheme = localStorage.getItem('theme');
    if (savedTheme) {
      document.documentElement.dataset.theme = savedTheme;
      return;
    }

    if (isLightPreferred()) {
      document.documentElement.dataset.theme = 'light';
    }
  }

  function toggleTheme() {
    if (themeWipeActive) return;

    const html = document.documentElement;
    const nextTheme = html.dataset.theme === 'dark' ? 'light' : 'dark';
    const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

    if (prefersReducedMotion || typeof document.startViewTransition !== 'function') {
      html.dataset.theme = nextTheme;
      localStorage.setItem('theme', nextTheme);
      return;
    }

    themeWipeActive = true;
    html.classList.add('theme-transition');

    const transition = document.startViewTransition(() => {
      html.dataset.theme = nextTheme;
      localStorage.setItem('theme', nextTheme);
    });

    transition.finished.finally(() => {
      html.classList.remove('theme-transition');
      themeWipeActive = false;
    });
  }

  function toggleSidebar() {
    if (isMobileViewport()) {
      sidebarOpen = !sidebarOpen;
      return;
    }

    sidebarCollapsed = !sidebarCollapsed;
  }

  function closeSidebar() {
    sidebarOpen = false;
  }

  function showToast(message) {
    toastMessage = message;
    toastVisible = true;

    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => {
      toastVisible = false;
    }, 2500);
  }

  function onNavigate(event) {
    const { slug } = event.detail;
    activeCat = slug;
    closeSidebar();

    const section = document.getElementById(slug);
    if (section) {
      section.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  }

  function startSplashExit(delayMs) {
    clearTimeout(splashTimer);
    clearTimeout(splashRemoveTimer);

    splashTimer = setTimeout(() => {
      splashHiding = true;
      splashRemoveTimer = setTimeout(() => {
        showSplash = false;
      }, 700);
    }, Math.max(0, delayMs));
  }

  async function fetchConfig() {
    const response = await fetch(CONFIG_URL, { cache: 'no-store' });
    if (!response.ok) {
      throw new Error(`Failed to fetch config (${response.status})`);
    }

    const payload = await response.json();
    if (!isValidConfig(payload)) {
      throw new Error('Invalid configure.json format');
    }

    return payload;
  }

  async function fetchRemoteStats() {
    try {
      const res = await fetch(STATS_URL);
      if (!res.ok) return;
      const data = await res.json();
      if (data.success) {
        totalRequestsAllTime = data.totalRequestsAllTime;
        averagePerDay = data.averagePerDay;
        chartHistory = data.last30Days || [];
        totalRequests30Days = data.totalRequests30Days || null;
        busiestDay = data.busiestDay || null;
        monthStats = data.month || null;
      }
    } catch (_) {
      // silent fail — stats tidak kritis
    }
  }

  async function loadData() {
    status = 'loading';
    errorMessage = '';
    splashHiding = false;
    showSplash = true;

    const startedAt = Date.now();

    try {
      const payload = await fetchConfig();
      categories = payload;
      status = 'ready';
      activeCat = 'hero';

      await tick();
      scheduleScrollSpy();

      const elapsed = Date.now() - startedAt;
      startSplashExit(MIN_SPLASH_MS - elapsed);
    } catch (error) {
      status = 'error';
      errorMessage = error?.message || 'Failed to load API documentation data.';

      const elapsed = Date.now() - startedAt;
      startSplashExit(MIN_SPLASH_MS - elapsed);
    }
  }

  function handleScroll() {
    showBackTop = window.scrollY > 400;
  }

  function initScrollSpy() {
    if (scrollSpyObserver) {
      scrollSpyObserver.disconnect();
    }

    const sections = document.querySelectorAll('.section-anchor');
    if (!sections.length) return;

    scrollSpyObserver = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            activeCat = entry.target.id;
          }
        }
      },
      { rootMargin: '-30% 0px -60% 0px' }
    );

    for (const section of sections) {
      scrollSpyObserver.observe(section);
    }
  }

  function scheduleScrollSpy() {
    clearTimeout(scrollSpyTimer);
    scrollSpyTimer = setTimeout(() => {
      initScrollSpy();
    }, 120);
  }

  onMount(() => {
    applyInitialTheme();
    loadData();
    fetchRemoteStats();
    handleScroll();
    window.addEventListener('scroll', handleScroll);
    window.addEventListener('resize', closeSidebar);

    return () => {
      window.removeEventListener('scroll', handleScroll);
      window.removeEventListener('resize', closeSidebar);
    };
  });

  afterUpdate(() => {
    if (status === 'ready') {
      scheduleScrollSpy();
    }
  });

  onDestroy(() => {
    clearTimeout(toastTimer);
    clearTimeout(splashTimer);
    clearTimeout(splashRemoveTimer);
    clearTimeout(scrollSpyTimer);

    if (scrollSpyObserver) {
      scrollSpyObserver.disconnect();
    }
  });

  $: categoriesWithApis = categories
    .filter((category) => Array.isArray(category.apis) && category.apis.length > 0)
    .map((category) => ({ ...category, slug: slugify(category.name) }));

  $: allApis = categoriesWithApis.flatMap((category) => category.apis);

  $: stats = [
    { n: allApis.length, l: 'Total Endpoints' },
    { n: categoriesWithApis.length, l: 'Categories' },
    {
      n: allApis.filter((api) => String(api.method || '').toUpperCase() === 'GET').length,
      l: 'GET Endpoints'
    },
    {
      n: allApis.filter((api) => String(api.method || '').toUpperCase() === 'POST').length,
      l: 'POST Endpoints'
    },
    { n: totalRequestsAllTime, l: 'Total Requests', remote: true },
    { n: averagePerDay, l: 'Daily', remote: true }
  ];

  $: getCount = allApis.filter((api) => String(api.method || '').toUpperCase() === 'GET').length;
  $: postCount = allApis.filter((api) => String(api.method || '').toUpperCase() === 'POST').length;
  $: totalCount = getCount + postCount || 1;
  $: getPct = Math.round((getCount / totalCount) * 100);
  $: postPct = 100 - getPct;

  $: chartData = chartHistory.length > 0 
    ? chartHistory.map(item => item.total)
    : [54200, 58100, 52900, 61400, 57800, 60200, averagePerDay || 63800];
  $: maxVal = Math.max(...chartData) * 1.05;
  $: minVal = Math.min(...chartData) * 0.95;
  $: yRange = maxVal - minVal || 1;
  $: points = chartData.map((val, i) => ({
    x: 10 + (i * 280 / (chartData.length - 1 || 1)),
    y: 110 - ((val - minVal) / yRange * 90)
  }));
  $: pathD = points.length ? 'M ' + points.map(p => `${p.x},${p.y}`).join(' L ') : '';
  $: areaD = points.length ? `${pathD} L ${points[points.length-1].x},115 L ${points[0].x},115 Z` : '';
  $: lastPoint = points.length ? points[points.length - 1] : { x: 0, y: 0 };
  
  $: chartDays = chartHistory.length > 0
    ? chartHistory.map(item => {
        const parts = item.date.split('-');
        return parts.length === 3 ? `${parts[2]}/${parts[1]}` : item.date;
      })
    : ['H-6', 'H-5', 'H-4', 'H-3', 'H-2', 'H-1', 'Hari Ini'];

  $: normalizedSearch = searchTerm.trim().toLowerCase();

  $: filteredSections = categoriesWithApis
    .map((category) => {
      const filteredApis = normalizedSearch
        ? category.apis.filter((api) => {
            const title = String(api.title || '').toLowerCase();
            const endpoint = String(api.endpointPath || '').toLowerCase();
            const description = String(api.description || '').toLowerCase();
            return (
              title.includes(normalizedSearch) ||
              endpoint.includes(normalizedSearch) ||
              description.includes(normalizedSearch)
            );
          })
        : category.apis;

      return {
        ...category,
        apis: filteredApis
      };
    })
    .filter((category) => category.apis.length > 0);
</script>

<div class="dot-pattern"></div>

{#if showSplash}
  <div id="splash" class:hide={splashHiding}>
    <div class="splash-logo">
      <span>F</span><span>e</span><span>r</span><span>d</span><span>e</span><span>v</span><span class="accent">A</span><span class="accent">P</span><span class="accent">I</span>
    </div>
    <div class="splash-bar">
      <div class="splash-progress"></div>
    </div>
    <div class="splash-sub">API Documentation</div>
  </div>
{/if}

<div class="app-root" class:visible={!showSplash}>
  {#if status === 'error'}
    <div class="error-state">
      <div class="error-card">
        <div class="error-title">Failed to Load Documentation</div>
        <div class="error-desc">{errorMessage}</div>
        <button class="btn btn-primary" type="button" on:click={loadData}>
          <i class="fa-solid fa-rotate-right"></i>Retry
        </button>
      </div>
    </div>
  {:else}
    <div
      class="overlay"
      class:show={sidebarOpen}
      id="overlay"
      role="button"
      tabindex="0"
      on:click={closeSidebar}
      on:keydown={(event) => (event.key === 'Enter' || event.key === ' ') && closeSidebar()}
    ></div>

<header class="header" class:sidebar-collapsed={sidebarCollapsed}>
  <div class="header-brand">
    <button class="mobile-menu-btn" id="menuBtn" on:click={toggleSidebar}>
  <i class="fa-solid" class:fa-bars={!sidebarOpen} class:fa-xmark={sidebarOpen}></i>
</button>
    <a
      href="#hero"
      class="logo"
      on:click|preventDefault={() => onNavigate({ detail: { slug: 'hero' } })}
    >
      <img
        src="https://cdn.ferdev.me/assets/img/brand_image.png"
        alt="FerDev"
        referrerpolicy="no-referrer"
        style="height: 32px; width: auto; display: block;"
        on:error={(e) => {
          e.currentTarget.style.display = 'none';
          e.currentTarget.nextElementSibling.style.display = 'inline';
        }}
      />
      <span class="logo-fallback" style="display: none;">
        FERDEV<span class="accent">API</span>
      </span>
    </a>
  </div>
  <div class="header-left">
    <div class="search-box">
      <i class="fa-solid fa-magnifying-glass"></i>
      <input
        type="text"
        id="globalSearch"
        placeholder="Search endpoints..."
        autocomplete="off"
        bind:value={searchTerm}
      />
    </div>
  </div>
  <div class="header-right">
    <a href="https://api.ferdev.me/register" target="_blank" class="icon-btn" title="Get API Key">
      <i class="fa-solid fa-key"></i>
    </a>
    <button class="theme-toggle" id="themeBtn" on:click={toggleTheme}>
      <i class="fa-solid fa-sun icon-sun"></i>
      <i class="fa-solid fa-moon icon-moon"></i>
    </button>
  </div>
</header>

    <Sidebar
      categories={categoriesWithApis}
      activeCat={activeCat}
      open={sidebarOpen}
      collapsed={sidebarCollapsed}
      on:navigate={onNavigate}
    />

    <main class="main" class:sidebar-collapsed={sidebarCollapsed}>
      <div class="hero section-anchor" id="hero">
        <div class="hero-label">&#9632; REST API Documentation</div>
        <div class="hero-title">
          <img
            src="https://cdn.ferdev.me/assets/img/brand_image.png"
            alt="FERDEV API"
            class="hero-logo-img"
            referrerpolicy="no-referrer"
            on:error={(e) => {
              e.currentTarget.style.display = 'none';
              e.currentTarget.nextElementSibling.style.display = 'block';
            }}
          />
          <span class="hero-logo-fallback" style="display: none;">
            FERDEV <span>API</span>
          </span>
        </div>
        <div class="hero-desc">
          A complete collection of REST APIs. AI, Downloader, Search, Tools, and more. All in one
          platform.
        </div>

        <!-- Stats Dashboard -->
        <div class="hero-dashboard" id="heroStats">
          <!-- Card 1: Overview & Distribution -->
          <div class="db-card">
            <div>
              <div class="db-card-title">
                <i class="fa-solid fa-chart-simple"></i> Platform Overview
              </div>
              <div class="db-grid">
                <div class="db-stat">
                  <div class="db-stat-num">{allApis.length}</div>
                  <div class="db-stat-label">Total Endpoints</div>
                </div>
                <div class="db-stat">
                  <div class="db-stat-num">{categoriesWithApis.length}</div>
                  <div class="db-stat-label">Categories</div>
                </div>
                <div class="db-stat">
                  <div class="db-stat-num remote-loading">
                    {#if totalRequestsAllTime === null}
                      <span class="stat-loading">
                        <span class="stat-dot"></span>
                        <span class="stat-dot"></span>
                        <span class="stat-dot"></span>
                      </span>
                    {:else}
                      <span class="db-stat-num remote-val">{fmt(totalRequestsAllTime)}</span>
                    {/if}
                  </div>
                  <div class="db-stat-label">Total Requests</div>
                </div>
                <div class="db-stat">
                  <div class="db-stat-num remote-loading">
                    {#if averagePerDay === null}
                      <span class="stat-loading">
                        <span class="stat-dot"></span>
                        <span class="stat-dot"></span>
                        <span class="stat-dot"></span>
                      </span>
                    {:else}
                      <span class="db-stat-num remote-val">{fmt(averagePerDay)}</span>
                    {/if}
                  </div>
                  <div class="db-stat-label">Daily Average</div>
                </div>
              </div>
            </div>
            
            <div class="db-divider"></div>
            
            <div class="dist-bar-wrapper">
              <div class="dist-bar-labels">
                <span class="dist-label-get">GET: {getCount} ({getPct}%)</span>
                <span class="dist-label-post">POST: {postCount} ({postPct}%)</span>
              </div>
              <div class="dist-progress-bar">
                <div class="dist-fill-get" style="width: {getPct}%"></div>
                <div class="dist-fill-post" style="width: {postPct}%"></div>
              </div>
            </div>
          </div>

          <!-- Card 2: Traffic Trend Chart -->
          <div class="db-card">
            <div class="db-card-title">
              <i class="fa-solid fa-wave-square"></i> Traffic Trend (All History)
            </div>
            
            <div class="chart-container">
              <svg class="chart-svg" viewBox="0 0 300 120" preserveAspectRatio="none">
                <defs>
                  <!-- Bar Gradient -->
                  <linearGradient id="chartGrad" x1="0" y1="1" x2="0" y2="0">
                    <stop offset="0%" stop-color="var(--accent)" />
                    <stop offset="100%" stop-color="var(--accent-2)" />
                  </linearGradient>
                </defs>
                
                <!-- Grid Lines -->
                <line x1="10" y1="20" x2="290" y2="20" class="chart-gridline" />
                <line x1="10" y1="65" x2="290" y2="65" class="chart-gridline" />
                <line x1="10" y1="110" x2="290" y2="110" class="chart-gridline" />
                
                <!-- Bar rects -->
                {#each chartData as val, i}
                  {@const barHeight = maxVal > 0 ? (val / maxVal) * 90 : 0}
                  {@const barWidth = (280 / chartData.length) * 0.75}
                  {@const x = 10 + (i * 280 / chartData.length) + ((280 / chartData.length) * 0.125)}
                  {@const y = 110 - barHeight}
                  <rect
                    x={x}
                    y={y}
                    width={barWidth}
                    height={barHeight}
                    rx="1.5"
                    fill="url(#chartGrad)"
                    class="chart-bar"
                  />
                  
                  <!-- Interactive Hover Tooltip zone -->
                  <rect
                    x={x}
                    y="10"
                    width={barWidth}
                    height="100"
                    fill="transparent"
                    style="cursor: pointer;"
                  >
                    <title>{chartDays[i]}: {val.toLocaleString()}</title>
                  </rect>
                {/each}

                <!-- SVG text labels for dates -->
                {#each points as pt, i}
                  {#if i === 0 || i === points.length - 1 || (points.length > 5 && i === Math.floor(points.length / 2))}
                    <text x={pt.x} y="118" fill="var(--text-muted)" font-size="8.5" font-family="sans-serif" text-anchor={i === 0 ? 'start' : (i === points.length - 1 ? 'end' : 'middle')}>
                      {chartDays[i]}
                    </text>
                  {/if}
                {/each}
              </svg>
            </div>
            
            <div class="db-divider"></div>
            
            <div class="chart-summary-grid">
              <div class="summary-item">
                <span class="summary-label">Total 30 Days</span>
                <span class="summary-val">{totalRequests30Days !== null ? fmt(totalRequests30Days) : '—'}</span>
              </div>
              <div class="summary-item">
                <span class="summary-label">Daily Average</span>
                <span class="summary-val">{averagePerDay !== null ? fmt(averagePerDay) : '—'}</span>
              </div>
              <div class="summary-item">
                <span class="summary-label">Busiest Day</span>
                <span class="summary-val">
                  {#if busiestDay}
                    {fmt(busiestDay.total)} <span class="summary-sub">({busiestDay.date.split('-').slice(1).reverse().join('/')})</span>
                  {:else}
                    —
                  {/if}
                </span>
              </div>
              <div class="summary-item">
                <span class="summary-label">Monthly Estimate</span>
                <span class="summary-val">{monthStats ? fmt(monthStats.estimatedTotal) : '—'}</span>
              </div>
            </div>
          </div>
        </div>

        <div class="usage-guide">
  <div class="hero-label">
    &#9632; Implementation Examples
  </div>

  <!-- POST Method -->
  <div class="usage-method">
    <div class="usage-method-header">
      <span class="method-badge method-post">POST</span>
      <span class="usage-method-desc">API Key is sent via headers in <code class="inline-code">Authorization Bearer</code>, parameters in JSON <code class="inline-code">body</code></span>
    </div>
    <div class="code-block">
      <div class="code-lang">JavaScript</div>
      <pre class="code-pre"><span class="c-kw">const</span> <span class="c-var">response</span> <span class="c-op">=</span> <span class="c-kw">await</span> <span class="c-fn">fetch</span><span class="c-punc">(</span><span class="c-str">'https://api.ferdev.me/endpoint'</span><span class="c-punc">, &#123;</span>
  <span class="c-prop">method</span><span class="c-op">:</span> <span class="c-str">'POST'</span><span class="c-punc">,</span>
  <span class="c-prop">headers</span><span class="c-op">:</span> <span class="c-punc">&#123;</span>
    <span class="c-str">'Content-Type'</span><span class="c-op">:</span> <span class="c-str">'application/json'</span><span class="c-punc">,</span>
    <span class="c-str">'Authorization'</span><span class="c-op">:</span> <span class="c-str">'Bearer </span><span class="c-key">YOUR_API_KEY</span><span class="c-str">'</span>
  <span class="c-punc">&#125;,</span>
  <span class="c-prop">body</span><span class="c-op">:</span> <span class="c-fn">JSON</span><span class="c-op">.</span><span class="c-fn">stringify</span><span class="c-punc">(&#123;</span>
    <span class="c-prop">prompt</span><span class="c-op">:</span> <span class="c-str">'Halo, siapa kamu?'</span>
  <span class="c-punc">&#125;)</span>
<span class="c-punc">&#125;);</span>

<span class="c-kw">const</span> <span class="c-var">data</span> <span class="c-op">=</span> <span class="c-kw">await</span> <span class="c-var">response</span><span class="c-op">.</span><span class="c-fn">json</span><span class="c-punc">();</span>
<span class="c-fn">console</span><span class="c-op">.</span><span class="c-fn">log</span><span class="c-punc">(</span><span class="c-var">data</span><span class="c-punc">);</span></pre>
    </div>
  </div>

  <!-- GET Method -->
  <div class="usage-method">
    <div class="usage-method-header">
      <span class="method-badge method-get">GET</span>
      <span class="usage-method-desc">API Key and parameters sent directly as <code class="inline-code">query string</code> in the URL</span>
    </div>
    <div class="code-block">
      <div class="code-lang">JavaScript</div>
      <pre class="code-pre"><span class="c-kw">const</span> <span class="c-var">response</span> <span class="c-op">=</span> <span class="c-kw">await</span> <span class="c-fn">fetch</span><span class="c-punc">(</span>
  <span class="c-str">'https://api.ferdev.me/endpoint?param=value&amp;apikey=</span><span class="c-key">YOUR_API_KEY</span><span class="c-str">'</span>
<span class="c-punc">);</span>

<span class="c-kw">const</span> <span class="c-var">data</span> <span class="c-op">=</span> <span class="c-kw">await</span> <span class="c-var">response</span><span class="c-op">.</span><span class="c-fn">json</span><span class="c-punc">();</span>
<span class="c-fn">console</span><span class="c-op">.</span><span class="c-fn">log</span><span class="c-punc">(</span><span class="c-var">data</span><span class="c-punc">);</span></pre>
    </div>
  </div>
</div>
        <!-- /Cara Penggunaan API -->

      </div>

      <div id="content">
        {#if filteredSections.length === 0 && status === 'ready'}
          <div class="no-results">
            <i class="fa-solid fa-magnifying-glass"></i>
            <p>No results for "<strong>{normalizedSearch}</strong>"</p>
          </div>
        {:else}
          {#each filteredSections as category}
            <div class="cat-section section-anchor" id={category.slug}>
              <div class="section-header">
                <div class="section-header-left">
                  <div class="section-icon"><i class={category.icon}></i></div>
                  <div class="section-title">{category.name}</div>
                  <div class="section-desc">{category.apis.length} endpoints available</div>
                </div>
                <div class="section-badge">{category.apis.length} APIs</div>
              </div>

              <div class="api-grid">
                {#each category.apis as api, apiIndex}
                  <ApiCard {api} index={apiIndex} baseUrl={BASE_URL} on:toast={(event) => showToast(event.detail.message)} />
                {/each}
              </div>
            </div>
          {/each}
        {/if}
      </div>
    </main>

    <div class="toast" class:show={toastVisible} id="toast">
      <i class="fa-solid fa-check"></i><span id="toastMsg">{toastMessage}</span>
    </div>
    <button class="back-top" class:show={showBackTop} id="backTop" on:click={() => window.scrollTo({ top: 0, behavior: 'smooth' })}>
      <i class="fa-solid fa-arrow-up"></i>
    </button>
  {/if}
</div>