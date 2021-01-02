
<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      accordion
      :expand-on-click-node="false"
      :default-expanded-keys="defaultExpandedKeys"
      draggable
      :allow-drop="allowDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="Category Name">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Category Icon">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Category Unit">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">Cancel</el-button>
        <el-button type="primary" @click="submit">Submit</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》from '《组件路径》';

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "childs",
        label: "name",
      },
      defaultExpandedKeys: [],
      dialogVisible: false,
      category: {
        catId: 0,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
        productCount: null,
      },
      dialogtye: "",
      dialogTitle: "",
      //assistance variable for calculating tree depth
      maxDepth: 0,
    };
  },

  methods: {
    //return tree, if:
    // 1. for type = inner, the draggingNode height + dropNode level < 3
    // 2. for type = prev || next, the draggingNode height - 1 + dropNode Level < 3
    //
    allowDrop(draggingNode, dropNode, type) {
      console.log(draggingNode, dropNode, type);
      if (type == "inner") {
        console.log("draggingNode height", this.calculate(draggingNode.data));
        return (this.calculate(draggingNode.data) + dropNode.data.catLevel) <= 3;
      } else if (type == "prev" || type == "next")
        return (this.calculate(draggingNode.data) - 1 + dropNode.data.catLevel) <= 3;
    },

    //caculate the subtree height represented by current node (the node number of the longest path in the subtree represented by this current node), using recuresive algorithm
    calculate(currentNode) {
      let max = 1;
      if (currentNode.childs == null || currentNode.childs.length == 0) {
        return 1;
      } else {
        for (var i = 0; i < currentNode.childs.length; i++) {
          let result = this.calculate(currentNode.childs[i]);
          max = this.compareAndSwap(max, result);
        }
        return max + 1;
      }
    },

    compareAndSwap(a, b) {
      return a > b ? a : b;
    },

    getMenus() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.list;
      });
    },

    resetCategory() {
      (this.category.catId = null),
        (this.category.name = ""),
        (this.category.parentCid = 0),
        (this.category.catLevel = 0),
        (this.category.showStatus = 1),
        (this.category.sort = 0),
        (this.category.icon = ""),
        (this.category.productUnit = ""),
        (this.category.productCount = 0);
    },

    append(data) {
      this.dialogVisible = true;
      this.dialogtye = "append";
      this.dialogTitle = "Append Category";
      this.resetCategory();
      this.category.parentCid = data.catId;
      // this.category.name = "";
      //to avoid data.catLevel is a string, use *1
      this.category.catLevel = data.catLevel * 1 + 1;
      // this.category = JSON.parse(JSON.stringify(data));
    },
    //Add commodity category
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.dialogVisible = false;
        this.$message({
          message: "Congrats, category is successfully added.",
          type: "success",
        });
        this.getMenus();
        this.defaultExpandedKeys = [this.category.parentCid];
      });
    },
    //edit category
    edit(data) {
      console.log(data);
      //re-retrieve the data from the database.
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.dialogVisible = true;
        this.dialogtye = "edit";
        this.dialogTitle = "Edit Category";
        // this.category.name = data.category.name;
        // this.category.catId = data.category.catId;
        // this.category.icon = data.category.icon;
        // this.category.productUnit = data.category.productUnit;
        // this.category.parentCid = data.category.parentCid;
        // this.category.catLevel = data.category.catLevel
        this.category = JSON.parse(JSON.stringify(data.category));
      });
    },
    //submit edited category
    //update category, other than the specified fields, all other will remain the same
    //as before
    update() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        //when use {} to construct a object, the field name and the value variable name is same, we //can only use the value variable.
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.dialogVisible = false;
        this.$message({
          message: "Congrats, category is successfully updated.",
          type: "success",
        });
        this.getMenus();
        this.defaultExpandedKeys = [this.category.parentCid];
      });
    },

    submit() {
      if (this.dialogtye == "append") {
        this.addCategory();
      } else if (this.dialogtye == "edit") {
        this.update();
      }
    },
    //remove category
    remove(node, data) {
      this.$confirm(
        `This will permanently delete the category: ${data.name}. Continue?`,
        "WARNING",
        {
          confirmButtonText: "OK",
          cancelButtonText: "Cancel",
          type: "warning",
        }
      )
        .then(() => {
          var ids = [data.catId];

          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "Congrats, category is successfully deleted.",
              type: "success",
            });
            this.defaultExpandedKeys = [node.parent.data.catId];
            this.getMenus();
          });
        })
        .catch(() => {});
    },
  },
  //  计算属性类似于data 概念
  computed: {},
  //  监控data 中的数据变化
  watch: {},

  //  生命周期- 创建完成（可以访问当前this 实例）
  created() {
    this.getMenus();
  },
  //  生命周期- 挂载完成（可以访问DOM 元素）
  mounted() {},
  beforeCreate() {}, // 生命周期- 创建之前
  beforeMount() {}, // 生命周期- 挂载之前
  beforeUpdate() {}, // 生命周期- 更新之前
  updated() {}, // 生命周期- 更新之后
  beforeDestroy() {}, // 生命周期- 销毁之前
  destroyed() {}, // 生命周期- 销毁完成
  activated() {}, // 如果页面有keep-alive 缓存功能，这个函数会触发
};
</script>
<style  scoped>
</style>