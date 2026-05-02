<!-- 行政區違規件數排行 — 水平 BarChart -->
<script setup>
import { ref, computed } from "vue";
import VueApexCharts from "vue3-apexcharts";

const props = defineProps([
	"chart_config",
	"activeChart",
	"series",
	"map_config",
	"map_filter",
	"map_filter_on",
]);

const emits = defineEmits([
	"filterByParam",
	"filterByLayer",
	"clearByParamFilter",
	"clearByLayerFilter",
	"fly",
]);

const COLORS = [
	"#C0392B","#E74C3C","#E67E22","#F39C12","#F1C40F",
	"#27AE60","#1ABC9C","#2980B9","#8E44AD","#607D8B",
	"#D35400","#2C3E50",
];

const chartHeight = computed(() => {
	const count = props.series?.[0]?.data?.length ?? 8;
	return Math.max(count * 40, 200) + "px";
});

const chartOptions = ref({
	chart: {
		offsetY: 8,
		stacked: false,
		toolbar: { show: false },
		zoom: { allowMouseWheelZoom: false },
		animations: { enabled: true, easing: "easeinout", speed: 600 },
	},
	colors: COLORS,
	dataLabels: {
		enabled: true,
		offsetX: 28,
		textAnchor: "start",
		formatter: (val) => val + " 件",
		style: { fontSize: "11px", fontWeight: 500, colors: ["#bbb"] },
		dropShadow: { enabled: false },
	},
	grid: { show: false },
	legend: { show: false },
	plotOptions: {
		bar: {
			borderRadius: 4,
			distributed: true,
			horizontal: true,
			barHeight: "65%",
			dataLabels: { hideOverflowingLabels: false },
		},
	},
	states: {
		hover: { filter: { type: "lighten", value: 0.15 } },
		active: { filter: { type: "darken", value: 0.2 } },
	},
	stroke: {
		colors: ["transparent"],
		show: true,
		width: 1,
	},
	tooltip: {
		custom: ({ series, seriesIndex, dataPointIndex, w }) =>
			`<div class="chart-tooltip">` +
			`<h6>${w.globals.labels[dataPointIndex]}</h6>` +
			`<span>${series[seriesIndex][dataPointIndex]} ${props.chart_config.unit}</span>` +
			`</div>`,
		followCursor: true,
	},
	xaxis: {
		axisBorder: { show: false },
		axisTicks: { show: false },
		labels: { show: false },
		type: "category",
	},
	yaxis: {
		labels: {
			style: { fontSize: "12px", colors: "#aaa" },
		},
	},
});

const selectedIndex = ref(null);

function handleClick(_e, _ctx, config) {
	if (!props.map_filter || !props.map_filter_on) return;
	const key = `${config.dataPointIndex}`;
	if (key !== selectedIndex.value) {
		emits("filterByParam", props.map_filter, props.map_config,
			config.w.globals.labels[config.dataPointIndex], "");
		selectedIndex.value = key;
	} else {
		emits("clearByParamFilter", props.map_config);
		selectedIndex.value = null;
	}
}
</script>

<template>
	<div v-if="activeChart === 'BarChart'" class="food-district-bar">
		<VueApexCharts
			type="bar"
			:height="chartHeight"
			width="100%"
			:options="chartOptions"
			:series="series"
			@data-point-selection="handleClick"
		/>
	</div>
</template>

<style scoped lang="scss">
.food-district-bar {
	width: 100%;
	height: 100%;
	overflow-y: auto;
	overflow-x: hidden;
	position: relative;
}
</style>
