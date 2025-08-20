<script setup lang="ts">
import { Head, Link } from '@inertiajs/vue3';
import { onMounted, ref } from 'vue';

// Reactive data
const reportData = ref<any>(null);
const loading = ref(true);
const error = ref<string | null>(null);

// AI Commentary state
const aiPrompt = ref('');
const aiResponse = ref('');
const aiLoading = ref(false);
const aiError = ref<string | null>(null);

// Fetch financial report data
const fetchFinancialData = async () => {
    try {
        loading.value = true;
        const response = await fetch('/api/financial-report');
        if (!response.ok) {
            throw new Error('Failed to fetch financial data');
        }
        const data = await response.json();
        reportData.value = data;
    } catch (err) {
        error.value = err instanceof Error ? err.message : 'An error occurred';
    } finally {
        loading.value = false;
    }
};

// Generate AI commentary
const generateCommentary = async () => {
    if (!aiPrompt.value.trim()) return;

    try {
        aiLoading.value = true;
        aiError.value = null;

        // Prepare financial data summary for AI analysis
        const financialSummary = prepareFinancialSummary();

        // Combine user prompt with financial data
        const fullPrompt = `Based on the following financial data for ${reportData.value?.company?.name}, please ${aiPrompt.value}

FINANCIAL DATA SUMMARY:
${financialSummary}

Please provide insights based on this actual financial data.`;

        const response = await fetch('/api/generate-commentary', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || '',
            },
            body: JSON.stringify({
                prompt: fullPrompt,
            }),
        });

        if (!response.ok) {
            throw new Error('Failed to generate commentary');
        }

        const data = await response.json();
        aiResponse.value = data.response || 'No response received';
    } catch (err) {
        aiError.value = err instanceof Error ? err.message : 'An error occurred';
    } finally {
        aiLoading.value = false;
    }
};

// Prepare financial data summary for AI analysis
const prepareFinancialSummary = () => {
    if (!reportData.value) return 'No financial data available';

    const { company, sections, summary } = reportData.value;

    let summaryText = `Company: ${company.name}\n`;
    summaryText += `Report Type: ${company.report_type}\n`;
    summaryText += `Period: ${company.period}\n\n`;

    // Add key financial metrics
    const netProfit = summary.find((s: any) => s.name === 'Net Profit');
    const operatingSurplus = summary.find((s: any) => s.name === 'Operating Surplus');

    if (netProfit) {
        const totalNetProfit = netProfit.values[netProfit.values.length - 1];
        summaryText += `Total Net Profit: ${formatCurrency(totalNetProfit)}\n`;
    }

    if (operatingSurplus) {
        const totalOperatingSurplus = operatingSurplus.values[operatingSurplus.values.length - 1];
        summaryText += `Total Operating Surplus: ${formatCurrency(totalOperatingSurplus)}\n\n`;
    }

    // Add section totals
    sections.forEach((section: any) => {
        if (section.total) {
            const totalValue = section.total.values[section.total.values.length - 1];
            summaryText += `${section.total.name}: ${formatCurrency(totalValue)}\n`;
        }
    });

    // Add key line items
    summaryText += '\nKey Revenue Sources:\n';
    sections.forEach((section: any) => {
        if (section.id === 'income') {
            section.subsections?.forEach((subsection: any) => {
                subsection.subsections?.forEach((subsubsection: any) => {
                    if (subsubsection.id === 'sheep_income') {
                        subsubsection.line_items?.forEach((item: any) => {
                            const totalValue = item.values[item.values.length - 1];
                            summaryText += `- ${item.name}: ${formatCurrency(totalValue)}\n`;
                        });
                    }
                });
            });
        }
    });

    summaryText += '\nKey Expenses:\n';
    sections.forEach((section: any) => {
        if (section.id === 'operating_expenses') {
            section.subsections?.forEach((subsection: any) => {
                subsection.line_items?.forEach((item: any) => {
                    const totalValue = item.values[item.values.length - 1];
                    if (totalValue > 0) {
                        summaryText += `- ${item.name}: ${formatCurrency(totalValue)}\n`;
                    }
                });
            });
        }
    });

    return summaryText;
};

// Format currency values
const formatCurrency = (value: number) => {
    if (value === 0) return '-';
    return new Intl.NumberFormat('en-NZ', {
        style: 'currency',
        currency: 'NZD',
        minimumFractionDigits: 0,
        maximumFractionDigits: 0,
    }).format(value);
};

onMounted(() => {
    fetchFinancialData();
});
</script>

<template>
    <Head title="Financial Report Challenge" />

    <div class="min-h-screen bg-gray-50 px-4 py-8">
        <div class="mx-auto max-w-7xl">
            <!-- Header -->
            <div class="mb-8">
                <Link href="/" class="mb-4 inline-flex items-center text-sm text-gray-600 hover:text-gray-900"> ← Back to Home </Link>

                <h1 class="mb-2 text-3xl font-bold text-gray-900">🌾 Figured Financial Report Challenge</h1>
                <p class="text-gray-600">Transform this basic data display into a beautiful, interactive financial report!</p>
            </div>

            <!-- Loading State -->
            <div v-if="loading" class="flex items-center justify-center py-12">
                <div class="h-12 w-12 animate-spin rounded-full border-b-2 border-blue-600"></div>
                <span class="ml-4 text-gray-600">Loading financial data...</span>
            </div>

            <!-- Error State -->
            <div v-else-if="error" class="rounded-lg border border-red-200 bg-red-50 p-6">
                <h2 class="mb-2 font-semibold text-red-800">Error Loading Data</h2>
                <p class="text-red-600">{{ error }}</p>
                <button @click="fetchFinancialData" class="mt-4 rounded bg-red-600 px-4 py-2 text-white hover:bg-red-700">Try Again</button>
            </div>

            <!-- Profit & Loss Report -->
            <div v-else-if="reportData" class="space-y-8">
                <!-- Company Header -->
                <div class="rounded-lg border border-gray-200 bg-white p-8 shadow-lg">
                    <div class="mb-6 border-b border-gray-200 pb-6 text-center">
                        <h2 class="mb-2 text-3xl font-bold text-gray-900">{{ reportData.company.name }}</h2>
                        <h3 class="mb-1 text-xl font-semibold text-blue-600">{{ reportData.company.report_type }}</h3>
                        <p class="text-sm text-gray-600">{{ reportData.company.basis }}</p>
                        <p class="text-sm text-gray-600">{{ reportData.company.period }}</p>
                        <p class="mt-1 text-xs text-gray-500">Actuals to {{ reportData.company.actuals_to }}</p>
                    </div>
                </div>

                <!-- Financial Report Table -->
                <div class="overflow-hidden rounded-lg border border-gray-200 bg-white shadow-lg">
                    <div class="p-6">
                        <h3 class="mb-4 text-lg font-semibold text-gray-900">Financial Performance</h3>

                        <!-- Table Container with horizontal scroll -->
                        <div class="overflow-x-auto">
                            <table class="w-full text-sm">
                                <!-- Table Header -->
                                <thead>
                                    <tr class="border-b-2 border-gray-300">
                                        <th class="sticky left-0 z-10 min-w-[200px] bg-white px-4 py-3 text-left font-semibold text-gray-700">
                                            Account
                                        </th>
                                        <th
                                            v-for="column in reportData.columns"
                                            :key="column.month"
                                            class="min-w-[100px] px-3 py-3 text-right font-semibold text-gray-700"
                                            :class="{
                                                'bg-blue-50 text-blue-800': column.type === 'Total',
                                                'text-green-700': column.type === 'Actual',
                                            }"
                                        >
                                            {{ column.month }}
                                            <div class="text-xs font-normal text-gray-500">{{ column.type }}</div>
                                        </th>
                                    </tr>
                                </thead>

                                <tbody>
                                    <!-- Income Section -->
                                    <template v-for="section in reportData.sections" :key="section.id">
                                        <!-- Main Section Header -->
                                        <tr class="border-b border-gray-200 bg-gray-50">
                                            <td class="sticky left-0 z-10 bg-gray-50 px-4 py-3 font-bold text-gray-900">
                                                {{ section.name }}
                                            </td>
                                            <td v-for="column in reportData.columns" :key="column.month" class="px-3 py-3"></td>
                                        </tr>

                                        <!-- Subsections -->
                                        <template v-for="subsection in section.subsections" :key="subsection.id">
                                            <!-- Subsection Header -->
                                            <tr class="border-b border-gray-100">
                                                <td class="sticky left-0 z-10 bg-white px-4 py-2 pl-8 font-semibold text-gray-800">
                                                    {{ subsection.name }}
                                                </td>
                                                <td v-for="column in reportData.columns" :key="column.month" class="px-3 py-2"></td>
                                            </tr>

                                            <!-- Sub-subsections -->
                                            <template v-if="subsection.subsections">
                                                <template v-for="subsubsection in subsection.subsections" :key="subsubsection.id">
                                                    <!-- Sub-subsection Header -->
                                                    <tr class="border-b border-gray-100">
                                                        <td class="sticky left-0 z-10 bg-white px-4 py-2 pl-12 text-sm font-medium text-gray-700">
                                                            {{ subsubsection.name }}
                                                        </td>
                                                        <td v-for="column in reportData.columns" :key="column.month" class="px-3 py-2"></td>
                                                    </tr>

                                                    <!-- Line Items -->
                                                    <tr
                                                        v-for="item in subsubsection.line_items"
                                                        :key="item.account_id"
                                                        class="border-b border-gray-50 hover:bg-gray-50"
                                                    >
                                                        <td
                                                            class="sticky left-0 z-10 bg-white px-4 py-2 pl-16 text-sm text-gray-600 hover:bg-gray-50"
                                                        >
                                                            {{ item.name }}
                                                        </td>
                                                        <td
                                                            v-for="(value, index) in item.values"
                                                            :key="index"
                                                            class="px-3 py-2 text-right text-sm"
                                                            :class="{
                                                                'bg-blue-50 font-semibold': index === item.values.length - 1,
                                                                'text-green-600': value > 0,
                                                                'text-red-600': value < 0,
                                                            }"
                                                        >
                                                            {{ formatCurrency(value) }}
                                                        </td>
                                                    </tr>
                                                </template>
                                            </template>

                                            <!-- Direct line items (for subsections without sub-subsections) -->
                                            <tr
                                                v-for="item in subsection.line_items || []"
                                                :key="item.account_id"
                                                class="border-b border-gray-50 hover:bg-gray-50"
                                            >
                                                <td class="sticky left-0 z-10 bg-white px-4 py-2 pl-12 text-sm text-gray-600 hover:bg-gray-50">
                                                    {{ item.name }}
                                                </td>
                                                <td
                                                    v-for="(value, index) in item.values"
                                                    :key="index"
                                                    class="px-3 py-2 text-right text-sm"
                                                    :class="{
                                                        'bg-blue-50 font-semibold': index === item.values.length - 1,
                                                        'text-green-600': value > 0,
                                                        'text-red-600': value < 0,
                                                    }"
                                                >
                                                    {{ formatCurrency(value) }}
                                                </td>
                                            </tr>

                                            <!-- Gross Profit for subsections that have it -->
                                            <tr v-if="subsection.gross_profit" class="border-b-2 border-blue-200 bg-blue-50 font-semibold">
                                                <td class="sticky left-0 z-10 bg-blue-50 px-4 py-3 pl-8 text-blue-800">
                                                    {{ subsection.gross_profit.name }}
                                                </td>
                                                <td
                                                    v-for="(value, index) in subsection.gross_profit.values"
                                                    :key="index"
                                                    class="px-3 py-3 text-right font-semibold text-blue-800"
                                                    :class="{
                                                        'bg-blue-100': index === subsection.gross_profit.values.length - 1,
                                                    }"
                                                >
                                                    {{ formatCurrency(value) }}
                                                </td>
                                            </tr>
                                        </template>

                                        <!-- Section Total -->
                                        <tr v-if="section.total" class="border-b-2 border-gray-300 bg-gray-100 font-bold">
                                            <td class="sticky left-0 z-10 bg-gray-100 px-4 py-4 text-gray-900">
                                                {{ section.total.name }}
                                            </td>
                                            <td
                                                v-for="(value, index) in section.total.values"
                                                :key="index"
                                                class="px-3 py-4 text-right font-bold"
                                                :class="{
                                                    'bg-gray-200': index === section.total.values.length - 1,
                                                    'text-green-700': value > 0,
                                                    'text-red-700': value < 0,
                                                    'text-gray-900': value === 0,
                                                }"
                                            >
                                                {{ formatCurrency(value) }}
                                            </td>
                                        </tr>
                                    </template>

                                    <!-- Summary Section -->
                                    <template v-for="summaryItem in reportData.summary" :key="summaryItem.name">
                                        <tr
                                            class="border-b-2 border-gray-400 text-lg font-bold"
                                            :class="{
                                                'bg-green-100': summaryItem.name === 'Net Profit',
                                                'bg-yellow-50': summaryItem.name === 'Operating Surplus',
                                            }"
                                        >
                                            <td
                                                class="sticky left-0 z-10 px-4 py-4"
                                                :class="{
                                                    'bg-green-100 text-green-800': summaryItem.name === 'Net Profit',
                                                    'bg-yellow-50 text-yellow-800': summaryItem.name === 'Operating Surplus',
                                                }"
                                            >
                                                {{ summaryItem.name }}
                                            </td>
                                            <td
                                                v-for="(value, index) in summaryItem.values"
                                                :key="index"
                                                class="px-3 py-4 text-right font-bold"
                                                :class="{
                                                    'bg-green-200': summaryItem.name === 'Net Profit' && index === summaryItem.values.length - 1,
                                                    'bg-yellow-100':
                                                        summaryItem.name === 'Operating Surplus' && index === summaryItem.values.length - 1,
                                                    'text-green-700': value > 0,
                                                    'text-red-700': value < 0,
                                                    'text-gray-900': value === 0,
                                                }"
                                            >
                                                {{ formatCurrency(value) }}
                                            </td>
                                        </tr>
                                    </template>
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>

                <!-- AI Commentary Demo -->
                <div class="rounded-lg border border-green-200 bg-green-50 p-6">
                    <h3 class="mb-3 text-lg font-semibold text-green-900">🤖 AI Financial Analysis (Prism Demo)</h3>
                    <p class="mb-4 text-sm text-green-700">
                        Ask AI to analyze the actual financial data from {{ reportData.company.name }}. The AI will review the complete P&L report and
                        provide insights.
                    </p>

                    <div class="space-y-4">
                        <div>
                            <label for="ai-prompt" class="mb-2 block text-sm font-medium text-green-800"> Ask AI about the financial data: </label>
                            <input
                                id="ai-prompt"
                                v-model="aiPrompt"
                                type="text"
                                placeholder="e.g., analyze the profitability trends, identify key revenue drivers, or assess expense management"
                                class="w-full rounded-lg border border-green-300 p-3 focus:border-transparent focus:ring-2 focus:ring-green-500"
                                :disabled="aiLoading"
                            />
                            <div class="mt-2 text-xs text-green-600">
                                <strong>Try these examples:</strong><br />
                                • "analyze the profitability trends and seasonal patterns"<br />
                                • "identify the main revenue drivers and suggest improvements"<br />
                                • "assess the expense management and cost structure"<br />
                                • "provide recommendations for improving financial performance"
                            </div>
                        </div>

                        <button
                            @click="generateCommentary"
                            :disabled="!aiPrompt.trim() || aiLoading"
                            class="rounded-lg bg-green-600 px-4 py-2 text-white hover:bg-green-700 disabled:cursor-not-allowed disabled:opacity-50"
                        >
                            {{ aiLoading ? 'Analyzing Financial Data...' : 'Generate Financial Analysis' }}
                        </button>

                        <div v-if="aiError" class="rounded-lg border border-red-200 bg-red-50 p-4">
                            <p class="text-sm text-red-800">{{ aiError }}</p>
                        </div>

                        <div v-if="aiResponse" class="rounded-lg border border-green-200 bg-white p-4">
                            <h4 class="mb-2 font-medium text-green-800">AI Financial Analysis:</h4>
                            <div class="prose prose-sm max-w-none text-gray-700">
                                <pre class="font-sans whitespace-pre-wrap">{{ aiResponse }}</pre>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Challenge Links -->
                <div class="border-t border-gray-200 py-8 text-center">
                    <div class="space-x-4">
                        <a
                            href="/CHALLENGE.md"
                            target="_blank"
                            class="inline-flex items-center rounded-lg bg-blue-600 px-6 py-3 text-white hover:bg-blue-700"
                        >
                            📋 Challenge Requirements
                        </a>
                        <a
                            href="/api/financial-report"
                            target="_blank"
                            class="inline-flex items-center rounded-lg bg-green-600 px-6 py-3 text-white hover:bg-green-700"
                        >
                            🔗 API Data
                        </a>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
