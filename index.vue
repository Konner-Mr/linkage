<!--
author: 康少
参数配置说明:
[{
    name: "",                                   //字段名称
    getOptionUrl: ``,                           //获取下拉选项数据接口
    linkage: false,                              //false => 一级级联字段init函数时获取下拉选项
    params: { brandType: "vehicle" } || [{ key: "brand", params: { brandCode: "brandCode" } }], 
                                                //获取数据时传参，对象：直接传参；数组：key是参数来源字段，params对象的key为参数名，value为数据来源的参数名，
    showKey: [
        {
            key: ["series", "model"],
            when: { brandType: "vehicle" }
        },
        {
            key: ["assembleSeries", "assemblyModel"],
            when: { brandType: "assembly_type" }
        }
    ],                                          //显示隐藏某些关联字段,key：关联字段； when：选中选项中brandType值为vehicle时显示关联字段，否则隐藏
    change_key: ["series", "assembleSeries"],   //级联字段key值
    key: "brand",                               //当前字段key值
    type: "select",                             //当前字段类型
    option: [],                                 //下拉选项
    value: "",                                  //选中数据值
    show_key: "brandName",                      //下拉选项显示的字段名
    value_key: "id",                            //下拉选项实际传输的值
    search_key: "brandId",                      //传参时的key值
    show: true                                  //是否显示改字段，不显示时不传参
}],
-->

<template>
  <div class="filterWrap">
    <div
      style="display:inline-block"
      class="px-1 mb-1"
      v-for="(item, index) in filterArray"
      :key="index"
      v-show="item.show"
    >
      <div class="rowSC" :style="{ width: item.width ? item.width : 'auto' }">
        <div class="mr-2" style="white-space: nowrap">
          <span class="text-danger" v-if="item.must">*</span>{{ item.name }}
        </div>
        <template v-if="item.type == 'select'">
          <el-select
            class="flex-1"
            filterable
            v-model="item.value"
            placeholder="请选择"
            clearable
            @change="selectChange(item)"
          >
            <el-option
              v-for="(o_item, o_index) in item.option"
              :key="o_index"
              :label="o_item[item.show_key]"
              :value="o_item[item.value_key]"
            >
            </el-option>
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
        <template v-if="item.type == 'time'">
          <el-date-picker
            v-model="item.value"
            type="daterange"
            value-format="yyyy-MM-dd"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
          >
          </el-date-picker>
        </template>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
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
  },
  data() {
    return {
      filterArray: []
    };
  },
  mounted() {
    if (this.autoInit) this.init();
  },
  watch: {
    /**
     * 监听字段变化返回参数集合**/
    filterArray: {
      handler(newV, oldV) {
        let params = new Object();
        newV.forEach(item => {
          if (item.search_key && item.show) {
            if (
              Object.prototype.toString.call(item.search_key) ==
              "[object Array]"
            ) {
              item.search_key.forEach((o_item, index) => {
                params[o_item] = item.value ? item.value[index] : "";
              });
            } else {
              params[item.search_key] = item.value;
            }
          }
        });
        this.$emit("paramsCall", params);
      },
      immediate: true,
      deep: true
    }
  },
  methods: {
    /**
     * 初始化**/
    init() {
      this.filterArray = this.field;
      return new Promise((resolve, reject) => {
        this.$nextTick(() => {
          let filter = this.filterArray,
            getArray = new Array();
          filter.forEach((item, index) => {
            if (item.getOptionUrl) {
              item.option = [];
            }
            item.value = item.type != "checkbox" ? "" : [];
            if (!item.linkage && item.getOptionUrl) {
              getArray.push(
                this.getOption(item.getOptionUrl, item.params, index)
              );
            }
          });
          if (getArray?.length > 0) {
            Promise.all(getArray).then(() => {
              resolve();
            });
          } else {
            resolve();
          }
        });
      });
    },
    /**
     * 获取下拉字段选项数据**/
    getOption(url, data, index) {
      return new Promise((resolve, reject) => {
        this.$set(this.filterArray[index], "value", "");
        this.$axios({
          url,
          method: "get",
          data
        })
          .then(res => {
            this.$set(
              this.filterArray[index],
              "option",
              res.data.records || res.data
            );
            resolve();
          })
          .catch(() => {
            resolve();
          });
      });
    },
    /**
     * 下拉选项触发change事件**/
    selectChange(e) {
      let o_index = e.option.findIndex(i => e.value == i[e.value_key]),
        filterArray = this.filterArray;
      //处理显示隐藏关联字段
      if (e.showKey) {
        this.show_hide_field(e, e.option[o_index]);
      }
      //处理联动查询
      if (e.change_key) {
        let change_key = e.change_key;
        change_key.forEach(c_item => {
          let index = filterArray.findIndex(c => c_item == c.key);
          //联动字段show为true
          if (index !== -1 && filterArray[index].show) {
            if (e.value !== "") {
              let url = filterArray[index].getOptionUrl,
                params = new Object(),
                config_params = filterArray[index].params;
              config_params.forEach(config_item => {
                let config_key = config_item.key,
                  config_child_params = config_item.params,
                  config_index = filterArray.findIndex(
                    c => config_key == c.key
                  );
                if (config_index !== -1) {
                  let params_index = filterArray[config_index].option.findIndex(
                    p_item =>
                      p_item[filterArray[config_index].value_key] ==
                      filterArray[config_index].value
                  );
                  for (let x in config_child_params) {
                    params[x] =
                      filterArray[config_index].option[params_index][
                        config_child_params[x]
                      ];
                  }
                }
              });
              this.getOption(url, params, index);
            } else {
              //清除关联字段选项和值
              this.$set(this.filterArray[index], "value", "");
              this.$set(this.filterArray[index], "option", []);
            }
          }
        });
      }
    },
    /**
     * 下拉选项触发关联字段显示隐藏**/
    show_hide_field(e, o_item) {
      e.showKey.forEach(item => {
        let key = item.key,
          when = item.when;
        for (let x in when) {
          //如果选中字段x == 指定值，则key中字段显示，否则隐藏，
          key.forEach(k_item => {
            let index = this.filterArray.findIndex(
              f_item => f_item.key === k_item
            );
            if (index != -1) {
              if (e.value !== "") {
                this.$set(
                  this.filterArray[index],
                  "show",
                  o_item[x] == when[x] ? true : false
                );
              }
              //清除所有关联字段选项和值
              this.$set(this.filterArray[index], "value", "");
              this.$set(this.filterArray[index], "option", []);
            }
          });
        }
      });
    }
  }
};
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
