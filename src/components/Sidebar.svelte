<script>
  import { createEventDispatcher } from 'svelte';

  export let categories = [];
  export let activeCat = 'hero';
  export let open = false;
  export let collapsed = false;

  const dispatch = createEventDispatcher();

  function navigate(slug) {
    dispatch('navigate', { slug });
  }
</script>

<nav class="sidebar" class:open class:collapsed id="sidebar">
  <div class="sidebar-label">Navigation</div>
  <a
    href="#hero"
    class="sidebar-item"
    class:active={activeCat === 'hero'}
    data-cat="hero"
    on:click|preventDefault={() => navigate('hero')}
  >
    <i class="fa-solid fa-house"></i><span class="sidebar-text">Overview</span>
  </a>
  <a
    href="https://api.ferdev.me/dashboard"
    class="sidebar-item"
    target="_blank"
    rel="noopener noreferrer"
  >
    <i class="fa-solid fa-dashboard"></i><span class="sidebar-text">Dashboard</span>
    <i class="fa-solid fa-arrow-up-right-from-square sidebar-text" style="font-size: 10px; opacity: 0.6;"></i>
  </a>
  <a
    href="https://status.ferdev.me"
    class="sidebar-item"
    target="_blank"
    rel="noopener noreferrer"
  >
    <i class="fa-solid fa-server"></i><span class="sidebar-text">Server Status</span>
    <i class="fa-solid fa-arrow-up-right-from-square sidebar-text" style="font-size: 10px; opacity: 0.6;"></i>
  </a>
  <div class="sidebar-divider"></div>
  <div class="sidebar-label">Categories</div>
  <div id="sidebarCats">
    {#each categories as cat}
      {@const slug = cat.slug}
      <a
        href={'#' + slug}
        class="sidebar-item"
        class:active={activeCat === slug}
        data-cat={slug}
        on:click|preventDefault={() => navigate(slug)}
      >
        <i class={cat.icon}></i><span class="sidebar-text">{cat.name}</span>
        <span class="sidebar-count">{cat.apis.length}</span>
      </a>
    {/each}
  </div>
  <div class="sidebar-divider"></div>
  <div class="sidebar-label">Information</div>
    <a
    href="https://api.ferdev.me/community"
    class="sidebar-item"
    target="_blank"
    rel="noopener noreferrer"
  >
    <i class="fa-solid fa-user-group"></i><span class="sidebar-text">Community</span>
    <i class="fa-solid fa-arrow-up-right-from-square sidebar-text" style="font-size: 10px; opacity: 0.6;"></i>
  </a>
    <a
    href="https://whatsapp.com/channel/0029Vb6klqTKQuJDtbifsl1y"
    class="sidebar-item"
    target="_blank"
    rel="noopener noreferrer"
  >
    <i class="fa-solid fa-screwdriver-wrench"></i><span class="sidebar-text">Changelog</span>
    <i class="fa-solid fa-arrow-up-right-from-square sidebar-text" style="font-size: 10px; opacity: 0.6;"></i>
  </a>
    <a
    href="https://tos.ferdev.me"
    class="sidebar-item"
    target="_blank"
    rel="noopener noreferrer"
  >
    <i class="fa-solid fa-scale-balanced"></i><span class="sidebar-text">Terms of Service</span>
    <i class="fa-solid fa-arrow-up-right-from-square sidebar-text" style="font-size: 10px; opacity: 0.6;"></i>
  </a>
</nav>
