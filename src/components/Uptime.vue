<template>
    <span v-if="pill" class="uptime-pill" :class="dotClass" :title="title">
        <span class="uptime-dot"></span>
        <span class="uptime-value">{{ uptime }}</span>
    </span>
    <span v-else :title="title">{{ uptime }}</span>
</template>

<script>
import { DOWN, MAINTENANCE, PENDING, UP } from "../util.ts";

export default {
    props: {
        monitor: { type: Object, default: null },
        type:    { type: String, default: null },
        pill:    { type: Boolean, default: false },
    },

    computed: {
        uptime() {
            if (this.type === "maintenance") return this.$t("statusMaintenance");

            const key = this.monitor.id + "_" + this.type;
            if (this.$root.uptimeList[key] !== undefined) {
                const result = Math.round(this.$root.uptimeList[key] * 10000) / 100;
                if (this.$route.path.startsWith("/status") && result > 100) return "100%";
                return result + "%";
            }
            return this.$t("notAvailableShort");
        },

        statusCode() {
            const hb = this.lastHeartBeat;
            return hb.status;
        },

        dotClass() {
            const s = this.lastHeartBeat.status;
            if (s === MAINTENANCE) return "dot-maintenance";
            if (s === DOWN)        return "dot-down";
            if (s === UP)          return "dot-up";
            if (s === PENDING)     return "dot-pending";
            return "dot-unknown";
        },

        lastHeartBeat() {
            const list = this.$root.lastHeartbeatList;
            if (this.monitor.id in list && list[this.monitor.id]) return list[this.monitor.id];
            return { status: -1 };
        },

        title() {
            if (this.type === "1y")  return this.$t("years", 1);
            if (this.type === "720") return this.$t("days", 30);
            return this.$t("hours", 24);
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.uptime-pill {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-family: $font-family-base;
    font-size: 0.78rem;
    font-weight: 700;
    color: $text-secondary;

    .dark & { color: $dark-font-color; }
}

.uptime-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
}

.uptime-value {
    line-height: 1;
}

.dot-up          .uptime-dot { background-color: $status-up; }
.dot-down        .uptime-dot { background-color: $status-down; }
.dot-pending     .uptime-dot { background-color: $status-pending; }
.dot-maintenance .uptime-dot { background-color: $status-maintenance; }
.dot-unknown     .uptime-dot { background-color: $status-unknown; }
</style>
