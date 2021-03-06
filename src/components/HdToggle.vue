<template>
  <div
    :class="{
      'hd-toggle--is-open': open,
      'hd-toggle--is-using-mouse': isUsingMouse,
    }"
    class="hd-toggle"
    @keydown="setUsingMouse(false)"
    @mousedown="setUsingMouse(true)"
  >
    <button
      :disabled="!canBeToggled"
      class="hd-toggle__control"
      type="button"
      @click="toggleOpen"
    >
      {{ title }}

      <HdIcon
        :src="chevronIcon"
        class="hd-toggle__control-icon"
      />
    </button>
    <div
      ref="body"
      :style="{
        maxHeight: `${maxHeight}px`,
        transitionDuration: `${transitionDuration}ms`,
      }"
      class="hd-toggle__body"
    >
      <slot />
    </div>
  </div>
</template>

<script>
import { OnResizeService } from 'homeday-blocks';
import HdIcon from 'homeday-blocks/src/components/HdIcon.vue';
import { chevron as chevronIcon } from 'homeday-assets';

export const TABINDEX_BACKUP_ATTRIBUTE = 'data-hd-tabindex-backup';

export default {
  name: 'HdToggle',
  components: {
    HdIcon,
  },
  props: {
    title: {
      type: String,
      default: '',
    },
    canBeToggled: {
      type: Boolean,
      default: true,
    },
    open: {
      type: Boolean,
      default: true,
    },
    transitionDuration: {
      type: Number,
      default: 300,
    },
  },
  data() {
    return {
      // bodyHeight is a random big number only initially,
      // it will be calculated on mounted
      bodyHeight: 10000,
      isUsingMouse: false,
      internalFocusRemoved: false,
      chevronIcon,
    };
  },
  computed: {
    maxHeight() {
      if (this.open) {
        return this.bodyHeight;
      }

      return 0;
    },
  },
  watch: {
    open: {
      handler(open) {
        this.ensureBodyScrolledToTop();

        this.$nextTick(() => {
          if (open) {
            this.enableInternalFocus();
          } else {
            this.disableInternalFocus();
          }
        });
      },
      immediate: true,
    },
  },
  mounted() {
    this.$nextTick(this.calculateBodyHeight);
    this.addResizeEvents();
  },
  beforeDestroy() {
    this.removeResizeEvents();
  },
  methods: {
    // Encapsulated in a method to be able to mock it in the tests
    getScrollHeight(el) {
      return el.scrollHeight;
    },
    toggleOpen() {
      if (this.canBeToggled === false) {
        return;
      }

      this.$emit('toggle', !this.open);
    },
    calculateBodyHeight() {
      if (!this.$refs.body) {
        return;
      }

      this.bodyHeight = this.getScrollHeight(this.$refs.body);
    },
    ensureBodyScrolledToTop() {
      if (!this.$refs.body) {
        return;
      }

      this.$refs.body.scrollTop = 0;
    },
    disableInternalFocus() {
      if (!this.$refs.body) {
        return;
      }

      const $innerFocusableElements = this.$refs.body.querySelectorAll(
        'a, button, [tabindex]',
      );

      if (!$innerFocusableElements.length) {
        return;
      }

      $innerFocusableElements.forEach((el) => {
        const currentTabIndex = el.getAttribute('tabindex');

        if (currentTabIndex !== null) {
          el.setAttribute(TABINDEX_BACKUP_ATTRIBUTE, currentTabIndex);
        }

        el.setAttribute('tabindex', '-1');
      });

      this.internalFocusRemoved = true;
    },
    enableInternalFocus() {
      if (!this.$refs.body || this.internalFocusRemoved === false) {
        return;
      }

      const $innerFocusableElements = this.$refs.body.querySelectorAll(
        `[${TABINDEX_BACKUP_ATTRIBUTE}]`,
      );

      if (!$innerFocusableElements.length) {
        return;
      }

      $innerFocusableElements.forEach((el) => {
        const oldTabIndex = el.getAttribute(TABINDEX_BACKUP_ATTRIBUTE);

        el.setAttribute('tabindex', oldTabIndex);
        el.removeAttribute(TABINDEX_BACKUP_ATTRIBUTE);
      });
    },
    addResizeEvents() {
      OnResizeService.onDebounced(this.calculateBodyHeight);
    },
    removeResizeEvents() {
      OnResizeService.offDebounced(this.calculateBodyHeight);
    },
    setUsingMouse(usingMouse) {
      this.isUsingMouse = usingMouse;
    },
  },
};
</script>

<style lang="scss" scoped>
@import 'homeday-blocks/src/styles/mixins.scss';

$_controlIconSize: 32px;

.hd-toggle {
  $_root: &;

  &__control {
    display: block;
    position: relative;
    width: 100%;
    min-height: $_controlIconSize;
    padding: 0 $_controlIconSize 0 0;
    margin: 0;
    border: 0;
    background-color: transparent;
    box-shadow: none;
    @include font('subtitle');
    text-align: left;
    transition: outline $time-s ease-in-out;

    @media (min-width: $break-tablet) {
      @include font('title');
    }

    &:not([disabled]) {
      cursor: pointer;
    }

    #{$_root}--is-using-mouse & {
      outline: 0;
    }
  }

  &__control-icon {
    display: block;
    position: absolute;
    top: 0;
    right: 0;
    width: $_controlIconSize;
    height: $_controlIconSize;
    transform: rotate(90deg);
    transition: transform ($time-s * 2) ease-in-out;

    #{$_root}--is-open & {
      transform: rotate(-90deg);
    }
  }

  &__body {
    overflow: hidden;
    transition-property: max-height;
    transition-timing-function: linear;
  }
}
</style>
