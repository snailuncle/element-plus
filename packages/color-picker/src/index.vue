<template>
  <div v-click-outside="hide" style="display: inline-block;">
    <el-popper
      ref="popper"
      v-model:visible="showPicker"
      effect="light"
      manual-mode
      trigger="click"
      :show-arrow="false"
      popper-class="el-color-picker__panel el-color-dropdown"
      :stop-popper-mouse-event="false"
      @after-leave="doDestroy"
    >
      <template #default>
        <div class="el-color-dropdown__main-wrapper">
          <hue-slider
            ref="hue"
            class="hue-slider"
            :color="color"
            vertical
          />
          <sv-panel ref="svPanel" :color="color" />
        </div>
        <alpha-slider v-if="showAlpha" ref="alpha" :color="color" />
        <predefine
          v-if="predefine"
          ref="predefine"
          :color="color"
          :colors="predefine"
        />
        <div class="el-color-dropdown__btns">
          <span class="el-color-dropdown__value">
            <el-input
              v-model="customInput"
              :validate-event="false"
              size="mini"
              @keyup.enter="handleConfirm"
              @blur="handleConfirm"
            />
          </span>
          <el-button
            size="mini"
            type="text"
            class="el-color-dropdown__link-btn"
            @click="clear"
          >
            {{ t('el.colorpicker.clear') }}
          </el-button>
          <el-button
            plain
            size="mini"
            class="el-color-dropdown__btn"
            @click="confirmValue"
          >
            {{ t('el.colorpicker.confirm') }}
          </el-button>
        </div>
      </template>
      <template #trigger>
        <div
          :class="[
            'el-color-picker',
            colorDisabled ? 'is-disabled' : '',
            colorSize ? `el-color-picker--${ colorSize }` : ''
          ]"
        >
          <div v-if="colorDisabled" class="el-color-picker__mask"></div>
          <div class="el-color-picker__trigger" @click="handleTrigger">
            <span class="el-color-picker__color" :class="{ 'is-alpha': showAlpha }">
              <span
                class="el-color-picker__color-inner"
                :style="{
                  backgroundColor: displayedColor
                }"
              ></span>
              <span v-if="!modelValue && !showPanelColor" class="el-color-picker__empty el-icon-close"></span>
            </span>
            <span v-show="modelValue || showPanelColor" class="el-color-picker__icon el-icon-arrow-down"></span>
          </div>
        </div>
      </template>
    </el-popper>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, computed, ref,
  nextTick, reactive, watch,
  provide,inject, onMounted,
} from 'vue'
import type { ComputedRef } from '@vue/reactivity'
import { ClickOutside } from '@element-plus/directives'
import Color from './color'
import SvPanel from './components/sv-panel.vue'
import HueSlider from './components/hue-slider.vue'
import AlphaSlider from './components/alpha-slider.vue'
import Predefine from './components/predefine.vue'
import ElPopper from '@element-plus/popper'
import ElButton from '@element-plus/button'
import ElInput from '@element-plus/input'
import { t } from '@element-plus/locale'
import { UPDATE_MODEL_EVENT }  from '@element-plus/utils/constants'
import { useGlobalConfig } from '@element-plus/utils/util'
import { isValidComponentSize } from '@element-plus/utils/validators'
import { elFormKey, elFormItemKey } from '@element-plus/form'

import type { PropType } from 'vue'
import type { ElFormContext, ElFormItemContext } from '@element-plus/form'

interface IUseOptions {
  currentColor: ComputedRef<string>
}

const OPTIONS_KEY = Symbol()

export const useOptions = () => {
  return inject<IUseOptions>(OPTIONS_KEY)
}

export default defineComponent( {
  name: 'ElColorPicker',
  components: {
    ElPopper,
    ElInput,
    SvPanel,
    HueSlider,
    AlphaSlider,
    ElButton,
    Predefine,
  },
  directives: {
    ClickOutside,
  },
  props: {
    modelValue: String,
    showAlpha: Boolean,
    colorFormat: String,
    disabled: Boolean,
    size: {
      type: String as PropType<ComponentSize>,
      validator: isValidComponentSize,
    },
    popperClass: String,
    predefine: Array,
  },
  emits: {
    change: null,
    'active-change': null,
    [UPDATE_MODEL_EVENT]: null,
  },
  setup(props, { emit }) {
    const ELEMENT = useGlobalConfig()
    const elForm = inject(elFormKey, {} as ElFormContext)
    const elFormItem = inject(elFormItemKey, {} as ElFormItemContext)

    const hue = ref(null)
    const svPanel = ref(null)
    const alpha = ref(null)
    const popper = ref(null)
    // data
    const color = reactive(new Color({
      enableAlpha: props.showAlpha,
      format: props.colorFormat,
    }))
    const showPicker = ref(false)
    const showPanelColor = ref(false)
    const customInput = ref('')
    // computed
    const displayedColor = computed(() => {
      if (!props.modelValue && !showPanelColor.value) {
        return 'transparent'
      }
      return displayedRgb(color, props.showAlpha)
    })
    const colorSize = computed(() => {
      return props.size || elFormItem.size || ELEMENT.size
    })
    const colorDisabled = computed(() => {
      return props.disabled || elForm.disabled
    })

    const currentColor = computed(() => {
      return !props.modelValue && !showPanelColor.value ? '' : color.value
    })
    // watch
    watch(() => props.modelValue, newVal => {
      if (!newVal) {
        showPanelColor.value = false
      } else if (newVal && newVal !== color.value) {
        color.fromString(newVal)
      }
    })
    watch(currentColor, val => {
      customInput.value = val
      emit('active-change', val)
      // showPanelColor.value = true
    })

    watch(() => color.value, () => {
      if (!props.modelValue && !showPanelColor.value) {
        showPanelColor.value = true
      }
    })

    // methods
    function displayedRgb(color, showAlpha) {
      if (!(color instanceof Color)) {
        throw Error('color should be instance of _color Class')
      }

      const { r, g, b } = color.toRgb()
      return showAlpha
        ? `rgba(${ r }, ${ g }, ${ b }, ${ color.get('alpha') / 100 })`
        : `rgb(${ r }, ${ g }, ${ b })`
    }
    function hide() {
      showPicker.value = false
      resetColor()
    }
    function resetColor() {
      nextTick(() => {
        if (props.modelValue) {
          color.fromString(props.modelValue)
        } else {
          showPanelColor.value = false
        }
      })
    }
    function handleTrigger() {
      if (colorDisabled.value) return
      showPicker.value = !showPicker.value
    }
    function handleConfirm() {
      color.fromString(customInput.value)
    }
    function confirmValue() {
      const value = color.value
      emit(UPDATE_MODEL_EVENT, value)
      emit('change', value)
      elFormItem.formItemMitt?.emit('el.form.change', value)
      showPicker.value = false
    }
    function clear() {
      showPicker.value = false
      emit(UPDATE_MODEL_EVENT, null)
      emit('change', null)
      if (props.modelValue !== null) {
        elFormItem.formItemMitt?.emit('el.form.change', null)
      }
      resetColor()
    }
    function doDestroy() {
      popper.value.doDestroy()
    }

    onMounted(() => {
      if (props.modelValue) {
        color.fromString(props.modelValue)
        customInput.value = currentColor.value
      }
    })
    watch(showPicker, () => {
      nextTick(() => {
        hue.value?.update()
        svPanel.value?.update()
        alpha.value?.update()
      })
    })

    provide<IUseOptions>(OPTIONS_KEY, {
      currentColor,
    })

    return {
      color,
      colorDisabled,
      colorSize,
      displayedColor,
      showPanelColor,
      showPicker,
      customInput,
      handleConfirm,
      hide,
      handleTrigger,
      clear,
      confirmValue,
      doDestroy,
      t,
      hue,
      svPanel,
      alpha,
      popper,
    }
  },
})
</script>
