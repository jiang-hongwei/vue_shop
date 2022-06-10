<template>
<div>
  <!--   面包屑导航区-->
  <el-breadcrumb separator-class="el-icon-arrow-right">
    <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
    <el-breadcrumb-item>商品管理</el-breadcrumb-item>
    <el-breadcrumb-item>商品分类</el-breadcrumb-item>
  </el-breadcrumb>
<!--  卡片视图区-->
  <el-card>
<!--    添加分类按钮-->
    <el-row>
      <el-col>
        <el-button type="primary" @click="showAddCateDialog">添加分类</el-button>
      </el-col>
    </el-row>
<!--    表格-->
  <tree-table class="treeTable" :data="catelist" :columns="columns" :selection-type="false" :expand-type="false" show-index index-text="#" border :show-row-hover="false">
<!--   是否有效 -->
    <template v-slot:isOk="scope" >
      <i class="el-icon-success" v-if="scope.row.cat_deleted === false" style="color: lightgreen"></i>
      <i class="el-icon-edit" v-else style="color: red"></i>
    </template>
<!--  排序  -->
    <template v-slot:order="scope">
    <el-tag size="mini" v-if="scope.row.cat_level === 0">一级</el-tag>
    <el-tag type="success" size="mini" v-else-if="scope.row.cat_level===1">二级</el-tag>
    <el-tag type="warning" size="mini" v-else>三级</el-tag>
    </template>
<!--    操作-->
    <template v-slot:opt="scope">
    <el-button type="primary" icon="el-icon-edit" size="mini">编辑</el-button>
    <el-button type="danger" icon="el-icon-search" size="mini">删除</el-button>
    </template>
  </tree-table>
<!--    分页-->
    <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="queryInfo.pagenum"
      :page-sizes="[1,2,3,10]"
      :page-size="queryInfo.pagesize"
      layout="total, sizes, prev, pager, next, jumper"
      :total="total">
    </el-pagination>
  </el-card>
<!--  添加分类的对话框-->
  <el-dialog
    title="添加分类"
    :visible.sync="addCateDialogVisible"
    width="50%"
    @close="addCateDialogClosed"
  >
<!--    添加分类的表单-->
    <el-form :model="addCateForm" :rules="addCateFormRules" ref="addCateFormRef" label-width="100px">
      <el-form-item label="分类名称" prop="cat_name">
        <el-input v-model="addCateForm.cat_name"></el-input>
      </el-form-item>
      <el-form-item label="父级分类">
        <el-cascader  :options="parentCateList" :props="cascaderProps" v-model="selectedKeys" @change="parentCateChanged" clearable > <!-- @change选中项发生改变就会触发 即移动鼠标就会触发 -->
        </el-cascader>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
    <el-button @click="addCateDialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="addCate">确 定</el-button>
  </span>
  </el-dialog>

</div>
</template>

<script>
export default {
  name: 'Cate',
  data(){
    return {
      //请求参数
      queryInfo:{
        type:3,
        pagenum: 1,
        pagesize: 5
      },
    //  商品分类数据列表
    catelist:[],
      //总数据条数
      total: 0,
      //为table指定列的定义
      columns:[
        {
          label:'分类名称',
          prop: 'cat_name'
        },
        {
          label: '是否有效',
          //表示将当前列定义为模板字符串
          type: 'template',
          //表示当前这一列使用的模板名称
          template: 'isOk'
        },
        {
          label: '排序',
          //表示将当前列定义为模板字符串
          type: 'template',
          //表示当前这一列使用的模板名称
          template: 'order'
        },
        {
          label: '操作',
          //表示将当前列定义为模板字符串
          type: 'template',
          //表示当前这一列使用的模板名称
          template: 'opt'
        }
      ],
      //控制添加分类对话框的显示
      addCateDialogVisible: false,
      //添加分类的白丹数据对象
      addCateForm: {
        //将要添加分类的名称
        cat_name: '',
        //父级分类ID
        cat_pid: 0,
        //分类等级，默认添加一级
        cat_level: 0
      },
      //添加分类表单的验证规则对象
      addCateFormRules:{
        cat_name:[
          { required: true, message: '请输入分类名称', trigger: 'blur' }
        ]
      },
      //父级分类列表
      parentCateList:[],
      // 指定级联选择器的配置对象
      cascaderProps: {
        expandTrigger: 'hover',/* 新版element要写在这里 */
        value: 'cat_id', /* 选中的是哪个属性 */
        label: 'cat_name', /* 显示的是哪个属性 */
        children: 'children' /* 嵌套的是哪个属性 */,
        checkStrictly: true
      },
      // 选中的父级分类的Id数组 要选两个 所以是数组
      selectedKeys: []
    }
  },
  created () {
    this.getCateList()
  },
  methods:{
  //  获取商品分类列表
    async getCateList(){
      const {data : res } = await this.$http.get('categories',{params : this.queryInfo})
      if (res.meta.status !== 200){
        return this.$message.error('获取商品分类失败')
      }
      //
      this.catelist = res.data.result
      this.total = res.data.total
    },
    //监听pagesize改变
    handleSizeChange(newSize){
      this.queryInfo.pagesize = newSize
      this.getCateList()
    },
    //监听pagenum的改变
    handleCurrentChange(newPage){
      this.queryInfo.pagenum = newPage
      this.getCateList()
    },
    //点击按钮展示添加分类对话框
    showAddCateDialog(){
      //先获取父级分类列表
      this.getParentCateList()
    this.addCateDialogVisible = true
    },
    //获取父级分类的数据列表
    async getParentCateList(){
      const {data : res} = await this.$http.get('categories',{params:{type:2}})
      if (res.meta.status !== 200){
        return this.$message.error('获取父级分类失败')
      }
      this.parentCateList = res.data
    },
    //选择项发展是变化
    parentCateChanged(){
    //  如果length大于0证明选中父级
      if (this.selectedKeys.length > 0) {
        //父级分类的ID
        this.addCateForm.cat_pid = this.selectedKeys[this.selectedKeys.length-1]
        //为当前分类的等级赋值
        this.addCateForm.cat_level = this.selectedKeys.length
      } else {
        //父级分类的ID
        this.addCateForm.cat_pid = 0
        //为当前分类的等级赋值
        this.addCateForm.cat_level = 0
      }
    },
    //点击按钮添加新的分类
    addCate(){
      this.$refs.addCateFormRef.validate(async valid=>{
        if(!valid) return
       const {data : res} = await this.$http.post('categories',this.addCateForm)

        if (res.meta.status !== 201) {
          this.$message.error('添加分类失败')
        }
        this.$message.success('添加分类成功')
        this.getCateList()
        this.addCateDialogVisible = false
      })

    },
    //监听对话框关闭事件，重置表单数据
    addCateDialogClosed(){
      /* 清空分类名称 即cat_name 下次打开就为空了 */
      this.$refs['addCateFormRef'].resetFields()

      //清空储存数据
      this.selectedKeys = []
      this.addCateForm.cat_pid = 0
      this.addCateForm.cat_level = 0
    }
  }
}
</script>

<style scoped lang="less">
.treeTable {
  margin-top: 15px;
}
.el-cascader {
  width: 100%;
}
</style>
