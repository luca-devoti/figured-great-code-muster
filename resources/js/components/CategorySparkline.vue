<script setup>
import { CategoryScale, Chart, LinearScale, LineController, LineElement, PointElement, Tooltip } from 'chart.js';
import { onActivated, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import { money, monthName } from '../format';

Chart.register(LineController, LineElement, PointElement, LinearScale, CategoryScale, Tooltip);

const props = defineProps({
    // [{ month: '2025-06-01', budget: 1234, actual: 1180 }, ...] sorted by month.
    entries: { type: Array, required: true },
});

const canvas = ref(null);
let chart = null;

function render() {
    if (!canvas.value) return;

    const labels = props.entries.map((e) => monthName(e.month));
    const budgetData = props.entries.map((e) => e.budget);
    const actualData = props.entries.map((e) => e.actual);

    if (chart) {
        chart.data.labels = labels;
        chart.data.datasets[0].data = budgetData;
        chart.data.datasets[1].data = actualData;
        chart.update();
        return;
    }

    chart = new Chart(canvas.value, {
        type: 'line',
        data: {
            labels,
            datasets: [
                {
                    label: 'Budget',
                    data: budgetData,
                    borderColor: '#90a4ae',
                    borderDash: [3, 3],
                    borderWidth: 1.5,
                    pointRadius: 0,
                    tension: 0.3,
                },
                {
                    label: 'Actual',
                    data: actualData,
                    borderColor: '#296fdc',
                    borderWidth: 2,
                    pointRadius: 0,
                    tension: 0.3,
                },
            ],
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: false,
            interaction: { intersect: false, mode: 'index' },
            scales: {
                x: { display: false },
                y: { display: false },
            },
            plugins: {
                legend: { display: false },
                tooltip: {
                    callbacks: {
                        label: (ctx) => `${ctx.dataset.label}: ${money(ctx.parsed.y)}`,
                    },
                },
            },
        },
    });
}

function destroy() {
    chart?.destroy();
    chart = null;
}

onMounted(render);
watch(() => props.entries, render);
onBeforeUnmount(destroy);

// This row lives inside a <KeepAlive>'d page - when the tab is hidden its
// container measures as 0x0. A plain resize() wasn't reliably picking up the
// restored size for these nested per-row canvases, so rebuild the chart from
// scratch on reactivation instead - more expensive but reliably correct.
onActivated(() => {
    destroy();
    render();
});
</script>

<template>
    <div class="h-9 w-32">
        <canvas ref="canvas"></canvas>
    </div>
</template>
