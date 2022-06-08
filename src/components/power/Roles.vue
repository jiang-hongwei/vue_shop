<template>
<div>
  <!--   面包屑导航区-->
  <el-breadcrumb separator-class="el-icon-arrow-right">
    <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
    <el-breadcrumb-item>权限管理</el-breadcrumb-item>
    <el-breadcrumb-item>角色列表</el-breadcrumb-item>
  </el-breadcrumb>
<!--  卡片视图-->
  <el-card>
<!--    添加角色按钮区域-->
    <el-row>
      <el-col>
        <el-button type="primary">添加角色</el-button>
      </el-col>
    </el-row>
<!--    角色列表-->
    <el-table :data="roleList" border stripe>
<!--      展开列-->
      <el-table-column type="expand">
        <template v-slot="scope">
          <el-row :class="['bdbottom',i1 === 0 ? 'bdtop' : '','vcenter']" v-for="(item1,i1) in scope.row.children" :key="item1.id">
            <!--          渲染一级权限-->
            <el-col :span="5">
              <el-tag closable @close="removeRightById(scope.row,item1.id)"> {{item1.authName}}</el-tag>
              <i class="el-icon-caret-right"></i>
            </el-col>

          <!--          渲染二级和三级权限-->
          <el-col :span="19">
<!--           通过for循环嵌套 渲染二级权限-->
            <el-row v-for="(item2,i2) in item1.children" :key="item2.id" :class="[i2 === 0? '' : 'bdtop','vcenter']">
              <el-col :span="6">
                <el-tag type="success" closable @close="removeRightById(scope.row,item2.id)">{{item2.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <el-col :span="18">
                <el-tag type="warning" v-for="(item3,i3) in item2.children" :key="item3.id" closable @close="removeRightById(scope.row,item3.id)">{{item3.authName}}</el-tag>
              </el-col>
            </el-row>
          </el-col>
          </el-row>
        </template>
      </el-table-column>
<!--      索引列表-->
    <el-table-column type="index">
    </el-table-column>
    <el-table-column label="角色名称" prop="roleName"></el-table-column>
    <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
    <el-table-column label="操作">
      <template v-slot="scope" width="300px">
        <el-button type="primary" icon="el-icon-search" size="mini" @click="">编辑</el-button>
        <el-button type="danger" icon="el-icon-delete" size="mini">删除</el-button>
        <el-button type="warning" icon="el-icon-setting" size="mini" @click="showSetRightDialog(scope.row)">分配权限</el-button>
      </template>
    </el-table-column>
    </el-table>
  </el-card>
  <!--    分配权限对话框-->
  <el-dialog
    title="分配权限"
    :visible.sync="setRightDialogVisible"
    width="50%"
    @close="serRightDialogClosed"
  >
<!--    树形控件-->
    <el-tree :data="rightslist" :props="treeProps" show-checkbox node-key="id" default-expand-all :default-checked-keys="defKeys"
    ref="treeRef"></el-tree>
    <span slot="footer" class="dialog-footer">
    <el-button @click="setRightDialogVisible = false">取 消</el-button>
    <el-button type="primary" @click="allotRights">确 定</el-button>
  </span>
  </el-dialog>

</div>
</template>

<script>
export default {
  name: 'Roles',
  data(){
    return {
      //所有角色列表
      roleList:[],
    //  控制分配权限对话框显示与隐藏
      setRightDialogVisible: false,
      //所有权限的数据
      rightslist:[],
    //  树形控件的属性绑定对象
      treeProps: {
        label:'authName',
        children: 'children'
      },
      //默认选中的节点ID
      defKeys:[],
      //当前即将分配权显的角色ID
      roleId:''
    }
  },
  created () {
    this.getRolesList()
  },
  methods:{
    async getRolesList(){
      const {data : res} = await this.$http.get('roles')
      if (res.meta.status !== 200){
        return this.$message.error('获取角色列表失败')
      }
      this.roleList = res.data
      console.log(res.data)
    },
    //根据ID删除权限
    async removeRightById(role,rightId){
    // 弹框提示用户是否要删除
     const confirmResult = await this.$confirm(
       '此操作将永久删除该文件, 是否继续?',
       '提示',
       {
         confirmButtonText: '确定',
         cancelButtonText: '取消',
         type: 'warning'
       }
      ).catch(err => err)
      if(confirmResult !== 'confirm'){
        return this.$message.info('取消了删除')
      }
     const {data : res} = await  this.$http.delete(`roles/${role.id}/rights/${rightId}`)
      if(res.meta.status !== 200){
        return this.$message.error('删除权限失败了！')
      }
      role.children = res.data
    },
    //展示分配权限对话框
    async showSetRightDialog(role){
      this.roleId = role.id
      //获取所有权限数据
      const {data : res} = await this.$http.get('rights/tree')
      if(res.meta.status !== 200){
        return this.$message.error('获取权限数据失败')
      }
      //获取到的权限数据保存到data中
      this.rightslist = res.data
      console.log(this.rightslist)
      //递归获取三级节点打的ID
      this.getLeafKeys(role,this.defKeys)
      this.setRightDialogVisible = true
    },
    //通过递归!!获取角色下所有的三级权限id
    getLeafKeys(node,arr){
      if(!node.children){
        return arr.push(node.id)
      }

      node.children.forEach(item=>
      this.getLeafKeys(item,arr)
      )
    },
  //监听分配权限的close事件
    serRightDialogClosed(){
      this.defKeys = []
    },
  //点击角色分配权限
    async allotRights(){
      const key = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys()
      ]
      // console.log(key)
      const idStr = key.join(',')
      const {data : res} = await this.$http.post(`roles/${this.roleId}/rights`,{rids:idStr})
      if (res.meta.status !== 200){
        return this.$message.error('更新权限失败！')
      }
      this.$message.success('分配权限成功！')
      this.getRolesList()
      this.setRightDialogVisible = false
    }
  }
}
</script>

<style scoped>
.el-tag {
  margin: 7px;
}
.bdtop {
  border-top: 1px solid #eeeeee;
}
.bdbottom {
  border-bottom: 1px solid #eeeeee;
}
.vcenter {
  display: flex;
  align-items: center;
}
</style>
