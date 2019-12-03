---
title: vue项目elementUI实现树状下拉选择组件（常见机构选择）
date: 2019-12-03 13:55:06
categories: 学习
tags: [vue]
---
# 一直在网上找各种各样的树状下拉组件，多方参考后自己封装
## 自定义组件 treeSelect.vue
```
<template>
  <div>
    <el-popover placement="bottom-start"
                trigger="click"
                :width="width"
                @hide="popverHide">
      <el-tree ref="tree"
               class="common-tree"
               :default-expand-all="defaultExpandAll"
               :style="treeStyle"
               :data="treeData"
               :props="treeProps"
               :show-checkbox="multiple"
               :node-key="nodeKey"
               :default-checked-keys="defaultCheckedKeys"
               :highlight-current="true"
               @node-click="handleNodeClick"
               @check-change="handleCheckChange">
        <el-select slot="reference" //reference 触发Popover显示的HTML元素
                   ref="select"
                   class="tree-select"
                   v-model="selectedData"
                   :multiple="multiple"
                   @click.native="isShowSelect = !isShowSelect"> 
          <el-option v-for="item in options"
                     :key="item.value"
                     :label="item.label"
                     :value="item.value"> 
          </el-option>
        </el-select>
      </el-tree>
    </el-popover>
  </div>
</template>

<script>
export default {
  name: "treeSelect",
  props: {
    //树结构数据--异步加载数据的话需要注释
    treeData: {
      type: Array,
      default () {
        return [];
      }
    },
    treeProps: {
      type: Object,
      default(){
        return {};
      }
    },
    // 配置是否可多选
    multiple: {
      type: Boolean,
      default () {
        return false;
      }
    },
    // 配置是否可清空选择
    clearable: {
      type: Boolean,
      default () {
        return false;
      }
    },
    nodeKey: {
      type: String,
      default () {
        return 'id';
      }
    },
    // 默认选中的节点key数组
    checkedKeys: {
      type: Array,
      default () {
        return [];
      }
    },
    width: {
      type: Number,
      default () {
        return 250;
      }
    },
    height: {
      type: Number,
      default () {
        return 300;
      }
    }
  },
  data() {
    return {
      defaultCheckKeys: [],
      isShowSelect: false,
      options: [],
      selectedData: [], //选中的节点
      treeStyle: 'width:' + this.width + 'px;' + 'height:' + this.height + 'px;',
      selectStyle: 'width:' + (this.width + 24) + 'px;',
      checkedIds: [],
      checkedData: [],
    };
  },
  created() {},
  mounted() {
    if (this.checkedKeys.length > 0) {
      if (this.multiple) {
        this.defaultCheckedKeys = this.checkedKeys;
        this.selectedData = this.checkedKeys.map((item) => {
          var node = this.$refs.tree.getNode(item);
          return node.label;
        });
      } else {
        var item = this.checkedKeys[0];
        this.$nextTick(() => {
          this.$refs.tree.setCurrentKey(item);
          var node = this.$refs.tree.getNode(item);
          this.selectedData = node.label;
        })
      }
    }
  },
  methods: {
    initData(treeData){
      this.treeData = treeData; //异步加载初始化data
    },
    popoverHide(){
      if (this.multiple) {
        this.checkedIds = this.$refs.tree.getCheckedKeys(); //所有被选中的节点的 key 所组成的数组数据
        this.checkedData = this.$refs.tree.getCheckedNodes(); //所有被选中的节点所组成的数组数据
      } else {
        this.checkedIds = this.$refs.tree.getCurrentKey();
        this.checkedData = this.$refs.tree.getCurrentNode();
      }
      this.$emit('popoverHide', this.checkedIds, this.checkedData);
    },
    //节点被点击时的回调,返回被点击的节点数据
    handleNodeClick (data, node) {
      if (!this.multiple) {
        let tmpMap = {};
        tmpMap.value = node.key;
        tmpMap.label = node.label;
        this.options = [];
        this.options.push(tmpMap);
        this.selectedData = node.label;
        this.isShowSelect = !this.isShowSelect;
      }
    },
    //节点选中状态发生变化时的回调
    handleCheckChange () {
      var checkedKeys = this.$refs.tree.getCheckedKeys(); //所有被选中的节点的 key 所组成的数组数据
      this.options = checkedKeys.map((item) => {
        var node = this.$refs.tree.getNode(item); //所有被选中的节点对应的node
        let tmpMap = {};
        tmpMap.value = node.key;
        tmpMap.label = node.label;
        return tmpMap;
      });
      this.selectedData = this.options.map((item) => {
        return item.label;
      });
    }
  },
  watch: {
    isShowSelect (val) {
      //隐藏select自带的下拉框
      this.$refs.select.blur();
    }
  }
}
</script>

<style scoped>
  .common-tree{
    overflow: auto;
  }
</style>
 
<style>
  .tree-select .el-select__tags .el-tag .el-tag__close{
    display: none;
  }
  .tree-select .el-select__tags .el-tag .el-icon-close{
    display: none;
  }
</style>
```

##调用方式 common.vue
```
<template>
  <div>
    <tree-select ref="demoTreeSelect"
                :defaultProps="defaultProps"
                :nodeKey="nodeKey" 
                :multiple="true"
                :defultCheckedKeys="defaultCheckedKeys"
                @popoverHide="popoverHide"></tree-select>
  </div>
</template>
import TreeSelect from "../treeSelect.vue";
export default {
  data() {
    defaultProps: {
      children: 'children',
      label: 'name'
    },
    nodeKey: 'code',
    defaultCheckedKeys: []
  },
  beforeMount() {
    //异步初始化数据
    //api接口调用返回值res.data
    this.$refs.demoTreeSelect.initData(res.data);
  },
  methods: {
    popoverHide(checkedIds, checkedData){
      console.log(checkedIds);
      console.log(checkedData);
    }
  }
}
```