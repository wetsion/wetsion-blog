---
title: 基于iview的自定义动态表单组件
description: 基于iview开发的动态表单组件
categories:
 - 前端
tags:
 - js
 - vue
---



> 近期在开发搜索管理系统的时候，由于业务需求需要，基于iview开发了一个动态表单组件

对于调用者，只需要传入表单项的定义，还有对应表单项的值，即可动态渲染出表单，同时支持配置表单是全部渲染还是手动逐步添加显示表单项

代码如下：

 - dynamic-form.vue

```html
<template>
  <Form ref="oweForm" :model="data" :label-width="80">
    <div v-if="!gradual">
      <FormItem v-for="(formItem, index) in formValue"
                :prop="formItem.name"
                :label="formItem.note"
                :key="index"
                :rules="formItem.rules">
        <Row>
          <Col span="18">
            <component :is="showItems[formItem.type]" v-model="data[formItem.name]" :info="formItem"/>
          </Col>
          <Col span="4" offset="1">
            <Poptip trigger="hover" :content="formItem.tips === '' ? '无' : formItem.tips">
              <Icon type="ios-help-circle-outline" size="18"/>
            </Poptip>
          </Col>
        </Row>
      </FormItem>
    </div>
    <div v-else>
      <FormItem v-for="(formItem, index) in currentGradualFormValue"
                :prop="formItem.name"
                :label="formItem.note"
                :key="index"
                :rules="formItem.rules">
        <Row>
          <Col span="14">
            <component :is="showItems[formItem.type]" v-model="data[formItem.name]" :info="formItem"/>
          </Col>
          <Col span="4" offset="1">
            <Poptip trigger="hover" :content="formItem.tips === '' ? '无' : formItem.tips">
              <Icon type="ios-help-circle-outline" size="18"/>
            </Poptip>
          </Col>
          <Col span="4" offset="1">
            <Button @click="removeFormValue(formItem.name)">Delete</Button>
          </Col>
        </Row>
      </FormItem>
      <Row>
        <Col span="16">
          <Select v-model="choosedFormValueName" style="width:200px">
            <Option v-for="item in gradualFromValue" :value="item.name" :key="item.name">{{ item.name }}</Option>
          </Select>
        </Col>
        <Col span="8">
          <Button @click="addFormValue">新增表单项</Button>
        </Col>
      </Row>
    </div>
  </Form>
</template>

<script>
import showItems from './form-items-map'
export default {
  name: 'DynamicForm',
  props: {
    value: {
      type: Object,
      default: () => ({})
    },
    form: {
      type: String,
      default: ''
    },
    // 是否可逐步添加表单项，默认否，展示全部表单项
    gradual: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      showItems,
      gradualFromValue: [],
      currentGradualFormValue: [],
      gradualFormMap: {},
      choosedFormValueName: ''
    }
  },
  watch: {
    form (val) {
      this.generateGradualFormValue()
    },
    value: {
      handler (nv, ov) {
        this.generateGradualFormValue()
      },
      deep: true
    }
  },
  computed: {
    formValue: {
      get () {
        let formValueArr = JSON.parse(this.form)
        for (let i = 0; i < formValueArr.length; i++) {
          let rules = []
          if (formValueArr[i].required) {
            rules.push({required: true, type: 'string', message: formValueArr[i].hintText, trigger: 'blur'})
          }
          // TODO 正则匹配校验待实现，判空待实现
          formValueArr[i].rules = rules
          // 如果有默认值，将默认值赋值给data[formItem]
          // if (formValueArr[i].conf.defaultValue) {
          //   this.data[formValueArr[i].name] = formValueArr[i].conf.defaultValue
          // }
        }
        return formValueArr
      }
    },
    data: {
      set (val) {
        this.$emit('input', val)
      },
      get () {
        return this.value
      }
    }
  },
  methods: {
    async validate () {
      console.log('df validate')
      return new Promise((resolve) => {
        this.$refs.oweForm.validate((valid) => {
          resolve(valid)
        })
      })
    },
    onValidate (callback) {},
    generateGradualFormValue () {
      this.gradualFromValue = []
      this.currentGradualFormValue = []
      let formValueArr = JSON.parse(this.form)
      for (let i = 0; i < formValueArr.length; i++) {
        let rules = []
        if (formValueArr[i].required) {
          rules.push({required: true, type: 'string', message: formValueArr[i].hintText, trigger: 'blur'})
        }
        // TODO 正则匹配校验待实现，判空待实现
        formValueArr[i].rules = rules

        this.gradualFormMap[formValueArr[i].name] = formValueArr[i]
        if (this.value.hasOwnProperty(formValueArr[i].name)) {
          this.currentGradualFormValue.push(formValueArr[i])
        }
      }
      this.gradualFromValue = formValueArr
    },
    /***
     * 往动态表单里添加表单项
     * @param name
     */
    addFormValue () {
      let canCreate = true
      this.currentGradualFormValue.forEach(item => {
        if (item.name === this.choosedFormValueName) {
          this.$Message.warning('该表单项已存在！')
          canCreate = false
        }
      })
      if (canCreate) {
        this.currentGradualFormValue.push(this.gradualFormMap[this.choosedFormValueName])
      }
    },
    /***
     * 删除动态表单项
     * @param name
     */
    removeFormValue (name) {
      let tempArr = []
      this.currentGradualFormValue.forEach(item => {
        if (item.name !== name) {
          tempArr.push(item)
        }
      })
      this.currentGradualFormValue = tempArr
    }
  },
  mounted () {
    this.generateGradualFormValue()
  }
}
</script>

<style scoped>

</style>

```

> `form`属性是一个数组转成的字符串，数组中每一个对象需要包含以下属性：
> - `required` 是否必填
> - `hintText` hintText文本
> - `name` 表单项的name
> - `note` 表单项的label
> - `type` 表单项的类型
> - `tips` 提示文本

 - form-items-map.js

```js
import FormInput from './form-items/form-input'
import FormCheckbox from './form-items/form-checkbox'
import FormRadio from './form-items/form-radio'
import FormSelect from './form-items/form-select'
import FormTextarea from './form-items/form-textarea'

export default {
  0: FormInput,
  1: FormTextarea,
  2: FormRadio,
  3: FormSelect,
  4: FormCheckbox
}
```

> `FormInput`、`FormCheckbox`、`FormRadio`、`FormSelect`、`FormTextarea`都是重写的表单项，具有统一的props


 - form-input.vue

```html
<template>
  <Input type="text" v-model="data" :placeholder="info.conf.hintText ? info.conf.hintText : ''"/>
</template>

<script>
export default {
  name: 'FormInput',
  props: {
    value: {
      type: String | Number,
      default: ''
    },
    info: {
      type: Object,
      default: () => ({})
    }
  },
  computed: {
    data: {
      get () {
        return this.value
      },
      set (val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>

<style scoped>
</style>
```

 - form-checkbox.vue

```html
<template>
  <CheckboxGroup v-model="data">
    <Checkbox v-for="(item, index) in info.conf.data" :label="item.return" :key="index">
      <span>{{ item.show }}</span>
    </Checkbox>
  </CheckboxGroup>
</template>

<script>
export default {
  name: 'FormCheckbox',
  props: {
    value: {
      type: String | Number,
      default: ''
    },
    info: {
      type: Object,
      default: () => ({})
    }
  },
  computed: {
    data: {
      get () {
        return this.value
      },
      set (val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>

<style scoped>
</style>
```

 - form-radio.vue

```html
<template>
  <RadioGroup v-model="data">
    <Radio v-for="(item, index) in info.conf.data" :label="item.return" :key="index">
      <span>{{ item.show }}</span>
    </Radio>
  </RadioGroup>
</template>

<script>
export default {
  name: 'FormRadio',
  props: {
    value: {
      type: String | Number,
      default: ''
    },
    info: {
      type: Object,
      default: () => ({})
    }
  },
  computed: {
    data: {
      get () {
        return this.value
      },
      set (val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>

<style scoped>
</style>
```

 - form-select.vue

```html
<template>
  <Select v-model="data" style="width:200px">
    <Option v-for="(item, index) in info.conf.data" :value="item.return" :key="index">{{ item.show }}</Option>
  </Select>
</template>

<script>
export default {
  name: 'FormSelect',
  props: {
    value: {
      type: String | Number,
      default: ''
    },
    info: {
      type: Object,
      default: () => ({})
    }
  },
  computed: {
    data: {
      get () {
        return this.value
      },
      set (val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>

<style scoped>
</style>
```

 - form-textarea.vue

```html
<template>
  <Input type="textarea" v-model="data" :placeholder="info.conf.hintText ? info.conf.hintText : ''"/>
</template>

<script>
export default {
  name: 'FormTextarea',
  props: {
    value: {
      type: String | Number,
      default: ''
    },
    info: {
      type: Object,
      default: () => ({})
    }
  },
  computed: {
    data: {
      get () {
        return this.value
      },
      set (val) {
        this.$emit('input', val)
      }
    }
  }
}
</script>

<style scoped>
</style>
```

至此一个动态表单组件就完成了，记录一哈，嘿嘿:yum:
