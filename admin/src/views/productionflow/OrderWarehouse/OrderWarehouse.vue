<template>
  <div>
    <!-- 库房备料 -->
    <el-row>
      <el-col :span="2">
        <!-- 选择合同及合同子表中的产品信息 -->
        <OrderConstractProductSel
          @refreshTableList="refreshTableList">
        </OrderConstractProductSel>
      </el-col>

      <el-col :span="22">
          <div class="icon">
            <i class="el-icon-view"><span>查看详情</span></i>
            <i class="el-icon-bell"><span>通知钉钉</span></i>
            <i class="el-icon-warning-outline"><span>展示</span></i>
            <i class="el-icon-date"><span>MRP确认</span></i>
          </div>
        <el-row><el-tag class="ordre-tag">当前选择订单合同:{{selectedConstractData == null ? '' : selectedConstractData.contractNo}}</el-tag></el-row>
        <!--销售发起备忘录数据-->
        <Table :columns="columns" 
          :select="select" 
          :top_btn="top_btn" 
          :act_btn="act_btn" 
          :replace="replace"
          @onButtonClick="onButtonClick">
        </Table>
      </el-col>
    </el-row>

    <!--库房备料详情Dialog-->
    <OrderWarehouseContent
      :replace="replace"
      :visiable="content.visiable" 
      :title="content.title"
      :updateRow="content.updateRow"
      @close="closeContent">
    </OrderWarehouseContent>

    <!--送审框Dialog通知钉钉-->
    <CommitDialog
      :visiable="commitDialogData.visiable"
      :paramData="commitDialogData.paramData"
      @close="closeCommitDialog"
      @commitMethod="executeDindDingInterface"
    ></CommitDialog>
    
    <!--订单字表详情数据-->
    <OrderProductInfoDetails
      :visiable="orderProductInfoDetails.visiable"
      :pkProductOrderB="orderProductInfoDetails.pkProductOrderB"
      @close="() => { orderProductInfoDetails.visiable = false }">
    </OrderProductInfoDetails>

  </div>
</template>

<script>
import Table from '@/components/viewcomp/Table'
import OrderWarehouseContent from "@/views/productionflow/OrderWarehouse/content"
import OrderConstractProductSel from "@/views/productionflow/comp/OrderConstractProductSel"
import OrderProductInfoDetails from "@/views/productionflow/comp/OrderProductInfoDetails"
import CommitDialog from "@/components/Commitdingding/CommitDialog";
import { sendDingMsgByPsndoc } from "@/api/pub/pub"; // 调用钉钉后端API接口
import { selectAllMaterielWareHouseAsPage, updateMaterielWareHouse } from '@/api/orderCompMage/OrderWarehouse'

export default {
  name: "ProductionFlowOrderWarehouseView",
  components: { Table, CommitDialog, OrderConstractProductSel, OrderProductInfoDetails, OrderWarehouseContent },
  data() {
    return {
      content: {
        visiable: false, // 内容界面显示控制
        title: '', // 标题
        action: null, // 执行Action
        updateRow: null, // 更新时带过来的temp
        pkProductOrder: null // 单据业务属性 合同订单主键
      },
      columns: [
        {prop: 'pkOmgWareHouse', label: '库房备料主键', primary: true},
        {prop: 'pkProductOrderB', label: '合同子表主键', show: false},
        {prop: 'contractNo', label: '合同编号', width: '110'},
        {prop: 'productCode', label: '产品编号', width: '120'},
        {prop: 'productName', label: '产品名称', width: '110'},
        {prop: 'customerCode', label: '客户编号', width: '110'},
        {prop: 'constractOrderType', label: '合同类型', width: '110'},
        {prop: 'productionState', label: '生产状态', width: '100'},
        {prop: 'constractProductNum', label: '订单数量', width: '110'},
        {prop: 'scheduledtime', label: '预计交货日期', width: '180'},
        {prop: 'billState', label: '单据状态', width: '110'},
        {prop: 'fileName', label: '文件名称'}
      ],
      select: {
        'select': selectAllMaterielWareHouseAsPage,
        'queryParam': { dr: 0 }
      },
      top_btn: [
      ],
      act_btn: [
        {
          width: 140,
          name: '查看产品数据',
          code: 'LOOKORDERBPRODUCT',
          action: null,
          typeLess: 'itag',
          iconPic: 'el-icon-view',
          btnSize: 'mini'
        },
        {
          name: '通知钉钉',
          code: 'COMMITDINGDING',
          action: null,
          typeLess: 'itag',
          iconPic: 'el-icon-bell',
          btnSize: 'mini'
        },
        {
          name: '详情',
          code: 'DETAILS',
          action: null,
          typeLess: 'itag',
          iconPic: 'el-icon-warning-outline',
          btnSize: 'mini'
        },
        {
          name: 'MRP运算',
          code: 'COUNTMRP',
          when: {'billState': [1]},
          typeLess: 'itag',
          iconPic:'el-icon-date',
          action: null,
          btnSize: 'mini'
        }
      ],
      replace: [
        {name: 'constractOrderType',replace: {'0':"普通合同",'1':"生产订单",'2':"OEM",'3':"DEM",'4':"贸易模式"}},
        {name: 'billState',replace: {'0':"自由态",'1':"进行中",'2':"已完成"}},
        {name: 'productionState',replace: {'0':"自由态",'1':"生产中",'2':"暂停(未启用)",'3':"生产完成",'4':"一键暂停"}}
      ],
      commitDialogData: {
        // 控制钉钉Dialog显示
        visiable: false,
        paramData: null
      },
      orderProductInfoDetails: {
        visiable: false,
        pkProductOrderB: null // 订单子表主键
      },
      selectedConstractData: null, // 当前选中的订单合同
    };
  },
  methods: {
    // 调用接口获取界面数据
    refreshTableList(constractData) {
      if (constractData) {
        this.selectedConstractData = constractData
        let queryParam = {
          pkProductOrder: constractData.pkProductOrder,
          dr: 0
        }
        let newSelect = {select: this.select.select, queryParam: queryParam }
        this.select = newSelect
      }
    },
    // 监听按钮
    onButtonClick(param, btnCode, unexParam) {
      if (param.btnCode === "LOOKORDERBPRODUCT") { // 查看订单中的产品详情
        this.orderProductInfoDetails.pkProductOrderB = param.pkProductOrderB
        this.orderProductInfoDetails.visiable = true
      }
      if (param.btnCode === "COMMITDINGDING") { // 推送消息到钉钉
        this.commitDialogData.visiable = true;
      }
      if (param.btnCode === "DETAILS") {
        this.content.updateRow = param.bindData
        this.openContent('库房备料', null)
      }
      if (param.btnCode === 'COUNTMRP') { // MRP运算
        if (param.bindData.pkOmgWareHouse) {
          let mrpParam = {
            pkOmgWareHouse: param.bindData.pkOmgWareHouse,
            billState: '2'
          }
          updateMaterielWareHouse(mrpParam).then((response) => {
            if (response && response.success) {
              this.$message({ showClose: true, message: '操作成功!', type: 'success' });
              this.refreshTableList(this.selectedConstractData)
            } else {
              this.$message({ showClose: true, message: `${response.errorMessage}!`, type: 'warning' });
            }
          })
        }
      }
    },
    openContent(title, action) { // 打开内容界面Dialog
      this.content.visiable = true
      this.content.title = title
      this.content.action = action
    },
    closeContent() { // 关闭内容界面Dialog
      this.refreshTableList(this.selectedConstractData)
      this.content.visiable = false
    },
    closeCommitDialog() {
      this.commitDialogData.visiable = false;
    },
    executeDindDingInterface(sendMsgParam) {
      // 调用钉钉接口
      sendDingMsgByPsndoc(sendMsgParam).then(response => {
        if (response && response.data.success) {
          // 调用接口后处理
          alert("调用钉钉接口成功");
        } else {
          alert("调用钉钉接口失败");
        }
      });
    }
  }
};
</script>

<style scoped>
.ordre-tag{
  margin-left: 10px;
  margin-top: 16px;
}
.icon{
  float: right;
  margin-top: 20px;
  margin-right: 500px;
  color: #409EFF;
}
div{
  font-size: 8px;
  color:  #606266;
}
i{
  font-size: 15px;
  padding-left: 10px;
  cursor:pointer
}
i span{
  font-size: 8px;
  padding-left: 5px;
}
i:hover{
  color: #409EFF;
  cursor: pointer;
}
</style>