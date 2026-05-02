<!-- 社群食安事件類型分布 — DonutChart -->
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

const steps = ref(100);

const parsedSeries = computed(() => {
	if (!props.series?.[0]?.data) return [];
	const data = props.series[0].data.slice(0, steps.value);
	return data.map((item) => item.y);
});

const parsedLabels = computed(() => {
	if (!props.series?.[0]?.data) return [];
	const data = props.series[0].data.slice(0, steps.value);
	return data.map((item) => item.x);
});

const total = computed(() =>
	parsedSeries.value.reduce((a, b) => a + b, 0)
);

const chartOptions = computed(() => ({
	chart: {
		toolbar: { show: false },
		animations: { enabled: true, easing: "easeinout", speed: 600 },
	},
	colors: [...(props.chart_config?.color ?? ["#E74C3C","#E67E22","#F1C40F","#3498DB","#9B59B6"])],
	dataLabels: {
		enabled: false,
	},
	labels: parsedLabels.value,
	legend: {
		show: true,
		position: "bottom",
		horizontalAlign: "center",
		fontSize: "12px",
		labels: { colors: "#ccc" },
		markers: { width: 10, height: 10, radius: 3 },
		itemMargin: { horizontal: 8, vertical: 2 },
		formatter: (label, opts) =>
			`${label} (${opts.w.globals.series[opts.seriesIndex]})`,
	},
	plotOptions: {
		pie: {
			donut: {
				size: "62%",
				labels: {
					show: true,
					total: {
						show: true,
						label: "總通報",
						fontSize: "12px",
						color: "#aaa",
						formatter: () => total.value + " 則",
					},
					value: {
						show: true,
						fontSize: "18px",
						fontWeight: 700,
						color: "#fff",
						formatter: (val) => val + " 則",
					},
				},
			},
		},
	},
	stroke: {
		colors: ["#1e2022"],
		width: 2,
	},
	tooltip: {
		custom: ({ series, seriesIndex, w }) =>
			`<div class="chart-tooltip">` +
			`<h6>${w.globals.labels[seriesIndex]}</h6>` +
			`<span>${series[seriesIndex]} ${props.chart_config?.unit ?? "則"}</span>` +
			`</div>`,
	},
}));
</script>

<template>
	<div v-if="activeChart === 'DonutChart'" class="food-category-donut">
		<VueApexCharts
			type="donut"
			height="260px"
			width="100%"
			:options="chartOptions"
			:series="parsedSeries"
		/>
	</div>
</template>

<style scoped lang="scss">
.food-category-donut {
	width: 100%;
	height: 100%;
	display: flex;
	align-items: center;
	justify-content: center;
	overflow: hidden;
	position: relative;
}
</style>
