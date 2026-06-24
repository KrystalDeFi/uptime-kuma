<template>
    <span class="status-chip" :class="chipClass">
        <span class="status-dot"></span>
        <span class="status-label">{{ text }}</span>
    </span>
</template>

<script>
export default {
    props: {
        /** Current status of monitor */
        status: {
            type: Number,
            default: 0,
        },
    },

    computed: {
        chipClass() {
            const map = { 0: "status-down", 1: "status-up", 2: "status-pending", 3: "status-maintenance" };
            return map[this.status] ?? "status-unknown";
        },

        text() {
            if (this.status === 0) return this.$t("Down");
            if (this.status === 1) return this.$t("Up");
            if (this.status === 2) return this.$t("Pending");
            if (this.status === 3) return this.$t("statusMaintenance");
            return this.$t("Unknown");
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.status-chip {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px 3px 7px;
    border-radius: $chip-radius;
    font-family: $font-family-base;
    font-size: 0.75rem;
    font-weight: 600;
    letter-spacing: 0.01em;
    white-space: nowrap;
    border: 1px solid transparent;
}

.status-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    flex-shrink: 0;
}

// ─── Status variants ─────────────────────────────────────────────────────────
.status-up {
    color: darken($status-up, 15%);
    background-color: rgba($status-up, 0.1);
    border-color: rgba($status-up, 0.2);

    .status-dot { background-color: $status-up; }

    .dark & {
        color: lighten($status-up, 15%);
        background-color: rgba($status-up, 0.15);
        border-color: rgba($status-up, 0.25);
    }
}

.status-down {
    color: darken($status-down, 10%);
    background-color: rgba($status-down, 0.08);
    border-color: rgba($status-down, 0.18);

    .status-dot { background-color: $status-down; }

    .dark & {
        color: lighten($status-down, 20%);
        background-color: rgba($status-down, 0.15);
        border-color: rgba($status-down, 0.25);
    }
}

.status-pending {
    color: darken($status-pending, 20%);
    background-color: rgba($status-pending, 0.08);
    border-color: rgba($status-pending, 0.18);

    .status-dot { background-color: $status-pending; }

    .dark & {
        color: lighten($status-pending, 20%);
        background-color: rgba($status-pending, 0.15);
        border-color: rgba($status-pending, 0.25);
    }
}

.status-maintenance {
    color: darken($status-maintenance, 10%);
    background-color: rgba($status-maintenance, 0.08);
    border-color: rgba($status-maintenance, 0.18);

    .status-dot { background-color: $status-maintenance; }

    .dark & {
        color: lighten($status-maintenance, 20%);
        background-color: rgba($status-maintenance, 0.15);
        border-color: rgba($status-maintenance, 0.25);
    }
}

.status-unknown {
    color: $text-secondary;
    background-color: rgba($status-unknown, 0.08);
    border-color: rgba($status-unknown, 0.15);

    .status-dot { background-color: $status-unknown; }

    .dark & {
        color: $dark-font-color;
        background-color: rgba($status-unknown, 0.12);
    }
}
</style>
