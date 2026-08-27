<template>
  <button
    v-if="!isMobileOpen"
    class="mobile-toggle"
    @click="isMobileOpen = true"
    aria-label="Open navigation"
  >
    <svg width="22" height="22" viewBox="0 0 20 20" fill="none">
      <path
        d="M3 5H17M3 10H17M3 15H17"
        stroke="currentColor"
        stroke-width="1.5"
        stroke-linecap="round"
      />
    </svg>
  </button>

  <div v-if="isMobileOpen" class="backdrop" @click="isMobileOpen = false"></div>

  <aside
    class="sidebar"
    :class="{ collapsed: effectiveCollapsed, 'is-open': isMobileOpen }"
  >
    <div class="sidebar-header">
      <div class="brand" v-if="!effectiveCollapsed">
        <h1>{{ t("nav.companyName") }}</h1>
        <span class="subtitle">{{ t("nav.subtitle") }}</span>
      </div>
      <div class="brand-mark" v-else>{{ brandInitial }}</div>
    </div>

    <nav class="sidebar-nav">
      <router-link
        v-for="item in navItems"
        :key="item.path"
        :to="item.path"
        class="nav-item"
        :class="{ active: $route.path === item.path }"
        @click="isMobileOpen = false"
        :title="effectiveCollapsed ? item.label : null"
      >
        <span class="nav-icon" v-html="item.icon"></span>
        <span class="nav-label" v-if="!effectiveCollapsed">{{
          item.label
        }}</span>
      </router-link>
    </nav>

    <div class="sidebar-footer">
      <div class="footer-row" v-if="!effectiveCollapsed">
        <LanguageSwitcher />
      </div>
      <div class="footer-row">
        <ProfileMenu
          @show-profile-details="$emit('show-profile-details')"
          @show-tasks="$emit('show-tasks')"
        />
      </div>
      <button
        class="collapse-toggle"
        v-if="!isMediumScreen"
        @click="toggleCollapsed"
      >
        <svg
          width="18"
          height="18"
          viewBox="0 0 20 20"
          fill="none"
          :style="{ transform: effectiveCollapsed ? 'rotate(180deg)' : 'none' }"
        >
          <path
            d="M12.5 4L7 10L12.5 16"
            stroke="currentColor"
            stroke-width="1.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
        <span v-if="!effectiveCollapsed">Collapse</span>
      </button>
    </div>
  </aside>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";
import { useI18n } from "../composables/useI18n";
import LanguageSwitcher from "./LanguageSwitcher.vue";
import ProfileMenu from "./ProfileMenu.vue";

defineEmits(["show-profile-details", "show-tasks"]);

const { t } = useI18n();

// Manual desktop preference (user-controlled, persisted)
const isCollapsed = ref(false);
const isMobileOpen = ref(false);

// Automatic icons-only tier for medium/tablet widths (769px-1100px).
// Independent of the manual preference above - when the viewport widens
// back past this breakpoint, the sidebar reverts to the stored preference.
const isMediumScreen = ref(false);
let mediumScreenQuery = null;
const handleMediumScreenChange = (event) => {
  isMediumScreen.value = event.matches;
};

const STORAGE_KEY = "sidebar-collapsed";

onMounted(() => {
  isCollapsed.value = localStorage.getItem(STORAGE_KEY) === "true";

  mediumScreenQuery = window.matchMedia(
    "(min-width: 769px) and (max-width: 1100px)",
  );
  isMediumScreen.value = mediumScreenQuery.matches;
  mediumScreenQuery.addEventListener("change", handleMediumScreenChange);
});

onUnmounted(() => {
  if (mediumScreenQuery) {
    mediumScreenQuery.removeEventListener("change", handleMediumScreenChange);
  }
});

const toggleCollapsed = () => {
  isCollapsed.value = !isCollapsed.value;
  localStorage.setItem(STORAGE_KEY, String(isCollapsed.value));
};

// Layout-facing collapsed state: forced icons-only on medium screens,
// otherwise follows the user's manual desktop preference.
const effectiveCollapsed = computed(
  () => isMediumScreen.value || isCollapsed.value,
);

const brandInitial = computed(() => {
  const name = t("nav.companyName");
  return name ? name.charAt(0) : "A";
});

const icons = {
  overview:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 11L8 6L11.5 9.5L17 4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M3 16H17" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>',
  inventory:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 6L10 3L17 6L10 9L3 6Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M3 6V14L10 17M17 6V14L10 17M10 9V17" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/></svg>',
  orders:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M5 3H15C15.5523 3 16 3.44772 16 4V17L13 15L10 17L7 15L4 17V4C4 3.44772 4.44772 3 5 3Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M7 8H13M7 11H13" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>',
  finance:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M10 2V18M14 5.5C14 4.11929 12.2091 3 10 3C7.79086 3 6 4.11929 6 5.5C6 6.88071 7.79086 8 10 8C12.2091 8 14 9.11929 14 10.5C14 11.8807 12.2091 13 10 13C7.79086 13 6 11.8807 6 10.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>',
  demand:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M3 17L7 10L11 13L17 4" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M13 4H17V8" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>',
  reports:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><path d="M5 3H15V17H5V3Z" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round"/><path d="M7.5 7H12.5M7.5 10H12.5M7.5 13H10.5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/></svg>',
  backlog:
    '<svg width="20" height="20" viewBox="0 0 20 20" fill="none"><circle cx="10" cy="10" r="7" stroke="currentColor" stroke-width="1.5"/><path d="M10 6V10L12.5 12" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>',
};

const navItems = computed(() => [
  { path: "/", label: t("nav.overview"), icon: icons.overview },
  { path: "/inventory", label: t("nav.inventory"), icon: icons.inventory },
  { path: "/orders", label: t("nav.orders"), icon: icons.orders },
  { path: "/spending", label: t("nav.finance"), icon: icons.finance },
  { path: "/demand", label: t("nav.demandForecast"), icon: icons.demand },
  { path: "/reports", label: "Reports", icon: icons.reports },
  { path: "/backlog", label: "Backlog", icon: icons.backlog },
]);
</script>

<style scoped>
.sidebar {
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  width: var(--sidebar-width-expanded);
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  position: sticky;
  top: 0;
  height: 100vh;
  align-self: flex-start;
  transition: width 0.2s ease;
  overflow-y: auto;
  z-index: 200;
}

.sidebar.collapsed {
  width: var(--sidebar-width-collapsed);
}

.sidebar-header {
  padding: 1.5rem 1.25rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  min-height: 70px;
  display: flex;
  align-items: center;
}

.brand h1 {
  font-size: 1.125rem;
  font-weight: 700;
  color: var(--sidebar-text-active);
  letter-spacing: -0.025em;
  white-space: nowrap;
}

.brand .subtitle {
  display: block;
  font-size: 0.75rem;
  color: var(--sidebar-text);
  font-weight: 400;
  margin-top: 0.25rem;
  white-space: nowrap;
}

.brand-mark {
  width: 32px;
  height: 32px;
  border-radius: var(--radius-md);
  background: var(--sidebar-accent);
  color: var(--sidebar-text-active);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.938rem;
}

.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
  padding: 0.75rem;
  overflow-y: auto;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.75rem;
  color: var(--sidebar-text);
  text-decoration: none;
  font-weight: 500;
  font-size: 0.875rem;
  border-radius: var(--radius-sm);
  border-left: 3px solid transparent;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.nav-item:hover {
  color: var(--sidebar-text-active);
  background: var(--sidebar-hover-bg);
}

.nav-item.active {
  color: var(--sidebar-text-active);
  background: var(--sidebar-hover-bg);
  border-left-color: var(--sidebar-accent);
}

.nav-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  width: 20px;
  height: 20px;
}

.nav-label {
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-footer {
  padding: 0.75rem;
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.footer-row {
  display: flex;
}

.collapse-toggle {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 0.75rem;
  background: none;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: var(--radius-sm);
  color: var(--sidebar-text);
  cursor: pointer;
  font-family: inherit;
  font-size: 0.813rem;
  font-weight: 500;
  transition: all 0.15s ease;
}

.collapse-toggle:hover {
  color: var(--sidebar-text-active);
  background: var(--sidebar-hover-bg);
}

/* When collapsed, constrain the reused ProfileMenu to icon-only so its
   name label/chevron don't overflow the narrow rail */
.sidebar.collapsed .footer-row {
  justify-content: center;
}

.sidebar.collapsed .footer-row :deep(.profile-button) {
  padding: 0.5rem;
  gap: 0;
}

.sidebar.collapsed .footer-row :deep(.profile-name),
.sidebar.collapsed .footer-row :deep(.chevron) {
  display: none;
}

.sidebar.collapsed .footer-row :deep(.dropdown-menu) {
  left: 100%;
  right: auto;
  margin-left: 0.5rem;
}

.mobile-toggle {
  display: none;
}

.backdrop {
  display: none;
}

@media (max-width: 768px) {
  .mobile-toggle {
    display: flex;
    align-items: center;
    justify-content: center;
    position: fixed;
    top: 1rem;
    left: 1rem;
    width: 40px;
    height: 40px;
    background: var(--sidebar-bg);
    color: var(--sidebar-text-active);
    border: none;
    border-radius: var(--radius-sm);
    cursor: pointer;
    z-index: 250;
  }

  .backdrop {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.4);
    z-index: 199;
  }

  .sidebar,
  .sidebar.collapsed {
    position: fixed;
    top: 0;
    left: 0;
    width: var(--sidebar-width-expanded);
    transform: translateX(-100%);
    transition: transform 0.2s ease;
  }

  .sidebar.is-open {
    transform: translateX(0);
  }
}
</style>
