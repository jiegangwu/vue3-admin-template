<template>
  <div class="errorLogContainer">
    <!--操作-->
    <div class="mr-3px rowSS">
      <el-button type="primary" @click="errorLogProd">错误日志测试</el-button>
      <el-button type="primary" @click="multiDelBtnClick">
        <!-- 感觉写法复杂了-->
        <el-icon style="vertical-align: middle">
          <Delete />
        </el-icon>
        <span style="vertical-align: middle">删除</span>
      </el-button>
      <!--条件搜索-->
      <el-form ref="refsearchForm" :inline="true" class="demo-searchForm ml-2">
        <el-form-item label-width="0px" label="" prop="errorLog" label-position="left">
          <el-input v-model="searchForm.errorLog" class="w-150px" placeholder="错误日志" />
        </el-form-item>
        <el-form-item label-width="0px" label="" prop="pageUrl" label-position="left">
          <el-input v-model="searchForm.pageUrl" class="w-200px" placeholder="页面路径" />
        </el-form-item>
        <el-form-item label-width="0px" label="" prop="createTime" label-position="left">
          <el-date-picker
            v-model="startEndArr"
            type="datetimerange"
            format="YYYY-MM-DD"
            value-format="YYYY-MM-DD HH:mm:ss"
            class="w-250px"
            range-separator="-"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            @change="dateTimePacking"
          />
        </el-form-item>
      </el-form>
      <!--查询按钮-->
      <el-button @click="searchBtnClick">查询</el-button>
    </div>
    <!--表格和分页-->
    <el-table
      id="resetElementDialog"
      ref="refuserTable"
      :height="`calc(100vh - ${settings.delWindowHeight})`"
      border
      :data="usertableData"
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" align="center" width="50" />
      <el-table-column align="center" prop="errorLog" label="错误日志" width="450">
        <template #default="{ row }">
          <div>{{ row.errorLog }}</div>
          <el-button type="text" size="small" @click="consoleToPlatform(row.errorLog)">
            click it console to platform to track
          </el-button>
        </template>
      </el-table-column>
      <el-table-column align="center" prop="pageUrl" label="页面路径" min-width="180" />
      <el-table-column align="center" prop="version" label="版本号" width="60" />
      <el-table-column align="center" prop="browserType" label="浏览器类型" min-width="180" />
      <el-table-column align="center" prop="createTime" label="创建时间" width="140" />
      <!--点击操作-->
      <el-table-column fixed="right" align="center" label="操作" width="80">
        <template #default="{ row }">
          <el-button type="text" size="small" @click="tableDelClick(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <!--分页-->
    <div class="columnCC mt-20px">
      <el-pagination
        :current-page="pageNum"
        :page-sizes="[10, 20, 50, 100]"
        :page-size="pageSize"
        layout="total, sizes, prev, pager, next, jumper"
        :total="totalPage"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />
    </div>
    <!--图片错误demo-->
    <img v-if="imgShow" src="http://img.png" />
  </div>
</template>

<script setup>
import { Delete } from '@element-plus/icons-vue'
import settings from '@/settings'
import bus from '@/utils/bus'
import packages from '/package.json'
const errorLogProd = () => {
  throw new Error('产生的错误日志')
}
const consoleToPlatform = (err) => {
  //加个custom不收集
  console.error(`custom${err}`)
}

//img loader err test
const imgShow = ref(false)
const errorLogImg = () => {
  imgShow.value = !imgShow.value
}

/*表格查询和筛选*/
const usertableData = ref([])
const searchForm = reactive({
  errorLog: '',
  pageUrl: `https://github.jzfai.top/${packages.name}`,
  createTime: '',
  id: ''
})

const totalPage = ref()
const startEndArr = ref([])
const dialogTitle = ref([])
const detailDialog = ref(false)

const selectPageReq = () => {
  const data = Object.assign(searchForm, {
    pageNum,
    pageSize
  })
  Object.keys(data).forEach((fItem) => {
    if (data[fItem] === '' || data[fItem] === null || data[fItem] === undefined) delete data[fItem]
  })
  const reqConfig = {
    url: '/integration-front/errorCollection/selectPage',
    method: 'get',
    data,
    isParams: true,
    bfLoading: false,
    isAlertErrorMsg: false
  }
  axiosReq(reqConfig).then((resData) => {
    usertableData.value = resData.data?.records
    totalPage.value = resData.data?.total
  })
}
const dateTimePacking = (timeArr) => {
  if (timeArr && timeArr.length === 2) {
    searchForm.startTime = timeArr[0]
    searchForm.endTime = timeArr[1]
  } else {
    searchForm.startTime = ''
    searchForm.endTime = ''
  }
}
onMounted(() => {
  selectPageReq()
  bus.on('reloadErrorPage', () => {
    selectPageReq()
  })
})
const searchBtnClick = () => {
  pageNum.value = 1
  selectPageReq()
}

/*添加和修改*/
/*详情*/
const detailData = ref({})
/*删除*/
const { elConfirm, elMessage } = useElement()
const deleteByIdReq = (id) => {
  return axiosReq({
    url: '/integration-front/errorCollection/deleteById',
    data: { id },
    isParams: true,
    method: 'delete',
    bfLoading: true
  })
}
const tableDelClick = async (row) => {
  await elConfirm('确定', `您确定要删除【${row.pageUrl}】吗？`)
    .then(() => {
      deleteByIdReq(row.id).then(() => {
        selectPageReq()
        elMessage(`【${row.pageUrl}】删除成功`)
      })
    })
    //不写catch会报错
    .catch(() => {})
}

/*批量删除*/
const multipleSelection = ref([])
const handleSelectionChange = (val) => {
  multipleSelection.value = val
}
const multiDelBtnClick = async () => {
  let rowDeleteIdArr = []
  let deleteNameTitle = ''
  rowDeleteIdArr = multipleSelection.value.map((mItem) => {
    deleteNameTitle = `${deleteNameTitle + mItem.pageUrl},`
    return mItem.id
  })
  if (rowDeleteIdArr.length === 0) {
    elMessage('表格选项不能为空', 'warning')
    return
  }
  const stringLength = deleteNameTitle.length - 1
  await elConfirm('删除', `您确定要删除【${deleteNameTitle.slice(0, stringLength)}】吗`)
  const data = rowDeleteIdArr
  axiosReq({
    url: `/integration-front/errorCollection/deleteBatchIds`,
    data,
    method: 'DELETE',
    bfLoading: true
  }).then(() => {
    elMessage('删除成功')
    selectPageReq()
  })
}

let { pageNum, pageSize, handleCurrentChange, handleSizeChange } = useTable(selectPageReq)
</script>

<style scoped lang="scss">
/*详情*/
.detail-container {
  flex-wrap: wrap;
}

.detail-container-item {
  min-width: 40%;
  margin-bottom: 20px;
}

.detailDialog-title {
  margin-bottom: 14px;
  font-weight: bold;
  font-size: 16px;
}
</style>
