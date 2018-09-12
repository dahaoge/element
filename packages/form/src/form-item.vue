<template>
  <div class="el-form-item" :class="[{
      'el-form-item--feedback': elForm && elForm.statusIcon,
      'is-error': validateState === 'error',
      'is-validating': validateState === 'validating',
      'is-success': validateState === 'success',
      'is-required': isRequired || required
    },
    sizeClass ? 'el-form-item--' + sizeClass : ''
  ]">
    <label :for="labelFor" class="el-form-item__label" :style="labelStyle" v-if="label || $slots.label">
      <slot name="label">{{label + form.labelSuffix}}</slot>
    </label>
    <div class="el-form-item__content" :style="contentStyle">
      <slot></slot>
      <transition name="el-zoom-in-top">
        <div v-if="validateState === 'error' && showMessage && form.showMessage" class="el-form-item__error" :class="{
            'el-form-item__error--inline': typeof inlineMessage === 'boolean'
              ? inlineMessage
              : (elForm && elForm.inlineMessage || false)
          }">
          {{validateMessage}}
        </div>
      </transition>
    </div>
  </div>
</template>
<script>
import AsyncValidator from 'async-validator';
import emitter from 'element-ui-hao/src/mixins/emitter';
import objectAssign from 'element-ui-hao/src/utils/merge';
import { noop, getPropByPath, strFormat } from 'element-ui-hao/src/utils/util';

export default {
  name: 'ElFormItem',

  componentName: 'ElFormItem',

  mixins: [emitter],

  provide() {
    return {
      elFormItem: this
    };
  },

  inject: ['elForm'],

  props: {
    dataIndex: {
      type: Number,
      default: null
    },
    validLabel: String,
    label: String,
    labelWidth: String,
    prop: String,
    required: {
      type: Boolean,
      default: undefined
    },
    rules: [Object, Array],
    error: String,
    validateStatus: String,
    for: String,
    inlineMessage: {
      type: [String, Boolean],
      default: ''
    },
    showMessage: {
      type: Boolean,
      default: true
    },
    size: String
  },
  watch: {
    error: {
      immediate: true,
      handler(value) {
        this.validateMessage = value;
        this.validateState = value ? 'error' : '';
      }
    },
    validateStatus(value) {
      this.validateState = value;
    }
  },
  computed: {
    labelFor() {
      return this.for || this.prop;
    },
    labelStyle() {
      const ret = {};
      if (this.form.labelPosition === 'top') return ret;
      const labelWidth = this.labelWidth || this.form.labelWidth;
      if (labelWidth) {
        ret.width = labelWidth;
      }
      return ret;
    },
    contentStyle() {
      const ret = {};
      const label = this.label;
      if (this.form.labelPosition === 'top' || this.form.inline) return ret;
      if (!label && !this.labelWidth && this.isNested) return ret;
      const labelWidth = this.labelWidth || this.form.labelWidth;
      if (labelWidth) {
        ret.marginLeft = labelWidth;
      }
      return ret;
    },
    form() {
      let parent = this.$parent;
      let parentName = parent.$options.componentName;
      while (parentName !== 'ElForm') {
        if (parentName === 'ElFormItem') {
          this.isNested = true;
        }
        parent = parent.$parent;
        parentName = parent.$options.componentName;
      }
      return parent;
    },
    fieldValue() {
      const model = this.form.model;
      if (!model || !this.prop) {
        return;
      }

      let path = this.prop;
      if (path.indexOf(':') !== -1) {
        path = path.replace(/:/, '.');
      }

      return getPropByPath(model, path, true).v;
    },
    inputType() {
      console.log(this.$children[0]);

      if (
        this.$children &&
        this.$children[0] &&
        this.$children[0].$options &&
        this.$children[0].$options._componentTag
      ) {
        const tagName = this.$children[0].$options._componentTag;
        const c = this.$children[0];
        // el-date-picker
        return tagName === 'el-input'
          ? this.$children[0].type === 'number' ? null : 'string'
          : tagName === 'el-date-picker'
            ? 'date'
            : tagName === 'el-select'
              ? c.multiple ? 'array' : 'string'
              : null;
      }
      return null;
    },
    isRequired() {
      let rules = this.getRules();
      let isRequired = false;

      if (rules && rules.length) {
        rules.every(rule => {
          if (rule.required) {
            isRequired = true;
            return false;
          }
          return true;
        });
      }
      return isRequired;
    },
    _formSize() {
      return this.elForm.size;
    },
    elFormItemSize() {
      return this.size || this._formSize;
    },
    sizeClass() {
      return this.elFormItemSize || (this.$ELEMENT || {}).size;
    }
  },
  data() {
    return {
      validateState: '',
      validateMessage: '',
      validateDisabled: false,
      validator: {},
      isNested: false,
      validMsg: {
        default: 'Validation error on field {label}',
        required: '请输入 {label}',
        whitespace: '{label} 不能为空',
        date: {
          format: '{label} 格式不正确',
          parse: '{label} 格式不正确 ',
          invalid: '{label} 格式不正确'
        },
        types: {
          string: '请输入合法的字符',
          number: '请输入数字',
          date: '时间格式不正确',
          integer: '请输入整数',
          float: '请输入小数',
          email: '邮件格式不正确',
          url: '网络地址格式不正确'
        },
        string: {
          len: '{label} 的长度必须是 {len}',
          min: '{label} 的长度必须大于 {min}',
          max: '{label} 的长度必须小于 {max}',
          range: '请输入 {min} 到 {max} 个字符'
        },
        number: {
          len: '{label} 的长度为 {len}',
          min: '{label} 必须大于 {min}',
          max: '{label} 必须小于 {max}',
          range: '请输入 {min} 到 {max} 之前的数字'
        },
        array: {
          len: '请输入选择至少{len}个选项',
          min: '请至少选择 {min} 个选项',
          max: '请最多选择 {max} 个选项',
          range: '请选择 {min} 到 {max} 个选项'
        }
      }
    };
  },
  methods: {
    validate(trigger, callback = noop) {
      this.validateDisabled = false;
      const rules = this.getFilteredRule(trigger);
      if ((!rules || rules.length === 0) && this.required === undefined) {
        callback();
        return true;
      }

      this.validateState = 'validating';

      let descriptor = {};
      let tempRules = [];
      let validType = '';
      if (rules && rules.length > 0) {
        rules.forEach(rule => {
          if (validType === '' && rule.type) validType = rule.type;
          delete rule.trigger;
        });
        tempRules = rules.map(item => {
          if (!validType) {
            validType = this.inputType ? this.inputType : 'string';
          }
          item.type = validType;

          if (
            item.message === null ||
            item.message === undefined ||
            item.message === ''
          ) {
            let template = '';
            if (item.required) {
              template = this.validMsg.required;
            } else if (item.whitespace) {
              this.validMsg.whitespace;
            } else if (item.default) {
              this.validMsg.default;
            } else if (
              item.type === 'string' ||
              item.type === 'number' ||
              item.type === 'array'
            ) {
              item.type = item.type ? item.type : 'string';
              template = item.len
                ? this.validMsg[item.type].len
                : item.min && item.max
                  ? this.validMsg[item.type].range
                  : item.min
                    ? this.validMsg[item.type].min
                    : item.max ? this.validMsg[item.type].max : '';
            } else {
              template = this.validMsg.types[item.type];
            }
            if (
              template !== null &&
              template !== undefined &&
              template !== ''
            ) {
              item.label = this.label
                ? this.label
                : this.validLabel ? this.validLabel : ' ';
              item.message = strFormat(template, item);
            }
          }
          return item;
        });
        /* if (validType === 'array') {
          if (
            tempRules.some(item => {
              return item.required;
            }) &&
            tempRules.some(item => {
              return !item.len && !item.min;
            })
          ) {
            tempRules.push({
              min: 1,
              message: '请至少选择 1 个选项',
              type: validType
            });
          }
        } */
      }

      descriptor[this.prop] = tempRules;

      const validator = new AsyncValidator(descriptor);
      const model = {};

      if (this.dataIndex != null && this.dataIndex !== undefined) {
        model[this.prop] = this.fieldValue[this.dataIndex];
      } else {
        model[this.prop] = this.fieldValue;
      }

      if (
        validType === 'float' ||
        validType === 'integer' ||
        validType === 'number'
      ) {
        model[this.prop] = parseFloat(model[this.prop]);
      }

      validator.validate(
        model,
        { firstFields: true },
        (errors, invalidFields) => {
          this.validateState = !errors ? 'success' : 'error';
          this.validateMessage = errors ? errors[0].message : '';

          callback(this.validateMessage, invalidFields);
          this.elForm && this.elForm.$emit('validate', this.prop, !errors);
        }
      );
    },
    clearValidate() {
      this.validateState = '';
      this.validateMessage = '';
      this.validateDisabled = false;
    },
    resetField() {
      this.validateState = '';
      this.validateMessage = '';

      let model = this.form.model;
      let value = this.fieldValue;
      let path = this.prop;
      if (path.indexOf(':') !== -1) {
        path = path.replace(/:/, '.');
      }

      let prop = getPropByPath(model, path, true);

      this.validateDisabled = true;
      if (Array.isArray(value)) {
        prop.o[prop.k] = [].concat(this.initialValue);
      } else {
        prop.o[prop.k] = this.initialValue;
      }

      this.broadcast('ElTimeSelect', 'fieldReset', this.initialValue);
    },
    getRules() {
      let formRules = this.form.rules;
      const selfRules = this.rules;
      const requiredRule =
        this.required !== undefined ? { required: !!this.required } : [];

      const prop = getPropByPath(formRules, this.prop || '');
      formRules = formRules ? prop.o[this.prop || ''] || prop.v : [];

      return [].concat(selfRules || formRules || []).concat(requiredRule);
    },
    getFilteredRule(trigger) {
      const rules = this.getRules();

      return rules
        .filter(rule => {
          if (!rule.trigger || trigger === '') return true;
          if (Array.isArray(rule.trigger)) {
            return rule.trigger.indexOf(trigger) > -1;
          } else {
            return rule.trigger === trigger;
          }
        })
        .map(rule => objectAssign({}, rule));
    },
    onFieldBlur() {
      this.validate('blur');
    },
    onFieldChange() {
      if (this.validateDisabled) {
        this.validateDisabled = false;
        return;
      }

      this.validate('change');
    }
  },
  mounted() {
    if (this.prop) {
      this.dispatch('ElForm', 'el.form.addField', [this]);

      let initialValue = this.fieldValue;
      if (Array.isArray(initialValue)) {
        initialValue = [].concat(initialValue);
      }
      Object.defineProperty(this, 'initialValue', {
        value: initialValue
      });

      let rules = this.getRules();

      if (rules.length || this.required !== undefined) {
        this.$on('el.form.blur', this.onFieldBlur);
        this.$on('el.form.change', this.onFieldChange);
      }
    }
  },
  beforeDestroy() {
    this.dispatch('ElForm', 'el.form.removeField', [this]);
  }
};
</script>
