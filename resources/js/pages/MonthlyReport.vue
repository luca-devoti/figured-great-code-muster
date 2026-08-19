<script setup>
import axios from 'axios';
import { Chart, registerables } from 'chart.js';
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import { money, monthName } from '../format';

Chart.register(...registerables);

// Brand colours, matched to the CSS custom properties in resources/css/app.css
// so the chart reads as part of the same app, not a bolted-on library default.
const CHART_COLORS = {
    budget: '#cfd8dc', // --color-fg-muted-grey
    favourable: '#13c366', // --color-fg-positive
    unfavourable: '#f7ba2a', // --color-fg-warning
    grid: '#eceff1', // --color-fg-pale-grey
    text: '#607d8b', // --color-fg-mid-grey
};

const farms = ref([]);
const selectedFarmId = ref(null);
const lines = ref([]);
const commentaries = ref([]);
const weather = ref([]);
const months = ref([]);
const selectedMonth = ref(null);
const commentaryDraft = ref('');
const saving = ref(false);
const savedAt = ref(null);
const loading = ref(true);

// AI commentary draft generation.
const aiInstructions = ref('');
const generating = ref(false);
const generateError = ref('');
// True whenever the textarea holds an AI draft the adviser hasn't reviewed yet (by
// editing it or saving it) - so an unread AI paragraph can never look identical to the
// adviser's own reviewed words before it's copied into a client email.
const aiDraftUnreviewed = ref(false);

// Open items elsewhere in the practice for the selected farm (unanswered emails,
// invoices not yet keyed in) - so the adviser can see there's an unresolved question or
// disputed invoice before finalising a report that numbers depend on it.
const emails = ref([]);
const invoices = ref([]);

// This month vs season-to-date view.
const viewMode = ref('month');

// Season trend chart: which category is plotted. Stays on auto-pick (the category that has
// moved furthest from budget) as the adviser changes month, so the chart keeps surfacing
// whatever's most worth a second look - until the adviser picks a category themselves, at
// which point their choice sticks until they switch farms.
const chartCategory = ref(null);
const chartCategoryTouched = ref(false);
const chartCanvas = ref(null);
let chartInstance = null;

function selectChartCategory(category) {
    chartCategoryTouched.value = true;
    chartCategory.value = category;
}

onMounted(async () => {
    const { data } = await axios.get('/api/farms');
    farms.value = data;
    selectedFarmId.value = data[0]?.id ?? null;

    // Fetched once - these lists are small and don't depend on the selected farm,
    // just which of their rows get matched to it below.
    const [emailsRes, invoicesRes] = await Promise.all([axios.get('/api/emails'), axios.get('/api/invoices')]);
    emails.value = emailsRes.data;
    invoices.value = invoicesRes.data.invoices;
});

watch(selectedFarmId, loadReport);
watch(selectedMonth, loadCommentaryDraft);

async function loadReport() {
    loading.value = true;
    const { data } = await axios.get(`/api/farms/${selectedFarmId.value}/report`);
    lines.value = data.lines;
    commentaries.value = data.commentaries;
    weather.value = data.weather;
    months.value = [...new Set(data.lines.map((l) => l.month))];
    // Keep the selected month if the new farm has it; otherwise start at the first.
    if (!months.value.includes(selectedMonth.value)) {
        selectedMonth.value = months.value[0];
    } else {
        loadCommentaryDraft();
    }
    loading.value = false;
}

function loadCommentaryDraft() {
    const existing = commentaries.value.find((c) => c.month === selectedMonth.value);
    commentaryDraft.value = existing?.body ?? '';
    savedAt.value = null;
    aiDraftUnreviewed.value = false;
}

// Any deliberate action on the draft - editing it or saving it - counts as the adviser
// having read it, so the "unreviewed" flag only ever covers the moment right after
// generation.
function touchCommentaryDraft() {
    aiDraftUnreviewed.value = false;
}

const monthLines = computed(() => lines.value.filter((l) => l.month === selectedMonth.value));

const monthWeather = computed(() => weather.value.find((w) => w.month === selectedMonth.value));

function variance(line) {
    return line.actual - line.budget;
}

function variancePercent(line) {
    if (line.budget === 0) return null;
    return ((line.actual - line.budget) / line.budget) * 100;
}

// Income/sales categories are favourable when actual > budget; expense categories are the
// opposite. Flat "over budget = warning" styling would flag good news (e.g. strong lamb
// sales) the same way as bad news (e.g. a feed bill blowout), which misleads the farmer
// reading the report.
function isIncomeCategory(category) {
    return /income|sales/i.test(category);
}

function isFavourable(line) {
    return isIncomeCategory(line.category) ? line.actual >= line.budget : line.actual <= line.budget;
}

function varianceClass(line) {
    if (!line.budget || Math.abs(variance(line)) <= line.budget * 0.1) return '';
    return isFavourable(line) ? 'font-semibold text-fg-positive-dark' : 'font-semibold text-fg-warning-text';
}

// All months of the season up to and including the selected one, in chronological order.
const seasonMonthsToDate = computed(() => {
    const idx = months.value.indexOf(selectedMonth.value);
    return idx === -1 ? months.value : months.value.slice(0, idx + 1);
});

// Season-to-date totals per category. Bruce (Windrow Cropping) explicitly asked for this in
// his emails about the bank meeting — a single bad month (e.g. January's flooded wheat
// harvest) can look very different once the months that followed are added back in.
const seasonLines = computed(() => {
    const inSeason = new Set(seasonMonthsToDate.value);
    const totals = new Map();
    for (const line of lines.value) {
        if (!inSeason.has(line.month)) continue;
        const running = totals.get(line.category) ?? { category: line.category, budget: 0, actual: 0 };
        running.budget += line.budget;
        running.actual += line.actual;
        totals.set(line.category, running);
    }
    return [...totals.values()];
});

const displayLines = computed(() => (viewMode.value === 'season' ? seasonLines.value : monthLines.value));

const periodLabel = computed(() => {
    if (viewMode.value !== 'season' || !seasonMonthsToDate.value.length) return '';
    return `${monthName(seasonMonthsToDate.value[0])} – ${monthName(selectedMonth.value)}`;
});

// Regional weather worth mentioning in a season narrative (droughts, floods, adverse
// events) beyond the selected month's own conditions.
function isNotableWeather(w) {
    return w.rainfall_pct_normal < 60 || w.rainfall_pct_normal > 150 || w.soil_moisture_deficit_mm > 100;
}

const seasonWeatherHighlights = computed(() => {
    const inSeason = new Set(seasonMonthsToDate.value);
    return weather.value
        .filter((w) => inSeason.has(w.month) && w.month !== selectedMonth.value && isNotableWeather(w))
        .map((w) => `${monthName(w.month)}: ${w.notes}`);
});

// Which months already have commentary, so the adviser can see the backlog at a glance
// instead of clicking through all 12 months per farm to find out what's left.
const monthsWithCommentary = computed(() => new Set(commentaries.value.map((c) => c.month)));

// Unanswered emails from the farmer, and invoices not yet keyed in, matched to the
// selected farm. Neither table stores a farm_id - emails are matched by sender name
// against the farm's owner (an email from someone else, e.g. a supplier, is a
// practice-wide concern rather than this farm's, so it's deliberately left out here),
// invoices by the account name printed on the scanned text.
const openEmailsForFarm = computed(() => {
    const farm = farms.value.find((f) => f.id === selectedFarmId.value);
    if (!farm) return [];
    return emails.value.filter((e) => e.from_name === farm.owner_name && !e.replied_at);
});

const openInvoicesForFarm = computed(() => {
    const farm = farms.value.find((f) => f.id === selectedFarmId.value);
    if (!farm) return [];
    return invoices.value.filter((i) => i.raw_text.toUpperCase().includes(farm.name.toUpperCase()) && !i.entered_at);
});

// Budgeted vs actual profit for whatever period is on screen (this month, or season to
// date) - the single number a farmer actually wants to know, ahead of the category detail.
const periodProfit = computed(() => {
    let budget = 0;
    let actual = 0;
    for (const line of displayLines.value) {
        const sign = isIncomeCategory(line.category) ? 1 : -1;
        budget += sign * line.budget;
        actual += sign * line.actual;
    }
    return { budget, actual, difference: actual - budget };
});

// Pick the category worth charting by default: whichever has moved furthest from budget
// for the season shown, so the trend chart opens already pointed at the thing worth
// discussing (e.g. Wheat Income for Windrow Cropping after a delayed harvest) instead of an
// alphabetically-first category nobody asked about. Re-picks as the adviser moves through
// months (a shortfall that gets resolved should stop being the headline), but leaves the
// adviser's own choice alone once they've made one.
watch(selectedFarmId, () => {
    chartCategoryTouched.value = false;
});

watch(seasonLines, (current) => {
    if (!current.length) {
        chartCategory.value = null;
        return;
    }
    if (chartCategoryTouched.value && current.some((l) => l.category === chartCategory.value)) return;
    chartCategory.value = [...current].sort((a, b) => Math.abs(b.actual - b.budget) - Math.abs(a.actual - a.budget))[0].category;
});

// The charted category's budget vs actual for every month of the season so far.
const chartMonthData = computed(() => {
    if (!chartCategory.value) return [];
    return seasonMonthsToDate.value.map((month) => {
        const line = lines.value.find((l) => l.month === month && l.category === chartCategory.value);
        return { month, budget: line?.budget ?? 0, actual: line?.actual ?? 0 };
    });
});

function renderChart() {
    if (!chartCanvas.value) return;
    if (chartInstance) {
        chartInstance.destroy();
        chartInstance = null;
    }
    if (!chartMonthData.value.length) return;

    const isIncome = isIncomeCategory(chartCategory.value);
    const actualColors = chartMonthData.value.map((d) => {
        if (!d.budget) return CHART_COLORS.text;
        const favourable = isIncome ? d.actual >= d.budget : d.actual <= d.budget;
        return favourable ? CHART_COLORS.favourable : CHART_COLORS.unfavourable;
    });

    chartInstance = new Chart(chartCanvas.value, {
        type: 'bar',
        data: {
            labels: chartMonthData.value.map((d) => monthName(d.month)),
            datasets: [
                {
                    label: 'Budget',
                    data: chartMonthData.value.map((d) => d.budget),
                    backgroundColor: CHART_COLORS.budget,
                    borderRadius: 3,
                },
                {
                    label: 'Actual',
                    data: chartMonthData.value.map((d) => d.actual),
                    backgroundColor: actualColors,
                    borderRadius: 3,
                },
            ],
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: { position: 'bottom', labels: { boxWidth: 12, font: { size: 11 } } },
                tooltip: { callbacks: { label: (ctx) => `${ctx.dataset.label}: ${money(ctx.raw)}` } },
            },
            scales: {
                x: { grid: { display: false }, ticks: { color: CHART_COLORS.text, font: { size: 11 } } },
                y: {
                    grid: { color: CHART_COLORS.grid },
                    ticks: {
                        color: CHART_COLORS.text,
                        font: { size: 11 },
                        callback: (v) => (Math.abs(v) >= 1000 ? `${v / 1000}k` : v),
                    },
                },
            },
        },
    });
}

watch(chartMonthData, async () => {
    await nextTick();
    renderChart();
});

onBeforeUnmount(() => {
    if (chartInstance) chartInstance.destroy();
});

async function saveCommentary() {
    saving.value = true;
    const { data } = await axios.put(`/api/farms/${selectedFarmId.value}/commentary`, {
        month: selectedMonth.value,
        body: commentaryDraft.value,
    });
    // Keep the local cache in sync so switching months and back shows the save.
    const index = commentaries.value.findIndex((c) => c.month === selectedMonth.value);
    if (index >= 0) commentaries.value[index] = data;
    else commentaries.value.push(data);
    saving.value = false;
    savedAt.value = new Date();
    aiDraftUnreviewed.value = false;
}

async function generateCommentary() {
    const farm = farms.value.find((f) => f.id === selectedFarmId.value);

    const lineBlock = (lineSet) =>
        lineSet
            .map(
                (l) =>
                    `- ${l.category} (${isIncomeCategory(l.category) ? 'income' : 'expense'}): budget ${money(l.budget)}, actual ${money(l.actual)}, variance ${money(l.actual - l.budget)}`,
            )
            .join('\n');

    const weatherBlock = monthWeather.value
        ? `Rainfall: ${monthWeather.value.rainfall_mm}mm (${monthWeather.value.rainfall_pct_normal}% of normal)
Mean temperature: ${monthWeather.value.mean_temp_c}°C
Soil moisture deficit: ${monthWeather.value.soil_moisture_deficit_mm}mm
Notes: ${monthWeather.value.notes}`
        : 'No weather data available for this month.';

    const highlightsBlock = seasonWeatherHighlights.value.length
        ? `\n\nOther notable regional weather so far this season:\n${seasonWeatherHighlights.value.join('\n')}`
        : '';

    const prompt = `Farm: ${farm.name} (${farm.type}), owner ${farm.owner_name}
Month: ${monthName(selectedMonth.value)}

This month's budget vs actual:
${lineBlock(monthLines.value)}

Season-to-date (${monthName(seasonMonthsToDate.value[0])} – ${monthName(selectedMonth.value)}):
${lineBlock(seasonLines.value)}

Regional weather conditions this month:
${weatherBlock}${highlightsBlock}
${aiInstructions.value.trim() ? `\nThe adviser wants this commentary to focus on: ${aiInstructions.value.trim()}` : ''}

Write the monthly commentary for this farm and month.`;

    generating.value = true;
    generateError.value = '';
    try {
        const { data } = await axios.post('/api/ai', {
            system: "You are a rural accounting adviser at Southdown Rural Accountants (Waikato, NZ) writing monthly commentary for a farm client. Write 2-4 short paragraphs in plain English: summarise what happened financially this month, call out notable budget variances and a plausible reason for them (referencing weather where relevant), and flag anything worth following up with the farmer. For income/sales categories, actual above budget is favourable; for expense categories, actual above budget is unfavourable - frame variances by what they mean for the farmer, not just their size. Where the season-to-date figures tell a different story than the single month on its own (e.g. a shortfall that was later recovered, or a gain that's still behind for the season), say so explicitly - farmers care about the season trend more than one month in isolation. Only use the figures given to you - never invent numbers. Do not use markdown formatting.",
            prompt,
        });
        commentaryDraft.value = data.text.trim();
        aiDraftUnreviewed.value = true;
    } catch (e) {
        generateError.value = e.response?.data?.error ?? e.message;
    } finally {
        generating.value = false;
    }
}
</script>

<template>
    <div>
        <div class="mb-4 flex items-end justify-between">
            <div>
                <h2 class="text-lg font-semibold">Monthly report</h2>
                <p class="text-sm text-fg-mid-grey">
                    Budget vs actual by month. Write commentary for each farm and month — June 2025 for Riverbend is
                    done as an example.
                </p>
            </div>
            <p
                v-if="!loading"
                class="rounded-full px-2.5 py-1 text-xs font-medium whitespace-nowrap"
                :class="
                    monthsWithCommentary.size === months.length
                        ? 'bg-fg-positive-15 text-fg-positive-dark'
                        : 'bg-fg-warning-15 text-fg-warning-text'
                "
            >
                {{ monthsWithCommentary.size }} / {{ months.length }} months have commentary
            </p>
        </div>

        <div class="mb-4 flex flex-wrap gap-3">
            <select v-model="selectedFarmId" class="rounded border border-fg-muted-grey bg-white px-3 py-1.5 text-sm">
                <option v-for="farm in farms" :key="farm.id" :value="farm.id">
                    {{ farm.name }} ({{ farm.type }} — {{ farm.owner_name }})
                </option>
            </select>
            <select v-model="selectedMonth" class="rounded border border-fg-muted-grey bg-white px-3 py-1.5 text-sm">
                <option v-for="month in months" :key="month" :value="month">
                    {{ monthsWithCommentary.has(month) ? '✓ ' : '' }}{{ monthName(month) }}
                </option>
            </select>
        </div>

        <p v-if="loading" class="text-fg-light-grey">Loading…</p>

        <!-- Unresolved items elsewhere in the practice for this farm - a report is only as
             good as the numbers behind it, and those numbers can still be in dispute on
             another screen. -->
        <div
            v-if="!loading && (openEmailsForFarm.length || openInvoicesForFarm.length)"
            class="mb-4 rounded border border-fg-warning-15 bg-fg-warning-15 p-3"
        >
            <p class="text-sm font-medium text-fg-warning-text">
                ⚠ Open items for this client — review before finalising this report
            </p>
            <ul class="mt-1 space-y-0.5 text-xs text-fg-dark-grey">
                <li v-for="e in openEmailsForFarm" :key="`email-${e.id}`">
                    Unanswered email — "{{ e.subject }}" (Inbox)
                </li>
                <li v-for="i in openInvoicesForFarm" :key="`invoice-${i.id}`">
                    Invoice not yet entered — {{ i.filename }} (Invoice entry)
                </li>
            </ul>
        </div>

        <div v-if="!loading" class="grid grid-cols-1 gap-4 lg:grid-cols-3">
            <div class="lg:col-span-2">
                <!-- The one number a farmer actually wants: did we make more or less than planned. -->
                <div class="mb-3 grid grid-cols-3 gap-2 text-center">
                    <div class="rounded border border-fg-muted-grey bg-white p-2.5">
                        <p class="text-xs text-fg-light-grey">Budgeted profit</p>
                        <p class="text-base font-semibold">{{ money(periodProfit.budget) }}</p>
                    </div>
                    <div class="rounded border border-fg-muted-grey bg-white p-2.5">
                        <p class="text-xs text-fg-light-grey">Actual profit</p>
                        <p class="text-base font-semibold">{{ money(periodProfit.actual) }}</p>
                    </div>
                    <div class="rounded p-2.5" :class="periodProfit.difference >= 0 ? 'bg-fg-positive-9' : 'bg-fg-danger-9'">
                        <p class="text-xs text-fg-light-grey">Difference</p>
                        <p
                            class="text-base font-semibold"
                            :class="periodProfit.difference >= 0 ? 'text-fg-positive-dark' : 'text-fg-danger-dark'"
                        >
                            {{ periodProfit.difference >= 0 ? '+' : '' }}{{ money(periodProfit.difference) }}
                        </p>
                    </div>
                </div>

                <div class="mb-2 flex flex-wrap items-center justify-between gap-2">
                    <div class="inline-flex overflow-hidden rounded border border-fg-muted-grey text-xs">
                        <button
                            class="px-2.5 py-1"
                            :class="viewMode === 'month' ? 'bg-fg-main-blue text-white' : 'bg-white text-fg-mid-grey hover:bg-fg-pale-grey'"
                            @click="viewMode = 'month'"
                        >
                            This month
                        </button>
                        <button
                            class="border-l border-fg-muted-grey px-2.5 py-1"
                            :class="viewMode === 'season' ? 'bg-fg-main-blue text-white' : 'bg-white text-fg-mid-grey hover:bg-fg-pale-grey'"
                            @click="viewMode = 'season'"
                        >
                            Season to date
                        </button>
                    </div>
                    <p v-if="periodLabel" class="text-xs text-fg-light-grey">{{ periodLabel }}</p>
                </div>

                <div class="overflow-x-auto rounded border border-fg-muted-grey bg-white">
                    <table class="w-full text-sm">
                        <thead class="bg-fg-super-pale-grey text-left text-fg-mid-grey">
                            <tr>
                                <th class="px-3 py-2 font-medium">Category</th>
                                <th class="px-3 py-2 text-right font-medium">Budget</th>
                                <th class="px-3 py-2 text-right font-medium">Actual</th>
                                <th class="px-3 py-2 text-right font-medium">Variance</th>
                                <th class="px-3 py-2 text-right font-medium">%</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr
                                v-for="line in displayLines"
                                :key="line.id ?? line.category"
                                class="border-t border-fg-pale-grey hover:bg-fg-pale-grey"
                            >
                                <td class="px-3 py-1.5">{{ line.category }}</td>
                                <td class="px-3 py-1.5 text-right font-mono text-xs">{{ money(line.budget) }}</td>
                                <td class="px-3 py-1.5 text-right font-mono text-xs">{{ money(line.actual) }}</td>
                                <td class="px-3 py-1.5 text-right font-mono text-xs" :class="varianceClass(line)">
                                    {{ money(variance(line)) }}
                                </td>
                                <td class="px-3 py-1.5 text-right font-mono text-xs text-fg-light-grey">
                                    {{ variancePercent(line) === null ? '—' : variancePercent(line).toFixed(1) + '%' }}
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>

                <!-- The season story for one category, visually - the table above answers "what",
                     this answers "is it actually a problem once the rest of the season is in". -->
                <div class="mt-4 rounded border border-fg-muted-grey bg-white p-4">
                    <div class="mb-3 flex flex-wrap items-center justify-between gap-2">
                        <h3 class="text-sm font-semibold">Season trend</h3>
                        <select
                            :value="chartCategory"
                            class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-xs"
                            @change="selectChartCategory($event.target.value)"
                        >
                            <option v-for="line in seasonLines" :key="line.category" :value="line.category">
                                {{ line.category }}
                            </option>
                        </select>
                    </div>
                    <p class="mb-2 text-xs text-fg-light-grey">
                        Budget vs actual, {{ monthName(seasonMonthsToDate[0]) }} – {{ monthName(selectedMonth) }}. Bars are
                        coloured green where {{ chartCategory }} helped the result and orange where it hurt it.
                    </p>
                    <div class="h-56">
                        <canvas ref="chartCanvas"></canvas>
                    </div>
                </div>
            </div>

            <div class="space-y-4">
            <div class="rounded border border-fg-muted-grey bg-white p-4">
                <label class="mb-1 block text-sm font-medium">Commentary — {{ monthName(selectedMonth) }}</label>
                <p class="mb-2 text-xs text-fg-light-grey">
                    What happened this month, and does anything need following up with the farmer?
                </p>

                <div class="mb-2 flex gap-2">
                    <input
                        v-model="aiInstructions"
                        type="text"
                        placeholder="Optional: what should this focus on? (e.g. mention the storm losses)"
                        class="flex-1 rounded border border-fg-muted-grey px-2 py-1.5 text-sm"
                        @keyup.enter="generateCommentary"
                    />
                    <button
                        class="shrink-0 rounded border border-fg-main-blue px-3 py-1.5 text-sm font-medium text-fg-main-blue hover:bg-fg-main-blue-9 disabled:opacity-50"
                        :disabled="generating || !monthLines.length"
                        @click="generateCommentary"
                    >
                        {{ generating ? 'Generating…' : '✨ Generate with AI' }}
                    </button>
                </div>
                <p v-if="generateError" class="mb-2 rounded bg-fg-danger-9 p-2 text-xs text-fg-danger-dark">{{ generateError }}</p>
                <p v-if="aiDraftUnreviewed" class="mb-2 rounded bg-fg-warning-15 p-2 text-xs font-medium text-fg-warning-text">
                    ✨ AI draft — read it over before saving or sharing with the farmer
                </p>

                <textarea
                    v-model="commentaryDraft"
                    rows="12"
                    class="w-full rounded border border-fg-muted-grey p-2 text-sm"
                    placeholder="Write the month's commentary…"
                    @input="touchCommentaryDraft"
                ></textarea>
                <div class="mt-2 flex items-center gap-3">
                    <button
                        class="rounded bg-fg-main-blue px-4 py-1.5 text-sm font-medium text-white hover:bg-fg-main-blue-hover disabled:opacity-50"
                        :disabled="saving || !commentaryDraft.trim()"
                        @click="saveCommentary"
                    >
                        {{ saving ? 'Saving…' : 'Save commentary' }}
                    </button>
                    <span v-if="savedAt" class="text-xs text-fg-positive-dark">Saved ✓</span>
                </div>
            </div>

            <!-- Regional climate summary for the selected month -->
            <div v-if="monthWeather" class="rounded border border-fg-muted-grey bg-white p-4">
                <h3 class="text-sm font-semibold">Regional conditions — {{ monthName(selectedMonth) }}</h3>
                <dl class="mt-2 grid grid-cols-3 gap-2 text-center">
                    <div class="rounded bg-fg-super-pale-grey p-2">
                        <dt class="text-xs text-fg-light-grey">Rainfall</dt>
                        <dd class="text-sm font-medium">
                            {{ monthWeather.rainfall_mm }} mm
                            <span class="block text-xs font-normal text-fg-light-grey">
                                {{ monthWeather.rainfall_pct_normal }}% of normal
                            </span>
                        </dd>
                    </div>
                    <div class="rounded bg-fg-super-pale-grey p-2">
                        <dt class="text-xs text-fg-light-grey">Mean temp</dt>
                        <dd class="text-sm font-medium">{{ monthWeather.mean_temp_c }}°C</dd>
                    </div>
                    <div class="rounded bg-fg-super-pale-grey p-2">
                        <dt class="text-xs text-fg-light-grey">Soil moisture deficit</dt>
                        <dd class="text-sm font-medium">{{ monthWeather.soil_moisture_deficit_mm }} mm</dd>
                    </div>
                </dl>
                <p class="mt-2 text-xs leading-relaxed text-fg-mid-grey">{{ monthWeather.notes }}</p>
            </div>
            </div>
        </div>
    </div>
</template>
