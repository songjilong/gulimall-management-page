<!--  -->
<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" type="success" @click="batchSave" round>提交拖拽</el-button>
    <el-button type="danger" @click="batchRemove" round>批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">添加</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">修改</el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >删除</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form ref="form" :model="category" label-width="80px">
        <el-form-item label="分类名称">
          <el-input v-model="category.name"></el-input>
        </el-form-item>
        <el-form-item label="图标地址">
          <el-input v-model="category.icon"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit"></el-input>
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
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';
export default {
  data() {
    return {
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name"
      },
      dialogVisible: false,
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: ""
      },
      title: "",
      submitType: "",
      maxLevel: 0,
      updateNodes: [],
      draggable: false,
      pCid: []
    };
  },
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get"
      }).then(({ data }) => {
        this.menus = data.data;
      });
    },
    submitData() {
      if (this.submitType == "add") {
        this.addCategory();
      }
      if (this.submitType == "edit") {
        this.editCategory();
      }
    },
    //添加三级菜单
    append(data) {
      //打开弹出框
      this.dialogVisible = true;
      this.title = "添加三级菜单";
      this.submitType = "add";
      //设置数据
      this.category.catId = null;
      this.category.name = "";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.category.icon = "";
      this.category.productUnit = "";
    },
    addCategory() {
      //发送请求添加数据
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false)
      })
        .then(({ data }) => {
          //关闭弹出框
          this.dialogVisible = false;
          //显示成功消息
          this.$message.success("菜单添加成功");
          //刷新页面
          this.getMenus();
          //展开当前父菜单
          this.expandedKey = [this.category.parentCid];
        })
        .catch(() => {
          this.$message.error("菜单添加失败");
        });
    },
    //修改三级菜单数据
    edit(data) {
      //打开弹出框
      this.dialogVisible = true;
      this.title = "修改三级菜单";
      this.submitType = "edit";
      //查出当前菜单数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get"
      }).then(({ data }) => {
        console.log("要修改的数据：", data);
        //展示数据
        this.category.catId = data.data.catId;
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    editCategory() {
      this.submitType = "edit";
      //解构出修改需要的数据
      const { catId, name, icon, productUnit } = this.category;
      console.log("解构的数据", { catId, name, icon, productUnit });
      //提交修改的数据
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      })
        .then(({ data }) => {
          //关闭弹出框
          this.dialogVisible = false;
          //显示成功消息
          this.$message.success("菜单修改成功");
          //刷新页面
          this.getMenus();
          //展开当前父菜单
          this.expandedKey = [this.category.parentCid];
        })
        .catch(() => {
          this.$message.error("菜单修改失败");
        });
    },
    //删除三级菜单
    remove(node, data) {
      let ids = [data.catId];
      this.$confirm(`是否删除「 ${data.name} 」?`, "警告", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false)
          })
            .then(({ data }) => {
              this.$message.error("菜单删除成功");
              //刷新页面
              this.getMenus();
              //展开当前父菜单
              this.expandedKey = [node.data.parentCid];
            })
            .catch(() => {
              this.$message.error("菜单删除失败");
            });
        })
        .catch(() => {
        });
      console.log("删除节点", node, data);
    },
    //批量保存
    batchSave() {
      //发送请求更新数据
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.successMsg("菜单拖动成功");
        //刷新页面
        this.getMenus();
        //展开当前父菜单
        this.expandedKey = this.pCid;
        //重置数据
        this.maxLevel = 0;
        this.updateNodes = [];
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      //被拖动结点的深度 + 放置到的结点的层级 <= 3 即可拖动
      // console.log("draggingNode, dropNode, type",draggingNode, dropNode, type);
      //算出当前结点下的最大层级
      this.nodeMaxLevel(draggingNode);
      //深度 = 最大层级 - 当前层级 + 1
      console.log("最大层级", this.maxLevel);
      console.log("拖拽的层级", draggingNode.level);
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log("深度", deep);
      if (type == "inner") {
        //目标层级 + 深度 是否小于等于3
        return dropNode.level + deep <= 3;
      } else {
        return dropNode.parent.level + deep <= 3;
      }
    },
    //算出当前结点下的最大层级
    nodeMaxLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          //递归计算子结点
          this.nodeMaxLevel(node.childNodes[i]);
        }
      } else {
        this.maxLevel = node.level;
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      //更新拖拽结点的父节点
      let pCid = 0;
      let siblings = null;
      //根据拖入的位置不同得到不同的父id、兄弟结点
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        pCid = dropNode.data.parentCid;
        siblings = dropNode.parent.childNodes;
      }
      this.pCid.push(pCid);
      //对结点排序
      for (let i = 0; i < siblings.length; i++) {
        //如果遍历的是正在拖拽的结点
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          //如果层级改变，修改为真实层级
          if (siblings[i].level != draggingNode.level) {
            //修改当前层级
            catLevel = siblings[i].level;
            //修改子结点层级
            this.updateChildrenLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log(this.updateNodes);
    },
    updateChildrenLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          this.updateNodes.push({
            catId: node.childNodes[i].data.catId,
            catLevel: node.childNodes[i].level
          });
          this.updateChildrenLevel(node.childNodes[i]);
        }
      }
    },
    batchRemove() {
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log(checkedNodes);
      let catIds = [];
      let names = [];
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
        names.push(checkedNodes[i].name);
      }
      this.$confirm(`是否删除「 ${names} 」?`, "警告", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false)
          })
            .then(({ data }) => {
              this.$message.success("批量删除成功");
              this.getMenus();
              this.expandedKey = [checkedNodes[0].parentCid];
            })
            .catch(() => {
              this.$message.error("批量删除失败");
            });
        })
        .catch(() => {
        });
    }
  },
  //import引入的组件需要注入到对象中才能使用
  components: {},
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    // this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  //如果页面有keep-alive缓存功能，这个函数会触发
  activated() {
    this.getMenus();
  }
};
</script>

<style>
.custom-tree-node {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 15px;
  padding-right: 10px;
}
</style>