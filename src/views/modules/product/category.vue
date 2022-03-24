

<template>
  <div>
    <!-- 防止误拖拽，设置一个按钮 -->
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <!-- 避免拖拽一次与数据库交互一次，设置批量保存 -->
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <!-- expand-on-click-node点击按钮是否有展开功能 -->
    <!-- 动态绑定默认展开的菜单,用于删除细节优化 -->
    <!-- draggable="true"表示可拖拽    @node-drop托拽成功后响应的事件 -->
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox="true"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      @node-drop="handleDrop"
      :allow-drop="allowDrop"
      ref="menuTree"
    >
      <!-- 该span会让每个结点后面有增加、删除功能 -->
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
            edit
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

    <!-- 确定表单的布局,复用一个对话框   title去绑定title值,区分对话框表头  submitData区分确定是执行修改逻辑还是添加逻辑 -->
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件，如组件、工具js、第三方插件等

export default {
  components: {},
  props: {},
  data() {
    return {
      pCId: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: "",
      // 区分对话框是由edit还是append触发的
      dialogType: "", // edit add
      // append提交的详细数据category
      category: {
        productUnit: "",
        icon: "",
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
      },
      dialogVisible: false,
      // 默认展开的菜单数组
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  computed: {},
  watch: {},
  methods: {
    batchDelete() {
      // 批量删除的菜单的ID
      let catIds = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("checkedNodes", checkedNodes);
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      // 删除之前先弹框提示
      this.$confirm(`是否批量删除${catIds}菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() =>{
        this.$http({
        url:this.$http.adornUrl('/product/category/delete'),
        method:'post',
        data:this.$http.adornData(catIds, false)
        }).then(({data}) => {
          this.$message({
          message: "菜单批量删除成功",
          type: "success",
        });
        this.getMenus(); 
        });
      }).catch(() => {

      });
    },

    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序修改成功",
          type: "success",
        });
        // 刷新菜单
        this.getMenus();
        this.expandedKey = this.pCId;
        this.updateNodes = [];
        this.maxLevel = 0;
        // this.pCId = 0;
      });
    },

    // 拖拽成功后触发的函数,拖拽会影响到父节点id 自身层级 排序,进行处理
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", dropNode.label, dropType);
      // 1 当前结点最新的父节点的ID
      let pCId = 0;
      // 当前结点的兄弟结点
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        // 以兄弟关系、前后关系进入: 父亲结点为进入结点的父亲结点
        // 当一级目录互相拖入，当前拖入结点无父ID，undefined设置为0
        pCId =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
      } else {
        // 如果是拖进去，那就是当前结点的ID就是其父亲ID
        pCId = dropNode.data.catId;
      }
      this.pCId.push(pCId);
      // 2 当前拖拽结点的最新顺序:inner方式，其兄弟就是拖入结点的孩子结点
      if (dropType == "inner") {
        siblings = dropNode.childNodes;
      } else {
        // 若是前后拖入，其兄弟节点就是父亲结点的所有子节点
        siblings = dropNode.parent.childNodes;
      }
      for (let i = 0; i < siblings.length; i++) {
        // 将所有需要修改的结点保存起来
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 如果遍历的正好是当前拖拽的结点，要修改父亲结点这个属性
          let catLevel = draggingNode.level; // 当前结点的默认层级
          if (siblings[i].level != draggingNode.level) {
            // 当前结点层级信息发生变化
            catLevel = siblings[i].level;
            // 递归继续修改子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCId,
            catLevel: catLevel,
          });
        } else {
          // 其余兄弟结点直接修改排序就可以了
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      // 3 当前拖拽结点的最新层级
    },
    // 递归修改结点的层级关系
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          // cnode 当前数据
          var cnode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cnode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      //  被拖动的当前结点以及所在的父节点总层数不能大于3
      // 1) 计算被拖动的当前结点总层数
      // draggingNode当前结点   dropNode:目的节点  type: 拖动的位置
      console.log("allowDrop:", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode);
      // 当前正在拖动的结点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      // 根据拖动类型判断拖动逻辑
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    // 递归计算当前结点的最大子节点深度
    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    // 区分确定按钮进而调用addCatcory或edit方法
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      }
    },

    // 写一个获取菜单的方法
    getMenus() {
      // 发请求的模板函数
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
        //   then是成功发送请求后的函数
      }).then(({ data }) => {
        console.log("成功获取到数据", data.data);
        this.menus = data.data;
      });
    },
    append(data) {
      console.log("append", data);
      // 打开对话框
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.dialogType = "add";
      this.title = "添加分类";

      // 因edit后,查询数据库进行回显,将catcory的值都进行了更新,添加需要将catcory的值都恢复默认
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;
      this.category.name = "";
    },

    edit(data) {
      this.dialogType = "edit";
      console.log("要修改的数据", data);
      this.dialogVisible = true;
      // 发送请求获取当前结点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        // 请求成功以后, 将真正数据进行回显
        console.log("要回显的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.sort = data.data.sort;
        this.category.showStatus = data.data.showStatus;
      });

      this.title = "修改分类";
    },

    remove(node, data) {
      var ids = [data.catId];
      // 删除之前先弹框提示
      this.$confirm(`是否删除${data.name}菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            // 删除成功弹窗提示
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            // 删除成功后，更新菜单
            this.getMenus();
            // 设置默认需要展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {
          // 点击取消后的逻辑
        });

      console.log("remove", node, data);
    },
    editCategory() {
      // 将编辑确实需要的数据解析出来封装发给后端,非必要数据就不发送了
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    // 点击append对话框的确定按钮后,触发添加三级分类的方法
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
  },
  // 第一次调用获取菜单的函数
  created() {
    this.getMenus();
  },
  mounted() {},
  beforeCreate() {},
  beforeMount() {},
  beforeUpdate() {},
  updated() {},
  beforeDestroy() {},
  destroyed() {},
  activated() {},
};
</script>
<style  scope>
</style>