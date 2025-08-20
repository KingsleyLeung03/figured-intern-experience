<script setup lang="ts">
import { defineProps } from 'vue';

const props = defineProps<{ reportData: any }>();

function formatCurrency(value: number) {
  const isNegative = value < 0;
  const absValue = Math.abs(value);
  const formatted = absValue.toLocaleString('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  });
  return isNegative ? `<span class='text-red-600'>(${formatted})</span>` : formatted;
}

function renderSection(section: any, columns: any[], level = 1) {
  const rows: any[] = [];
  // Section header
  rows.push({
    type: 'section',
    name: section.name,
    level,
    values: section.total?.values || null,
    isTotal: !!section.total,
  });
  // Subsections
  if (section.subsections) {
    for (const sub of section.subsections) {
      rows.push(...renderSection(sub, columns, level + 1));
    }
  }
  // Line items
  if (section.line_items) {
    for (const item of section.line_items) {
      rows.push({
        type: 'line_item',
        name: item.name,
        level: level + 1,
        values: item.values,
      });
    }
  }
  // Gross profit (for sheep)
  if (section.gross_profit) {
    rows.push({
      type: 'gross_profit',
      name: section.gross_profit.name,
      level: level + 1,
      values: section.gross_profit.values,
      isTotal: true,
    });
  }
  return rows;
}

function getSummaryRows(summary: any[], columns: any[]) {
  return summary.map((row: any) => ({
    type: 'summary',
    name: row.name,
    level: 1,
    values: row.values,
    isTotal: true,
  }));
}
</script>

<template>
  <div v-if="reportData" class="bg-white rounded-lg shadow border border-gray-200 p-6 mt-8">
    <!-- Company Info -->
    <div class="mb-6">
      <h2 class="text-2xl font-bold text-gray-900">{{ reportData.company.name }}</h2>
      <div class="text-sm text-gray-600">{{ reportData.company.report_type }} | {{ reportData.company.period }}</div>
      <div class="text-xs text-gray-500">Basis: {{ reportData.company.basis }} | Actuals to: {{ reportData.company.actuals_to }}</div>
    </div>

    <!-- Hierarchical Table -->
    <div class="overflow-x-auto">
      <table class="min-w-full text-sm border">
        <thead>
          <tr class="bg-gray-100">
            <th class="px-4 py-2 text-left font-semibold text-gray-700" style="min-width: 200px;">Section</th>
            <th v-for="col in reportData.columns" :key="col.month" class="px-4 py-2 text-right font-semibold text-gray-700">{{ col.month }}</th>
          </tr>
        </thead>
        <tbody>
          <!-- Render all sections recursively -->
          <template v-for="section in reportData.sections" :key="section.id">
            <template v-for="row in renderSection(section, reportData.columns)" :key="row.name + row.level + row.type">
              <tr :class="[row.isTotal ? 'bg-blue-50 font-bold border-t-2 border-blue-200' : row.type === 'section' ? 'bg-gray-50 font-semibold' : '', 'hover:bg-gray-100']">
                <td :style="{ paddingLeft: `${row.level * 16}px` }" class="px-4 py-2 text-gray-900">
                  {{ row.name }}
                </td>
                <template v-for="(value, idx) in row.values || []" :key="idx">
                  <td class="px-4 py-2 text-right" v-html="formatCurrency(value)"></td>
                </template>
              </tr>
            </template>
          </template>
          <!-- Summary Rows -->
          <template v-for="row in getSummaryRows(reportData.summary, reportData.columns)" :key="row.name">
            <tr class="bg-green-50 font-bold border-t-2 border-green-200">
              <td class="px-4 py-2 text-gray-900">{{ row.name }}</td>
              <template v-for="(value, idx) in row.values || []" :key="idx">
                <td class="px-4 py-2 text-right" v-html="formatCurrency(value)"></td>
              </template>
            </tr>
          </template>
        </tbody>
      </table>
    </div>
  </div>
</template>
