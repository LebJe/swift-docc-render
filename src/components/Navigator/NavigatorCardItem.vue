<!--
  This source file is part of the Swift.org open source project

  Copyright (c) 2022-2023 Apple Inc. and the Swift project authors
  Licensed under Apache License v2.0 with Runtime Library Exception

  See https://swift.org/LICENSE.txt for license information
  See https://swift.org/CONTRIBUTORS.txt for Swift project authors
-->

<template>
  <BaseNavigatorCardItem
    :class="{ expanded, active: isActive, 'is-group': isGroupMarker }"
    :style="{ '--nesting-index': item.depth }"
    :data-nesting-index="item.depth"
    :id="`container-${item.uid}`"
    :aria-hidden="isRendered ? null : 'true'"
    :hideNavigatorIcon="isGroupMarker"
    @keydown.left.native.prevent="handleLeftKeydown"
    @keydown.right.exact.native.prevent="handleRightKeydown"
    @keydown.enter.native.prevent="clickReference"
    @keydown.alt.right.native.prevent="toggleEntireTree"
  >
    <template #depth-spacer>
      <span
        hidden
        :id="usageLabel"
      >
        {{ $t('filter.navigate') }}
      </span>
      <button
        v-if="isParent"
        class="tree-toggle"
        tabindex="-1"
        :aria-labelledby="item.uid"
        :aria-expanded="expanded ? 'true': 'false'"
        :aria-describedby="ariaDescribedBy"
        @click.exact.prevent="toggleTree"
        @click.alt.prevent="toggleEntireTree"
        @click.meta.prevent="toggleSiblings"
      >
        <InlineChevronRightIcon
          class="icon-inline chevron"
          :class="{ rotate: expanded, animating: idState.isOpening }"
        />
      </button>
    </template>
    <template #navigator-icon="{ className }">
      <TopicTypeIcon
        v-if="!apiChange"
        :key="item.uid"
        :type="item.type"
        :image-override="item.icon ? navigatorReferences[item.icon] : null"
        :shouldCalculateOptimalWidth="false"
        :class="className"
      />
      <span
        v-else
        :class="[{ [`changed changed-${apiChange}`]: apiChange }, className ]"
      />
    </template>
    <template #title-container>
      <span
        v-if="isParent"
        hidden
        :id="parentLabel"
      >{{ $tc(
        'filter.parent-label',
        item.childUIDs.length,
        {
          'number-siblings': item.index + 1,
          'total-siblings': item.siblingsCount,
          'parent-siblings': item.parent,
          'number-parent': item.childUIDs.length
        }
      ) }}</span>
      <span
        v-if="!isParent"
        :id="siblingsLabel"
        hidden
      >
        {{ $t('filter.siblings-label', {
          'number-siblings': item.index + 1,
          'total-siblings': item.siblingsCount,
          'parent-siblings': item.parent
        }) }}
      </span>
      <component
        :is="refComponent"
        :id="item.uid"
        :class="{ bolded: isBold }"
        :url="isGroupMarker ? null : (item.path || '')"
        :tabindex="isFocused ? '0' : '-1'"
        :aria-describedby="`${ariaDescribedBy} ${usageLabel}`"
        class="leaf-link"
        ref="reference"
        @click.exact.native="handleClick"
        @click.alt.native.prevent="toggleEntireTree"
      >
        <HighlightMatches
          :text="item.title"
          :matcher="filterPattern"
        />
      </component>
      <Badge v-if="isDeprecated" variant="deprecated" />
      <Badge v-else-if="isBeta" variant="beta" />
    </template>
  </BaseNavigatorCardItem>
</template>

<script>
import InlineChevronRightIcon from 'theme/components/Icons/InlineChevronRightIcon.vue';
import BaseNavigatorCardItem from 'docc-render/components/Navigator/BaseNavigatorCardItem.vue';
import TopicTypeIcon from 'docc-render/components/TopicTypeIcon.vue';
import HighlightMatches from 'docc-render/components/Navigator/HighlightMatches.vue';
import Reference from 'docc-render/components/ContentNode/Reference.vue';
import Badge from 'docc-render/components/Badge.vue';
import { TopicTypes } from 'docc-render/constants/TopicTypes';
import { ChangeTypesOrder } from 'docc-render/constants/Changes';
import { IdState } from 'vue-virtual-scroller';
import { waitFrames } from 'docc-render/utils/loading';

export default {
  name: 'NavigatorCardItem',
  mixins: [
    IdState({
      idProp: vm => vm.item.uid,
    }),
  ],
  components: {
    BaseNavigatorCardItem,
    HighlightMatches,
    TopicTypeIcon,
    InlineChevronRightIcon,
    Reference,
    Badge,
  },
  props: {
    isRendered: {
      type: Boolean,
      default: false,
    },
    item: {
      type: Object,
      required: true,
    },
    expanded: {
      type: Boolean,
      default: false,
    },
    filterPattern: {
      type: RegExp,
      default: undefined,
    },
    isActive: {
      type: Boolean,
      default: false,
    },
    isBold: {
      type: Boolean,
      default: false,
    },
    apiChange: {
      type: String,
      default: null,
      validator: v => ChangeTypesOrder.includes(v),
    },
    isFocused: {
      type: Boolean,
      default: () => false,
    },
    enableFocus: {
      type: Boolean,
      default: true,
    },
    navigatorReferences: {
      type: Object,
      default: () => ({}),
    },
  },
  idState() {
    return {
      // special state to track opening animations for a few seconds, after toggling on/off
      isOpening: false,
    };
  },
  computed: {
    isGroupMarker: ({ item: { type } }) => type === TopicTypes.groupMarker,
    isParent: ({ item, isGroupMarker }) => !!item.childUIDs.length && !isGroupMarker,
    parentLabel: ({ item }) => `label-parent-${item.uid}`,
    siblingsLabel: ({ item }) => `label-${item.uid}`,
    usageLabel: ({ item }) => `usage-${item.uid}`,
    ariaDescribedBy: ({
      isParent, parentLabel, siblingsLabel,
    }) => (
      isParent ? `${parentLabel}` : `${siblingsLabel}`
    ),
    isBeta: ({ item: { beta } }) => !!beta,
    isDeprecated: ({ item: { deprecated } }) => !!deprecated,
    refComponent: ({ isGroupMarker }) => (isGroupMarker ? 'h3' : Reference),
  },
  methods: {
    toggleTree() {
      this.idState.isOpening = true;
      this.$emit('toggle', this.item);
    },
    toggleEntireTree() {
      this.idState.isOpening = true;
      this.$emit('toggle-full', this.item);
    },
    toggleSiblings() {
      this.idState.isOpening = true;
      this.$emit('toggle-siblings', this.item);
    },
    handleLeftKeydown() {
      // if not expanded, go to the nearest parent
      if (!this.expanded) {
        this.$emit('focus-parent', this.item);
        return;
      }
      // close the tree otherwise
      this.toggleTree();
    },
    handleRightKeydown() {
      // if we are already expanded, dont do anything
      if (this.expanded || !this.isParent) return;
      // otherwise expand
      this.toggleTree();
    },
    clickReference() {
      (this.$refs.reference.$el || this.$refs.reference).click();
    },
    focusReference() {
      (this.$refs.reference.$el || this.$refs.reference).focus();
    },
    handleClick() {
      if (this.isGroupMarker) return;
      this.$emit('navigate', this.item.uid);
    },
  },
  watch: {
    async isFocused(newVal) {
      await waitFrames(8);
      if (newVal && this.isRendered && this.enableFocus) {
        this.focusReference();
      }
    },
    async expanded() {
      // wait for 9 frames (60hz * 0.15ms = 9), for animations queues to finish.
      await waitFrames(9);
      // set the opening animation as ended
      this.idState.isOpening = false;
    },
  },
};
</script>

<style scoped lang='scss'>
@import 'docc-render/styles/_core.scss';

$tree-toggle-padding: $nav-card-horizontal-spacing-small;
$chevron-width: $nav-card-horizontal-spacing;

.is-group {
  .leaf-link {
    color: var(--color-figure-gray-secondary);
    font-weight: $font-weight-semibold;
    // groups dont need the overlay
    &:after {
      display: none;
    }
  }
}

.navigator-icon {
  display: flex;
  flex: 0 0 auto;

  &.changed {
    border: none;
    width: 1em;
    height: 1em;
    z-index: 0;

    &:after {
      top: 50%;
      left: 50%;
      right: auto;
      bottom: auto;
      transform: translate(-50%, -50%);
      background-image: $modified-svg;

      @include prefers-dark {
        background-image: $modified-dark-svg;
      }
      margin: 0;
    }

    &-added::after {
      background-image: $added-svg;

      @include prefers-dark {
        background-image: $added-dark-svg;
      }
    }

    &-deprecated::after {
      background-image: $deprecated-svg;

      @include prefers-dark {
        background-image: $deprecated-dark-svg;
      }
    }
  }
}

.leaf-link {
  color: var(--color-figure-gray);
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
  max-width: 100%;
  display: inline;
  vertical-align: middle;
  @include font-styles(body-reduced-tight);

  @include on-keyboard-focus {
    outline: none;
  }

  &:hover {
    text-decoration: none;
  }

  &.bolded {
    font-weight: $font-weight-semibold;
  }

  &:after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
  }
}

.extended-content {
  @include font-styles(body-reduced);
  color: var(--color-figure-gray-secondary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.tree-toggle {
  overflow: hidden;
  position: absolute;
  width: 100%;
  height: 100%;
  padding-right: $tree-toggle-padding;
  box-sizing: border-box;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: flex-end;
}

.chevron {
  width: $chevron-width;

  &.animating {
    transition: transform 0.15s ease-in;
  }

  &.rotate {
    transform: rotate(90deg);
  }
}
</style>
