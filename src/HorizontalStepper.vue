<template>
    <div class="stepper-box">
        <div class="top">
            <div class="divider-line" :style="{width: `${(100/(steps.length) * (steps.length - 1)) - 10}%`}"></div>
            <div class="steps-wrapper">
                <template v-if="topButtons">
                    <div v-if="currentStep.index > 0" class="stepper-button-top previous" @click="backStep()">
                        <font-awesome-icon class="icon" :icon="iconBack" />
                    </div>
                </template>
                <template v-for="(step, index) in steps">
                    <div :class="['step', isStepActive(index, step)]" :key="index" :style="{width: `${100 / steps.length}%`}">
                        <div class="circle">
                            <font-awesome-icon class="icon" :icon="step.completed ? iconCheck : step.icon" />
                        </div>
                        <div class="step-title">
                            <h6>{{step.title}}</h6>
                            <p class="step-subtitle">{{step.subtitle}}</p>
                        </div>
                    </div>
                </template>
                <div v-if="topButtons" :class="['stepper-button-top next', !canContinue ? 'deactivated' : '']" @click="nextStep()">
                    <font-awesome-icon class="icon" :icon="iconNext" />
                </div>
            </div>
        </div>
        <div class="content">
            <transition :enter-active-class="enterAnimation" :leave-active-class="leaveAnimation" mode="out-in">
                <!--If keep alive-->
                <keep-alive v-if="keepAliveData">
                    <component :is="steps[currentStep.index].component" :clickedNext="nextButton[currentStep.name]" @can-continue="proceed" @change-next="changeNextBtnValue" :current-step="currentStep"></component>
                </keep-alive>
                <!--If not show component and destroy it in each step change-->
                <component v-else :is="steps[currentStep.index].component" :clickedNext="nextButton[currentStep.name]" @can-continue="proceed" @change-next="changeNextBtnValue" :current-step="currentStep"></component>
            </transition>
        </div>
        <div v-if="!hideFooter" :class="['bottom', (currentStep.index > 0) ? '' : 'only-next']">
            <div v-if="currentStep.index > 0" class="stepper-button previous" @click="backStep()">
                <font-awesome-icon class="icon" :icon="iconBack" />
                <span class="pl">{{ 'back' | translate(locale) }}</span>
            </div>
            <div :class="['stepper-button next', !canContinue ? 'deactivated' : '']" @click="nextStep()">
                <span class="pr">{{ (finalStep) ? 'finish' : 'next' | translate(locale) }}</span>
                <font-awesome-icon class="icon" :icon="iconNext" />
            </div>
        </div>
    </div>
</template>

<script>
import translations from "./Translations.js";
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
import { faAngleLeft } from '@fortawesome/free-solid-svg-icons'
import { faAngleRight } from '@fortawesome/free-solid-svg-icons'
import { faCheck } from '@fortawesome/free-solid-svg-icons'
import { faInfo } from '@fortawesome/free-solid-svg-icons'
import { faFile } from '@fortawesome/free-solid-svg-icons'

export default {
  filters: {
    translate: function(value, locale) {
      return translations[locale][value];
    }
  },

  components: {
    FontAwesomeIcon
  },

  props: {
    locale: {
      type: String,
      default: "en"
    },
    topButtons: {
      type: Boolean,
      default: false
    },
    steps: {
      type: Array,
      default: function() {
        return [
          {
            icon: faFile,
            name: "first",
            title: "Sample title 1",
            subtitle: "Subtitle sample"
          },
          {
            icon: faInfo,
            name: "second",
            title: "Sample title 2",
            subtitle: "Subtitle sample"
          }
        ];
      }
    },
    keepAlive: {
      type: Boolean,
      default: true
    },
    reset: {
      type: Boolean,
      default: false
    },
    hideFooter: {
      type: Boolean,
      default: false
    },
    initialStep: {
      type: Number,
      default: 0
    }
  },

  data() {
    return {
      currentStep: {},
      previousStep: {},
      nextButton: {},
      canContinue: false,
      finalStep: false,
      keepAliveData: this.keepAlive,
      iconNext: faAngleRight,
      iconBack: faAngleLeft,
      iconCheck: faCheck
    };
  },

  computed: {
    enterAnimation() {
      if (this.currentStep.index < this.previousStep.index) {
        return "animated quick fadeInLeft";
      } else {
        return "animated quick fadeInRight";
      }
    },
    leaveAnimation() {
      if (this.currentStep.index > this.previousStep.index) {
        return "animated quick fadeOutLeft";
      } else {
        return "animated quick fadeOutRight";
      }
    }
  },

  methods: {
    isStepActive(index, step) {
      if (this.currentStep.index === index) {
        return "activated";
      } else {
        return "deactivated";
      }
    },

    activateStep(index, back = false) {
      if (this.steps[index]) {
        this.previousStep = this.currentStep;
        this.currentStep = {
          name: this.steps[index].name,
          index: index
        };

        if (index + 1 === this.steps.length) {
          this.finalStep = true;
        } else {
          this.finalStep = false;
        }

        if (!back) {
          this.$emit("completed-step", this.previousStep);
        }
      }
      this.$emit("active-step", this.currentStep);
    },

    nextStepAction() {
      this.nextButton[this.currentStep.name] = true;
      if (this.canContinue) {
        if (this.finalStep) {
          this.$emit("stepper-finished", this.currentStep);
        }
        let currentIndex = this.currentStep.index + 1;

        this.activateStep(currentIndex);
      }
      this.canContinue = false;
      this.$forceUpdate();
    },

    nextStep () {

      if (!this.$listeners || !this.$listeners['before-next-step']) {
        this.nextStepAction()
      }

      this.canContinue = false;

      this.$emit("before-next-step", { currentStep: this.currentStep }, (next = true) => {
        this.canContinue = true;
        if (next) {
          this.nextStepAction()
        }
      });
    },
    backStep() {
      this.$emit("clicking-back");
      let currentIndex = this.currentStep.index - 1;
      if (currentIndex >= 0) {
        this.activateStep(currentIndex, true);
      }
    },

    proceed(payload) {
      this.canContinue = payload.value;
    },

    changeNextBtnValue(payload) {
      this.nextButton[this.currentStep.name] = payload.nextBtnValue;
      this.$forceUpdate();
    },

    init() {
      // Initiate stepper
      if (this.initialStep >= 0 && this.initialStep < this.steps.length) {
        this.activateStep(this.initialStep);
      } else {
        this.activateStep(0);
      }

      this.steps.forEach(step => {
        this.nextButton[step.name] = false;
      });
    }
  },

  watch: {
    reset(val) {
      if(!val) {
        return;
      }

      this.keepAliveData = false;

      this.init();
      this.previousStep = {};

      this.$nextTick(() => {
        this.keepAliveData = this.keepAlive;
        this.$emit('reset', true);
      });

    }
  },

  beforeMount() {
    this.init();
  }
};
</script>

<style src="./HorizontalStepper.scss" scoped lang="scss">

</style>
<style scoped>
.pr {
  padding-right: 0.25rem;
}
.pl {
  padding-left: 0.25rem;
}
.icon {
  font-family: "FontAwesome";
  font-weight: normal;
  font-style: normal;
  font-size: 16px;
  line-height: 1;
  letter-spacing: normal;
  text-transform: none;
  display: inline-block;
  white-space: nowrap;
  word-wrap: normal;
  direction: ltr;
  -webkit-font-feature-settings: "liga";
  -webkit-font-smoothing: antialiased;
}
</style>
