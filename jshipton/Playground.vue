<template>
  <v-app theme="dark">
    <v-container class="py-6" fluid>
      <div class="d-flex ga-4 align-start">

        <!-- Subject: the component being changed -->
        <div class="flex-0-0-auto">
          <div class="text-overline text-medium-emphasis mb-3">v-heatmap</div>

          <div class="d-flex flex-column ga-4">
            <div v-for="variant in heatmapVariants" :key="variant.label">
              <code class="text-caption d-block mb-2">{{ variant.label }}</code>
              <v-heatmap v-bind="{ ...shared, ...variant.props }" />
            </div>
          </div>
        </div>

        <!-- Reference: the existing convention being mirrored -->
        <div class="flex-0-0-auto">
          <div class="text-overline text-medium-emphasis mb-3">
            v-date-picker &mdash; existing convention
          </div>

          <div class="d-flex flex-column ga-4">
            <div v-for="day in [undefined, 1, 6]" :key="String(day)">
              <code class="text-caption d-block mb-2">
                {{ day === undefined ? 'default' : `first-day-of-week="${day}"` }}
              </code>
              <v-date-picker
                :first-day-of-week="day"
                class="border rounded"
                hide-header
                show-adjacent-months
              />
            </div>
          </div>
        </div>

      </div>
    </v-container>
  </v-app>
</template>

<script setup>
  import { useDate } from 'vuetify'

  const adapter = useDate()
  const start = adapter.date('2026-06-01')
  const items = Array.from({ length: 122 }, (_, i) => {
    const date = adapter.addDays(start, i)
    return { date, value: (i * 7) % 13 }
  })

  const shared = {
    items,
    rows: [0, 1, 2, 3, 4, 5, 6],
    itemRow: v => adapter.toJsDate(v.date).getDay(),
    groupBy: v => adapter.format(v.date, 'monthAndYear'),
    groupGap: 16,
    cellSize: 18,
    gap: 3,
    thresholds: [
      { min: 1, color: '#2d5f3a' },
      { min: 4, color: '#3f8a4f' },
      { min: 8, color: '#5cb86a' },
      { min: 11, color: '#9be3a3' },
    ],
  }

  const heatmapVariants = [
    { label: 'default', props: {} },
    { label: 'first-day-of-week="1"', props: { firstDayOfWeek: '1' } },
    { label: 'first-day-of-week="6" group-gap="0"', props: { firstDayOfWeek: '6', groupGap: 0 } },
  ]
</script>