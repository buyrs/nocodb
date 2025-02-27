<script lang="ts" setup>
import dayjs from 'dayjs'
import { UITypes } from 'nocodb-sdk'
import {
  ActiveViewInj,
  IsCalendarInj,
  IsFormInj,
  IsGalleryInj,
  IsGridInj,
  IsKanbanInj,
  MetaInj,
  ReloadViewDataHookInj,
  ReloadViewMetaHookInj,
  type Row as RowType,
  computed,
  extractPkFromRow,
  inject,
  provide,
  ref,
  rowDefaultData,
} from '#imports'

const meta = inject(MetaInj, ref())

const view = inject(ActiveViewInj, ref())

const reloadViewMetaHook = inject(ReloadViewMetaHookInj)

const reloadViewDataHook = inject(ReloadViewDataHookInj)

const { isMobileMode } = useGlobal()

provide(IsFormInj, ref(false))

provide(IsGalleryInj, ref(false))

provide(IsGridInj, ref(false))

provide(IsKanbanInj, ref(false))

provide(IsCalendarInj, ref(true))

const {
  activeCalendarView,
  calendarRange,
  calDataType,
  loadCalendarMeta,
  loadCalendarData,
  loadSidebarData,
  isCalendarDataLoading,
  selectedDate,
  selectedMonth,
  activeDates,
  pageDate,
  showSideMenu,
  selectedDateRange,
  paginateCalendarView,
} = useCalendarViewStoreOrThrow()

const calendarRangeDropdown = ref(false)

const router = useRouter()

const route = useRoute()

const expandedFormOnRowIdDlg = computed({
  get() {
    return !!route.query.rowId
  },
  set(val) {
    if (!val)
      router.push({
        query: {
          ...route.query,
          rowId: undefined,
        },
      })
  },
})

const expandedFormDlg = ref(false)

const expandedFormRow = ref<RowType>()

const expandedFormRowState = ref<Record<string, any>>()

const expandRecord = (row: RowType, state?: Record<string, any>) => {
  const rowId = extractPkFromRow(row.row, meta.value!.columns!)
  if (rowId) {
    router.push({
      query: {
        ...route.query,
        rowId,
      },
    })
  } else {
    expandedFormRow.value = row
    expandedFormRowState.value = state
    expandedFormDlg.value = true
  }
}

const newRecord = (row: RowType) => {
  // TODO: The default values has to be filled based on the active calendar view
  // and selected sidebar filter option
  expandRecord({
    row: {
      ...rowDefaultData(meta.value?.columns),
      ...row.row,
    },
    oldRow: {},
    rowMeta: {
      new: true,
    },
  })
}

onMounted(async () => {
  await loadCalendarMeta()
  await loadCalendarData()
  if (!activeCalendarView.value) {
    activeCalendarView.value = 'month'
  }
})

reloadViewMetaHook?.on(async () => {
  await loadCalendarMeta()
})

reloadViewDataHook?.on(async () => {
  await Promise.all([loadCalendarData(), loadSidebarData()])
})

const goToToday = () => {
  selectedDate.value = dayjs()
  pageDate.value = dayjs()
  selectedMonth.value = dayjs()
  selectedDateRange.value = {
    start: dayjs().startOf('week'),
    end: dayjs().endOf('week'),
  }

  document?.querySelector('.nc-calendar-today')?.scrollIntoView({
    behavior: 'smooth',
    block: 'center',
  })
}

const headerText = computed(() => {
  switch (activeCalendarView.value) {
    case 'day':
      return dayjs(selectedDate.value).format('D MMMM YYYY')
    case 'week':
      if (selectedDateRange.value.start.isSame(selectedDateRange.value.end, 'month')) {
        return `${selectedDateRange.value.start.format('D')} - ${selectedDateRange.value.end.format('D MMM YY')}`
      } else if (selectedDateRange.value.start.isSame(selectedDateRange.value.end, 'year')) {
        return `${selectedDateRange.value.start.format('D MMM')} - ${selectedDateRange.value.end.format('D MMM YY')}`
      } else {
        return `${selectedDateRange.value.start.format('D MMM YY')} - ${selectedDateRange.value.end.format('D MMM YY')}`
      }
    case 'month':
      return dayjs(selectedMonth.value).format('MMMM YYYY')
    case 'year':
      return dayjs(selectedDate.value).format('YYYY')
  }
})
</script>

<template>
  <div class="flex h-full flex-row" data-testid="nc-calendar-wrapper">
    <div class="flex flex-col w-full">
      <div class="flex justify-between p-3 items-center border-b-1 border-gray-200" data-testid="nc-calendar-topbar">
        <div class="flex justify-start gap-3 items-center">
          <NcTooltip>
            <template #title> {{ $t('labels.previous') }}</template>
            <NcButton
              v-e="`['c:calendar:calendar-${activeCalendarView}-prev-btn']`"
              data-testid="nc-calendar-prev-btn"
              size="small"
              type="secondary"
              @click="paginateCalendarView('prev')"
            >
              <component :is="iconMap.doubleLeftArrow" class="h-4 w-4" />
            </NcButton>
          </NcTooltip>

          <NcDropdown v-model:visible="calendarRangeDropdown" :auto-close="false" :trigger="['click']">
            <NcButton :class="{ '!w-22': activeCalendarView === 'year' }" class="w-45" full-width size="small" type="secondary">
              <div class="flex px-2 w-full items-center justify-between">
                <span class="font-bold text-center text-brand-500" data-testid="nc-calendar-active-date">{{ headerText }}</span>
                <component :is="iconMap.arrowDown" class="h-4 w-4 text-gray-700" />
              </div>
            </NcButton>

            <template #overlay>
              <div v-if="calendarRangeDropdown" class="min-w-[22.1rem]" @click.stop>
                <NcDateWeekSelector
                  v-if="activeCalendarView === ('day' as const)"
                  v-model:active-dates="activeDates"
                  v-model:page-date="pageDate"
                  v-model:selected-date="selectedDate"
                />
                <NcDateWeekSelector
                  v-else-if="activeCalendarView === ('week' as const)"
                  v-model:active-dates="activeDates"
                  v-model:page-date="pageDate"
                  v-model:selected-week="selectedDateRange"
                  is-week-picker
                />
                <NcMonthYearSelector
                  v-else-if="activeCalendarView === ('month' as const)"
                  v-model:page-date="pageDate"
                  v-model:selected-date="selectedMonth"
                />
                <NcMonthYearSelector
                  v-else-if="activeCalendarView === ('year' as const)"
                  v-model:page-date="pageDate"
                  v-model:selected-date="selectedDate"
                  is-year-picker
                />
              </div>
            </template>
          </NcDropdown>
          <NcTooltip>
            <template #title> {{ $t('labels.next') }}</template>
            <NcButton
              v-e="`['c:calendar:calendar-${activeCalendarView}-next-btn']`"
              data-testid="nc-calendar-next-btn"
              size="small"
              type="secondary"
              @click="paginateCalendarView('next')"
            >
              <component :is="iconMap.doubleRightArrow" class="h-4 w-4" />
            </NcButton>
          </NcTooltip>
          <NcButton
            v-if="!isMobileMode"
            v-e="`['c:calendar:calendar-${activeCalendarView}-today-btn']`"
            data-testid="nc-calendar-today-btn"
            size="small"
            type="secondary"
            @click="goToToday"
          >
            <span class="text-gray-700">
              {{ $t('activity.goToToday') }}
            </span>
          </NcButton>
          <span class="opacity-0" data-testid="nc-active-calendar-view">
            {{ activeCalendarView }}
          </span>
        </div>
        <NcTooltip>
          <template #title> {{ $t('activity.toggleSidebar') }}</template>
          <NcButton
            v-if="!isMobileMode"
            v-e="`['c:calendar:calendar-${activeCalendarView}-toggle-sidebar']`"
            data-testid="nc-calendar-side-bar-btn"
            size="small"
            type="secondary"
            @click="showSideMenu = !showSideMenu"
          >
            <component :is="iconMap.sidebar" class="h-4 w-4 text-gray-700 transition-all" />
          </NcButton>
        </NcTooltip>
      </div>
      <template v-if="calendarRange">
        <LazySmartsheetCalendarYearView v-if="activeCalendarView === 'year'" />
        <template v-if="!isCalendarDataLoading">
          <LazySmartsheetCalendarMonthView
            v-if="activeCalendarView === 'month'"
            @expand-record="expandRecord"
            @new-record="newRecord"
          />
          <LazySmartsheetCalendarWeekViewDateField
            v-else-if="activeCalendarView === 'week' && calDataType === UITypes.Date"
            @expand-record="expandRecord"
            @new-record="newRecord"
          />
          <LazySmartsheetCalendarWeekViewDateTimeField
            v-else-if="activeCalendarView === 'week' && calDataType === UITypes.DateTime"
            @expand-record="expandRecord"
            @new-record="newRecord"
          />
          <LazySmartsheetCalendarDayViewDateField
            v-else-if="activeCalendarView === 'day' && calDataType === UITypes.Date"
            @expand-record="expandRecord"
            @new-record="newRecord"
          />
          <LazySmartsheetCalendarDayViewDateTimeField
            v-else-if="activeCalendarView === 'day' && calDataType === UITypes.DateTime"
            @expand-record="expandRecord"
            @new-record="newRecord"
          />
        </template>

        <div v-if="isCalendarDataLoading && activeCalendarView !== 'year'" class="flex w-full items-center h-full justify-center">
          <GeneralLoader size="xlarge" />
        </div>
      </template>
      <template v-else>
        <div class="flex w-full items-center h-full justify-center">
          {{ $t('activity.noRange') }}
        </div>
      </template>
    </div>
    <LazySmartsheetCalendarSideMenu
      v-if="!isMobileMode"
      :visible="showSideMenu"
      @expand-record="expandRecord"
      @new-record="newRecord"
    />
  </div>

  <Suspense>
    <LazySmartsheetExpandedForm
      v-if="expandedFormRow && expandedFormDlg"
      v-model="expandedFormDlg"
      :meta="meta"
      :row="expandedFormRow"
      :state="expandedFormRowState"
      :view="view"
    />
  </Suspense>

  <Suspense>
    <LazySmartsheetExpandedForm
      v-if="expandedFormOnRowIdDlg"
      v-model="expandedFormOnRowIdDlg"
      :meta="meta"
      :row="{ row: {}, oldRow: {}, rowMeta: {} }"
      :row-id="route.query.rowId"
      :view="view"
    />
  </Suspense>
</template>
