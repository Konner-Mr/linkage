# linkage
该组件是一个简单的Vue3表单组件，支持渲染input、select、data-picker等，可自行扩展，组件依赖于Element-plus
select：
  支持远程搜索
  支持级联

使用场景：表格过滤条件

使用方式：
field字段配置可参照index.vue组件内说明
<Linkage ref="linkageRef" :field="field" @paramsCall="setParams" />

const linkageRef = ref(null)

linkageRef.value.init().then(()=>{})
