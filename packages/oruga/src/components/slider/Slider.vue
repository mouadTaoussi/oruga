<script setup lang="ts" generic="IsRange extends boolean = false">
import { computed, ref, watch } from "vue";

import OSliderThumb from "./SliderThumb.vue";
import OSliderTick from "./SliderTick.vue";

import { getOption } from "@/utils/config";
import { isTrueish } from "@/utils/helpers";
import { defineClasses, useProviderParent } from "@/composables";

import type { SliderComponent } from "./types";
import type { SelectProps } from "./props";

/**
 * A slider to select a value or range from a given range
 * @displayName Slider
 * @requires ./SliderTick.vue
 * @style _slider.scss
 */
defineOptions({
    isOruga: true,
    name: "OSlider",
    configField: "slider",
});

const props = withDefaults(defineProps<SelectProps<IsRange>>(), {
    override: undefined,
    modelValue: undefined,
    // range: false,
    min: 0,
    max: 100,
    step: 1,
    variant: () => getOption("slider.variant"),
    size: () => getOption("slider.size"),
    ticks: false,
    tooltip: () => getOption("slider.tooltip", true),
    tooltipVariant: () => getOption("slider.tooltipVariant"),
    tooltipAlways: false,
    rounded: () => getOption("slider.rounded", false),
    disabled: false,
    lazy: false,
    customFormatter: undefined,
    biggerSliderFocus: false,
    indicator: false,
    format: () => getOption("slider.format", "raw"),
    locale: () => getOption("locale"),
    ariaLabel: () => getOption("slider.ariaLabel"),
});

type ModelValue = typeof props.modelValue;

const emits = defineEmits<{
    /**
     * modelValue prop two-way binding
     * @param value {number | number[]} updated modelValue prop
     */
    (e: "update:modelValue", value: ModelValue): void;
    /**
     * on value change event
     * @param value {number | number[]} updated modelValue prop
     */
    (e: "change", value: ModelValue): void;
    /**
     * on dragging event
     * @param value {number | number[]} updated modelValue prop
     * */
    (e: "dragging", value: ModelValue): void;
    /** on drag start event */
    (e: "dragstart"): void;
    /** on drag end event */
    (e: "dragend"): void;
}>();

const sliderRef = ref();
const thumbStartRef = ref();
const thumbEndRef = ref();

// Provided data is a computed ref to enjure reactivity.
const provideData = computed<SliderComponent>(() => ({
    max: props.max,
    min: props.min,
}));

/** Provide functionalities and data to child item components */
useProviderParent(undefined, { data: provideData });

const valueStart = ref<number>(null);
const valueEnd = ref<number>(null);
const dragging = ref(false);

const isThumbReversed = ref();
const isTrackClickDisabled = ref();

const minValue = computed(() => Math.min(valueStart.value, valueEnd.value));

const maxValue = computed(() => Math.max(valueStart.value, valueEnd.value));

const vmodel = computed<ModelValue>(
    () =>
        (isTrueish(props.range)
            ? [minValue.value, maxValue.value]
            : valueStart.value || 0) as ModelValue,
);

/** update vmodel value on internal value change */
watch([valueStart, valueEnd], () => {
    if (isTrueish(props.range))
        isThumbReversed.value = valueStart.value > valueEnd.value;
    if (!props.lazy || !dragging.value)
        emits("update:modelValue", vmodel.value); // update external vmodel
    if (dragging.value) emits("dragging", vmodel.value);
});

/** When min, max or v-model is changed set the new active step. */
watch(
    [() => props.min, () => props.max, () => props.modelValue],
    () => setValues(props.modelValue),
    { immediate: true }, // initialise valueStart and valueEnd
);

function setValues(newValue: number | number[]): void {
    if (props.min > props.max) return;

    if (Array.isArray(newValue)) {
        const smallValue =
            typeof newValue[0] !== "number" || isNaN(newValue[0])
                ? props.min
                : Math.min(Math.max(props.min, newValue[0]), props.max);
        const largeValue =
            typeof newValue[1] !== "number" || isNaN(newValue[1])
                ? props.max
                : Math.max(Math.min(props.max, newValue[1]), props.min);
        valueStart.value = isThumbReversed.value ? largeValue : smallValue;
        valueEnd.value = isThumbReversed.value ? smallValue : largeValue;
    } else {
        valueStart.value = isNaN(newValue)
            ? props.min
            : Math.min(props.max, Math.max(props.min, newValue));
        valueEnd.value = null;
    }
}

const tickValues = computed(() => {
    if (!props.ticks || props.min > props.max || props.step === 0) return [];
    const result = [];
    for (let i = props.min + props.step; i < props.max; i = i + props.step) {
        result.push(i);
    }
    return result;
});

const barSize = computed(() =>
    isTrueish(props.range)
        ? `${
              (100 * (maxValue.value - minValue.value)) /
              (props.max - props.min)
          }%`
        : `${
              (100 * (valueStart.value - props.min)) / (props.max - props.min)
          }%`,
);

const barStart = computed(() =>
    isTrueish(props.range)
        ? `${(100 * (minValue.value - props.min)) / (props.max - props.min)}%`
        : "0%",
);

const barStyle = computed(() => ({
    width: barSize.value,
    left: barStart.value,
}));

function getSliderSize(): number {
    return sliderRef.value.getBoundingClientRect().width;
}

function onSliderClick(event: MouseEvent): void {
    if (props.disabled || isTrackClickDisabled.value) return;
    const sliderOffsetLeft = sliderRef.value.getBoundingClientRect().left;
    const percent =
        ((event.clientX - sliderOffsetLeft) / getSliderSize()) * 100;
    const targetValue = props.min + (percent * (props.max - props.min)) / 100;
    const diffFirst = Math.abs(targetValue - valueStart.value);
    if (!isTrueish(props.range)) {
        if (diffFirst < props.step / 2) return;
        thumbStartRef.value.setPosition(percent);
    } else {
        const diffSecond = Math.abs(targetValue - valueEnd.value);
        if (diffFirst <= diffSecond) {
            if (diffFirst < props.step / 2) return;
            thumbStartRef.value.setPosition(percent);
        } else {
            if (diffSecond < props.step / 2) return;
            thumbEndRef.value.setPosition(percent);
        }
    }
    emits("change", vmodel.value);
}

function onDragStart(): void {
    dragging.value = true;
    emits("dragstart");
}

function onDragEnd(): void {
    isTrackClickDisabled.value = true;
    // avoid triggering onSliderClick after dragend
    setTimeout(() => (isTrackClickDisabled.value = false));
    dragging.value = false;
    emits("dragend");
    if (props.lazy) emits("update:modelValue", vmodel.value);
}

// --- Computed Component Classes ---

const rootClasses = defineClasses(
    ["rootClass", "o-slide"],
    [
        "sizeClass",
        "o-slide--",
        computed(() => props.size),
        computed(() => !!props.size),
    ],
    [
        "disabledClass",
        "o-slide--disabled",
        null,
        computed(() => props.disabled),
    ],
);

const trackClasses = defineClasses(["trackClass", "o-slide__track"]);

const fillClasses = defineClasses(
    ["fillClass", "o-slide__fill"],
    [
        "variantClass",
        "o-slide__fill--",
        computed(() => props.variant),
        computed(() => !!props.variant),
    ],
);

const thumbClasses = defineClasses(
    ["thumbClass", "o-slide__thumb"],
    ["thumbDraggingClass", "o-slide__thumb--dragging", null, dragging],
    [
        "thumbRoundedClass",
        "o-slide__thumb--rounded",
        null,
        computed(() => props.rounded),
    ],
);

const thumbWrapperClasses = defineClasses(
    ["thumbWrapperClass", "o-slide__thumb-wrapper"],
    [
        "thumbWrapperDraggingClass",
        "o-slide__thumb-wrapper--dragging",
        null,
        dragging,
    ],
);

// --- Expose Public Functionalities ---

/** expose functionalities for programmatic usage */
defineExpose({ value: vmodel });
</script>

<template>
    <div :class="rootClasses" data-oruga="slider" @click="onSliderClick">
        <div ref="sliderRef" :class="trackClasses">
            <div :class="fillClasses" :style="barStyle" />
            <template v-if="ticks">
                <o-slider-tick
                    v-for="(val, key) in tickValues"
                    :key="key"
                    :value="val"
                    :tick-class="tickClass"
                    :tick-hidden-class="tickHiddenClass"
                    :tick-label-class="tickLabelClass" />
            </template>

            <!--
                @slot Define additional slider ticks here
             -->
            <slot />

            <o-slider-thumb
                ref="thumbStartRef"
                v-model="valueStart"
                :slider-props="props"
                :slider-size="getSliderSize"
                :thumb-classes="thumbClasses"
                :thumb-wrapper-classes="thumbWrapperClasses"
                @change="emits('change', vmodel)"
                @dragstart="onDragStart"
                @dragend="onDragEnd" />

            <o-slider-thumb
                v-if="isTrueish(props.range)"
                ref="thumbEndRef"
                v-model="valueEnd"
                :slider-props="props"
                :slider-size="getSliderSize"
                :thumb-classes="thumbClasses"
                :thumb-wrapper-classes="thumbWrapperClasses"
                @change="emits('change', vmodel)"
                @dragstart="onDragStart"
                @dragend="onDragEnd" />
        </div>
    </div>
</template>
