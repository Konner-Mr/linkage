<!--
author: Mr.Kon
参数配置说明:
[{
    key: "brand",                               //当前字段key值
    name: "",                                   //字段名称
    ………级联字段配置start………
    getOptionUrl: ``,                           //获取下拉选项数据接口
    linkage: false,                             //false => 一级级联字段init函数时获取下拉选项
    params: [{ key: "brand", params: { brandCode: "brandCode" } }], 
                                                //获取选项数据时传参，key是参数来源字段(对应字段配置中的key)，params对象中【属性名】为【传参名】，【属性值】为【key字段】下【选中选项的其他属性的值】，
    default_params:{},                          //默认参数，和params合并传输
    change_key: ["series"],                     //级联字段key值
    empty_key:[],                               //级联字段，选项值为空的时候对其关联字段的option、value进行清空
    type: "select",                             //当前字段类型 【input/select/date/dateTime】
    option: [],                                 //下拉选项
    default_option:[],                          //默认选项，拼接在选项后面
    show_key: "brandName",                      //下拉选项显示的字段名
    value_key: "id",                            //下拉选项实际传输的值
    noClearable: true,                          //不需要清空键，默认false
    remote: true,                               //远程搜索开关
    remoteMethod:remoteMethod,                  //v => 输入的值
    ………级联字段end………
    value: "",                                  //选中数据值
    search_key: "brandId",                      //传参时的key值
    noEmpty: true,                              //初始化不需要清空value值
    show: true                                  //是否显示改字段，不显示时不传参
}],
远程搜索例子：函数内部可自定义，本案例中调用到组件的getIndex，getOption
* remoteMethod => 自定义函数名
* e => 输入的值
* key => 字段名称
const remoteMethod = (e, key = 'series') => {
  let index = linkage.value.getIndex(key)
  linkage.value.getOption('knowledge/inner/brand/query', { brandType: 'vehicle,assembly_type' }, index)
}
-->

<template>
  <div class="filterWrap">
    <div style="display: inline-block" class="px-1 mb-1" v-for="(item, index) in field" :key="index" v-show="item.show">
      <div class="rowSC" :style="{ width: item.width ? item.width : 'auto' }">
        <div class="mr-2" style="white-space: nowrap">
          <span class="text-danger" v-if="item.must">*</span>
          {{ item.name }}
        </div>
        <template v-if="item.type == 'select'">
          <el-select
            class="flex-1"
            filterable
            v-model="item.value"
            placeholder="请选择"
            :remote="item.remote"
            :remote-method="item.remoteMethod"
            :clearable="!item.noClearable ? true : false"
            @change="selectChange(item)"
          >
            <el-option
              v-for="(o_item, o_index) in item.option"
              :key="o_index"
              :label="o_item[item.show_key]"
              :value="o_item[item.value_key]"
            ></el-option>
          </el-select>
        </template>
        <template v-if="item.type == 'input'">
          <el-input
            v-model="item.value"
            class="flex-1"
            :placeholder="item.placeholder ? item.placeholder : '请输入'"
            clearable
          ></el-input>
        </template>
        <template v-if="item.type == 'date'">
          <el-date-picker
            v-model="item.value"
            type="daterange"
            value-format="YYYY-MM-DD"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
          ></el-date-picker>
        </template>
        <template v-if="item.type == 'dateTime'">
          <el-date-picker
            v-model="item.value"
            type="datetimerange"
            value-format="YYYY-MM-DD hh:mm:ss"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            :disabled-date="disabledDate"
          ></el-date-picker>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import request from '@/utils/axiosReq'
import { defineExpose } from 'vue'
const props = defineProps({
  /**
   * 级联过滤字段**/
  field: {
    type: Array,
    default: () => []
  },
  /**
   * 是否直接初始化**/
  autoInit: {
    type: Boolean,
    default: false
  }
})
const emits = defineEmits(['paramsCall'])

//禁止选择的日期
const disabledDate = (time) => {
  return time.getTime() > Date.now() - 8.64e6
}

/**
 * 通过watch把参数集合组装好抛出去给父组件
 */
watch(
  () => props.field,
  (newV, oldV) => {
    let params = new Object()
    newV.forEach((item) => {
      if (item.search_key && item.show) {
        if (Object.prototype.toString.call(item.search_key) == '[object Array]') {
          item.search_key.forEach((o_item, index) => {
            params[o_item] = item.value ? item.value[index] : ''
          })
        } else {
          params[item.search_key] = item.value
        }
      }
    })
    emits('paramsCall', params)
  },
  {
    deep: true
  }
)

/**
 * 初始化
 */
const init = () => {
  return new Promise(async (resolve, reject) => {
    let f = props.field
    for (let x = 0; x < f.length; x++) {
      if (f[x].getOptionUrl) f[x].option = []
      if (f[x].noEmpty) f[x].value = f[x].type != 'checkbox' ? '' : []
      if (!f[x].linkage && f[x].getOptionUrl) {
        await getOption(f[x].getOptionUrl, f[x].default_params, x)
      }
      if (x == f.length - 1) resolve()
    }
  })
}

/**
 * 获取下拉字段选项数据**/
const getOption = (url, params, index, method = 'get') => {
  return new Promise((resolve, reject) => {
    props.field[index].value = ''
    request({ url, method, data: params, isParams: true })
      .then((res) => {
        let data = res.data.records || res.data,
          default_option = props.field[index].default_option
        props.field[index].option = default_option ? [...data, ...default_option].flat() : data
        resolve()
      })
      .catch(() => {
        resolve()
      })
  })
}

/**
 * 下拉选择
 */
const selectChange = (e) => {
  let f = props.field
  if (e.change_key) {
    e.change_key.forEach((a) => {
      //关联字段索引值
      let index = f.findIndex((c) => a == c.key)
      if (index !== -1) {
        if (e.value !== '') {
          let url = f[index].getOptionUrl,
            params = new Object(),
            config_params = f[index].params
          config_params.forEach((config_item) => {
            let config_key = config_item.key,
              config_child_params = config_item.params,
              config_index = f.findIndex((c) => config_key == c.key)
            if (config_index !== -1) {
              let params_index = f[config_index].option.findIndex(
                (p_item) => p_item[f[config_index].value_key] == f[config_index].value
              )
              for (let x in config_child_params) {
                params[x] = f[config_index].option[params_index][config_child_params[x]]
              }
            }
          })
          if (f[index].default_params) params = Object.assign(f[index].default_params, params)
          getOption(url, params, index)
        } else {
          let e_key = e.empty_key
          if (e_key) {
            e_key.forEach((e) => {
              let e_i = f.findIndex((c) => e == c.key)
              if (e_i !== -1) {
                f[e_i].option = f[e_i].default_option ? f[e_i].default_option : []
                f[e_i].value = ''
              }
            })
          }
        }
      }
    })
  }
}

/**
 * 获取字段在props.field中的索引值
 * key不能为空
 */
const getIndex = (key) => {
  return props.field.findIndex((item) => item.key === key)
}

defineExpose({ init, getOption, getIndex })
</script>

<style lang="scss" scoped>
.filterWrap {
  display: inline-block;
}
.rowSC {
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: center;
}

.flex-wrap {
  flex-wrap: wrap;
}

.px-1 {
  padding-left: 10px;
  padding-right: 10px;
}
.mb-1 {
  margin-bottom: 10px;
}

.flex-1 {
  flex: 1;
}

.mr-2 {
  margin-right: 20px;
}
</style>
