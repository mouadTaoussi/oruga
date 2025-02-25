<script
    setup
    lang="ts"
    generic="
        T extends string | number | object,
        IsMultiple extends boolean = false
    ">
import {
    computed,
    nextTick,
    ref,
    watch,
    onUnmounted,
    type Component,
} from "vue";

import PositionWrapper from "../utils/PositionWrapper.vue";

import { getOption } from "@/utils/config";
import { vTrapFocus } from "@/directives/trapFocus";
import { toCssDimension, isMobileAgent, isTrueish } from "@/utils/helpers";
import { isClient } from "@/utils/ssr";
import {
    unrefElement,
    defineClasses,
    useProviderParent,
    useMatchMedia,
    useEventListener,
    useClickOutside,
    useVModel,
} from "@/composables";

import type { DynamicComponent } from "@/types";
import type { DropdownComponent } from "./types";
import type { DropdownProps } from "./props";

/**
 * Dropdowns are very versatile, can used as a quick menu or even like a select for discoverable content
 * @displayName Dropdown
 * @requires ./DropdownItem.vue
 * @style _dropdown.scss
 */
defineOptions({
    isOruga: true,
    name: "ODropdown",
    configField: "dropdown",
});

const props = withDefaults(defineProps<DropdownProps<T, IsMultiple>>(), {
    override: undefined,
    modelValue: undefined,
    // multiple: false,
    active: false,
    label: undefined,
    disabled: false,
    inline: false,
    scrollable: false,
    maxHeight: () => getOption("dropdown.maxHeight", 200),
    position: () => getOption("dropdown.position", "bottom-left"),
    mobileModal: () => getOption("dropdown.mobileModal", true),
    animation: () => getOption("dropdown.animation", "fade"),
    trapFocus: () => getOption("dropdown.trapFocus", true),
    checkScroll: () => getOption("dropdown.checkScroll", false),
    expanded: false,
    menuId: null,
    menuTabindex: null,
    menuTag: () => getOption<DynamicComponent>("dropdown.menuTag", "div"),
    triggerTag: () => getOption<DynamicComponent>("dropdown.triggerTag", "div"),
    triggers: () => getOption("dropdown.triggers", ["click"]),
    delay: undefined,
    closeable: () =>
        getOption("dropdown.closeable", ["escape", "outside", "content"]),
    tabindex: 0,
    ariaRole: () => getOption("dropdown.ariaRole", "list"),
    mobileBreakpoint: () => getOption("dropdown.mobileBreakpoint"),
    teleport: () => getOption("dropdown.teleport", false),
});

type ModelValue = typeof props.modelValue;

const emits = defineEmits<{
    /**
     * modelValue prop two-way binding
     * @param value {string | number | object | array} updated modelValue prop
     */
    (e: "update:modelValue", value: ModelValue): void;
    /**
     * active prop two-way binding
     * @param value {boolean} updated active prop
     */
    (e: "update:active", value: boolean): void;
    /**
     * on change event - fired after update:modelValue
     * @param value {string | number | object | array} selected value
     */
    (e: "change", value: ModelValue): void;
    /**
     * on close event
     * @param method {string} close method
     */
    (e: "close", method: string): void;
    /** the list inside the dropdown reached the start */
    (e: "scroll-start"): void;
    /** the list inside the dropdown reached it's end */
    (e: "scroll-end"): void;
}>();

/** The selected item value */
// const vmodel = defineModel<ModelValue>({ default: undefined });
const vmodel = useVModel<ModelValue>();

/** The active state of the dropdown, use v-model:active to make it two-way binding */
const isActive = defineModel<boolean>("active", { default: false });

const autoPosition = ref(props.position);

/** update autoPosition on prop change */
watch(
    () => props.position,
    (v) => (autoPosition.value = v),
);

const { isMobile } = useMatchMedia(props.mobileBreakpoint);

// check if mobile modal should be shown
const isMobileModal = computed(
    () => isMobile.value && props.mobileModal && !props.inline,
);

// check if client is mobile native
const isMobileNative = computed(() => props.mobileModal && isMobileAgent.any());

const menuStyle = computed(() => ({
    maxHeight: props.scrollable ? toCssDimension(props.maxHeight) : null,
    overflow: props.scrollable ? "auto" : null,
}));

const hoverable = computed(() => props.triggers.indexOf("hover") >= 0);

// --- Event Handler ---

const contentRef = ref<HTMLElement | Component>();
const triggerRef = ref<HTMLElement>();

const eventCleanups = [];
let timer: NodeJS.Timeout;

const cancelOptions = computed(() =>
    typeof props.closeable === "boolean"
        ? props.closeable
            ? ["escape", "outside", "content"]
            : []
        : props.closeable,
);

watch(
    isActive,
    (value) => {
        // on active set event handler
        if (value && isClient) {
            if (cancelOptions.value.indexOf("outside") >= 0) {
                // set outside handler
                eventCleanups.push(
                    useClickOutside(contentRef, onClickedOutside, {
                        ignore: [triggerRef],
                        immediate: true,
                        passive: true,
                    }),
                );
            }

            if (cancelOptions.value.indexOf("escape") >= 0) {
                // set keyup handler
                eventCleanups.push(
                    useEventListener("keyup", onKeyPress, document, {
                        immediate: true,
                    }),
                );
            }
        } else if (!value) {
            // on close cleanup event handler
            eventCleanups.forEach((fn) => fn());
            eventCleanups.length = 0;
        }
    },
    { immediate: true, flush: "post" },
);

onUnmounted(() => {
    // on close cleanup event handler
    eventCleanups.forEach((fn) => fn());
    eventCleanups.length = 0;
});

/** Close dropdown if clicked outside. */
function onClickedOutside(): void {
    if (!isActive.value || props.inline) return;
    if (cancelOptions.value.indexOf("outside") < 0) return;
    emits("close", "outside");
    isActive.value = false;
}

/** Keypress event that is bound to the document */
function onKeyPress(event: KeyboardEvent): void {
    if (isActive.value && (event.key === "Escape" || event.key === "Esc")) {
        if (cancelOptions.value.indexOf("escape") < 0) return;
        emits("close", "escape");
        isActive.value = false;
    }
}

function onClick(): void {
    if (props.triggers.indexOf("click") < 0) return;
    toggle();
}

function onContextMenu(event: MouseEvent): void {
    if (props.triggers.indexOf("contextmenu") < 0) return;
    event.preventDefault();
    open();
}

function onFocus(): void {
    if (props.triggers.indexOf("focus") < 0) return;
    open();
}

const isHovered = ref(false);
function onHover(): void {
    if (!isMobileNative.value && props.triggers.indexOf("hover") >= 0) {
        isHovered.value = true;
        open();
    }
}
function onHoverLeave(): void {
    if (!isMobileNative.value && isHovered.value) {
        isHovered.value = false;
        onClose();
    }
}

/** Toggle dropdown if it's not disabled. */
function toggle(): void {
    if (props.disabled) return;
    if (isActive.value) isActive.value = !isActive.value;
    // if not active, toggle after clickOutside event
    // this fixes toggling programmatic
    else nextTick(() => (isActive.value = !isActive.value));
}

function open(): void {
    if (props.disabled) return;
    if (props.delay) {
        timer = setTimeout(() => {
            isActive.value = true;
            timer = null;
        }, props.delay);
    } else {
        isActive.value = true;
    }
}

function onClose(): void {
    if (cancelOptions.value.indexOf("content") < 0) return;
    emits("close", "content");
    isActive.value = !props.closeable;
    if (timer && props.closeable) clearTimeout(timer);
}

// --- InfitiveScroll Feature ---

if (isClient && props.checkScroll)
    useEventListener("scroll", checkDropdownScroll, contentRef);

/** Check if the scroll list inside the dropdown reached the top or it's end. */
function checkDropdownScroll(): void {
    const dropdown = unrefElement(contentRef);
    if (dropdown.clientHeight !== dropdown.scrollHeight) {
        if (
            dropdown.scrollTop + dropdown.clientHeight >=
            dropdown.scrollHeight
        ) {
            emits("scroll-end");
        } else if (dropdown.scrollTop <= 0) {
            emits("scroll-start");
        }
    }
}

// --- Dependency Injection Feature ---

/**
 * Click listener from DropdownItem.
 *   1. Set new selected item.
 *   2. Emit input event to update the user v-model.
 *   3. Close the dropdown.
 */
function selectItem(value: T): void {
    if (isTrueish(props.multiple)) {
        if (vmodel.value && Array.isArray(vmodel.value)) {
            if (!vmodel.value.includes(value)) {
                // add a value
                vmodel.value = [...vmodel.value, value] as ModelValue;
            } else {
                // remove a value
                vmodel.value = vmodel.value.filter(
                    (val) => val !== value,
                ) as ModelValue;
            }
        } else {
            // init new value array
            vmodel.value = [value] as ModelValue;
        }
        // emit change after vmodel has changed
        nextTick(() => emits("change", vmodel.value));
    } else {
        if (vmodel.value !== value) {
            // update a single value
            vmodel.value = value as ModelValue;
            // emit change after vmodel has changed
            nextTick(() => emits("change", vmodel.value));
        }
    }
    if (!props.multiple) {
        if (cancelOptions.value.indexOf("content") < 0) return;
        emits("close", "content");
        isActive.value = false;
        isHovered.value = false;
    }
}

// Provided data is a computed ref to enjure reactivity.
const provideData = computed<DropdownComponent<T>>(() => ({
    props,
    selected: vmodel.value,
    selectItem,
}));

/** Provide functionalities and data to child item components */
useProviderParent(contentRef, { data: provideData });

// --- Computed Component Classes ---

const rootClasses = defineClasses(
    ["rootClass", "o-drop"],
    ["disabledClass", "o-drop--disabled", null, computed(() => props.disabled)],
    ["expandedClass", "o-drop--expanded", null, computed(() => props.expanded)],
    ["inlineClass", "o-drop--inline", null, computed(() => props.inline)],
    [
        "mobileClass",
        "o-drop--mobile",
        null,
        computed(() => isMobileModal.value && !hoverable.value),
    ],
    [
        "positionClass",
        "o-drop--position-",
        autoPosition,
        computed(() => !!autoPosition.value),
    ],
    [
        "activeClass",
        "o-drop--active",
        null,
        computed(() => isActive.value || props.inline),
    ],
    ["hoverableClass", "o-drop--hoverable", null, hoverable],
);

const triggerClasses = defineClasses(["triggerClass", "o-drop__trigger"]);

const positionWrapperClasses = defineClasses([
    "teleportClass",
    "o-drop--teleport",
    null,
    computed(() => !!props.teleport),
]);

const menuMobileOverlayClasses = defineClasses([
    "menuMobileOverlayClass",
    "o-drop__overlay",
]);

const menuClasses = defineClasses(
    ["menuClass", "o-drop__menu"],
    [
        "menuPositionClass",
        "o-drop__menu--",
        autoPosition,
        computed(() => !!autoPosition.value),
    ],
    [
        "menuActiveClass",
        "o-drop__menu--active",
        null,
        computed(() => isActive.value || props.inline),
    ],
);

// --- Expose Public Functionalities ---

/** expose functionalities for programmatic usage */
defineExpose({ $trigger: triggerRef, $content: contentRef, value: vmodel });
</script>

<template>
    <div
        data-oruga="dropdown"
        :class="rootClasses"
        @mouseleave="onHoverLeave"
        @focusout="onHoverLeave">
        <component
            :is="triggerTag"
            v-if="!inline"
            ref="triggerRef"
            :tabindex="disabled ? null : tabindex"
            :class="triggerClasses"
            :aria-haspopup="ariaRole === 'list' ? true : ariaRole"
            @click="onClick"
            @contextmenu="onContextMenu"
            @mouseenter="onHover"
            @focus.capture="onFocus">
            <!--
                @slot Override the trigger element, default is label prop
                @binding {boolean} active - dropdown active state
            -->
            <slot name="trigger" :active="isActive">
                {{ label }}
            </slot>
        </component>

        <PositionWrapper
            v-slot="{ setContent }"
            v-model:position="autoPosition"
            :teleport="teleport"
            :class="[...rootClasses, ...positionWrapperClasses]"
            :trigger="triggerRef"
            :disabled="!isActive"
            default-position="bottom"
            :disable-positioning="!isMobileModal">
            <transition :name="animation">
                <div
                    v-if="isMobileModal"
                    v-show="isActive"
                    :tabindex="-1"
                    :class="menuMobileOverlayClasses"
                    :aria-hidden="disabled || !isActive" />
            </transition>

            <transition :name="animation">
                <component
                    :is="menuTag"
                    v-show="(!disabled && (isActive || isHovered)) || inline"
                    :id="menuId"
                    :ref="(el) => (contentRef = setContent(el))"
                    v-trap-focus="trapFocus"
                    :tabindex="menuTabindex"
                    :class="menuClasses"
                    :style="menuStyle"
                    :role="ariaRole"
                    :aria-hidden="disabled || !isActive"
                    :aria-modal="!inline && trapFocus">
                    <!--
                        @slot Place dropdown items here
                        @binding {boolean} active - dropdown active state
                        @binding {boolean} toggle - toggle active state function
                    -->
                    <slot :active="isActive" :toggle="toggle" />
                </component>
            </transition>
        </PositionWrapper>
    </div>
</template>
