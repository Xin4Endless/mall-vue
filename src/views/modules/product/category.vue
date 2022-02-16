<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree ref="menuTree"
             :allow-drop="allowDrop"
             :data="categoryData"
             :default-expanded-keys="expandedKey"
             :draggable="draggable"
             :expand-on-click-node="false"
             :props="defaultProps"
             node-key="catId"
             show-checkbox
             @node-drop="handleDrop">
      <span slot-scope="{ node, data }" class="custom-tree-node">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            size="mini"
            type="text"
            @click="() => append(data)">
            添加
          </el-button>
           <el-button
             size="mini"
             type="text"
             @click="() => update(data)">
            修改
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            size="mini"
            type="text"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :close-on-click-modal="false"
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称：" label-width="20%">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标：" label-width="20%">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位：" label-width="20%">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitBtn">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
export default {
  data() {
    return {
      updateNodes: [],
      pCid: [],
      draggable: false,
      category: {
        catId: null,
        name: '',
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: '',
        icon: ''
      },
      maxLevel: 0,
      categoryData: [],
      expandedKey: [],
      dialogVisible: false,
      dialogType: '',
      dialogTitle: '',
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  created() {
    this.getCategoryData()
  },
  methods: {
    batchDelete() {
      let catIds = []
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      console.log('被选中的元素', checkedNodes)
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId)
      }
      this.$confirm(`是否批量删除【${catIds}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              message: '菜单批量删除成功',
              type: 'success'
            })
            this.getCategoryData()
          })
        })
        .catch(() => {
        })
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({data}) => {
        this.$message({
          message: '菜单顺序等修改成功',
          type: 'success'
        })
        // 刷新出新的菜单
        this.getCategoryData()
        // 设置需要默认展开的菜单
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.maxLevel = 0
        // this.pCid = 0;
      })
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)
      // 1、当前节点最新的父节点id
      let pCid = 0
      let siblings = null
      if (dropType === 'before' || dropType === 'after') {
        pCid =
          dropNode.parent.data.catId === undefined
            ? 0
            : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)

      // 2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId === draggingNode.data.catId) {
          // 如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level
          if (siblings[i].level !== draggingNode.level) {
            // 当前节点的层级发生变化
            catLevel = siblings[i].level
            // 修改他子节点的层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({catId: siblings[i].data.catId, sort: i})
        }
      }

      // 3、当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // 1、被拖动的当前节点以及所在的父节点总层数不能大于3

      // 1）、被拖动的当前节点总层数
      console.log('allowDrop:', draggingNode, dropNode, type)
      //
      this.countNodeLevel(draggingNode)
      // 当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度：', deep)

      //   this.maxLevel
      if (type === 'inner') {
        // console.log(
        //   `this.maxLevel：${this.maxLevel}；draggingNode.data.catLevel：${draggingNode.data.catLevel}；dropNode.level：${dropNode.level}`
        // );
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel(node) {
      // 找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    getCategoryData() {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => {
        console.log(data.data)
        this.categoryData = data.data
      })
    },
    submitBtn() {
      if (this.dialogType === 'add') {
        this.addCategory()
      } else if (this.dialogType === 'update') {
        this.updateCategory()
      }
      // this.category = {}
    },
    append(data) {
      this.dialogVisible = true
      this.dialogType = 'add'
      this.dialogTitle = '添加分类'
      this.category.parentCid = data.catId
      this.category.catLevel = parseInt(data.catLevel) + 1
    },
    update(data) {
      this.dialogVisible = true
      this.dialogType = 'update'
      this.dialogTitle = '修改分类'
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get',
        params: this.$http.adornParams({})
      }).then(({data}) => {
        this.category = data.category
      })
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        this.$message.success('保存成功')
        this.dialogVisible = false
        this.getCategoryData()
        this.expandedKey = [this.category.parentCid]
      })
    },
    updateCategory() {
      // 解构
      var {catId, name, icon, productUnit} = this.category
      // K、V一致的时候可以省略 :V
      var data = {catId, name, icon, productUnit}
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData(data, false)
      }).then(({data}) => {
        this.$message.success('保存成功')
        this.dialogVisible = false
        this.getCategoryData()
        this.expandedKey = [this.category.parentCid]
        this.category = {}
        console.log(this.expandedKey)
      })
    },
    // node 当前节点的信息
    remove(node, data) {
      var ids = [data.catId]
      this.$confirm(`是否删除 ${data.name} 分类`, '提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() =>
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            this.$message.success('删除成功')
            // 刷新菜单
            this.getCategoryData()
            // 设置默认展开节点
            this.expandedKey = [node.parent.data.catId]
          }).catch(({data}) => {
            this.$message.error('删除失败')
          })
        )
        .catch()
    }
  }
}
</script>
