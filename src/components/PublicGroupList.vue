<template>
    <!-- Group List -->
    <Draggable v-model="$root.publicGroupList" :disabled="!editMode" item-key="id" :animation="100">
        <template #item="group">
            <div class="mb-5" data-testid="group">
                <!-- Group Title -->
                <h2 class="group-title">
                    <font-awesome-icon v-if="editMode && showGroupDrag" icon="arrows-alt-v" class="action drag me-3" />
                    <font-awesome-icon
                        v-if="editMode"
                        icon="times"
                        class="action remove me-3"
                        @click="removeGroup(group.index)"
                    />
                    <span class="collapse-toggle" @click="toggleGroup(group.element)">
                        <font-awesome-icon
                            icon="chevron-down"
                            class="chevron me-2"
                            :class="{ collapsed: isGroupCollapsed(group.element) }"
                        />
                    </span>
                    <Editable
                        v-model="group.element.name"
                        :contenteditable="editMode"
                        tag="span"
                        :class="{ 'collapse-toggle': !editMode }"
                        data-testid="group-name"
                        @click="!editMode && toggleGroup(group.element)"
                    />
                </h2>

                <transition name="slide-fade-up">
                    <div v-if="!isGroupCollapsed(group.element)" class="shadow-box monitor-list mt-4 position-relative">
                        <div v-if="group.element.monitorList.length === 0" class="text-center no-monitor-msg">
                            {{ $t("No Monitors") }}
                        </div>

                        <!-- Monitor List -->
                        <!-- animation is not working, no idea why -->
                        <Draggable
                            v-model="group.element.monitorList"
                            class="monitor-list"
                            group="same-group"
                            :disabled="!editMode"
                            :animation="100"
                            item-key="id"
                        >
                            <template #item="monitor">
                                <div class="item" data-testid="monitor">
                                    <!-- Edit mode actions -->
                                    <div v-if="editMode" class="edit-actions">
                                        <font-awesome-icon icon="arrows-alt-v" class="action drag" />
                                        <font-awesome-icon
                                            icon="times"
                                            class="action remove"
                                            @click="removeMonitor(group.index, monitor.index)"
                                        />
                                        <font-awesome-icon
                                            icon="cog"
                                            class="action link-active btn-link"
                                            data-testid="monitor-settings"
                                            @click="$refs.monitorSettingDialog.show(group, monitor)"
                                        />
                                    </div>

                                    <!-- Main monitor row -->
                                    <div class="monitor-row">
                                        <!-- Name + tags -->
                                        <div class="monitor-meta">
                                            <a
                                                v-if="showLink(monitor)"
                                                :href="monitor.element.url"
                                                class="monitor-name"
                                                target="_blank"
                                                rel="noopener noreferrer"
                                                data-testid="monitor-name"
                                            >
                                                {{ monitor.element.name }}
                                            </a>
                                            <span v-else class="monitor-name" data-testid="monitor-name">
                                                {{ monitor.element.name }}
                                            </span>
                                            <div class="monitor-tags">
                                                <Tag
                                                    v-if="showCertificateExpiry && monitor.element.certExpiryDaysRemaining"
                                                    :item="{
                                                        name: $t('Cert Exp.'),
                                                        value: formattedCertExpiryMessage(monitor),
                                                        color: certExpiryColor(monitor),
                                                    }"
                                                    :size="'sm'"
                                                />
                                                <template v-if="showTags">
                                                    <Tag
                                                        v-for="tag in monitor.element.tags"
                                                        :key="tag"
                                                        :item="tag"
                                                        :size="'sm'"
                                                        data-testid="monitor-tag"
                                                    />
                                                </template>
                                            </div>
                                        </div>

                                        <!-- Heartbeat bar -->
                                        <div :key="$root.userHeartbeatBar" class="monitor-heartbeat">
                                            <HeartbeatBar size="mid" :monitor-id="monitor.element.id" />
                                        </div>

                                        <!-- Status / uptime in a fixed-width slot on the right -->
                                        <div class="monitor-uptime-slot">
                                            <Status
                                                v-if="showOnlyLastHeartbeat"
                                                :status="statusOfLastHeartbeat(monitor.element.id)"
                                            />
                                            <Uptime v-else :monitor="monitor.element" type="24" :pill="true" />
                                        </div>
                                    </div>
                                </div>
                            </template>
                        </Draggable>
                    </div>
                </transition>
            </div>
        </template>
    </Draggable>
    <MonitorSettingDialog ref="monitorSettingDialog" />
</template>

<script>
import MonitorSettingDialog from "./MonitorSettingDialog.vue";
import Draggable from "vuedraggable";
import HeartbeatBar from "./HeartbeatBar.vue";
import Uptime from "./Uptime.vue";
import Tag from "./Tag.vue";
import Status from "./Status.vue";

export default {
    components: {
        MonitorSettingDialog,
        Draggable,
        HeartbeatBar,
        Uptime,
        Tag,
        Status,
    },
    props: {
        /** Are we in edit mode? */
        editMode: {
            type: Boolean,
            required: true,
        },
        /** Should tags be shown? */
        showTags: {
            type: Boolean,
        },
        /** Should expiry be shown? */
        showCertificateExpiry: {
            type: Boolean,
        },
        /** Should only the last heartbeat be shown? */
        showOnlyLastHeartbeat: {
            type: Boolean,
        },
    },
    data() {
        return {};
    },
    computed: {
        showGroupDrag() {
            return this.$root.publicGroupList.length >= 2;
        },
    },
    methods: {
        /**
         * Toggle collapsed state for a group
         * @param {object} group Group to toggle
         * @returns {void}
         */
        toggleGroup(group) {
            if (!this.$router) {
                return;
            }

            const groupId = this.getGroupIdentifier(group);
            const collapsed = this.getCollapsedList();
            const index = collapsed.indexOf(groupId);

            if (index >= 0) {
                collapsed.splice(index, 1);
            } else {
                collapsed.push(groupId);
            }

            const query = { ...this.$route.query };
            if (collapsed.length > 0) {
                query.collapse = collapsed;
            } else {
                delete query.collapse;
            }

            this.$router.push({ query }).catch(() => {});
        },

        /**
         * Check if a group is collapsed
         * @param {object} group Group to check
         * @returns {boolean} Whether the group is collapsed
         */
        isGroupCollapsed(group) {
            return this.getCollapsedList().includes(this.getGroupIdentifier(group));
        },

        /**
         * Get list of collapsed group identifiers from the query param.
         * Vue Router normalises repeated params (?collapse=1&collapse=2) into an array.
         * @returns {string[]} Collapsed group identifiers
         */
        getCollapsedList() {
            const raw = this.$route.query.collapse;
            if (!raw) {
                return [];
            }
            // Normalise to array: a single query param is a string, repeated params are already an array
            return [].concat(raw);
        },

        /**
         * Remove the specified group
         * @param {number} index Index of group to remove
         * @returns {void}
         */
        removeGroup(index) {
            this.$root.publicGroupList.splice(index, 1);
        },

        /**
         * Remove a monitor from a group
         * @param {number} groupIndex Index of group to remove monitor from
         * @param {number} index Index of monitor to remove
         * @returns {void}
         */
        removeMonitor(groupIndex, index) {
            this.$root.publicGroupList[groupIndex].monitorList.splice(index, 1);
        },

        /**
         * Should a link to the monitor be shown?
         * Attempts to guess if a link should be shown based upon if
         * sendUrl is set and if the URL is default or not.
         * @param {object} monitor Monitor to check
         * @param {boolean} ignoreSendUrl Should the presence of the sendUrl
         * property be ignored. This will only work in edit mode.
         * @returns {boolean} Should the link be shown
         */
        showLink(monitor, ignoreSendUrl = false) {
            // We must check if there are any elements in monitorList to
            // prevent undefined errors if it hasn't been loaded yet
            if (this.$parent.editMode && ignoreSendUrl && Object.keys(this.$root.monitorList).length) {
                return (
                    this.$root.monitorList[monitor.element.id].type === "http" ||
                    this.$root.monitorList[monitor.element.id].type === "keyword" ||
                    this.$root.monitorList[monitor.element.id].type === "json-query"
                );
            }
            return monitor.element.sendUrl && monitor.element.url && monitor.element.url !== "https://";
        },

        /**
         * Returns formatted certificate expiry or Bad cert message
         * @param {object} monitor Monitor to show expiry for
         * @returns {string} Certificate expiry message
         */
        formattedCertExpiryMessage(monitor) {
            if (monitor?.element?.validCert && monitor?.element?.certExpiryDaysRemaining) {
                return this.$t("days", monitor.element.certExpiryDaysRemaining);
            } else if (monitor?.element?.validCert === false) {
                return this.$t("noOrBadCertificate");
            } else {
                return this.$t("unknownDays");
            }
        },

        /**
         * Returns the status of the last heartbeat
         * @param {number} monitorId Id of the monitor to get status for
         * @returns {number} Status of the last heartbeat
         */
        statusOfLastHeartbeat(monitorId) {
            let heartbeats = this.$root.heartbeatList[monitorId] ?? [];
            let lastHeartbeat = heartbeats[heartbeats.length - 1];
            return lastHeartbeat?.status;
        },

        /**
         * Returns certificate expiry color based on days remaining
         * @param {object} monitor Monitor to show expiry for
         * @returns {string} Color for certificate expiry
         */
        certExpiryColor(monitor) {
            if (monitor?.element?.validCert && monitor.element.certExpiryDaysRemaining > 7) {
                return "#059669";
            }
            return "#DC2626";
        },

        /**
         * Get unique identifier for a group
         * @param {object} group object
         * @returns {string} group identifier
         */
        getGroupIdentifier(group) {
            if (group.id !== undefined && group.id !== null) {
                return group.id.toString();
            }
            return `group${this.$root.publicGroupList.indexOf(group)}`;
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars";

.no-monitor-msg {
    position: absolute;
    width: 100%;
    top: 20px;
    left: 0;
}

.monitor-list {
    min-height: 46px;
}

// ─── Monitor row ─────────────────────────────────────────────────────────────
.item {
    padding: 10px 16px;
}

.edit-actions {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 6px;
}

.monitor-row {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
}

.monitor-meta {
    flex: 1 1 0;
    min-width: 0;
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 3px;
}

.monitor-name {
    font-weight: 600;
    font-size: 0.9rem;
    color: $text-primary;
    text-decoration: none;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;

    &:hover { color: $primary; }

    .dark & { color: $dark-font-color; }
}

.monitor-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
}

.monitor-heartbeat {
    flex: 0 0 40%;
    min-width: 0;

    @media (max-width: 767px) {
        flex: 0 0 30%;
    }
}

.monitor-uptime-slot {
    flex: 0 0 64px;
    display: flex;
    justify-content: flex-end;
    align-items: center;
}

.btn-link {
    color: #bbbbbb;
}

.link-active {
    color: $primary;
}

.flip-list-move {
    transition: transform 0.5s;
}

.no-move {
    transition: transform 0s;
}

.drag {
    color: #bbb;
    cursor: grab;
}

.remove {
    color: $danger;
}

.group-title {
    span {
        display: inline-block;
        min-width: 15px;
    }
}

.collapse-toggle {
    cursor: pointer;
    padding: 2px;
}

.chevron {
    font-size: 0.8em;
    color: #bbb;
    transition: all 0.2s $easing-in;

    &.collapsed {
        transform: rotate(-90deg);
    }
}

.mobile {
    .item {
        padding: 13px 0 10px;
    }
}

.bg-maintenance {
    background-color: $maintenance;
}
</style>
