<template>
  <div class="top">
    <div class="top-op">
      <div class="btn">
        <el-upload
          :show-file-list="false"
          :width-credentials="true"
          :multiple="true"
          :http-request="addFile"
          :accept="fileAccept"
        >
          <el-button type="primary">
            <span class="iconfont icon-upload"></span>
            上传
          </el-button>
        </el-upload>
      </div>
      <el-button type="success" @click="newFolder" v-if="category == 'all'">
        <span class="iconfont icon-folder-add"></span>
        新建文件夹
      </el-button>
      <el-button type="danger" :disabled="selectFileIdList.length == 0" @click="delFileBatch">
        <span class="iconfont icon-del"></span>
        批量删除
      </el-button>
      <el-button type="warning" :disabled="selectFileIdList.length == 0" @click="moveFolderBatch">
        <span class="iconfont icon-move"></span>
        批量移动
      </el-button>
      <div class="search-panel">
        <el-input
          clearable
          placeholder="请输入文件名搜索"
          v-model="fileNameFuzzy"
          @keyup.enter="search"
        >
          <template #suffix>
            <i class="iconfont -icon-search" @click="search"></i>
          </template>
        </el-input>
      </div>
      <div class="iconfont icon-refresh" @click="reload"></div>
    </div>
    <!-- 导航 -->
    <Navigation ref="navigationref" @navChange="navChange"></Navigation>
  </div>
  <div class="file-list" v-if="tableData.list && tableData.list.length > 0">
    <Table
      ref="dataTableRef"
      :columns="columns"
      :dataSource="tableData"
      :fetch="loadDataList"
      :initFetch="false"
      :options="tableOptions"
      @rowSelected="rowSelected"
    >
      <template #fileName="{ index, row }">
        <div class="file-item" @mouseenter="showOP(row)" @mouseleave="cancelShowOp(row)">
          <!-- fileType 1：视频 2：音频 3：图片 status 0：转码中 1：转码失败 2.转码成功-->
          <template v-if="(row.fileType == 3 || row.fileType == 1) && row.status == 2">
            <Icon :cover="row.fileCover" :width="32"></Icon>
          </template>
          <template v-else>
            <!--folderType 0：文件 1：目录-->
            <Icon v-if="row.folderType == 0" :fileType="row.fileType"></Icon>
            <Icon v-if="row.folderType == 1" :fileType="0"></Icon>
          </template>
          <span class="file-name" v-if="!row.showEdit" :title="row.fileName">
            <span @click="preview(row)">{{ row.fileName }}</span>
            <span v-if="row.status == 0" class="transfer-status">转码中</span>
            <span v-if="row.status == 1" class="transfer-status transfer-fail">转码失败</span>
          </span>
          <div class="edit-panel" v-if="row.showEdit">
            <el-input
              v-model.trim="row.fileNameReal"
              ref="editNameRef"
              :maxLength="190"
              @keyup.enter="saveNameEdit(index)"
            >
              <template #suffix>{{ row.fileSuffix }}</template>
            </el-input>
            <!-- 新建文件夹 -->
            <!-- 输入东西才能点对号 -->
            <span
              :class="['iconfont icon-right1', row.fileNameReal ? '' : 'no-allow']"
              @click="saveNameEdit(index)"
            ></span>
            <!-- 叉号 -->
            <span class="iconfont icon-error" @click="cancelNameEdit(index)"></span>
          </div>
          <span class="op">
            <template v-if="row.showOp && row.fileId && row.status == 2">
              <span class="iconfont icon-share1" @click="share(row)">分享</span>
              <span class="iconfont icon-download" v-if="row.folderType == 0" @click="download(row)"
                >下载
              </span>
              <span class="iconfont icon-del" @click="delFile(row)">删除</span>
              <span class="iconfont icon-edit" @click="editFileName(index)">重命名</span>
              <span class="iconfont icon-move" @click="moveFolder(row)">移动</span>
            </template>
          </span>
        </div>
      </template>
      <template #fileSize="{ index, row }">
        <span v-if="row.fileSize">{{ Utils.sizeToStr(row.fileSize) }}</span>
      </template>
    </Table>
  </div>
  <div v-else class="no-data">
    <div class="no-data-inner">
      <Icon iconName="no_data" :width="120" fit="fill"></Icon>
      <div class="tips">当前目录为空，上传你的第一个文件吧</div>
      <div class="op-list">
        <el-upload
          :show-file-list="false"
          :width-credentials="true"
          :multiple="true"
          :http-request="addFile"
          :accept="fileAccept"
        >
          <div class="op-item">
            <Icon iconName="file" :width="60"></Icon>
            <div>上传文件</div>
          </div>
        </el-upload>
        <div class="op-item" v-if="category == 'all'" @click="newFolder">
          <Icon iconName="folder" :width="60"></Icon>
          <div>新建目录</div>
        </div>
      </div>
    </div>
  </div>
  <FoldersSelect ref="foldersSelectRef" @folderSelect="moveFolderDone"></FoldersSelect>
  <!-- 预览 -->
  <Preview ref="previewRef"></Preview>
  <ShareFile ref="shareRef"></ShareFile>
</template>

<script setup>
import ShareFile from './ShareFile.vue'
import CategoryInfo from '@/js/CategoryInfo.js'
import { ref, reactive, getCurrentInstance, nextTick, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
const { proxy } = getCurrentInstance()
const route = useRoute()
const router = useRouter()
const emit = defineEmits(['addFile'])
const addFile = (fileData) => {
  emit('addFile', {
    file: fileData.file,
    filePid: currentFolder.value.fileId,
  })
}
// 添加文件回调
const reload = () => {
  showLoading.value = true
  loadDataList()
}
defineExpose({ reload })
// 当前目录
const currentFolder = ref({ fileId: 0 })
const api = {
  loadDataList: '/file/loadDataList',
  rename: '/file/rename',
  newFoloder: '/file/newFoloder',
  getFoloderInfo: '/file/getFoloderInfo',
  delFile: '/file/delFile',
  changeFileFolder: '/file/changeFileFolder',
  createDownloadUrl: '/file/createDownloadUrl',
  downLoad: '/api/file/download',
}
const fileAccept = computed(() => {
  const categoryItem = CategoryInfo[category.value]
  return categoryItem ? categoryItem.accept : '*'
})
const columns = [
  {
    label: '文件名',
    prop: 'fileName',
    scopedSlots: 'fileName',
  },
  {
    label: '修改时间',
    prop: 'lastUpdateTime',
    width: 200,
  },
  {
    label: '大小',
    prop: 'fileSize',
    scopedSlots: 'fileSize',
    width: 200,
  },
]
// 搜索
const search = () => {
  showLoading.value = true
  loadDataList()
}
const tableData = ref({})
const dataTableRef = ref()
const tableOptions = ref({
  extHeight: 50,
  selectType: 'checkbox',
})
const fileNameFuzzy = ref()
const showLoading = ref(true)
const category = ref()
const loadDataList = async () => {
  let params = {
    // 页码
    pageNo: tableData.value.pageNo,
    // 分页大小
    pageSize: tableData.value.pageSize,
    // 文件名
    fileNameFuzzy: fileNameFuzzy.value,
    // 分类
    category: category.value,
    // 文件父id
    filePid: currentFolder.value.fileId,
  }
  if (params.category != 'all') {
    delete params.filePid
  }
  let result = await proxy.Request({
    url: api.loadDataList,
    showLoading: showLoading.value,
    params: params,
  })
  if (!result) {
    return
  }
  tableData.value = result.data
}

// 展示操作按钮
const showOP = (row) => {
  tableData.value.list.forEach((item) => {
    item.showOp = false
  })
  row.showOp = true
}
const cancelShowOp = (row) => {
  row.showOP = false
}

const editing = ref(false)
const editNameRef = ref()
// 新建文件夹
const newFolder = () => {
  // 已经处于编辑状态
  if (editing.value) {
    return
  }
  // 新建文件夹时其他文件禁止编辑
  tableData.value.list.forEach((element) => {
    element.showEdit = false
  })
  // 当前处于编辑状态
  editing.value = true
  // 开头插入一个新的对象
  tableData.value.list.unshift({
    showEdit: true,
    fileType: 0,
    fileId: '',
    filePid: currentFolder.value.fileId,
  })
  nextTick(() => {
    // 将焦点设置在编辑框上
    editNameRef.value.focus()
  })
}
// 取消编辑
const cancelNameEdit = (index) => {
  const fileData = tableData.value.list[index]
  if (fileData.fileId) {
    // 取消编辑状态
    fileData.showEdit = false
  } else {
    // 数组中删除该文件数据对象
    tableData.value.list.splice(index, 1)
  }
  // 不再处于编辑状态
  editing.value = false
}
// 新建目录
const saveNameEdit = async (index) => {
  const { fileId, filePid, fileNameReal } = tableData.value.list[index]
  if (fileNameReal == '' || fileNameReal.indexOf('/') != -1) {
    proxy.Message.error('文件名不能为空且不能含斜杠')
    return
  }
  // 重命名文件.
  let url = api.rename
  if (fileId == '') {
    // 新建文件夹
    url = api.newFoloder
  }
  let result = await proxy.Request({
    url: url,
    params: {
      fileId: fileId,
      filePid: filePid,
      fileName: fileNameReal,
    },
  })
  if (!result) {
    return
  }
  tableData.value.list[index] = result.data
  // 结束编辑
  editing.value = false
}
const editFileName = (index) => {
  //  新建文件夹时重命名其他文件应该把新建的文件夹删除
  if (tableData.value.list[0].fileId == '') {
    tableData.value.list.splice(0, 1)
    index--
  }
  tableData.value.list.forEach((element) => {
    element.showEdit = false
  })
  let currentData = tableData.value.list[index]
  currentData.showEdit = true
  // 编辑文件
  if (currentData.folderType == 0) {
    // 文件名中去掉扩展名后的部分
    currentData.fileNameReal = currentData.fileName.substring(0, currentData.fileName.indexOf('.'))
    // 文件名的扩展名部分
    currentData.fileSuffix = currentData.fileName.substring(currentData.fileName.indexOf('.'))
  }
  // 编辑文件夹 
  else {
    // 文件名
    currentData.fileNameReal = currentData.fileName
    // 空字符串
    currentData.fileSuffix = ''
  }
  editing.value = true
  nextTick(() => {
    editNameRef.value.focus()
  })
}
// 多选
const selectFileIdList = ref([])
const rowSelected = (rows) => {
  selectFileIdList.value = []
  rows.forEach((item) => {
    selectFileIdList.value.push(item.fileId)
  })
}
const delFile = async (row) => {
  proxy.Confirm(`你确定要删除${row.fileName}吗？`, async () => {
    let result = await proxy.Request({
      url: api.delFile,
      params: {
        fileIds: row.fileId,
      },
    })
    if (!result) {
      return
    }
    loadDataList()
  })
}
// 批量删除
const delFileBatch = () => {
  if (selectFileIdList.value.length == 0) {
    return
  }
  proxy.Confirm('你确定要删除这些文件吗？删除的文件可在10天内恢复', async () => {
    let result = await proxy.Request({
      url: api.delFile,
      params: {
        fileIds: selectFileIdList.value.join(','),
      },
    })
    if (!result) {
      return
    }
    loadDataList()
  })
}

const foldersSelectRef = ref()
const currentMoveFile = ref({})
// 单个移动
const moveFolder = (data) => {
  currentMoveFile.value = data
  foldersSelectRef.value.showFolderDialog(currentFolder.value.fileId)
}
// 批量移动
const moveFolderBatch = () => {
  currentMoveFile.value = {}
  foldersSelectRef.value.showFolderDialog(currentFolder.value.fileId)
}

const moveFolderDone = async (folderId) => {
  if (currentFolder.value.fileId == folderId) {
    proxy.Message.warning('文件正在当前目录，无需移动')
  }
  let fileIdsArray = []
  if (currentMoveFile.value.fileId) {
    fileIdsArray.push(currentMoveFile.value.fileId)
  } else {
    fileIdsArray = fileIdsArray.concat(selectFileIdList.value)
  }
  let result = await proxy.Request({
    url: api.changeFileFolder,
    params: {
      fileIds: fileIdsArray,
      filePid: folderId,
    },
  })
  if (!result) {
    return
  }
  foldersSelectRef.value.close()
  loadDataList()
}
// 预览
const previewRef = ref()
const navigationref = ref()
const preview = (data) => {
  if (data.folderType == 1) {
    navigationref.value.openFolder(data)
  }
  // 文件
  if (data.status !== 2) {
    proxy.Message.warning('文件未完成转码，无法预览')
    return
  }
  previewRef.value.showPreview(data, 0)
}

const navChange = (data) => {
  const { categoryId, curFolder } = data
  category.value = categoryId
  currentFolder.value = curFolder
  loadDataList()
}
// 下载
const download = async (row) => {
  let result = await proxy.Request({
    url: api.createDownloadUrl + '/' + row.fileId,
  })
  if (!result) {
    return
  }
  window.location.href = api.downLoad + '/' + result.data
}

// 分享
const shareRef = ref()
const share = (row) => {
  shareRef.value.show(row)
}
</script>

<style lang="scss" scoped>
@import '@/assets/file.list.scss';
</style>
