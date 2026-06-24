<template>
    <div class="ping-chart">
        <div class="chart-header">
            <span class="chart-title">{{ $t("Response") }}</span>

            <!-- Inline legend shown only for the multi-line (stats) view -->
            <div v-if="chartPeriodHrs !== '0'" class="chart-legend">
                <span class="legend-item">
                    <span class="legend-line" style="background:#126331"></span>
                    {{ $t("minPing") }}
                </span>
                <span class="legend-item">
                    <span class="legend-line" style="background:#5CDD8B"></span>
                    {{ $t("avgPing") }}
                </span>
                <span class="legend-item">
                    <span class="legend-line" style="background:#21b55a"></span>
                    {{ $t("maxPing") }}
                </span>
            </div>

            <div class="period-options">
                <button
                    type="button"
                    class="btn btn-period-toggle dropdown-toggle"
                    data-bs-toggle="dropdown"
                    aria-expanded="false"
                >
                    {{ chartPeriodOptions[chartPeriodHrs] }}
                </button>
                <ul class="dropdown-menu dropdown-menu-end">
                    <li v-for="(item, key) in chartPeriodOptions" :key="key">
                        <button
                            type="button"
                            class="dropdown-item"
                            :class="{ active: chartPeriodHrs == key }"
                            @click="chartPeriodHrs = key"
                        >
                            {{ item }}
                        </button>
                    </li>
                </ul>
            </div>
        </div>
        <div class="chart-wrapper" :class="{ loading: loading }">
            <Line :data="chartData" :options="chartOptions" />
        </div>
    </div>
</template>

<script lang="js">
import {
    BarController,
    BarElement,
    Chart,
    Filler,
    LinearScale,
    LineController,
    LineElement,
    PointElement,
    TimeScale,
    Tooltip,
    Legend,
} from "chart.js";
import "chartjs-adapter-dayjs-4";
import { Line } from "vue-chartjs";
import { UP, DOWN, PENDING, MAINTENANCE } from "../util.ts";

Chart.register(
    LineController,
    BarController,
    LineElement,
    PointElement,
    TimeScale,
    BarElement,
    LinearScale,
    Tooltip,
    Filler,
    Legend
);

export default {
    components: { Line },
    props: {
        /** ID of monitor */
        monitorId: {
            type: Number,
            required: true,
        },
    },
    data() {
        return {
            loading: false,

            // Time period for the chart to display, in hours
            // Initial value is 0 as a workaround for triggering a data fetch on created()
            chartPeriodHrs: "0",

            chartPeriodOptions: {
                0: this.$t("recent"),
                3: "3h",
                6: "6h",
                24: "24h",
                168: "1w",
            },

            chartRawData: null,
            chartDataFetchInterval: null,
        };
    },
    computed: {
        chartOptions() {
            return {
                responsive: true,
                maintainAspectRatio: false,
                onResize: (chart) => {
                    chart.canvas.parentNode.style.position = "relative";
                    if (screen.width < 576) {
                        chart.canvas.parentNode.style.height = "275px";
                    } else if (screen.width < 768) {
                        chart.canvas.parentNode.style.height = "320px";
                    } else if (screen.width < 992) {
                        chart.canvas.parentNode.style.height = "300px";
                    } else {
                        chart.canvas.parentNode.style.height = "250px";
                    }
                },
                layout: {
                    padding: {
                        left: 10,
                        right: 30,
                        top: 30,
                        bottom: 10,
                    },
                },

                elements: {
                    point: {
                        // Hide points on chart unless mouse-over
                        radius: 0,
                        hitRadius: 100,
                    },
                },
                scales: {
                    x: {
                        type: "time",
                        time: {
                            minUnit: "minute",
                            round: "second",
                            tooltipFormat: "YYYY-MM-DD HH:mm:ss",
                            displayFormats: {
                                minute: "HH:mm",
                                hour: "MM-DD HH:mm",
                            },
                        },
                        ticks: {
                            sampleSize: 3,
                            maxRotation: 0,
                            autoSkipPadding: 30,
                            padding: 3,
                        },
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                            offset: false,
                        },
                    },
                    y: {
                        title: {
                            display: true,
                            text: this.$t("respTime"),
                        },
                        offset: false,
                        grid: {
                            color: this.$root.theme === "light" ? "rgba(0,0,0,0.1)" : "rgba(255,255,255,0.1)",
                        },
                    },
                    y1: {
                        display: false,
                        position: "right",
                        grid: {
                            drawOnChartArea: false,
                        },
                        min: 0,
                        max: 1,
                        offset: false,
                    },
                },
                bounds: "ticks",
                plugins: {
                    tooltip: {
                        mode: "nearest",
                        intersect: false,
                        padding: 10,
                        backgroundColor: this.$root.theme === "light" ? "rgba(212,232,222,1.0)" : "rgba(32,42,38,1.0)",
                        bodyColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        titleColor: this.$root.theme === "light" ? "rgba(12,12,18,1.0)" : "rgba(220,220,220,1.0)",
                        // No longer rely solely on datasetIndex === 0; we want to hide tooltips only for the bars
                        filter: function (tooltipItem) {
                            const ds = tooltipItem?.chart?.data?.datasets?.[tooltipItem.datasetIndex];
                            return ds && ds.type !== "bar";
                        },
                        callbacks: {
                            label: (context) => {
                                const label = context.dataset.label;
                                return `${label} ${new Intl.NumberFormat().format(context.parsed.y)} ms`;
                            },
                        },
                    },
                    legend: {
                        display: false,
                    },
                },
            };
        },
        chartData() {
            if (this.chartPeriodHrs === "0") {
                return this.getChartDatapointsFromHeartbeatList();
            } else {
                return this.getChartDatapointsFromStats();
            }
        },
    },
    watch: {
        // Update chart data when the selected chart period changes
        chartPeriodHrs: function (newPeriod) {
            if (this.chartDataFetchInterval) {
                clearInterval(this.chartDataFetchInterval);
                this.chartDataFetchInterval = null;
            }

            // eslint-disable-next-line eqeqeq
            if (newPeriod == "0") {
                this.heartbeatList = null;
                this.$root.storage()["chart-period"] = newPeriod;
            } else {
                this.loading = true;

                let period;
                try {
                    period = parseInt(newPeriod);
                } catch (e) {
                    // Invalid period
                    period = 24;
                }

                this.$root.getMonitorChartData(this.monitorId, period, (res) => {
                    if (!res.ok) {
                        this.$root.toastError(res.msg);
                    } else {
                        this.chartRawData = res.data;
                        this.$root.storage()["chart-period"] = newPeriod;
                    }
                    this.loading = false;
                });

                this.chartDataFetchInterval = setInterval(
                    () => {
                        this.$root.getMonitorChartData(this.monitorId, period, (res) => {
                            if (res.ok) {
                                this.chartRawData = res.data;
                            }
                        });
                    },
                    5 * 60 * 1000
                );
            }
        },
    },
    created() {
        // Load chart period from storage if saved
        let period = this.$root.storage()["chart-period"];
        if (period != null) {
            // Has this ever been not a string?
            if (typeof period !== "string") {
                period = period.toString();
            }
            this.chartPeriodHrs = period;
        } else {
            this.chartPeriodHrs = "0";
        }
    },
    beforeUnmount() {
        if (this.chartDataFetchInterval) {
            clearInterval(this.chartDataFetchInterval);
        }
    },
    methods: {
        // Get color of bar chart for this datapoint
        getBarColorForDatapoint(datapoint) {
            if (datapoint.maintenance != null) {
                // Target is in maintenance
                return "rgba(23,71,245,0.41)";
            } else if (datapoint.down === 0) {
                // Target is up, no need to display a bar
                return "#000";
            } else if (datapoint.up === 0) {
                // Target is down
                return "rgba(220, 53, 69, 0.41)";
            } else {
                // Show yellow for mixed status
                return "rgba(245, 182, 23, 0.41)";
            }
        },
        // push datapoint to chartData
        pushDatapoint(datapoint, avgPingData, minPingData, maxPingData, downData, colorData) {
            const x = this.$root.unixToDateTime(datapoint.timestamp);

            // Show ping values if it was up in this period
            avgPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing != null ? datapoint.avgPing : null,
            });
            minPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing != null ? datapoint.minPing : null,
            });
            maxPingData.push({
                x,
                y: datapoint.up > 0 && datapoint.avgPing != null ? datapoint.maxPing : null,
            });
            downData.push({
                x,
                y: datapoint.down + (datapoint.maintenance || 0),
            });

            colorData.push(this.getBarColorForDatapoint(datapoint));
        },
        // get the average of a set of datapoints
        getAverage(datapoints) {
            const totalUp = datapoints.reduce((total, current) => total + current.up, 0);
            const totalDown = datapoints.reduce((total, current) => total + current.down, 0);
            const totalMaintenance = datapoints.reduce((total, current) => total + (current.maintenance || 0), 0);
            const totalPing = datapoints.reduce((total, current) => total + current.avgPing * current.up, 0);
            const minPing = datapoints.reduce((min, current) => Math.min(min, current.minPing), Infinity);
            const maxPing = datapoints.reduce((max, current) => Math.max(max, current.maxPing), 0);

            // Find the middle timestamp to use
            let midpoint = Math.floor(datapoints.length / 2);

            return {
                timestamp: datapoints[midpoint].timestamp,
                up: totalUp,
                down: totalDown,
                maintenance: totalMaintenance > 0 ? totalMaintenance : undefined,
                avgPing: totalUp > 0 ? totalPing / totalUp : 0,
                minPing,
                maxPing,
            };
        },
        getChartDatapointsFromHeartbeatList() {
            // Render chart using heartbeatList
            let lastHeartbeatTime;
            const monitorInterval = this.$root.monitorList[this.monitorId]?.interval;
            let pingData = []; // Ping Data for Line Chart, y-axis contains ping time
            let downData = []; // Down Data for Bar Chart, y-axis is 1 if target is down (red color), under maintenance (blue color) or pending (orange color), 0 if target is up
            let colorData = []; // Color Data for Bar Chart

            let heartbeatList =
                (this.monitorId in this.$root.heartbeatList && this.$root.heartbeatList[this.monitorId]) || [];

            for (const beat of heartbeatList) {
                const beatTime = this.$root.toDayjs(beat.time);
                const x = beatTime.format("YYYY-MM-DD HH:mm:ss");

                // Insert empty datapoint to separate big gaps
                if (lastHeartbeatTime && monitorInterval) {
                    const diff = Math.abs(beatTime.diff(lastHeartbeatTime));
                    if (diff > monitorInterval * 1000 * 10) {
                        // Big gap detected
                        const gapX = [
                            lastHeartbeatTime.add(monitorInterval, "second").format("YYYY-MM-DD HH:mm:ss"),
                            beatTime.subtract(monitorInterval, "second").format("YYYY-MM-DD HH:mm:ss"),
                        ];

                        for (const x of gapX) {
                            pingData.push({
                                x,
                                y: null,
                            });
                            downData.push({
                                x,
                                y: null,
                            });
                            colorData.push("#000");
                        }
                    }
                }

                pingData.push({
                    x,
                    y: beat.status === UP ? beat.ping : null,
                });
                downData.push({
                    x,
                    y: beat.status === DOWN || beat.status === MAINTENANCE || beat.status === PENDING ? 1 : 0,
                });
                switch (beat.status) {
                    case MAINTENANCE:
                        colorData.push("rgba(23 ,71, 245, 0.41)");
                        break;
                    case PENDING:
                        colorData.push("rgba(245, 182, 23, 0.41)");
                        break;
                    default:
                        colorData.push("rgba(220, 53, 69, 0.41)");
                }

                lastHeartbeatTime = beatTime;
            }

            return {
                datasets: [
                    {
                        // Line Chart
                        data: pingData,
                        fill: "origin",
                        tension: 0.2,
                        borderWidth: 1.5,
                        borderColor: "#4ABF74",
                        backgroundColor: "#4ABF7438",
                        yAxisID: "y",
                        label: this.$t("avgPing"),
                    },
                    {
                        // Bar Chart
                        type: "bar",
                        data: downData,
                        borderColor: "#00000000",
                        backgroundColor: colorData,
                        yAxisID: "y1",
                        barThickness: "flex",
                        barPercentage: 1,
                        categoryPercentage: 1,
                        inflateAmount: 0.05,
                        label: "status",
                    },
                ],
            };
        },
        getChartDatapointsFromStats() {
            // Render chart using UptimeCalculator data
            let lastHeartbeatTime;
            const monitorInterval = this.$root.monitorList[this.monitorId]?.interval;

            let avgPingData = []; // Ping Data for Line Chart, y-axis contains ping time
            let minPingData = []; // Ping Data for Line Chart, y-axis contains ping time
            let maxPingData = []; // Ping Data for Line Chart, y-axis contains ping time
            let downData = []; // Down Data for Bar Chart, y-axis is number of down datapoints in this period
            let colorData = []; // Color Data for Bar Chart

            const period = parseInt(this.chartPeriodHrs);
            let aggregatePoints = period > 6 ? 12 : 4;

            let aggregateBuffer = [];

            if (this.chartRawData) {
                for (const datapoint of this.chartRawData) {
                    // Empty datapoints are ignored
                    if (datapoint.up === 0 && datapoint.down === 0 && datapoint.maintenance === 0) {
                        continue;
                    }

                    const beatTime = this.$root.unixToDayjs(datapoint.timestamp);

                    // Insert empty datapoint to separate big gaps
                    if (lastHeartbeatTime && monitorInterval) {
                        const diff = Math.abs(beatTime.diff(lastHeartbeatTime));
                        const oneSecond = 1000;
                        const oneMinute = oneSecond * 60;
                        const oneHour = oneMinute * 60;
                        if (
                            (period <= 24 && diff > Math.max(oneMinute * 10, monitorInterval * oneSecond * 10)) ||
                            (period > 24 && diff > Math.max(oneHour * 10, monitorInterval * oneSecond * 10))
                        ) {
                            // Big gap detected
                            // Clear the aggregate buffer
                            if (aggregateBuffer.length > 0) {
                                const average = this.getAverage(aggregateBuffer);
                                this.pushDatapoint(average, avgPingData, minPingData, maxPingData, downData, colorData);
                                aggregateBuffer = [];
                            }

                            const gapX = [
                                lastHeartbeatTime.subtract(monitorInterval, "second").format("YYYY-MM-DD HH:mm:ss"),
                                this.$root.unixToDateTime(datapoint.timestamp + 60),
                            ];

                            for (const x of gapX) {
                                avgPingData.push({
                                    x,
                                    y: null,
                                });
                                minPingData.push({
                                    x,
                                    y: null,
                                });
                                maxPingData.push({
                                    x,
                                    y: null,
                                });
                                downData.push({
                                    x,
                                    y: null,
                                });
                                colorData.push("#000");
                            }
                        }
                    }

                    if (datapoint.up > 0 && this.chartRawData.length > aggregatePoints * 2) {
                        // Aggregate Up data using a sliding window
                        aggregateBuffer.push(datapoint);

                        if (aggregateBuffer.length === aggregatePoints) {
                            const average = this.getAverage(aggregateBuffer);
                            this.pushDatapoint(average, avgPingData, minPingData, maxPingData, downData, colorData);
                            // Remove the first half of the buffer
                            aggregateBuffer = aggregateBuffer.slice(Math.floor(aggregatePoints / 2));
                        }
                    } else {
                        // datapoint is fully down or too few datapoints, no need to aggregate
                        // Clear the aggregate buffer
                        if (aggregateBuffer.length > 0) {
                            const average = this.getAverage(aggregateBuffer);
                            this.pushDatapoint(average, avgPingData, minPingData, maxPingData, downData, colorData);
                            aggregateBuffer = [];
                        }

                        this.pushDatapoint(datapoint, avgPingData, minPingData, maxPingData, downData, colorData);
                    }

                    lastHeartbeatTime = beatTime;
                }
                // Clear the aggregate buffer if there are still datapoints
                if (aggregateBuffer.length > 0) {
                    const average = this.getAverage(aggregateBuffer);
                    this.pushDatapoint(average, avgPingData, minPingData, maxPingData, downData, colorData);
                    aggregateBuffer = [];
                }
            }

            return {
                datasets: [
                    {
                        // minimum ping chart
                        data: minPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderWidth: 1.5,
                        borderColor: "#126331",
                        backgroundColor: "#2F9C5914",
                        yAxisID: "y",
                        label: this.$t("minPing"),
                    },
                    {
                        // average ping chart
                        data: avgPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderWidth: 1.5,
                        borderColor: "#5CDD8B",
                        backgroundColor: "#5CDD8B06",
                        yAxisID: "y",
                        label: this.$t("avgPing"),
                    },
                    {
                        // maximum ping chart
                        data: maxPingData,
                        fill: "origin",
                        tension: 0.2,
                        borderWidth: 1.5,
                        borderColor: "#21b55a",
                        backgroundColor: "#1E7A4214",
                        yAxisID: "y",
                        label: this.$t("maxPing"),
                    },
                    {
                        // Bar Chart
                        type: "bar",
                        data: downData,
                        borderColor: "#00000000",
                        backgroundColor: colorData,
                        yAxisID: "y1",
                        barThickness: "flex",
                        barPercentage: 1,
                        categoryPercentage: 1,
                        inflateAmount: 0.05,
                        label: "status",
                    },
                ],
            };
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.ping-chart {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.chart-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
}

.chart-legend {
    display: flex;
    align-items: center;
    gap: 12px;
    flex: 1;
}

.legend-item {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    font-size: 0.75rem;
    font-weight: 600;
    color: $text-secondary;

    .dark & { color: $dark-font-color; }
}

.legend-line {
    display: inline-block;
    width: 18px;
    height: 2px;
    border-radius: 1px;
    flex-shrink: 0;
}

.chart-title {
    font-family: $font-family-base;
    font-size: 0.75rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: $text-secondary;

    .dark & { color: $dark-font-color; }
}

.period-options {
    position: relative;
    z-index: 10;

    .dropdown-menu {
        min-width: 80px;
        padding: 4px;
        border-radius: $input-radius;
        border: 1px solid $border-light;
        box-shadow: $shadow-float;
        font-size: 0.8rem;

        .dark & {
            background: $dark-surface;
            border-color: $dark-border-color;
        }

        .dropdown-item {
            border-radius: calc($input-radius - 2px);
            padding: 4px 12px;
            font-family: $font-family-base;
            font-size: 0.8rem;
            font-weight: 500;
            color: $text-primary;

            &:hover { background-color: $surface-subtle; }

            &.active {
                background-color: $text-primary;
                color: white;
            }

            .dark & {
                color: $dark-font-color;
                &:hover { background: $dark-bg; }
            }

            .dark &.active {
                background: $primary;
                color: $dark-font-color2;
            }
        }
    }
}

.btn-period-toggle {
    font-family: $font-family-base;
    font-size: 0.8rem;
    font-weight: 600;
    padding: 3px 10px;
    background: transparent;
    border: 1px solid $border-light;
    border-radius: $input-radius;
    color: $text-secondary;
    transition: border-color 0.12s ease, background-color 0.12s ease;

    &:hover {
        background-color: $surface-subtle;
        border-color: #cbd5e1;
        color: $text-primary;
    }

    &::after { vertical-align: 0.15em; }

    .dark & {
        color: $dark-font-color;
        border-color: $dark-border-color;

        &:hover {
            background-color: $dark-surface;
        }
    }
}

.chart-wrapper {
    &.loading { filter: blur(10px); }
}

.form-select {
    width: unset;
    display: inline-flex;
}
</style>
