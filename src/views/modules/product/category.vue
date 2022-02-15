<template>
  <div>
    <el-tree :data="categoryData"
             :props="defaultProps"
             node-key="catId"
             show-checkbox
             draggable
             :expand-on-click-node="false"
             :default-expanded-keys="expandedKey"
             :allow-drop="allowDrop">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)">
            添加
          </el-button>
           <el-button
             type="text"
             size="mini"
             @click="() => update(data)">
            修改
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)">
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="dialogTitle"
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
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
    allowDrop(draggingNode, dropNode, type) {

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
        params: this.$http.adornParams({
        })
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
