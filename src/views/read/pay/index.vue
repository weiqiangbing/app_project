<template>
  <div class="app-container pay-wrap">	 
    <!-- 筛选 -->
    <filter-inputs ref="filterDom" @handleFilter="handleFilter" @handleDownload="handleDownload" @handleShowRow="handleShowRow" @handleShowEcharts="handleShowEcharts" :tableData="tableData" :checkedList="checkedList" :downloadLoading="downloadLoading" :filterConfig="pageConfig.payData" :groupListData="groupListData" @getEchartData="getEchartData" :echartDatas="echartDatas"></filter-inputs>
    <!-- 显示隐藏列 -->
    <!-- <div v-show="isShowRow" class="checked-show-row-list">
      <el-checkbox-group 
        v-model="checkedList"
        @change="checkListChange">
        <el-checkbox v-for="(item) in defaultCheckedList" :label="item.prop" v-bind:key="item.prop">{{item.label}}</el-checkbox>
      </el-checkbox-group>
    </div>     -->
    <el-dialog
      title="请选择你想要显示的列进行勾选"
      class="check_th_clss"
      :modal="false"
      top="5vh"
      :visible.sync="isShowRow"
      width="620px">
      <el-checkbox-group 
        v-model="checkedList"
        @change="checkListChange">
        <el-checkbox v-for="(item) in defaultCheckedList" :label="item.prop" v-bind:key="item.prop" v-if="filterAuth(item.prop)">{{item.prop=='fc'?'推广渠道':item.label}}</el-checkbox>
      </el-checkbox-group>
    </el-dialog>
    <!-- 主题内容 -->
    <div class="main-table" v-loading="loading">    
      <el-table
        class="border-top-20"
        v-if="!groupChaged"
        show-summary
        :summary-method="getSummaries"
        :data="tableData"
        @sort-change="sortChange"       
        style="width: 100%">
        <!-- <template v-for="(item) in tableColumn">
          <el-table-column
            v-bind:key="item.prop"
            v-if="checkedSum[item.prop] && show_field.indexOf(item.prop) != -1"
            :prop="item.prop"
            :label="item.prop=='fc'?'推广渠道':item.label"
            :show-overflow-tooltip="true"
            :fixed="fixedColumn(item)"
            :width="item.width ? item.width : ''"
            :formatter="formatTdValue"
            :render-header="(h, obj)=>handleRenderHeader(h,obj,item)"
            :sortable="sortSum[item.prop] ? 'custom' : false"
            align="center">
          </el-table-column>          
        </template> -->

        <template v-for="key in show_field">
          <el-table-column
            v-for="item in tableColumn"
            :key="item.prop"
            v-if="item.prop==key && checkedList.indexOf(item.prop) != -1 && filterAuth(item.prop)"
            :prop="item.prop"
            :label="item.prop=='fc'?'推广渠道':item.label"
            :show-overflow-tooltip="true"
            :fixed="fixedColumn(item)"
            :width="item.width ? item.width : ''"
            :formatter="formatTdValue"
            :render-header="(h, obj)=>handleRenderHeader(h,obj,item)"
            :sortable="sortSum[item.prop] || item.prop.indexOf('field_') != -1 ? 'custom' : false"
            align="center">
          </el-table-column>
        </template>
      </el-table>
    </div>
    <!-- 分页 -->
    <div class="pagination">
      <el-pagination
        v-if="refreshPagination"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :background="true"
        :page-sizes="[10, 20, 50, 100,200]"
        :page-size="pageCount"
        :current-page="currentPage"                       
        layout="total, sizes, prev, pager, next, jumper"
        :total="total_nums">
      </el-pagination>
    </div>
    <!-- 图表弹框 -->
    <chart-line ref="chartLine" :tableData="tableData" :isShowSelect="isShowSelect" :listQuery="listQuery" :checkedList="checkedList" @closeFn="closeFn"></chart-line>
  </div>	
</template>

<script>
import FilterInputs from '@/components/FilterInputs'
import ChartLine from '@/components/Charts/index'
import { getCheckedZh, formatJson, addTremParams, filterObjToArr } from '@/utils/common'
import { getList } from '@/api/read/pay'
import { pageConfig } from '@/utils/pageConfig'
import { tableColumns } from '@/utils/tableColumns'
import { pageTableConfig } from '@/utils/pageTableConfig'
import { setPayList, getPayList, removePayList} from '@/utils/auth'
import {getSpecialName} from '@/utils/assist'
import { pageSelectTh } from '@/utils/selectedThs'
import {getPageAuth} from '@/utils/power'

export default {
  name: 'Pay',
  components: {
    FilterInputs,
    ChartLine
  },
	data() {
		return {
      loading: true,
      groupChaged: false,
      oldGroups:'',
      totalInfos:{},
      isShowRow: false,
      isShowSelect: false,		
      downloadLoading: false, // 导出状态
      refreshPagination: true, // 页数增加的时候，因为缓存的缘故，需要刷新Pagination组件
			total_nums: 0, // 分组内的用户总数
			currentPage: 1, // 默认获取第一页的数据
      pageCount: 10, // 每页10条数据     
      groupListData: [],
      defaultCheckedList: [], // 隐藏列默认值      
      checkedList: [],
      tableData: [],  
      echartDatas:[],
      allData: [], // 导出数据
      show_field: [], // 本次返回数据所包含的列
			tableColumn: [...tableColumns],
      listQuery: {},
      siteList1: {},
      new_show_fields:[],
      authArrData:[],
      pageConfig: JSON.parse(JSON.stringify(pageConfig)),
      noRepeatSum: JSON.parse(JSON.stringify(pageTableConfig.payData.noRepeatSum)),
      sortSum: JSON.parse(JSON.stringify(pageTableConfig.payData.sortSum)), 
		}
	},
	created() {
    let pageCount = localStorage.getItem('ind_pay_pagecount')
    if(pageCount){
      this.pageCount = Number(pageCount)
    }  
    // this.listData()	
  },
  mounted() {
    this.siteList1 = this.$refs.filterDom.siteList1
  },
	methods: {  
    // 去空值
    filterNullVal(obj){
      const newObj = {}
      for (const key in obj) {        
        if(obj[key] != ''){
          newObj[key] = obj[key]
        }
      }
      return newObj
    }, 
    fixedColumn(item){
      if(this.listQuery.group){
        if(this.listQuery.group.indexOf(item.prop) != -1){
          return true
        }
      }
      return false
    }, 
    filterAuth(key){
      return getPageAuth(key, this.authArrData)
    }, 
    computeThs(num){
      this.defaultCheckedList = []
      this.tableColumn = [...tableColumns]
      let hasArr = []
      let reteArr = []
      if(num){
        this.new_show_fields.forEach(r=>{
          this.tableColumn.push({label: r.value, prop: r.key})
          reteArr.push({label: r.value, prop: r.key})
          hasArr.push(r.key)
        })  
      }

      for (let i = 0; i < this.show_field.length; i++) {
        for (let j = 0; j < this.tableColumn.length; j++) {
          if(this.show_field[i] == this.tableColumn[j].prop && hasArr.indexOf(this.tableColumn[j].prop)==-1){
            this.defaultCheckedList.push({prop: this.tableColumn[j].prop, label: this.tableColumn[j].label})
          }
        }
      }
      this.defaultCheckedList = this.defaultCheckedList.concat(reteArr)
      if(getPayList().length > 0){
        this.checkedList = getPayList()
      }else{
        this.checkedList = pageSelectTh.paydataTh
      } 
    },    
    listData(isEch) {		
      const currentPage = this.currentPage - 1
      this.loading = true
      let params = {
        page: currentPage,
        count: this.pageCount,
        start_time: this.listQuery.start_time,
        end_time: this.listQuery.end_time,
        order: this.listQuery.order,
        book_id: this.listQuery.book_id, 
        site_id: this.listQuery.site_id,
        pay_channel: this.listQuery.pay_channel,
        platform: this.listQuery.platform,
        time_type: this.listQuery.time_type,
        group: this.listQuery.group,  
        time_unit: this.listQuery.time_unit,
        user_type: this.listQuery.user_type,
        user_timeline: this.listQuery.user_timeline,
        searches_user: this.listQuery.searches_user,
        searches_book: this.listQuery.searches_book,
      }
      if(isEch){
        params.count = 2500
        params.order = ''
      }
      if(this.listQuery.group != this.oldGroups){
        this.groupChaged = true
        this.$nextTick(()=>{
          this.groupChaged = false
        })
      }
      let tremPraam = addTremParams(this.listQuery)
      params = Object.assign(params, tremPraam)
      const newParams = this.filterNullVal(params)

        getList(newParams).then(res => {          
          if(!isEch){
            this.oldGroups = this.listQuery.group
            this.tableData = [...res.data.collection]
            this.show_field = filterObjToArr(res.data.show_fields)
            this.groupListData = res.data.group_list ? [...res.data.group_list]	: []
            this.total_nums = res.data.total_nums
            this.totalInfos = res.data.total_info
            this.new_show_fields = res.data.new_show_fields
            if(res.data.new_show_fields && res.data.new_show_fields.length>0){
              this.computeThs(1)
            }else{
              this.computeThs(0)
            }
          }else{
            this.echartDatas = [...res.data.collection]
            this.$nextTick(()=>{
              this.$refs.filterDom.handleShowEcharts()
            })
            if(res.data.total_nums>2500){
              this.$message.warning('还有'+(res.data.total_nums-2500)+'条数据未统计到')
            }
          } 
        setTimeout(() => {
          this.loading = false
        }, 250)	
      }).catch(err => {
        this.loading = false
      })   
    },    
    checkListChange(){
      setPayList(this.checkedList)
    }, 
    handleFilter(listQuery){
      listQuery.order = 'pay_money desc'
      this.currentPage = 1
      this.listQuery = {...listQuery}
      this.listData()
    },
    getEchartData(filter){
        this.listQuery = filter
        this.listData(true)
    },
    // ===== 导出数据 start =====
    handleDownload(data) {
      // console.log(this.checkedList);
      // return
      let obj = getCheckedZh(this.defaultCheckedList, this.checkedList)
      const currentPage = this.currentPage - 1
      const checkedEnStr = obj.checkedEn.join(',')
      this.downloadLoading = true
      let params = {
        page: currentPage,
        count: -1,
        //field: checkedEnStr,
        start_time: data.start_time,
        end_time: data.end_time,
        order: data.order,
        book_id: data.book_id, 
        site_id: data.site_id,
        pay_channel: data.pay_channel,
        platform: data.platform,
        time_type: data.time_type,
        group: data.group,  
        time_unit: data.time_unit,
        user_type: data.user_type,
        user_timeline: data.user_timeline,
        searches_user: data.searches_user,
        searches_book: data.searches_book,
        outsite: data.outsite,
        section_id: data.section_id,
        class_id: data.class_id,
        author_name: data.author_name
      }
      let tremPraam = addTremParams(data)
      params = Object.assign(params, tremPraam)
      params.download_field = this.checkedList.join(',') 
      const newParams = this.filterNullVal(params)
      getList(newParams).then(res => { 
        // console.log('res', res)
        const blob = new Blob([res])     
        const fileName = '充值统计.csv'   
        if('download' in document.createElement('a')){
          const elink = document.createElement('a')
          elink.download = fileName
          elink.style.display = 'none'
          elink.href = URL.createObjectURL(blob)
          document.body.appendChild(elink)
          elink.click()
          URL.revokeObjectURL(elink.href)
          document.body.removeChild(elink)
        }else{
          navigator.msSaveBlob(blob, fileName)
        }
        this.downloadLoading = false
      }).catch(err => {
        this.downloadLoading = false
        this.$message.warning('导出失败！')
      })  
    },
    getSummaries(param) {
        const { columns, data } = param;
        const sums = [];
        columns.forEach((column, index) => {
          if (index === 0) {
            sums[index] = '总汇';
            return;
          }
          for(let i in this.totalInfos){
            if(column.property==i){
              sums[index] = this.totalInfos[i]
            }
          }
        });
        return sums;
      },
    // ===== 导出数据 end =====
    handleShowRow(isShowRow){
      this.isShowRow = isShowRow
    },
    formatTdValue(row,col,val){
      return getSpecialName(col.property, val)
    },
    handleShowEcharts(groupStr){ 
      if(this.tableData.length){
        this.isShowSelect = true
      }else{
        this.$message.warning('查询数据后方可绘图')
      }
    },  
    closeFn(){
      this.isShowSelect = false
    },
    // 设置table分页条数
    handleSizeChange(val){
      localStorage.setItem('ind_pay_pagecount', val)
      this.currentPage = 1
      this.pageCount = val
			this.loading = true
			this.listData()
			// this.loading = false
    },
    // 切换table分页
		handleCurrentChange(val) {
			this.currentPage = val
			this.loading = true
			this.listData()
			// this.loading = false
    },
    sortChange(data) {    
      this.$nextTick(()=>{
        const { prop, order } = data
        let orders = ''
        if (order === 'ascending') {
          orders = 'asc'
        }else if(order === 'descending'){
          orders = 'desc'
        }else if(prop == null){
          this.listQuery.order = '' 
          this.currentPage = 1
          this.listData()    
          return false
        }
        this.listQuery.order = prop + ' ' + orders
        this.currentPage = 1
        this.listData()
      })      
    },
    handleRenderHeader(h, { column, $index }, item){
      return h(
        'div',{
          style: 'padding: 0;margin: 0;text-align: left;white-space:normal; word-break:break-all;line-height: 1;position: relative;top: 3px;'
        },
        [ 
          h('span', column.label),
          h('el-popover', 
            { props: { placement: 'top', width: '200',trigger: 'hover', content: item.placeholder?item.placeholder:column.label }},
            [
              h('span', { slot: 'reference',class:'font-normal'},[
                  h('i', {class:'el-icon-question',style: 'color:#409eff;margin-left:5px;font-size:15px;'}),
              ])
            ]
          )
        ]
      )
    },

	}
}
</script>

<style lang="scss">
.pay-wrap{
  .el-icon-arrow-down {
    font-size: 12px;
  }
  .demonstration {
    display: block;
    color: #8492a6;
    font-size: 14px;
    margin-bottom: 20px;
  }
  .table{
    padding-top: 20px;

  }
  .pagination {
    display: flex;
    justify-content: flex-start;
    margin-top: 20px;
  }
  .border-top-20{
    padding-top: 10px;
  }
  .el-dialog__body{
    /* min-height: 400px; */
    padding: 5px 30px 20px;
  }
  .checked-show-row-list{
    padding-top: 30px;
  }
  .el-checkbox__label{
    padding-left: 5px;
  }
  .check_th_clss {
    /deep/ .el-checkbox{
      width: 110px;
    }
  }
}
</style>
