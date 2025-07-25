<script lang="ts">
import { PropType, defineComponent } from 'vue';
import { _EDIT, _VIEW } from '@shell/config/query-params';
import { addObject, removeObject } from '@shell/utils/array';
import cloneDeep from 'lodash/cloneDeep';
import { generateRandomAlphaString } from '@shell/utils/string';

export default defineComponent({
  name: 'Checkbox',

  props: {
    /**
     * The checkbox value.
     */
    value: {
      type:    [Boolean, Array, String] as PropType<boolean | boolean[] | string | string[]>,
      default: false
    },

    /**
     * The checkbox label.
     */
    label: {
      type:    String,
      default: null
    },

    /**
     * The i18n key to use for the checkbox label.
     */
    labelKey: {
      type:    String,
      default: null
    },

    /**
     * Random ID generated for binding label to input.
     */
    id: {
      type:    String,
      default: generateRandomAlphaString(12)
    },

    /**
     * Disable the checkbox.
     */
    disabled: {
      type:    Boolean,
      default: false
    },

    /**
     * Display an indeterminate state. Useful for cases where a checkbox might
     * be the parent to child checkboxes, and we need to show that a subset of
     * children are checked.
     */
    indeterminate: {
      type:    Boolean,
      default: false
    },

    /**
     * The checkbox editing mode.
     * @values _EDIT, _VIEW
     */
    mode: {
      type:    String,
      default: _EDIT
    },

    /**
     * The contents of the checkbox tooltip.
     */
    tooltip: {
      type:    [String, Object],
      default: null
    },

    /**
     * The i18n key to use for the checkbox tooltip.
     */
    tooltipKey: {
      type:    String,
      default: null
    },

    /**
     * A custom value to use when the checkbox is checked.
     */
    valueWhenTrue: {
      type:    [Boolean, String, Number],
      default: true
    },

    /**
     * The i18n key to use for the checkbox description.
     */
    descriptionKey: {
      type:    String,
      default: null
    },

    /**
     * The checkbox description.
     */
    description: {
      type:    String,
      default: null
    },

    /**
     * Primary checkbox displays label so that it stands out more
     */
    primary: {
      type:    Boolean,
      default: false
    },

    /**
     * Use this for usage of checkboxes that don't present a label.
     * Used for cases such as table checkboxes (group or row)
     */
    alternateLabel: {
      type:    String,
      default: undefined
    },

    /**
     * Inherited global identifier prefix for tests
     * Define a term based on the parent component to avoid conflicts on multiple components
     */
    componentTestid: {
      type:    String,
      default: 'checkbox'
    },
  },

  emits: ['update:value'],

  data() {
    return { describedById: `described-by-${ generateRandomAlphaString(12) }` };
  },

  computed: {
    ariaDescribedBy(): string | undefined {
      const inheritedDescribedBy = this.$attrs['aria-describedby'];
      const internalDescribedBy = this.descriptionKey || this.description ? this.describedById : undefined;

      if (inheritedDescribedBy && internalDescribedBy) {
        return `${ inheritedDescribedBy } ${ internalDescribedBy }`;
      } else if (inheritedDescribedBy || internalDescribedBy) {
        return `${ inheritedDescribedBy || internalDescribedBy }`;
      }

      return undefined;
    },
    /**
     * Determines if the checkbox is disabled.
     * @returns boolean: True when the disabled prop is true or when mode is
     * View.
     */
    isDisabled(): boolean {
      return (this.disabled || this.mode === _VIEW);
    },
    /**
     * Determines if the checkbox is checked when using custom values or
     * multiple values.
     * @returns boolean: True when at least one value is true in a collection or
     * when value matches `this.valueWhenTrue`.
     */
    isChecked(): boolean {
      return this.isMulti(this.value) ? this.findTrueValues(this.value) : this.value === this.valueWhenTrue;
    },

    /**
     * Determines if the Labeled Input should display a tooltip.
     */
    hasTooltip(): boolean {
      return !!this.tooltip || !!this.tooltipKey;
    },

    replacementLabel(): string | undefined {
      if (!this.label && !this.labelKey && this.alternateLabel) {
        return this.alternateLabel;
      }

      return undefined;
    },

    idForLabel():string {
      return `${ generateRandomAlphaString(12) }-checkbox-label`;
    }
  },

  methods: {
    /**
     * Toggles the checked state for the checkbox and emits an 'input' event.
     */
    clicked(event: MouseEvent | KeyboardEvent): boolean | void {
      if ((event.target as HTMLLinkElement).tagName === 'A' && (event.target as HTMLLinkElement).href) {
        // Ignore links inside the checkbox label so you can click them
        return true;
      }

      event.stopPropagation();
      event.preventDefault();

      if (this.isDisabled) {
        return;
      }

      const customEvent = {
        bubbles:    true,
        cancelable: false,
        shiftKey:   event.shiftKey,
        altKey:     event.altKey,
        ctrlKey:    event.ctrlKey,
        metaKey:    event.metaKey
      };

      const click = new CustomEvent('click', customEvent);

      // Flip the value
      const value = cloneDeep(this.value);

      if (this.isMulti(value)) {
        if (this.isChecked) {
          removeObject(value, this.valueWhenTrue);
        } else {
          addObject(value, this.valueWhenTrue);
        }
        this.$emit('update:value', value);
      } else if (this.isString(this.valueWhenTrue)) {
        if (this.isChecked) {
          this.$emit('update:value', null);
        } else {
          this.$emit('update:value', this.valueWhenTrue);
        }
      } else {
        this.$emit('update:value', !value);
        this.$el.dispatchEvent(click);
      }
    },

    /**
     * Determines if there are multiple values for the checkbox.
     */
    isMulti(value: boolean | boolean[] | string | string[]): value is boolean[] {
      return Array.isArray(value);
    },

    isString(value: boolean | number | string): value is boolean {
      return typeof value === 'string';
    },

    /**
     * Finds the first true value for multiple checkboxes.
     * @param value A collection of values for the checkbox.
     */
    findTrueValues(value: boolean[]): boolean {
      return value.find((v) => v === this.valueWhenTrue) || false;
    }
  }
});
</script>

<template>
  <div
    class="checkbox-outer-container"
    data-checkbox-ctrl
    :class="{
      'v-popper--has-tooltip': hasTooltip,
    }"
  >
    <label
      class="checkbox-container"
      :class="{ 'disabled': isDisabled}"
      @keydown.enter.prevent="clicked($event)"
      @keydown.space.prevent="clicked($event)"
      @click="clicked($event)"
    >
      <input
        :id="id"
        :checked="isChecked"
        :value="valueWhenTrue"
        type="checkbox"
        tabindex="-1"
        @click.stop.prevent
        @keyup.enter.stop.prevent
      >
      <span
        class="checkbox-custom"
        :class="{indeterminate: indeterminate}"
        :tabindex="isDisabled ? -1 : 0"
        :aria-disabled="isDisabled"
        :aria-label="replacementLabel"
        :aria-checked="!!value"
        :aria-labelledby="labelKey || label ? idForLabel : undefined"
        :aria-describedby="ariaDescribedBy"
        role="checkbox"
      />
      <span
        v-if="$slots.label || label || labelKey || hasTooltip"
        class="checkbox-label"
        :class="{ 'checkbox-primary': primary }"
      >
        <slot name="label">
          <span
            v-if="labelKey"
            :id="idForLabel"
          >
            <t
              :k="labelKey"
              :raw="true"
            />
          </span>
          <span
            v-else-if="label"
            :id="idForLabel"
          >{{ label }}</span>
          <i
            v-if="tooltipKey"
            v-clean-tooltip="{content: t(tooltipKey), triggers: ['hover', 'touch', 'focus']}"
            v-stripped-aria-label="t(tooltipKey)"
            class="checkbox-info icon icon-info icon-lg"
            :data-testid="componentTestid + '-info-icon'"
            :tabindex="isDisabled ? -1 : 0"
            role="tooltip"
          />
          <i
            v-else-if="tooltip"
            v-clean-tooltip="{content: tooltip, triggers: ['hover', 'touch', 'focus']}"
            v-stripped-aria-label="tooltip"
            class="checkbox-info icon icon-info icon-lg"
            :data-testid="componentTestid + '-info-icon'"
            :tabindex="isDisabled ? -1 : 0"
            role="tooltip"
          />
        </slot>
      </span>
    </label>
    <div
      v-if="descriptionKey || description"
      class="checkbox-outer-container-description"
    >
      <t
        v-if="descriptionKey"
        :id="describedById"
        :k="descriptionKey"
      />
      <template v-else-if="description">
        <p :id="describedById">
          {{ description }}
        </p>
      </template>
    </div>
    <div class="checkbox-outer-container-extra">
      <slot name="extra" />
    </div>
  </div>
</template>

<style lang='scss'>
$fontColor: var(--input-label);

.checkbox-outer-container {
  display: inline-flex;
  flex-direction: column;
  &-description {
    color: $fontColor;
    font-size: 14px;
    margin-left: 19px;
    margin-top: 5px;
    opacity: 0.8;
  }
  &-extra {
    font-size: 14px;
    margin-left: 19px;
    margin-top: 5px;
  }
}

// NOTE: SortableTable depends on the names of this class, do not arbitrarily change.
.checkbox-container {
  position: relative;
  display: inline-flex;
  align-items: center;
  margin: 0;
  cursor: pointer;
  user-select: none;
  border-radius: var(--border-radius);

  .checkbox-label {
    color: var(--input-label);
    display: inline-flex;
    margin: 0px 10px 0px 5px;

    &.checkbox-primary {
      color: inherit;
      font-weight: 600;
    }
  }

  .checkbox-info {
    line-height: normal;
    margin-left: 4px;

    &:focus-visible {
      @include focus-outline;
      outline-offset: 2px;
    }
  }

 .checkbox-custom {
    height: 14px;
    width: 14px;
    background-color: var(--body-bg);
    border-radius: var(--border-radius);
    border: 1px solid var(--border);
    flex-shrink: 0;

    &:focus-visible {
      @include focus-outline;
      outline-offset: 2px;
      border-radius: 0;
    }
  }

  input {
    // display: none;
    opacity: 0;
    position: absolute;
    z-index: -1;
  }

  input:focus-visible ~ .checkbox-custom {
    @include focus-outline;
    outline-offset: 2px;
    border-radius: 0;
  }

  input:checked ~ .checkbox-custom {
    background-color:var(--primary);
    -webkit-transform: rotate(0deg) scale(1);
    -ms-transform: rotate(0deg) scale(1);
    transform: rotate(0deg) scale(1);
    opacity:1;
    border: 1px solid var(--primary);
  }

  // Custom Checkbox tick
  .checkbox-custom::after {
    position: absolute;
    content: "";
    left: 0px;
    top: 0px;
    height: 0px;
    width: 0px;
    border-radius: var(--border-radius);
    border: solid;
    border-color: var(--input-text);
    border-width: 0 3px 3px 0;
    -webkit-transform: rotate(0deg) scale(0);
    -ms-transform: rotate(0deg) scale(0);
    transform: rotate(0deg) scale(0);
    opacity:1;
  }

  input:checked ~ .checkbox-custom::after {
    -webkit-transform: rotate(45deg) scale(1);
    -ms-transform: rotate(45deg) scale(1);
    transform: rotate(45deg) scale(1);
    opacity:1;
    left: 4px;
    width: 4px;
    height: 10px;
    border: solid;
    border-color: var(--checkbox-tick);
    border-width: 0 2px 2px 0;
    background-color: transparent;
  }

  input:checked ~ .checkbox-custom.indeterminate::after {
    -webkit-transform:  scale(1);
    -ms-transform:  scale(1);
    transform:  scale(1);
    opacity:1;
    left: 3px;
    top:2px;
    width: 6px;
    height: 5px;
    border: solid;
    border-color: var(--checkbox-tick);
    border-width: 0 0 2px 0;
    background-color: transparent;
  }

  // Disabled styles
  &.disabled {
    .checkbox-custom {
      background-color: var(--checkbox-disabled-bg);
      border-color: var(--checkbox-disabled-bg);
    }
    input:checked ~ .checkbox-custom {
      background-color: var(--checkbox-disabled-bg);
      border-color: var(--checkbox-disabled-bg);
      &::after {
        border-color: var(--checkbox-tick-disabled);
      }
    }
  }

  &.disabled {
    cursor: not-allowed;
  }

  .checkbox-view {
    display: flex;
    flex-direction: column;
    LABEL {
      color: $fontColor;
    }
  }
}
</style>
