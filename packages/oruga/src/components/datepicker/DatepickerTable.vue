<script setup lang="ts">
import { computed, ref, type PropType } from "vue";

import ODatepickerTableRow from "./DatepickerTableRow.vue";

import { isTrueish, isDefined } from "@/utils/helpers";
import { defineClasses } from "@/composables";

import { useDatepickerMixins } from "./useDatepickerMixins";
import { weekBuilder } from "./utils";
import type { DatepickerProps, DatepickerEvent, FocusedDate } from "./types";

defineOptions({
    name: "ODatepickerTable",
    configField: "datepicker",
});

const props = defineProps({
    modelValue: {
        type: [Date, Array] as PropType<Date | Date[]>,
        default: undefined,
    },
    focusedDate: { type: Object as PropType<FocusedDate>, required: true },
    dayNames: { type: Array as PropType<string[]>, required: true },
    monthNames: { type: Array as PropType<string[]>, required: true },
    pickerProps: {
        type: Object as PropType<DatepickerProps>,
        required: true,
    },
});

const emits = defineEmits<{
    /** modelValue prop two-way binding */
    (e: "update:modelValue", value: Date | Date[]): void;
    /** focusedDate prop two-way binding */
    (e: "update:focusedDate", value: FocusedDate): void;
    (e: "range-start", value: Date): void;
    (e: "range-end", value: Date): void;
    (e: "week-number-click", value: number): void;
}>();

const { isDateSelectable } = useDatepickerMixins(props.pickerProps);

const focusedDateModel = defineModel<FocusedDate>("focusedDate");

const selectedBeginDate = ref<Date>();
const selectedEndDate = ref<Date>();
const hoveredEndDate = ref<Date>();

const datepicker = computed<DatepickerProps>(() => props.pickerProps);

const visibleDayNames = computed(() => {
    const visibleDayNames = [];
    let index = datepicker.value.firstDayOfWeek;
    while (visibleDayNames.length < props.dayNames.length) {
        const currentDayName = props.dayNames[index % props.dayNames.length];
        visibleDayNames.push(currentDayName);
        index++;
    }
    if (datepicker.value.showWeekNumber) visibleDayNames.unshift("");
    return visibleDayNames;
});

/** Return array of all events in the specified month */
const eventsInThisMonth = computed(() => {
    if (!datepicker.value.events) return [];
    return datepicker.value.events
        .map((event) =>
            !event.date && event instanceof Date ? { date: event } : event,
        )
        .filter(
            (event) =>
                event.date.getMonth() === focusedDateModel.value.month &&
                event.date.getFullYear() === focusedDateModel.value.year,
        );
});

/** Return array of all weeks in the specified month */
const weeksInThisMonth = computed(() => {
    validateFocusedDay();
    const month = focusedDateModel.value.month;
    const year = focusedDateModel.value.year;
    const weeksInThisMonth = [];

    let startingDay = 1;

    while (weeksInThisMonth.length < 6) {
        const newWeek = weekBuilder(
            startingDay,
            month,
            year,
            datepicker.value.firstDayOfWeek,
        );
        weeksInThisMonth.push(newWeek);
        startingDay += 7;
    }

    return weeksInThisMonth;
});

function eventsInThisWeek(week: Date[]): DatepickerEvent[] {
    if (!datepicker.value.events) return [];
    return eventsInThisMonth.value.filter((event) => {
        const stripped = new Date(event.date);
        stripped.setHours(0, 0, 0, 0);
        const timed = stripped.getTime();
        return week.some((weekDate) => weekDate.getTime() === timed);
    });
}

const hoveredDateRange = computed(() => {
    if (!isTrueish(datepicker.value.range) || selectedEndDate.value) return [];
    return (
        hoveredEndDate.value < selectedBeginDate.value
            ? [hoveredEndDate.value, selectedBeginDate.value]
            : [selectedBeginDate.value, hoveredEndDate.value]
    ).filter(isDefined);
});

function validateFocusedDay(): void {
    const currentDate = new Date(
        focusedDateModel.value.year,
        focusedDateModel.value.month,
        focusedDateModel.value.day,
    );
    if (isDateSelectable(currentDate, focusedDateModel.value.month)) return;

    let day = 0;
    // Number of days in the current month
    const monthDays = new Date(
        focusedDateModel.value.year,
        focusedDateModel.value.month + 1,
        0,
    ).getDate();
    let firstFocusable = null;
    while (!firstFocusable && ++day < monthDays) {
        const date = new Date(
            focusedDateModel.value.year,
            focusedDateModel.value.month,
            day,
        );
        if (isDateSelectable(date, focusedDateModel.value.month)) {
            firstFocusable = currentDate;
            focusedDateModel.value = {
                day: date.getDate(),
                month: date.getMonth(),
                year: date.getFullYear(),
            };
        }
    }
}

// --- Event Handlers ---

/** Emit input event with selected date as payload for v-model in parent */
function onSelectedDate(date: Date): void {
    if (datepicker.value.disabled) return;
    else if (isTrueish(datepicker.value.range)) handleSelectRangeDate(date);
    else if (isTrueish(datepicker.value.multiple))
        handleSelectMultipleDates(date);
    else emits("update:modelValue", date);
}

/*
 * If both begin and end dates are set, reset the end date and set the begin date.
 * If only begin date is selected, emit an array of the begin date and the new date.
 * If not set, only set the begin date.
 */
function handleSelectRangeDate(date: Date): void {
    if (selectedBeginDate.value && selectedEndDate.value) {
        selectedBeginDate.value = date;
        selectedEndDate.value = undefined;
        emits("range-start", date);
    } else if (selectedBeginDate.value && !selectedEndDate.value) {
        if (selectedBeginDate.value > date) {
            selectedEndDate.value = selectedBeginDate.value;
            selectedBeginDate.value = date;
        } else {
            selectedEndDate.value = date;
        }
        emits("range-end", date);
        emits("update:modelValue", [
            selectedBeginDate.value,
            selectedEndDate.value,
        ]);
    } else {
        selectedBeginDate.value = date;
        emits("range-start", date);
    }
}

/*
 * If selected date already exists list of selected dates, remove it from the list
 * Otherwise, add date to list of selected dates
 */
function handleSelectMultipleDates(date: Date): void {
    let multipleSelectedDates = Array.isArray(props.modelValue)
        ? props.modelValue
        : [];
    const multipleSelect = multipleSelectedDates.filter(
        (selectedDate) =>
            selectedDate.getDate() === date.getDate() &&
            selectedDate.getFullYear() === date.getFullYear() &&
            selectedDate.getMonth() === date.getMonth(),
    );
    if (multipleSelect.length) {
        multipleSelectedDates = multipleSelectedDates.filter(
            (selectedDate) =>
                selectedDate.getDate() !== date.getDate() ||
                selectedDate.getFullYear() !== date.getFullYear() ||
                selectedDate.getMonth() !== date.getMonth(),
        );
    } else {
        multipleSelectedDates = [...multipleSelectedDates, date];
    }
    emits("update:modelValue", multipleSelectedDates);
}

function onRangeHoverEndDate(date: Date): void {
    hoveredEndDate.value = date;
}

function onChangeFocus(date: Date): void {
    focusedDateModel.value = {
        day: date.getDate(),
        month: date.getMonth(),
        year: date.getFullYear(),
    };
}

// --- Computed Component Classes ---

const tableClasses = defineClasses(["tableClass", "o-dpck__table"]);

const tableHeadClasses = defineClasses([
    "tableHeadClass",
    "o-dpck__table__head",
]);

const tableCellClasses = defineClasses([
    "tableCellClass",
    "o-dpck__table__cell",
]);

const tableHeadCellClasses = defineClasses([
    "tableHeadCellClass",
    "o-dpck__table__head-cell",
]);

const tableBodyClasses = defineClasses([
    "tableBodyClass",
    "o-dpck__table__body",
]);
</script>

<template>
    <section :class="tableClasses">
        <header :class="tableHeadClasses">
            <div
                v-for="(day, index) in visibleDayNames"
                :key="index"
                :class="[...tableCellClasses, ...tableHeadCellClasses]">
                <span>{{ day }}</span>
            </div>
        </header>
        <div :class="tableBodyClasses">
            <o-datepicker-table-row
                v-for="(week, index) in weeksInThisMonth"
                :key="index"
                :selected-date="modelValue"
                :day="focusedDateModel.day"
                :week="week"
                :month="focusedDateModel.month"
                :events="eventsInThisWeek(week)"
                :hovered-date-range="hoveredDateRange"
                :picker-props="props.pickerProps"
                @select="onSelectedDate"
                @hover-enddate="onRangeHoverEndDate"
                @change-focus="onChangeFocus"
                @week-number-click="$emit('week-number-click', $event)" />
        </div>
    </section>
</template>
