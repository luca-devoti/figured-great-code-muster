<script setup>
import axios from 'axios';
import { CategoryScale, Chart, Legend, LinearScale, LineController, LineElement, PointElement, Tooltip } from 'chart.js';
import { computed, nextTick, onActivated, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import CategorySparkline from '../components/CategorySparkline.vue';
import { money, monthName } from '../format';

Chart.register(LineController, LineElement, PointElement, LinearScale, CategoryScale, Tooltip, Legend);

const farms = ref([]);
const selectedFarmId = ref(null);
const farm = ref(null);
const lines = ref([]);
const weather = ref([]);
const loading = ref(true);

const generating = ref(false);
const generateError = ref('');
const summary = ref(null);

onMounted(async () => {
    const { data } = await axios.get('/api/farms');
    farms.value = data;
    // Pick up whatever farm was last selected on the Monthly Report page, if any.
    const stored = Number(localStorage.getItem('selectedFarmId'));
    selectedFarmId.value = data.find((f) => f.id === stored)?.id ?? data[0]?.id ?? null;
});

watch(selectedFarmId, onFarmChange);

async function onFarmChange(id) {
    if (!id) return;
    localStorage.setItem('selectedFarmId', id);
    summary.value = null;
    generateError.value = '';
    loading.value = true;
    const { data } = await axios.get(`/api/farms/${id}/report`);
    farm.value = data.farm;
    lines.value = data.lines;
    weather.value = data.weather;
    loading.value = false;
    await nextTick();
    renderMainChart();
}

// Group the flat list of report lines by category, in first-seen order.
const categories = computed(() => {
    const map = new Map();
    for (const line of lines.value) {
        if (!map.has(line.category)) map.set(line.category, []);
        map.get(line.category).push(line);
    }
    for (const entries of map.values()) entries.sort((a, b) => a.month.localeCompare(b.month));
    return map;
});

const months = computed(() => [...new Set(lines.value.map((l) => l.month))].sort());

// Simple heuristic: report-line categories that name an income stream contain
// "Income" or "Sales" (Milk Income, Lamb Sales, Wheat Income...); everything
// else in this table is a cost line.
function isIncomeCategory(name) {
    return /income|sales/i.test(name);
}

function categoryTotals(entries) {
    const budget = entries.reduce((sum, l) => sum + l.budget, 0);
    const actual = entries.reduce((sum, l) => sum + l.actual, 0);
    return { budget, actual, variance: actual - budget };
}

function isFavourable(categoryName, variance) {
    return isIncomeCategory(categoryName) ? variance >= 0 : variance <= 0;
}

const incomeRows = computed(() =>
    [...categories.value.entries()]
        .filter(([name]) => isIncomeCategory(name))
        .map(([name, entries]) => ({ name, entries, ...categoryTotals(entries) })),
);

const costRows = computed(() =>
    [...categories.value.entries()]
        .filter(([name]) => !isIncomeCategory(name))
        .map(([name, entries]) => ({ name, entries, ...categoryTotals(entries) })),
);

const netPosition = computed(() => {
    const sum = (rows, key) => rows.reduce((total, r) => total + r[key], 0);
    const budgetIncome = sum(incomeRows.value, 'budget');
    const actualIncome = sum(incomeRows.value, 'actual');
    const budgetCost = sum(costRows.value, 'budget');
    const actualCost = sum(costRows.value, 'actual');
    return {
        budgetNet: budgetIncome - budgetCost,
        actualNet: actualIncome - actualCost,
    };
});

// Month-by-month net (income minus cost), then a running year-to-date total -
// this is what answers "am I ahead or behind, and when did that happen?".
const monthlyNet = computed(() =>
    months.value.map((month) => {
        let budgetIncome = 0;
        let actualIncome = 0;
        let budgetCost = 0;
        let actualCost = 0;
        for (const line of lines.value) {
            if (line.month !== month) continue;
            if (isIncomeCategory(line.category)) {
                budgetIncome += line.budget;
                actualIncome += line.actual;
            } else {
                budgetCost += line.budget;
                actualCost += line.actual;
            }
        }
        return { month, budgetNet: budgetIncome - budgetCost, actualNet: actualIncome - actualCost };
    }),
);

const cumulativeNet = computed(() => {
    let budgetRunning = 0;
    let actualRunning = 0;
    return monthlyNet.value.map((m) => {
        budgetRunning += m.budgetNet;
        actualRunning += m.actualNet;
        return { month: m.month, budgetCumulative: budgetRunning, actualCumulative: actualRunning };
    });
});

const mainChartCanvas = ref(null);
let mainChart = null;

function renderMainChart() {
    if (!mainChartCanvas.value || !cumulativeNet.value.length) return;

    const labels = cumulativeNet.value.map((c) => monthName(c.month));
    const budgetData = cumulativeNet.value.map((c) => c.budgetCumulative);
    const actualData = cumulativeNet.value.map((c) => c.actualCumulative);

    if (mainChart) {
        mainChart.data.labels = labels;
        mainChart.data.datasets[0].data = budgetData;
        mainChart.data.datasets[1].data = actualData;
        mainChart.update();
        return;
    }

    mainChart = new Chart(mainChartCanvas.value, {
        type: 'line',
        data: {
            labels,
            datasets: [
                {
                    label: 'Budgeted profit (year to date)',
                    data: budgetData,
                    borderColor: '#90a4ae',
                    borderDash: [5, 5],
                    borderWidth: 2,
                    pointRadius: 2,
                    tension: 0.2,
                },
                {
                    label: 'Actual profit (year to date)',
                    data: actualData,
                    borderColor: '#296fdc',
                    borderWidth: 2.5,
                    pointRadius: 2,
                    tension: 0.2,
                },
            ],
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            interaction: { intersect: false, mode: 'index' },
            plugins: {
                legend: { position: 'bottom', labels: { boxWidth: 12, font: { size: 11 } } },
                tooltip: {
                    callbacks: {
                        label: (ctx) => `${ctx.dataset.label}: ${money(ctx.parsed.y)}`,
                    },
                },
            },
            scales: {
                y: {
                    ticks: { callback: (value) => money(value) },
                    grid: { color: '#eceff1' },
                },
                x: {
                    grid: { display: false },
                },
            },
        },
    });
}

function destroyMainChart() {
    mainChart?.destroy();
    mainChart = null;
}

onBeforeUnmount(destroyMainChart);

// The whole app is wrapped in <KeepAlive>, so switching away from this tab
// hides rather than unmounts it. A hidden container measures as 0x0, and a
// plain resize() didn't reliably pick up the restored size, so rebuild the
// chart from scratch on reactivation instead - otherwise it comes back blank.
onActivated(() => {
    destroyMainChart();
    renderMainChart();
});

// Seasonal domain knowledge, one block per farm type - only the block matching
// the selected farm is sent, so Claude never mixes up e.g. dairy and arable
// calendars.
const FARM_CALENDARS = {
    Dairy: `Dairy: the season runs roughly June to May. Cows calve June-August (start of the season), so vet/animal health and wages costs are often higher in that window. Milk production - and milk income - is low over calving, builds through spring as pasture grows (peaking roughly October-December), then tapers as cows are progressively "dried off" over autumn ahead of the next calving. Feed costs are typically highest when pasture is short (early season, or during a dry spell) and lowest at the spring growth peak. A drought or dry spell will usually show up as HIGHER feed costs (buying in supplement) and LOWER milk income (less pasture, less milk) in the same window.`,
    'Sheep & Beef': `Sheep & Beef: lambs are typically born in spring (roughly August-October, after winter mating), then sold ("marketed") from around November through into autumn, with sale numbers usually building over summer as lambs reach weight and tapering off by autumn. Wool income usually arrives in one or two lump sums a year, tied to shearing dates, rather than every month. Cattle sales tend to be lower-frequency, larger one-off transactions rather than a steady monthly income.`,
    Arable: `Arable / cropping: income is concentrated in the harvest window for each crop (commonly summer, roughly December-March for wheat and barley in New Zealand) and is usually $0 for the rest of the year. Costs run the other way: seed and cultivation costs cluster around planting time (autumn for winter-sown crops, and again in autumn for the next season), fertiliser and agrichemical costs build through the growing season, and contracting costs (machinery hire, e.g. for harvest) spike sharply in the harvest months. A wet spell right around harvest can delay machinery and push income from one month into the next.`,
};

function buildSystemPrompt(farmType) {
    const calendar =
        FARM_CALENDARS[farmType] ??
        'No seasonal calendar notes are available for this farm type - rely only on the weather notes and timing patterns in the figures themselves, and say so if you are not sure.';

    return `You are a rural accounting adviser at Southdown Rural Accountants in the Waikato, New Zealand. You are turning a farm's detailed budget-vs-actual accounts into a short, plain-English explanation that the farm's OWNER (not an accountant) can read and understand at a glance.

AUDIENCE
The reader has no accounting background. Never use a technical term (variance, GST, accrual, drawings, etc.) without briefly explaining it in plain words the first time it appears. Keep sentences short. Write like you are talking to the farmer over the kitchen table, not writing a compliance report.

OUTPUT FORMAT - respond with ONLY valid JSON, no markdown code fences, no text outside the JSON, matching exactly this shape:
{
  "headline": "one short, plain-English sentence summarising the year",
  "how_the_year_went": "2-4 sentences giving an overview of income and costs across the year",
  "highlights": [
    { "title": "short label for this highlight", "explanation": "1-3 plain-English sentences" }
  ],
  "questions_to_ask_your_adviser": ["short question", "short question"]
}
Include 3 to 5 items in "highlights" and 0 to 3 items in "questions_to_ask_your_adviser" (it is fine to leave the questions list empty if there is genuinely nothing unclear).

HARD RULES - THESE MATTER
1. Only ever state a dollar figure that appears verbatim in the data given to you below. Never estimate, round differently, or invent a number.
2. Every explanation you give for WHY something happened must be backed by evidence you can point to in the data below: the weather notes for that month, an offsetting change in a neighbouring month, or the farm's seasonal calendar (below). If you cannot find that evidence, say plainly "it's not clear from the figures why this happened" and put it in "questions_to_ask_your_adviser" instead of guessing.
3. Do not give tax, legal, or investment advice. If something looks tax-related, say it is "worth checking with your adviser" rather than explaining tax rules yourself.

HOW TO READ A VARIANCE
"Budget" is what the farm planned to earn or spend. "Actual" is what really happened. The gap between them is called a "variance" - explain this in plain words the first time you use it. A few genuine reasons a variance shows up (only use one of these if the data actually supports it):
- Weather: a wet, dry, or extreme month (check the weather notes given to you for the months in question) can delay farm work, cut pasture or crop growth, or force extra spending (e.g. buying in feed during a drought, or hiring extra machinery after a storm).
- Timing shift: sometimes money budgeted for one month actually lands in the month before or after (e.g. an invoice paid late, produce harvested or sold later than planned because of weather). A good sign of this is one month running well over budget in a category while a neighbouring month for the SAME category runs correspondingly under budget, roughly netting out across the two. If you spot this pattern, say so - it is usually reassuring news, not something to worry about.
- Farm calendar / seasonality: farm income and costs are naturally lumpy across the year, not spread evenly. See the calendar notes below for what is normal for this farm type.
- Genuinely unclear: if none of the above fit the data you were given, say the figures do not explain the cause and suggest the farmer ask their adviser. Do not invent a plausible-sounding reason.

NEW ZEALAND FARM CALENDAR - context for what is a NORMAL seasonal pattern for THIS farm, so you do not mistake an expected seasonal swing for a problem:
${calendar}

TONE
Be warm and direct, like a trusted adviser, not a corporate report. It is fine to say "nothing to worry about here" when that is true - the farmer does not want to be alarmed by normal seasonal swings. Only flag things genuinely worth their attention.`;
}

function buildDataBlock() {
    const parts = [];
    parts.push(`Farm: ${farm.value.name} (${farm.value.type}), owner ${farm.value.owner_name}`);
    parts.push(`Financial year shown: ${monthName(months.value[0])} - ${monthName(months.value[months.value.length - 1])}`);
    parts.push('');
    parts.push('BUDGET VS ACTUAL BY CATEGORY (NZD):');
    for (const [category, entries] of categories.value) {
        parts.push(`${category}:`);
        for (const line of entries) {
            parts.push(`  ${monthName(line.month)}: budget ${money(line.budget)}, actual ${money(line.actual)}`);
        }
    }
    parts.push('');
    parts.push('WEATHER NOTES BY MONTH (regional, same for all farms):');
    for (const w of weather.value) {
        parts.push(
            `${monthName(w.month)}: ${w.rainfall_mm}mm rainfall (${w.rainfall_pct_normal}% of normal), mean temp ${w.mean_temp_c}°C, soil moisture deficit ${w.soil_moisture_deficit_mm}mm. ${w.notes}`,
        );
    }
    return parts.join('\n');
}

function parseSummary(text) {
    const tryParse = (s) => {
        try {
            return JSON.parse(s);
        } catch {
            return null;
        }
    };

    const cleaned = text
        .trim()
        .replace(/^```(?:json)?\s*/i, '')
        .replace(/```\s*$/i, '')
        .trim();

    let parsed = tryParse(cleaned);
    if (!parsed) {
        const match = cleaned.match(/\{[\s\S]*\}/);
        if (match) parsed = tryParse(match[0]);
    }

    if (!parsed || typeof parsed.headline !== 'string' || typeof parsed.how_the_year_went !== 'string' || !Array.isArray(parsed.highlights)) {
        return null;
    }

    return {
        headline: parsed.headline,
        how_the_year_went: parsed.how_the_year_went,
        highlights: parsed.highlights.filter((h) => h && typeof h.title === 'string' && typeof h.explanation === 'string'),
        questions: Array.isArray(parsed.questions_to_ask_your_adviser)
            ? parsed.questions_to_ask_your_adviser.filter((q) => typeof q === 'string')
            : [],
    };
}

async function generateSummary() {
    generating.value = true;
    generateError.value = '';
    summary.value = null;
    try {
        const { data } = await axios.post('/api/ai', {
            system: buildSystemPrompt(farm.value.type),
            prompt: buildDataBlock(),
        });
        const parsed = parseSummary(data.text);
        if (!parsed) {
            generateError.value = "Claude's reply wasn't in the expected format - try generating again.";
        } else {
            summary.value = parsed;
        }
    } catch (e) {
        generateError.value = e.response?.data?.error ?? e.message;
    } finally {
        generating.value = false;
    }
}
</script>

<template>
    <div class="mx-auto max-w-3xl">
        <div class="mb-4">
            <h2 class="text-lg font-semibold">Client view</h2>
            <p class="text-sm text-fg-mid-grey">
                A plain-English, no-jargon look at the farm's year — built to hand to the farmer, not just an
                accountant. The figures below come straight from the accounts; the summary is a Claude-drafted
                first pass to read over before sharing.
            </p>
        </div>

        <select v-model="selectedFarmId" class="mb-4 rounded border border-fg-muted-grey bg-white px-3 py-1.5 text-sm">
            <option v-for="f in farms" :key="f.id" :value="f.id">{{ f.name }} ({{ f.type }} — {{ f.owner_name }})</option>
        </select>

        <p v-if="loading" class="text-fg-light-grey">Loading…</p>

        <div v-else class="space-y-4">
            <!-- AI-drafted summary -->
            <div class="rounded border border-fg-muted-grey bg-white p-5">
                <div class="mb-3 flex items-center justify-between">
                    <h3 class="text-sm font-semibold">Plain-English summary</h3>
                    <button
                        class="shrink-0 rounded bg-fg-main-blue px-3 py-1.5 text-sm font-medium text-white hover:bg-fg-main-blue-hover disabled:opacity-50"
                        :disabled="generating || !lines.length"
                        @click="generateSummary"
                    >
                        {{ generating ? 'Explaining…' : summary ? '✨ Regenerate' : '✨ Explain this year in plain English' }}
                    </button>
                </div>

                <p v-if="generateError" class="mb-3 rounded bg-fg-danger-9 p-3 text-sm text-fg-danger-dark">{{ generateError }}</p>

                <p v-if="!summary && !generating && !generateError" class="text-sm text-fg-light-grey">
                    Click the button to have Claude draft a plain-English explanation of {{ farm?.name }}'s year, grounded
                    in the actual figures and this year's weather.
                </p>

                <div v-if="summary" class="space-y-4">
                    <p class="text-base font-medium text-fg-dark-blue">{{ summary.headline }}</p>
                    <p class="text-sm leading-relaxed text-fg-dark-grey">{{ summary.how_the_year_went }}</p>

                    <div v-if="summary.highlights.length" class="space-y-2">
                        <div
                            v-for="(h, i) in summary.highlights"
                            :key="i"
                            class="rounded border border-fg-pale-grey bg-fg-super-pale-grey p-3"
                        >
                            <p class="text-sm font-semibold">{{ h.title }}</p>
                            <p class="mt-1 text-sm leading-relaxed text-fg-dark-grey">{{ h.explanation }}</p>
                        </div>
                    </div>

                    <div v-if="summary.questions.length">
                        <p class="text-sm font-semibold">Worth asking your adviser about</p>
                        <ul class="mt-1 list-disc space-y-1 pl-5 text-sm text-fg-dark-grey">
                            <li v-for="(q, i) in summary.questions" :key="i">{{ q }}</li>
                        </ul>
                    </div>

                    <p class="text-xs text-fg-light-grey">Drafted by AI from this farm's figures — read it over before sharing with the client.</p>
                </div>
            </div>

            <!-- The real numbers, rendered directly from the database - never touched by AI. -->
            <div class="rounded border border-fg-muted-grey bg-white p-5">
                <h3 class="mb-3 text-sm font-semibold">Your year at a glance — {{ monthName(months[0]) }} to {{ monthName(months[months.length - 1]) }}</h3>

                <p class="mb-1 text-xs font-medium text-fg-light-grey">Profit so far this year, budget vs actual</p>
                <div class="mb-4 h-64 w-full">
                    <canvas ref="mainChartCanvas"></canvas>
                </div>

                <div class="mb-4 grid grid-cols-3 gap-3 text-center">
                    <div class="rounded bg-fg-super-pale-grey p-3">
                        <p class="text-xs text-fg-light-grey">Budgeted profit</p>
                        <p class="text-lg font-semibold">{{ money(netPosition.budgetNet) }}</p>
                    </div>
                    <div class="rounded bg-fg-super-pale-grey p-3">
                        <p class="text-xs text-fg-light-grey">Actual profit</p>
                        <p class="text-lg font-semibold">{{ money(netPosition.actualNet) }}</p>
                    </div>
                    <div
                        class="rounded p-3"
                        :class="netPosition.actualNet >= netPosition.budgetNet ? 'bg-fg-positive-9' : 'bg-fg-danger-9'"
                    >
                        <p class="text-xs text-fg-light-grey">Difference</p>
                        <p
                            class="text-lg font-semibold"
                            :class="netPosition.actualNet >= netPosition.budgetNet ? 'text-fg-positive-dark' : 'text-fg-danger-dark'"
                        >
                            {{ money(netPosition.actualNet - netPosition.budgetNet) }}
                        </p>
                    </div>
                </div>

                <p class="mb-1 text-xs font-medium text-fg-light-grey">Income</p>
                <table class="mb-4 w-full text-sm">
                    <tbody>
                        <tr v-for="row in incomeRows" :key="row.name" class="border-t border-fg-pale-grey">
                            <td class="py-1">{{ row.name }}</td>
                            <td class="py-1"><CategorySparkline :entries="row.entries" /></td>
                            <td class="py-1 text-right font-mono text-xs text-fg-light-grey">budget {{ money(row.budget) }}</td>
                            <td class="py-1 text-right font-mono text-xs">actual {{ money(row.actual) }}</td>
                            <td
                                class="py-1 pl-2 text-right font-mono text-xs font-medium"
                                :class="isFavourable(row.name, row.variance) ? 'text-fg-positive-dark' : 'text-fg-danger-dark'"
                            >
                                {{ row.variance >= 0 ? '+' : '' }}{{ money(row.variance) }}
                            </td>
                        </tr>
                    </tbody>
                </table>

                <p class="mb-1 text-xs font-medium text-fg-light-grey">Costs</p>
                <table class="w-full text-sm">
                    <tbody>
                        <tr v-for="row in costRows" :key="row.name" class="border-t border-fg-pale-grey">
                            <td class="py-1">{{ row.name }}</td>
                            <td class="py-1"><CategorySparkline :entries="row.entries" /></td>
                            <td class="py-1 text-right font-mono text-xs text-fg-light-grey">budget {{ money(row.budget) }}</td>
                            <td class="py-1 text-right font-mono text-xs">actual {{ money(row.actual) }}</td>
                            <td
                                class="py-1 pl-2 text-right font-mono text-xs font-medium"
                                :class="isFavourable(row.name, row.variance) ? 'text-fg-positive-dark' : 'text-fg-danger-dark'"
                            >
                                {{ row.variance >= 0 ? '+' : '' }}{{ money(row.variance) }}
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</template>
