<template>
    <div tabindex="-1" class="dropdown" @focusin="openMenu" @focusout="handleFocusOut">
        <button type="button" class="filter-dropdown-status" :class="{ active: filterActive }" tabindex="0">
            <div class="px-1 d-flex align-items-center">
                <slot name="status"></slot>
            </div>
            <span class="px-1">
                <font-awesome-icon icon="angle-down" />
            </span>
        </button>
        <ul class="filter-dropdown-menu" :class="{ open: open }">
            <slot name="dropdown"></slot>
        </ul>
    </div>
</template>

<script>
export default {
    components: {},
    props: {
        filterActive: {
            type: Boolean,
            required: true,
        },
    },
    emits: ["openMenu"],
    data() {
        return {
            open: false,
        };
    },
    methods: {
        openMenu() {
            this.$emit("openMenu");
            this.open = true;
        },

        handleFocusOut(e) {
            if (e.relatedTarget != null && this.$el.contains(e.relatedTarget)) {
                return;
            }
            this.open = false;
        },
    },
};
</script>

<style lang="scss">
@import "../assets/vars.scss";
@import "../assets/app.scss";

.filter-dropdown-menu {
    z-index: 100;
    transition: opacity 0.15s ease;
    padding: 5px 0 !important;
    border-radius: $card-radius;
    overflow: hidden;
    position: absolute;
    inset: 0 auto auto 0;
    margin: 0;
    transform: translate(0, 40px);
    box-shadow: $shadow-float;
    border: 1px solid $border-light;
    visibility: hidden;
    list-style: none;
    height: 0;
    opacity: 0;
    background: $surface-raised;

    &.open { height: unset; visibility: inherit; opacity: 1; }

    .dropdown-item {
        padding: 7px 14px;
        font-family: $font-family-base;
        font-size: 0.875rem;
        font-weight: 500;
        cursor: pointer;
        &:hover { background-color: $surface-subtle; }
        &:focus { background-color: $highlight-white; outline: none; }
    }

    .dark & {
        background-color: $dark-surface;
        border-color: $dark-border-color;
        color: $dark-font-color;
        box-shadow: 0 4px 24px rgba(0,0,0,.5);

        .dropdown-item {
            color: $dark-font-color;
            &:hover { background-color: $dark-bg; }
            &:focus { background-color: $dark-bg2; }
            &.active { color: $dark-font-color2; background-color: $highlight !important; }
        }
    }
}

.filter-dropdown-status {
    // Uses the same shape as .btn-outline-normal
    @extend .btn-outline-normal;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    margin-left: 0;
    height: 34px;
}

.filter-active { color: $primary; }
</style>
