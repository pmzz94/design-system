<template>
  <div
    ref="guidedTourPortal"
    class="guided-tour-portal"
    :class="{ centeredTooltip }"
    :style="`z-index: ${zIndex}`"
    v-show="showGuidedTour"
  >
    <div v-show="showSpotlight" ref="spotlight" class="spotlight">
      <!-- Spotlight div -->
    </div>
    <div
      v-if="currentStep"
      ref="tooltip"
      class="tooltip"
      :style="{ opacity: showTooltip ? 1 : 0 }"
    >
      <div class="tooltip__progress-bar">
        <div
          class="tooltip__progress-bar__step"
          v-for="(step, index) of steps"
          :key="index"
          :style="{
            width: `calc(100% / ${steps.length})`,
            backgroundColor:
              steps.indexOf(currentStep) >= index
                ? 'var(--color-secondary)'
                : '',
          }"
        ></div>
      </div>
      <div class="tooltip__box">
        <div class="tooltip__box__header">
          <template v-if="!isStepIntro && !isStepOutro">
            <BIMDataButton
              class="tooltip__box__header__btn-close"
              width="0px"
              height="0px"
              ghost
              rounded
              icon
              @click="close"
            >
              <BIMDataIcon name="close" size="xxs" />
            </BIMDataButton>
          </template>
        </div>
        <div class="tooltip__box__content">
          <template v-if="currentStep.layout">
            <component :is="currentStep.layout" v-bind="currentStep.props" />
          </template>
          <template v-else>
            <div class="tooltip__box__content__title">
              {{ currentStep.props.title }}
            </div>
            <div
              class="tooltip__box__content__image"
              :style="{
                background: `var(--color-silver-light) url(${currentStep.props.img}) no-repeat ${currentStep.props.imgPosition} / ${currentStep.props.imgSize}`,
              }"
            />
            <div class="tooltip__box__content__text">
              {{ currentStep.props.content }}
            </div>
          </template>
        </div>
        <div class="tooltip__box__footer">
          <template v-if="isStepIntro">
            <div class="tooltip__box__footer__btn-skip">
              <BIMDataButton
                width="0px"
                height="0px"
                color="granite"
                @click="close"
              >
                Skip
              </BIMDataButton>
            </div>
          </template>
          <template v-else>
            <div class="tooltip__box__footer__btn-prev">
              <BIMDataButton
                width="0px"
                height="0px"
                color="primary"
                fill
                radius
                @click="clickPrev"
              >
                <BIMDataIcon
                  name="chevron"
                  size="xxs"
                  fill
                  color="white"
                  :rotate="180"
                />
              </BIMDataButton>
            </div>
          </template>
          <div class="tooltip__box__footer__step-counter">
            <span>{{ stepIndex + 1 }}</span>
            <span>/{{ steps.length }}</span>
          </div>
          <template v-if="isStepOutro">
            <div class="tooltip__box__footer__btn-start">
              <BIMDataButton
                width="0px"
                height="0px"
                color="granite"
                @click="close"
              >
                Commencer
              </BIMDataButton>
            </div>
          </template>
          <template v-else>
            <div class="tooltip__box__footer__btn-next">
              <BIMDataButton
                width="0px"
                height="0px"
                color="primary"
                fill
                radius
                @click="clickNext"
              >
                <BIMDataIcon name="chevron" size="xxs" fill color="white" />
              </BIMDataButton>
            </div>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {
  scrollToTarget,
  setSpotlightPosition,
  setTooltipPosition,
} from "./guided-tour-utils.js";

import BIMDataButton from "../../BIMDataComponents/BIMDataButton/BIMDataButton.vue";
import BIMDataIcon from "../../BIMDataComponents/BIMDataIcon/BIMDataIcon.vue";

export default {
  components: {
    BIMDataButton,
    BIMDataIcon,
  },
  props: {
    tours: {
      type: Array,
      default: () => [],
    },
    tourToDisplay: {
      type: String,
      default: () => "",
    },
    elementToObserve: {
      type: [Object, HTMLElement],
      default: () => {},
    },
    zIndex: {
      type: Number,
      default: () => 10000,
    },
    router: {
      type: Object,
    },
  },
  emits: ["set-completed-tour"],
  data() {
    return {
      steps: [],
      observer: null,
      showGuidedTour: false,
      movingForward: true,
      currentTarget: null,
      showSpotlight: true,
      showTooltip: false,
      stepIndex: 0,
      console,
    };
  },
  computed: {
    currentStep() {
      return this.steps[this.stepIndex];
    },
    nextStep() {
      return this.steps[this.stepIndex + 1];
    },
    prevStep() {
      return this.steps[this.stepIndex - 1];
    },
    centeredTooltip() {
      return this.currentStep && !this.currentStep.target;
    },
    isStepIntro() {
      return this.stepIndex === 0;
    },
    isStepOutro() {
      return this.stepIndex === this.steps.length - 1;
    },
  },
  watch: {
    steps() {
      this.stepIndex = 0;
    },
    async currentStep(step) {
      try {
        if (!step) return;

        if (step.target) {
          this.currentTarget = this.getDomElements(step);
        } else {
          // display a centered tooltip
          this.showSpotlight = true;
          this.showTooltip = true;
          return;
        }

        if (step.clickable) {
          (
            this.currentTarget.elementToClick || this.currentTarget.element
          ).addEventListener(
            "click",
            () => {
              if (this.nextStep.target) {
                this.mutationObserverManager();
              } else {
                this.next();
              }
            },
            { once: true }
          );
        }

        scrollToTarget(this.currentTarget.element, this.elementToObserve);
        setSpotlightPosition(this.currentTarget.element, this.$refs.spotlight);
        setTooltipPosition(this.currentTarget.element, this.$refs.tooltip);

        this.showSpotlight = true;
        this.showTooltip = true;
      } catch {
        this.closeGuidedTour();
      }
    },
  },
  mounted() {
    const tour = this.tours.find(t => t.name === this.tourToDisplay);
    if (tour) {
      this.openGuidedTour(tour.steps);
    } else {
      console.warn(`GuideTour: unknown tour ${this.tourToDisplay}`);
    }
  },
  unmounted() {
    this.mutationObserver.disconnect();
  },
  destroyed() {
    this.mutationObserver.disconnect();
  },
  methods: {
    clickNext() {
      if (this.currentStep.clickable) {
        (
          this.currentTarget.elementToClick || this.currentTarget.element
        ).click();
      } else {
        this.next();
      }
      this.resetSettings();
    },
    clickPrev() {
      this.movingForward = false;
      if (this.prevStep.clickable) {
        if (this.prevStep.routeName) {
          // eslint-disable-next-line vue/no-mutating-props
          this.router.push({ name: this.prevStep.routeName });
          this.mutationObserverManager();
        } else {
          const { elementToClick, element } = this.getDomElements(
            this.prevStep
          );
          (elementToClick || element).addEventListener(
            "click",
            () => {
              this.mutationObserverManager(this.prevStep, this.prev);
            },
            { once: true }
          );
          (elementToClick || element).click();
        }
      } else {
        this.prev();
      }
      this.resetSettings();
    },
    openGuidedTour(arg) {
      this.steps = arg.map(step => {
        return {
          ...step,
          layout: step.layout ? Object.freeze(step.layout) : null,
        };
      });
      this.showGuidedTour = true;
    },
    getDomElements(step, elementToWatch = document) {
      const {
        target,
        targetDetail,
        targetToClick,
        targetToClickDetail,
        targetToClickBack,
        targetToClickBackDetail,
      } = step;

      let element, elementToClick;

      if (Array.isArray(target)) {
        element = target.map(t =>
          elementToWatch.querySelector(`[data-guide=${t}]`)
        );
      } else if (typeof target === "string") {
        element = elementToWatch.querySelector(
          `[data-guide=${target}] ${targetDetail ? targetDetail : ""}`
        );
      }
      if (
        targetToClick ||
        (!this.movingForward && (targetToClickBack || targetToClickBackDetail))
      ) {
        elementToClick = elementToWatch.querySelector(
          `[data-guide-click=${targetToClickBack || targetToClick}] ${
            targetToClickBackDetail
              ? targetToClickBackDetail
              : targetToClickDetail
              ? targetToClickDetail
              : ""
          }`
        );
      }
      console.log("element", element);
      console.log("elementToClick", elementToClick);

      return {
        element,
        elementToClick,
      };
    },
    closeGuidedTour() {
      this.showGuidedTour = false;
      this.steps = [];
    },
    next() {
      this.stepIndex++;
    },
    prev() {
      this.stepIndex--;
    },
    close() {
      this.showTooltip = false;
      this.closeGuidedTour();
      this.$emit("set-completed-tour", this.tourToDisplay);
    },
    resetSettings() {
      this.movingForward = true;
      this.currentTarget = null;

      this.showSpotlight = false;
      this.showTooltip = false;

      this.$refs.tooltip.style.removeProperty("left");
      this.$refs.tooltip.style.removeProperty("top");

      this.$refs.spotlight.style.removeProperty("height");
      this.$refs.spotlight.style.removeProperty("width");
      this.$refs.spotlight.style.removeProperty("left");
      this.$refs.spotlight.style.removeProperty("top");
    },
    mutationObserverManager() {
      const stepToReach = this.movingForward ? this.nextStep : this.prevStep;
      const goToStep = this.movingForward ? this.next : this.prev;

      this.observer = new MutationObserver(() =>
        this.domToDisplay(stepToReach, goToStep)
      );
      this.observer.observe(this.elementToObserve, {
        childList: true,
        subtree: true,
      });
    },

    domToDisplay(stepToReach, goToStep) {
      const { element, elementToClick } = this.getDomElements(
        stepToReach,
        this.elementToObserve
      );

      console.log("elementToClick in domToDisplay", elementToClick);

      const isAnHTMLElement = elem => elem instanceof HTMLElement;

      const isAnArrayOfHTMLElement = elem =>
        Array.isArray(elem) && element.every(e => e instanceof HTMLElement);

      if (isAnHTMLElement(element) || isAnArrayOfHTMLElement(element)) {
        goToStep();
        this.observer.disconnect();
      }
    },
  },
};
</script>

<style scoped lang="scss" src="./_BIMDataGuidedTour.scss"></style>
