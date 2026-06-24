<template>
    <transition name="slide-fade" appear>
        <div v-if="monitor">
            <router-link v-if="group !== ''" :to="monitorURL(monitor.parent)" class="group-breadcrumb">
                <font-awesome-icon icon="layer-group" class="group-icon" />
                {{ group }}
            </router-link>
            <h1>
                {{ monitor.name }}
                <div class="monitor-id">
                    <div class="hash">#</div>
                    <div>{{ monitor.id }}</div>
                </div>
            </h1>
            <!-- eslint-disable-next-line vue/no-v-html-->
            <p v-if="monitor.description" v-html="descriptionHTML"></p>
            <div class="d-flex">
                <div class="tags">
                    <Tag
                        v-for="tag in monitor.tags"
                        :key="tag.id"
                        :item="tag"
                        :size="'sm'"
                        :scrollable="true"
                        :constrained="true"
                    />
                </div>
            </div>
            <p class="url">
                <a
                    v-if="
                        monitor.type === 'http' ||
                        monitor.type === 'keyword' ||
                        monitor.type === 'json-query' ||
                        monitor.type === 'real-browser' ||
                        monitor.type === 'websocket-upgrade'
                    "
                    :href="monitor.url"
                    target="_blank"
                    rel="noopener noreferrer"
                >
                    {{ filterPassword(monitor.url) }}
                </a>
                <span v-if="monitor.type === 'port'">TCP Port {{ monitor.hostname }}:{{ monitor.port }}</span>
                <span v-if="monitor.type === 'ping'">Ping: {{ monitor.hostname }}</span>
                <span v-if="monitor.type === 'globalping'">
                    <a v-if="monitor.subtype === 'http'" :href="monitor.url" target="_blank" rel="noopener noreferrer">
                        {{ filterPassword(monitor.url) }}
                    </a>
                    <span v-if="monitor.hostname">{{ monitor.hostname }}</span>
                    <br />
                    <span>{{ $t("Location") }}:</span>
                    <span class="keyword">{{ monitor.location }}</span>
                    <br />
                    <span v-if="monitor.subtype === 'dns'">
                        [{{ monitor.dns_resolve_type }}]
                        <br />
                        <span>{{ $t("Last Result") }}:</span>
                        <span class="keyword">{{ monitor.dns_last_result }}</span>
                    </span>
                </span>
                <span v-if="monitor.type === 'keyword'">
                    <br />
                    <span>{{ $t("Keyword") }}:</span>
                    <span class="keyword">{{ monitor.keyword }}</span>
                    <span v-if="monitor.invertKeyword" alt="Inverted keyword" class="keyword-inverted">↧</span>
                </span>
                <span v-if="monitor.type === 'json-query'">
                    <br />
                    <span>{{ $t("Json Query") }}:</span>
                    <span class="keyword">{{ monitor.jsonPath }}</span>
                    <br />
                    <span>{{ $t("Expected Value") }}:</span>
                    <span class="keyword">{{ monitor.expectedValue }}</span>
                </span>
                <span v-if="monitor.type === 'dns'">
                    [{{ monitor.dns_resolve_type }}] {{ monitor.hostname }}
                    <br />
                    <span>{{ $t("Last Result") }}:</span>
                    <span class="keyword">{{ monitor.dns_last_result }}</span>
                </span>
                <span v-if="monitor.type === 'docker'">Docker container: {{ monitor.docker_container }}</span>
                <span v-if="monitor.type === 'gamedig'">
                    Gamedig - {{ monitor.game }}: {{ monitor.hostname }}:{{ monitor.port }}
                </span>
                <span v-if="monitor.type === 'grpc-keyword'">
                    gRPC - {{ filterPassword(monitor.grpcUrl) }}
                    <br />
                    <span>{{ $t("Keyword") }}:</span>
                    <span class="keyword">{{ monitor.keyword }}</span>
                </span>
                <span v-if="monitor.type === 'mongodb'">{{ filterPassword(monitor.databaseConnectionString) }}</span>
                <span v-if="monitor.type === 'mqtt'">
                    MQTT: {{ monitor.hostname }}:{{ monitor.port }}/{{ monitor.mqttTopic }}
                </span>
                <span v-if="monitor.type === 'mysql'">{{ filterPassword(monitor.databaseConnectionString) }}</span>
                <span v-if="monitor.type === 'oracledb'">
                    {{
                        $t("oracledbConnectionString", {
                            connectionString: filterPassword(monitor.databaseConnectionString),
                        })
                    }}
                </span>
                <span v-if="monitor.type === 'postgres'">{{ filterPassword(monitor.databaseConnectionString) }}</span>
                <span v-if="monitor.type === 'push'">
                    Push:
                    <a :href="pushURL" target="_blank" rel="noopener noreferrer">{{ pushURL }}</a>
                </span>
                <span v-if="monitor.type === 'radius'">Radius: {{ filterPassword(monitor.hostname) }}</span>
                <span v-if="monitor.type === 'redis'">{{ filterPassword(monitor.databaseConnectionString) }}</span>
                <span v-if="monitor.type === 'sqlserver'">
                    SQL Server: {{ filterPassword(monitor.databaseConnectionString) }}
                </span>
                <span v-if="monitor.type === 'steam'">
                    Steam Game Server: {{ monitor.hostname }}:{{ monitor.port }}
                </span>
            </p>

            <div class="functions">
                <button v-if="monitor.active" class="btn btn-normal" @click="pauseDialog">
                    <font-awesome-icon icon="pause" />
                    {{ $t("Pause") }}
                </button>
                <button
                    v-if="!monitor.active"
                    class="btn btn-primary"
                    :disabled="monitor.forceInactive"
                    @click="resumeMonitor"
                >
                    <font-awesome-icon icon="play" />
                    {{ $t("Resume") }}
                </button>
                <router-link :to="'/edit/' + monitor.id" class="btn btn-normal">
                    <font-awesome-icon icon="edit" />
                    {{ $t("Edit") }}
                </router-link>
                <router-link :to="'/clone/' + monitor.id" class="btn btn-normal">
                    <font-awesome-icon icon="clone" />
                    {{ $t("Clone") }}
                </router-link>
                <div class="functions-divider" />
                <button class="btn btn-outline-danger btn-delete" @click="deleteDialog">
                    <font-awesome-icon icon="trash" />
                    {{ $t("Delete") }}
                </button>
            </div>

            <div class="shadow-box">
                <div class="heartbeat-header">
                    <Status :status="lastHeartBeat.status" data-testid="monitor-status" />
                    <span class="word">
                        {{ $t("checkEverySecond", [monitor.interval]) }} ({{ secondsToHumanReadableFormat(monitor.interval) }})
                    </span>
                </div>
                <HeartbeatBar :monitor-id="monitor.id" />
            </div>

            <!-- Push Examples -->
            <div v-if="monitor.type === 'push'" class="shadow-box big-padding">
                <a href="#" @click="pushMonitor.showPushExamples = !pushMonitor.showPushExamples">
                    {{ $t("pushViewCode") }}
                </a>

                <transition name="slide-fade" appear>
                    <div v-if="pushMonitor.showPushExamples" class="mt-3">
                        <select id="push-current-example" v-model="pushMonitor.currentExample" class="form-select">
                            <optgroup :label="$t('programmingLanguages')">
                                <option value="csharp">C#</option>
                                <option value="go">Go</option>
                                <option value="java">Java</option>
                                <option value="javascript-fetch">JavaScript (fetch)</option>
                                <option value="php">PHP</option>
                                <option value="python">Python</option>
                                <option value="typescript-fetch">TypeScript (fetch)</option>
                            </optgroup>
                            <optgroup :label="$t('pushOthers')">
                                <option value="bash-curl">Bash (curl)</option>
                                <option value="powershell">PowerShell</option>
                                <option value="docker">Docker</option>
                            </optgroup>
                        </select>

                        <prism-editor
                            v-model="pushMonitor.code"
                            class="css-editor mt-3"
                            :highlight="pushExampleHighlighter"
                            line-numbers
                            readonly
                        ></prism-editor>
                    </div>
                </transition>
            </div>

            <!-- Stats -->
            <div class="shadow-box stats">
                <div class="stats-grid">
                    <div v-if="monitor.type !== 'group'" class="stat-item">
                        <span class="stat-label">{{ pingTitle() }} <span class="stat-period">({{ $t("Current") }})</span></span>
                        <span class="stat-value">
                            <a href="#" @click.prevent="showPingChartBox = !showPingChartBox">
                                <CountUp :value="ping" />
                            </a>
                        </span>
                    </div>
                    <div v-if="monitor.type !== 'group'" class="stat-item">
                        <span class="stat-label">{{ pingTitle(true) }} <span class="stat-period">({{ $t("hours", 24) }})</span></span>
                        <span class="stat-value"><CountUp :value="avgPing" /></span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-label">{{ $t("Uptime") }} <span class="stat-period">({{ $t("hours", 24) }})</span></span>
                        <span class="stat-value"><Uptime :monitor="monitor" type="24" /></span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-label">{{ $t("Uptime") }} <span class="stat-period">({{ $t("days", 30) }})</span></span>
                        <span class="stat-value"><Uptime :monitor="monitor" type="720" /></span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-label">{{ $t("Uptime") }} <span class="stat-period">({{ $t("years", 1) }})</span></span>
                        <span class="stat-value"><Uptime :monitor="monitor" type="1y" /></span>
                    </div>
                    <div v-if="tlsInfo" class="stat-item">
                        <span class="stat-label">{{ $t("Cert Exp.") }} <span class="stat-period">(<Datetime :value="tlsInfo.certInfo.validTo" date-only />)</span></span>
                        <span class="stat-value">
                            <a href="#" @click.prevent="toggleCertInfoBox = !toggleCertInfoBox">{{ $t("days", tlsInfo.certInfo.daysRemaining) }}</a>
                            <font-awesome-icon
                                v-if="tlsInfo.hostnameMatchMonitorUrl === false"
                                class="cert-info-warn"
                                icon="exclamation-triangle"
                                :title="$t('certHostnameMismatch')"
                            />
                        </span>
                    </div>
                    <div v-if="domainInfo" class="stat-item">
                        <span class="stat-label">{{ $t("labelDomainExpiry") }} <span class="stat-period">(<Datetime :value="domainInfo.expiresOn" date-only />)</span></span>
                        <span class="stat-value">{{ $t("days", domainInfo.daysRemaining) }}</span>
                    </div>
                </div>
            </div>

            <!-- Cert Info Box -->
            <transition name="slide-fade" appear>
                <div v-if="showCertInfoBox" class="shadow-box big-padding text-center">
                    <div class="row">
                        <div class="col">
                            <certificate-info :certInfo="tlsInfo.certInfo" :valid="tlsInfo.valid" />
                        </div>
                    </div>
                </div>
            </transition>

            <!-- Ping Chart -->
            <div v-if="showPingChartBox" class="shadow-box ping-chart-wrapper">
                <PingChart :monitor-id="monitor.id" />
            </div>

            <!-- Screenshot -->
            <div v-if="monitor.type === 'real-browser'" class="shadow-box">
                <div class="row">
                    <div class="col-md-6 zoom-cursor">
                        <img
                            :src="screenshotURL"
                            style="width: 100%"
                            alt="screenshot of the website"
                            @click="showScreenshotDialog"
                        />
                    </div>
                    <ScreenshotDialog ref="screenshotDialog" :imageURL="screenshotURL" />
                </div>
            </div>

            <div class="shadow-box table-shadow-box">
                <div class="table-header">
                    <span class="table-title">{{ $t("Events") }}</span>
                    <div class="dropdown">
                        <button
                            class="btn btn-sm btn-outline-normal dropdown-toggle"
                            type="button"
                            data-bs-toggle="dropdown"
                        >
                            <font-awesome-icon icon="trash" />
                            {{ $t("Clear Data") }}
                        </button>
                        <ul class="dropdown-menu dropdown-menu-end">
                            <li>
                                <button type="button" class="dropdown-item" @click="clearEventsDialog">
                                    {{ $t("Events") }}
                                </button>
                            </li>
                            <li>
                                <button type="button" class="dropdown-item" @click="clearHeartbeatsDialog">
                                    {{ $t("Heartbeats") }}
                                </button>
                            </li>
                        </ul>
                    </div>
                </div>
                <table class="table table-borderless table-hover">
                    <thead>
                        <tr>
                            <th>{{ $t("Status") }}</th>
                            <th>{{ $t("DateTime") }}</th>
                            <th>{{ $t("Message") }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(beat, index) in displayedRecords" :key="index" style="padding: 10px">
                            <td><Status :status="beat.status" /></td>
                            <td :class="{ 'border-0': !beat.msg }">
                                <Datetime :value="beat.time" />
                            </td>
                            <td class="border-0">{{ beat.msg }}</td>
                        </tr>

                        <tr v-if="importantHeartBeatListLength === 0">
                            <td colspan="3">
                                {{ $t("No important events") }}
                            </td>
                        </tr>
                    </tbody>
                </table>

                <div class="d-flex justify-content-center kuma_pagination">
                    <pagination
                        v-model="page"
                        :records="importantHeartBeatListLength"
                        :per-page="perPage"
                        :options="paginationConfig"
                    />
                </div>
            </div>

            <Confirm ref="confirmPause" :yes-text="$t('Yes')" :no-text="$t('No')" @yes="pauseMonitor">
                {{ $t("pauseMonitorMsg") }}
            </Confirm>

            <Confirm
                ref="confirmDelete"
                btn-style="btn-danger"
                :yes-text="$t('Yes')"
                :no-text="$t('No')"
                @yes="deleteMonitor"
            >
                <div v-if="monitor && monitor.type === 'group'">
                    <div>{{ $t("deleteGroupMsg") }}</div>
                    <div v-if="hasChildren" class="form-check">
                        <input
                            id="delete-children-checkbox"
                            v-model="deleteChildrenMonitors"
                            class="form-check-input"
                            type="checkbox"
                        />
                        <label class="form-check-label" for="delete-children-checkbox">
                            {{ $t("deleteChildrenMonitors", childrenCount) }}
                        </label>
                    </div>
                </div>
                <div v-else>
                    {{ $t("deleteMonitorMsg") }}
                </div>
            </Confirm>

            <Confirm
                ref="confirmClearEvents"
                btn-style="btn-danger"
                :yes-text="$t('Yes')"
                :no-text="$t('No')"
                @yes="clearEvents"
            >
                {{ $t("clearEventsMsg") }}
            </Confirm>

            <Confirm
                ref="confirmClearHeartbeats"
                btn-style="btn-danger"
                :yes-text="$t('Yes')"
                :no-text="$t('No')"
                @yes="clearHeartbeats"
            >
                {{ $t("clearHeartbeatsMsg") }}
            </Confirm>
        </div>
    </transition>
</template>

<script>
import { defineAsyncComponent } from "vue";
import { useToast } from "vue-toastification";
const toast = useToast();
import Confirm from "../components/Confirm.vue";
import HeartbeatBar from "../components/HeartbeatBar.vue";
import Status from "../components/Status.vue";
import Datetime from "../components/Datetime.vue";
import CountUp from "../components/CountUp.vue";
import Uptime from "../components/Uptime.vue";
import Pagination from "v-pagination-3";
const PingChart = defineAsyncComponent(() => import("../components/PingChart.vue"));
import Tag from "../components/Tag.vue";
import CertificateInfo from "../components/CertificateInfo.vue";
import { getMonitorRelativeURL } from "../util.ts";
import { URL } from "whatwg-url";
import DOMPurify from "dompurify";
import { marked } from "marked";
import { getResBaseURL, timeDurationFormatter } from "../util-frontend";
import { highlight, languages } from "prismjs/components/prism-core";
import "prismjs/components/prism-clike";
import "prismjs/components/prism-javascript";
import "prismjs/components/prism-css";
import { PrismEditor } from "vue-prism-editor";
import "vue-prism-editor/dist/prismeditor.min.css";
import ScreenshotDialog from "../components/ScreenshotDialog.vue";

export default {
    components: {
        Uptime,
        CountUp,
        Datetime,
        HeartbeatBar,
        Confirm,
        Status,
        Pagination,
        PingChart,
        Tag,
        CertificateInfo,
        PrismEditor,
        ScreenshotDialog,
    },
    data() {
        return {
            page: 1,
            perPage: 25,
            heartBeatList: [],
            toggleCertInfoBox: false,
            showPingChartBox: true,
            paginationConfig: {
                hideCount: true,
                chunksNavigation: "scroll",
            },
            cacheTime: Date.now(),
            importantHeartBeatListLength: 0,
            displayedRecords: [],
            pushMonitor: {
                showPushExamples: false,
                currentExample: "javascript-fetch",
                code: "",
            },
            deleteChildrenMonitors: false,
        };
    },
    computed: {
        monitor() {
            let id = this.$route.params.id;
            return this.$root.monitorList[id];
        },

        /**
         * Get the count of children monitors for this group
         * @returns {number} Number of children monitors
         */
        childrenCount() {
            if (!this.monitor || this.monitor.type !== "group") {
                return 0;
            }
            const children = Object.values(this.$root.monitorList).filter((m) => m.parent === this.monitor.id);
            return children.length;
        },

        /**
         * Check if the monitor is a group and has children
         * @returns {boolean} True if monitor is a group with children
         */
        hasChildren() {
            return this.childrenCount > 0;
        },

        lastHeartBeat() {
            // Also trigger screenshot refresh here
            // eslint-disable-next-line vue/no-side-effects-in-computed-properties
            this.cacheTime = Date.now();

            if (this.monitor.id in this.$root.lastHeartbeatList && this.$root.lastHeartbeatList[this.monitor.id]) {
                return this.$root.lastHeartbeatList[this.monitor.id];
            }

            return {
                status: -1,
            };
        },

        ping() {
            if (this.lastHeartBeat.ping || this.lastHeartBeat.ping === 0) {
                return this.lastHeartBeat.ping;
            }

            return this.$t("notAvailableShort");
        },

        avgPing() {
            if (this.$root.avgPingList[this.monitor.id] || this.$root.avgPingList[this.monitor.id] === 0) {
                return this.$root.avgPingList[this.monitor.id];
            }

            return this.$t("notAvailableShort");
        },

        status() {
            if (this.$root.statusList[this.monitor.id]) {
                return this.$root.statusList[this.monitor.id];
            }

            return {};
        },

        tlsInfo() {
            // Add: this.$root.tlsInfoList[this.monitor.id].certInfo
            // Fix: TypeError: Cannot read properties of undefined (reading 'validTo')
            // Reason: TLS Info object format is changed in 1.8.0, if for some reason, it cannot connect to the site after update to 1.8.0, the object is still in the old format.
            if (this.$root.tlsInfoList[this.monitor.id] && this.$root.tlsInfoList[this.monitor.id].certInfo) {
                return this.$root.tlsInfoList[this.monitor.id];
            }

            return null;
        },

        domainInfo() {
            return this.$root.domainInfoList[this.monitor.id] || null;
        },

        showCertInfoBox() {
            return this.tlsInfo != null && this.toggleCertInfoBox;
        },

        group() {
            return this.monitor.path.slice(0, -1).join(" / ");
        },

        pushURL() {
            return this.$root.baseURL + "/api/push/" + this.monitor.pushToken + "?status=up&msg=OK&ping=";
        },

        screenshotURL() {
            return getResBaseURL() + this.monitor.screenshot + "?time=" + this.cacheTime;
        },

        descriptionHTML() {
            if (this.monitor.description != null) {
                return DOMPurify.sanitize(marked(this.monitor.description));
            } else {
                return "";
            }
        },
    },

    watch: {
        page(to) {
            this.getImportantHeartbeatListPaged();
        },

        monitor(to) {
            this.getImportantHeartbeatListLength();
        },
        "monitor.type"() {
            if (this.monitor && this.monitor.type === "push") {
                this.loadPushExample();
            }
        },
        "pushMonitor.currentExample"() {
            this.loadPushExample();
        },
    },

    mounted() {
        this.getImportantHeartbeatListLength();

        this.$root.emitter.on("newImportantHeartbeat", this.onNewImportantHeartbeat);

        if (this.monitor && this.monitor.type === "push") {
            if (this.lastHeartBeat.status === -1) {
                this.pushMonitor.showPushExamples = true;
            }
            this.loadPushExample();
        }
    },

    beforeUnmount() {
        this.$root.emitter.off("newImportantHeartbeat", this.onNewImportantHeartbeat);
    },

    methods: {
        getResBaseURL,
        /**
         * Request a test notification be sent for this monitor
         * @returns {void}
         */
        testNotification() {
            this.$root.getSocket().emit("testNotification", this.monitor.id);
            this.$root.toastSuccess("Test notification is requested.");
        },

        /**
         * Show dialog to confirm pause
         * @returns {void}
         */
        pauseDialog() {
            this.$refs.confirmPause.show();
        },

        /**
         * Resume this monitor
         * @returns {void}
         */
        resumeMonitor() {
            this.$root.getSocket().emit("resumeMonitor", this.monitor.id, (res) => {
                this.$root.toastRes(res);
            });
        },

        /**
         * Request that this monitor is paused
         * @returns {void}
         */
        pauseMonitor() {
            this.$root.getSocket().emit("pauseMonitor", this.monitor.id, (res) => {
                this.$root.toastRes(res);
            });
        },

        /**
         * Show dialog to confirm deletion
         * @returns {void}
         */
        deleteDialog() {
            this.$refs.confirmDelete.show();
        },

        /**
         * Show Screenshot Dialog
         * @returns {void}
         */
        showScreenshotDialog() {
            this.$refs.screenshotDialog.show();
        },

        /**
         * Show dialog to confirm clearing events
         * @returns {void}
         */
        clearEventsDialog() {
            this.$refs.confirmClearEvents.show();
        },

        /**
         * Show dialog to confirm clearing heartbeats
         * @returns {void}
         */
        clearHeartbeatsDialog() {
            this.$refs.confirmClearHeartbeats.show();
        },

        /**
         * Request that this monitor is deleted
         * @returns {void}
         */
        deleteMonitor() {
            this.$root.deleteMonitor(this.monitor.id, this.deleteChildrenMonitors, (res) => {
                this.$root.toastRes(res);
                if (res.ok) {
                    this.$router.push("/dashboard");
                }
            });
        },

        /**
         * Request that this monitors events are cleared
         * @returns {void}
         */
        clearEvents() {
            this.$root.clearEvents(this.monitor.id, (res) => {
                if (res.ok) {
                    this.getImportantHeartbeatListLength();
                } else {
                    toast.error(res.msg);
                }
            });
        },

        /**
         * Request that this monitors heartbeats are cleared
         * @returns {void}
         */
        clearHeartbeats() {
            this.$root.clearHeartbeats(this.monitor.id, (res) => {
                if (!res.ok) {
                    toast.error(res.msg);
                }
            });
        },

        /**
         * Return the correct title for the ping stat
         * @param {boolean} average Is the statistic an average?
         * @returns {string} Title formatted dependent on monitor type
         */
        pingTitle(average = false) {
            let translationPrefix = "";
            if (average) {
                translationPrefix = "Avg. ";
            }

            if (this.monitor.type === "http" || this.monitor.type === "keyword" || this.monitor.type === "json-query") {
                return this.$t(translationPrefix + "Response");
            }

            return this.$t(translationPrefix + "Ping");
        },

        /**
         * Get URL of monitor
         * @param {number} id ID of monitor
         * @returns {string} Relative URL of monitor
         */
        monitorURL(id) {
            return getMonitorRelativeURL(id);
        },

        /**
         * Filter and hide password in URL for display
         * @param {string} urlString URL to censor
         * @returns {string} Censored URL
         */
        filterPassword(urlString) {
            try {
                let parsedUrl = new URL(urlString);
                if (parsedUrl.password !== "") {
                    parsedUrl.password = "******";
                }
                return parsedUrl.toString();
            } catch (e) {
                // Handle SQL Server
                return urlString.replaceAll(/Password=(.+);/gi, "Password=******;");
            }
        },

        /**
         * Retrieves the length of the important heartbeat list for this monitor.
         * @returns {void}
         */
        getImportantHeartbeatListLength() {
            if (this.monitor) {
                this.$root.getSocket().emit("monitorImportantHeartbeatListCount", this.monitor.id, (res) => {
                    if (res.ok) {
                        this.importantHeartBeatListLength = res.count;
                        this.getImportantHeartbeatListPaged();
                    }
                });
            }
        },

        /**
         * Retrieves the important heartbeat list for the current page.
         * @returns {void}
         */
        getImportantHeartbeatListPaged() {
            if (this.monitor) {
                const offset = (this.page - 1) * this.perPage;
                this.$root
                    .getSocket()
                    .emit("monitorImportantHeartbeatListPaged", this.monitor.id, offset, this.perPage, (res) => {
                        if (res.ok) {
                            this.displayedRecords = res.data;
                        }
                    });
            }
        },

        /**
         * Updates the displayed records when a new important heartbeat arrives.
         * @param {object} heartbeat - The heartbeat object received.
         * @returns {void}
         */
        onNewImportantHeartbeat(heartbeat) {
            if (heartbeat.monitorID === this.monitor?.id) {
                if (this.page === 1) {
                    this.displayedRecords.unshift(heartbeat);
                    if (this.displayedRecords.length > this.perPage) {
                        this.displayedRecords.pop();
                    }
                    this.importantHeartBeatListLength += 1;
                }
            }
        },

        /**
         * Highlight the example code
         * @param {string} code Code
         * @returns {string} Highlighted code
         */
        pushExampleHighlighter(code) {
            return highlight(code, languages.js);
        },

        loadPushExample() {
            this.pushMonitor.code = "";
            this.$root.getSocket().emit("getPushExample", this.pushMonitor.currentExample, (res) => {
                let code = res.code
                    .replace("60", this.monitor.interval)
                    .replace("https://example.com/api/push/key?status=up&msg=OK&ping=", this.pushURL);
                this.pushMonitor.code = code;
            });
        },

        secondsToHumanReadableFormat(seconds) {
            return timeDurationFormatter.secondsToHumanReadableFormat(seconds);
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.group-breadcrumb {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    margin-bottom: 8px;
    padding: 3px 10px 3px 8px;
    border-radius: $chip-radius;
    background-color: rgba($primary, 0.08);
    border: 1px solid rgba($primary, 0.2);
    font-size: 0.78rem;
    font-weight: 700;
    color: darken(#22c55e, 10%);
    text-decoration: none;
    letter-spacing: 0.01em;
    transition: background-color 0.12s ease;

    &:hover {
        background-color: rgba($primary, 0.15);
        color: darken(#22c55e, 15%);
    }

    .dark & {
        background-color: rgba($primary, 0.12);
        border-color: rgba($primary, 0.25);
        color: $primary;
    }
}

.group-icon {
    font-size: 0.7rem;
    opacity: 0.8;
}

.form-check {
    margin-top: 16px;
}

// ─── Action buttons row ───────────────────────────────────────────────────────
.functions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 8px;
    margin-bottom: 16px;

    .btn {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        font-size: 0.875rem;
        font-weight: 600;
        padding: 0.42rem 1rem;
    }

    .functions-divider {
        flex: 1;
    }

    .btn-delete {
        color: #ef4444;
        border-color: #fca5a5;

        &:hover {
            background-color: #fef2f2;
            border-color: #ef4444;
            color: #b91c1c;
        }

        .dark & {
            color: #f87171;
            border-color: rgba(#ef4444, 0.35);
            &:hover { background-color: rgba(#ef4444, 0.1); }
        }
    }
}

@media (max-width: 767px) {
    .badge {
        margin-top: 14px;
    }
}

@media (max-width: 550px) {
    .ping-chart-wrapper {
        padding: 12px !important;
    }
}


.url {
    color: $primary;
    margin-bottom: 20px;
    font-weight: bold;

    a {
        color: $primary;
    }
}

.shadow-box {
    padding: 20px;
    margin-top: 25px;
}

.heartbeat-header {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 10px;
}

.word {
    color: $secondary-text;
    font-size: 0.8rem;
}

table {
    font-size: 14px;

    tr {
        transition: all ease-in-out 0.2ms;
    }
}

.stats {
    padding: 16px 20px;
}

.stats-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 0;
}

.stat-item {
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding: 10px 20px 10px 0;
    min-width: 110px;
    flex: 1 1 110px;

    // divider between items
    & + .stat-item {
        padding-left: 20px;
        border-left: 1px solid $border-light;

        .dark & { border-left-color: $dark-border-color; }
    }
}

.stat-label {
    font-size: 0.72rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-secondary;
    white-space: nowrap;
}

.stat-period {
    font-weight: 400;
    text-transform: none;
    letter-spacing: 0;
    color: $text-secondary;
    opacity: 0.7;
}

.stat-value {
    font-size: 1.4rem;
    font-weight: 800;
    letter-spacing: -0.03em;
    line-height: 1.1;
    color: $text-primary;

    a { color: inherit; text-decoration: none; &:hover { color: $primary; } }

    .dark & { color: $dark-font-color; }
}

@media (max-width: 600px) {
    .stats-grid { gap: 0; }
    .stat-item {
        flex: 1 1 50%;
        padding: 10px 12px 10px 0;
        border-left: none !important;

        &:nth-child(even) {
            padding-left: 12px;
            border-left: 1px solid $border-light !important;
            .dark & { border-left-color: $dark-border-color !important; }
        }
    }
}

.keyword {
    color: black;
}

.table-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 12px;
}

.table-title {
    font-family: $font-family-base;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-secondary;

    .dark & { color: $dark-font-color; }
}

.dark {
    .keyword {
        color: $dark-font-color;
    }

    .keyword-inverted {
        color: $dark-font-color;
    }

}

.tags {
    margin-bottom: 0.5rem;
    max-width: 95vw;
}

.tags > div:first-child {
    margin-left: 0 !important;
}

.monitor-id {
    display: inline-flex;
    font-size: 0.7em;
    margin-left: 0.3em;
    color: $secondary-text;
    flex-direction: row;
    flex-wrap: nowrap;

    .hash {
        user-select: none;
    }

    .dark & {
        opacity: 0.7;
    }
}

.cert-info-warn {
    margin-left: 4px;
    opacity: 0.5;

    .dark & {
        opacity: 0.7;
    }
}
</style>
