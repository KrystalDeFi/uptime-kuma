<template>
    <div :class="classes">
        <div v-if="!$root.socket.connected && !$root.socket.firstConnect" class="lost-connection">
            <div class="container-fluid">
                {{ $root.connectionErrorMsg }}
                <div v-if="$root.showReverseProxyGuide">
                    {{ $t("Using a Reverse Proxy?") }}
                    <a href="https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy" target="_blank">
                        {{ $t("Check how to config it for WebSocket") }}
                    </a>
                </div>
            </div>
        </div>

        <!-- Desktop header -->
        <header v-if="!$root.isMobile" class="d-flex flex-wrap justify-content-center py-3 mb-3 border-bottom app-header">
            <router-link
                to="/dashboard"
                class="d-flex align-items-center mb-3 mb-md-0 me-md-auto text-dark text-decoration-none ms-4"
            >
                <img src="https://krystal.app/assets/images/logos/krystal.svg" alt="Krystal" class="brand-logo" />
            </router-link>

            <a
                v-if="hasNewVersion"
                target="_blank"
                href="https://github.com/louislam/uptime-kuma/releases"
                class="btn btn-primary me-3"
            >
                <font-awesome-icon icon="arrow-alt-circle-up" />
                {{ $t("New Update") }}
            </a>

            <ul class="nav nav-pills">
                <li v-if="$root.loggedIn" class="nav-item me-2">
                    <router-link to="/manage-status-page" class="nav-link">
                        <font-awesome-icon icon="stream" />
                        {{ $t("Status Pages") }}
                    </router-link>
                </li>
                <li v-if="$root.loggedIn" class="nav-item me-2">
                    <router-link to="/dashboard" class="nav-link">
                        <font-awesome-icon icon="tachometer-alt" />
                        {{ $t("Dashboard") }}
                    </router-link>
                </li>
                <li v-if="$root.loggedIn" class="nav-item">
                    <div class="dropdown dropdown-profile-pic">
                        <div class="nav-link" data-bs-toggle="dropdown">
                            <div class="profile-pic">{{ $root.usernameFirstChar }}</div>
                            <font-awesome-icon icon="angle-down" />
                        </div>

                        <!-- Header's Dropdown Menu -->
                        <ul class="dropdown-menu">
                            <!-- Username -->
                            <li>
                                <i18n-t
                                    v-if="$root.username != null"
                                    tag="span"
                                    keypath="signedInDisp"
                                    class="dropdown-item-text"
                                >
                                    <strong>{{ $root.username }}</strong>
                                </i18n-t>
                                <span v-if="$root.username == null" class="dropdown-item-text">
                                    {{ $t("signedInDispDisabled") }}
                                </span>
                            </li>

                            <li><hr class="dropdown-divider" /></li>

                            <!-- Functions -->
                            <li>
                                <router-link
                                    to="/maintenance"
                                    class="dropdown-item"
                                    :class="{ active: $route.path.includes('manage-maintenance') }"
                                >
                                    <font-awesome-icon icon="wrench" />
                                    {{ $t("Maintenance") }}
                                </router-link>
                            </li>

                            <li>
                                <router-link
                                    to="/settings/general"
                                    class="dropdown-item"
                                    :class="{ active: $route.path.includes('settings') }"
                                >
                                    <font-awesome-icon icon="cog" />
                                    {{ $t("Settings") }}
                                </router-link>
                            </li>

                            <li>
                                <a
                                    href="https://github.com/louislam/uptime-kuma/wiki"
                                    class="dropdown-item"
                                    target="_blank"
                                >
                                    <font-awesome-icon icon="info-circle" />
                                    {{ $t("Help") }}
                                </a>
                            </li>

                            <li v-if="$root.loggedIn && $root.socket.token !== 'autoLogin'">
                                <button class="dropdown-item" @click="$root.logout">
                                    <font-awesome-icon icon="sign-out-alt" />
                                    {{ $t("Logout") }}
                                </button>
                            </li>
                        </ul>
                    </div>
                </li>
            </ul>
        </header>

        <!-- Mobile header -->
        <header v-else class="d-flex flex-wrap justify-content-center pt-2 pb-2 mb-3 app-header">
            <router-link to="/dashboard" class="d-flex align-items-center text-dark text-decoration-none">
                <img src="https://krystal.app/assets/images/logos/krystal.svg" alt="Krystal" class="brand-logo" />
            </router-link>
        </header>

        <main>
            <router-view v-if="$root.loggedIn" />
            <Login v-if="!$root.loggedIn && $root.allowLoginDialog" />
        </main>

        <!-- Mobile Only -->
        <div v-if="$root.isMobile" style="width: 100%; height: calc(60px + env(safe-area-inset-bottom))" />
        <nav v-if="$root.isMobile && $root.loggedIn" class="bottom-nav">
            <router-link to="/dashboard" class="nav-link">
                <div><font-awesome-icon icon="tachometer-alt" /></div>
                {{ $t("Home") }}
            </router-link>

            <router-link to="/list" class="nav-link">
                <div><font-awesome-icon icon="list" /></div>
                {{ $t("List") }}
            </router-link>

            <router-link to="/add" class="nav-link">
                <div><font-awesome-icon icon="plus" /></div>
                {{ $t("Add") }}
            </router-link>

            <router-link to="/settings" class="nav-link">
                <div><font-awesome-icon icon="cog" /></div>
                {{ $t("Settings") }}
            </router-link>
        </nav>

        <button
            v-if="numActiveToasts != 0"
            type="button"
            class="btn btn-normal clear-all-toast-btn"
            @click="clearToasts"
        >
            <font-awesome-icon icon="times" />
        </button>
    </div>
</template>

<script>
import Login from "../components/Login.vue";
import compareVersions from "compare-versions";
import { useToast } from "vue-toastification";
const toast = useToast();

export default {
    components: {
        Login,
    },

    data() {
        return {
            toastContainer: null,
            numActiveToasts: 0,
            toastContainerObserver: null,
        };
    },

    computed: {
        // Theme or Mobile
        classes() {
            const classes = {};
            classes[this.$root.theme] = true;
            classes["mobile"] = this.$root.isMobile;
            return classes;
        },

        hasNewVersion() {
            if (this.$root.info.latestVersion && this.$root.info.version) {
                return compareVersions(this.$root.info.latestVersion, this.$root.info.version) >= 1;
            } else {
                return false;
            }
        },
    },

    watch: {},

    mounted() {
        this.toastContainer = document.querySelector(".bottom-right.toast-container");

        // Watch the number of active toasts
        this.toastContainerObserver = new MutationObserver((mutations) => {
            for (const mutation of mutations) {
                if (mutation.type === "childList") {
                    this.numActiveToasts = mutation.target.children.length;
                }
            }
        });

        if (this.toastContainer != null) {
            this.toastContainerObserver.observe(this.toastContainer, { childList: true });
        }
    },

    beforeUnmount() {
        this.toastContainerObserver.disconnect();
    },

    methods: {
        /**
         * Clear all toast notifications.
         * @returns {void}
         */
        clearToasts() {
            toast.clear();
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.nav-link {
    font-family: $font-family-base;
    font-weight: 600;
    font-size: 0.875rem;
    border-radius: $btn-radius;
    padding: 0.42rem 0.875rem;
    transition: background-color 0.12s ease, color 0.12s ease;
    // White-ish on the dark header
    color: rgba(255, 255, 255, 0.65);

    &:hover {
        background-color: rgba(255, 255, 255, 0.08);
        color: #ffffff;
    }

    &.router-link-active,
    &.active {
        background-color: rgba(255, 255, 255, 0.14);
        color: #ffffff !important;
        font-weight: 700;
    }

    &.status-page {
        background-color: rgba(255, 255, 255, 0.1);
    }
}

.bottom-nav {
    z-index: 1000;
    position: fixed;
    bottom: 0;
    height: calc(62px + env(safe-area-inset-bottom));
    width: 100%;
    left: 0;
    background-color: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    box-shadow: 0 -1px 0 rgba(0, 0, 0, 0.06), 0 -4px 16px rgba(0, 0, 0, 0.04);
    text-align: center;
    white-space: nowrap;
    padding: 0 10px env(safe-area-inset-bottom);

    a {
        text-align: center;
        width: 25%;
        display: inline-block;
        height: 100%;
        padding: 8px 10px 0;
        font-family: "Manrope", BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        font-size: 0.72rem;
        font-weight: 600;
        color: #94a3b8;
        overflow: hidden;
        text-decoration: none;
        transition: color 0.15s ease;

        &.router-link-exact-active,
        &.active {
            color: $primary;
            font-weight: 700;
        }

        div {
            font-size: 20px;
        }
    }
}

main {
    min-height: calc(100vh - 160px);
}

.app-header {
    background-color: #0f172a;
    border-bottom-color: rgba(255, 255, 255, 0.08) !important;
}

.brand-logo {
    height: 28px;
    width: auto;
    display: block;
    // SVG is dark — invert to white on the dark header
    filter: brightness(0) invert(1);
}

.nav {
    margin-right: 25px;
}

.lost-connection {
    padding: 5px;
    background-color: crimson;
    color: white;
    position: fixed;
    width: 100%;
    z-index: 99999;
}

// Profile Pic Button with Dropdown
.dropdown-profile-pic {
    user-select: none;

    .nav-link {
        cursor: pointer;
        display: flex;
        gap: 6px;
        align-items: center;
        background-color: transparent;
        border: 1px solid rgba(255, 255, 255, 0.2);
        border-radius: $btn-radius;
        padding: 0.38rem 0.75rem;
        transition: border-color 0.12s ease, background-color 0.12s ease;
        color: rgba(255, 255, 255, 0.85) !important;

        &:hover {
            background-color: rgba(255, 255, 255, 0.08);
            border-color: rgba(255, 255, 255, 0.35);
        }
    }

    .dropdown-menu {
        padding-left: 0;
        padding-bottom: 4px;
        padding-top: 4px;
        margin-top: 6px !important;
        border-radius: $card-radius;
        overflow: hidden;
        border: 1px solid $border-light;
        box-shadow: $shadow-float;

        .dropdown-divider {
            margin: 4px 0;
            border-top: 1px solid $border-light;
            background-color: transparent;
        }

        .dropdown-item-text {
            font-family: $font-family-base;
            font-size: 0.78rem;
            font-weight: 600;
            color: $text-secondary;
            padding-bottom: 0.5rem;
        }

        .dropdown-item {
            font-family: $font-family-base;
            font-size: 0.875rem;
            font-weight: 500;
            padding: 0.6rem 1rem;
            color: $text-primary;
            transition: background-color 0.1s ease;
            &:hover { background-color: $surface-subtle; }
        }

        .dark & {
            background-color: $dark-surface;
            color: $dark-font-color;
            border-color: $dark-border-color;
            box-shadow: 0 4px 32px rgba(0,0,0,.5);

            .dropdown-divider { border-top-color: $dark-border-color; }

            .dropdown-item {
                color: $dark-font-color;
                &.active { color: $dark-font-color2; background-color: $highlight !important; }
                &:hover { background-color: $dark-bg; }
            }
        }
    }

    .profile-pic {
        display: flex;
        align-items: center;
        justify-content: center;
        color: #052e16;
        background-color: $primary;
        width: 28px;
        height: 28px;
        margin-right: 4px;
        border-radius: 50%;
        font-family: $font-family-base;
        font-weight: 800;
        font-size: 11px;
        flex-shrink: 0;
    }
}

.dark {
    .bottom-nav {
        background-color: rgba($dark-bg, 0.9);
        backdrop-filter: blur(12px);
        -webkit-backdrop-filter: blur(12px);
        box-shadow: 0 -1px 0 $dark-border-color;

        a {
            color: #475569;

            &.router-link-exact-active,
            &.active {
                color: $primary;
            }
        }
    }
}

.clear-all-toast-btn {
    position: fixed;
    right: 1em;
    bottom: 1em;
    font-size: 1.2em;
    padding: 9px 15px;
    width: 48px;
    box-shadow: 2px 2px 30px rgba(0, 0, 0, 0.2);
    z-index: 100;

    .dark & {
        box-shadow: 2px 2px 30px rgba(0, 0, 0, 0.5);
    }
}

@media (max-width: 770px) {
    .clear-all-toast-btn {
        bottom: 72px;
    }
}
</style>
