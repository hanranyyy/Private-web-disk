<!-- 上传 -->
<template>
  <div class="uploader-panel">
    <div class="uploader-title">
      <span>上传任务</span>
      <span class="tips">(仅展示本次上传任务)</span>
    </div>
    <div class="file-list">
      <div v-for="(item, index) in fileList" class="file-item">
        <div class="upload-panel">
          <div class="file-name">
            {{ item.fileName }}
          </div>
          <div class="progress">
            <!-- 上传中、秒传时、结束上传时显示进度条 -->
            <el-progress
              :percentage="item.uploadProgress"
              v-if="
                item.status == STATUS.uploading.value ||
                item.status == STATUS.upload_seconds.value ||
                item.status == STATUS.upload_finish.value
              "
            ></el-progress>
          </div>
          <div class="upload-status">
            <!-- 图标 -->
            <span
              :class="['iconfont', 'icon-' + STATUS[item.status].icon]"
              :style="{ color: STATUS[item.status].color }"
            ></span>
            <!-- 状态描述 -->
            <span class="status" :style="{ color: STATUS[item.status].color }">
              {{ item.status == 'fail' ? item.errorMsg : STATUS[item.status].desc }}
            </span>
            <!-- 上传中 -->
            <!-- Utils.sizeToStr 用于将文件大小转换为可读性更好的格式 -->
            <span class="upload-info" v-if="item.status == STATUS.uploading.value">
              {{ Utils.sizeToStr(item.uploadSize) }}/{{ Utils.sizeToStr(item.totalSize) }}
            </span>
          </div>
        </div>
        <div class="op">
          <!-- Md5 -->
          <el-progress
            type="circle"
            :width="50"
            :percentage="item.md5Progress"
            v-if="item.status == STATUS.init.value"
          ></el-progress>
          <div class="op-btn">
            <span v-if="item.status == STATUS.uploading.value">
              <Icon
                :width="28"
                class="btn-item"
                iconName="upload"
                v-if="item.pause"
                title="上传"
                @click="startUpload(item.uid)"
              ></Icon>
              <Icon
                v-else
                :width="28"
                class="btn-item"
                iconName="pause"
                title="暂停"
                @click="pauesUpload(item.uid)"
              ></Icon>
            </span>
            <Icon
              v-if="
                item.status !== STATUS.init.value &&
                item.status !== STATUS.upload_seconds.value &&
                item.status !== STATUS.upload_finish.value
              "
              :width="28"
              class="del btn-item"
              iconName="del"
              title="删除"
              @click="delUpload(item.uid, index)"
            ></Icon>
            <Icon
              v-if="
                item.status == STATUS.upload_seconds.value ||
                item.status == STATUS.upload_finish.value
              "
              :width="28"
              class="clean btn-item"
              iconName="clean"
              title="清除"
              @click="delUpload(item.uid, index)"
            ></Icon>
          </div>
        </div>
      </div>
      <div v-if="fileList.length == 0">
        <NoData msg="暂无上传任务"></NoData>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, getCurrentInstance, nextTick } from 'vue'
const { proxy } = getCurrentInstance()
import SparkMD5 from 'spark-md5'
const api = {
  uploadFile: '/file/uploadFile',
}
const STATUS = {
  emptyfile: {
    value: 'emptyfile',
    desc: '文件为空',
    color: '#F75000',
    icon: 'close',
  },
  fail: {
    value: 'fail',
    desc: '上传失败',
    color: '#F75000',
    icon: 'close',
  },
  init: {
    value: 'init',
    desc: '解析中',
    color: '#e6a23c',
    icon: 'clock',
  },
  uploading: {
    value: 'uploading',
    desc: '上传中',
    color: '#409eff',
    icon: 'upload',
  },
  upload_finish: {
    value: 'upload_finish',
    desc: '上传完成',
    color: '#b7c23a',
    icon: 'ok',
  },
  upload_seconds: {
    value: 'upload_seconds',
    desc: '秒传',
    color: '#b7c23a',
    icon: 'ok',
  },
}
// 一个分片5MB
const chunkSize = 1024 * 1024
const fileList = ref([])
const delList = ref([])
const addFile = async (file, filePid) => {
  const fileItem = {
    // 文件信息
    file: file,
    // 文件UID
    uid: file.uid,
    // md5进度
    md5Progress: 0,
    // md5值
    md5: null,
    // 文件名
    fileName: file.name,
    // 上传状态
    status: STATUS.init.value,
    // 上传大小
    uploadSize: 0,
    //上传总大小
    totalSize: file.size,
    // 上传进度
    uploadProgress: 0,
    // 暂停，
    pause: false,
    // 当前分片
    chunkIndex: 0,
    // 文件父级ID
    filePid: filePid,
    // 错误信息
    errorMsg: null,
  }
  fileList.value.unshift(fileItem)
  // 文件为空
  if (fileItem.totalSize == 0) {
    fileItem.status = STATUS.emptyfile.value
    return
  }
  let md5FileUid = await computeMd5(fileItem)
  if (md5FileUid == null) {
    return
  }
  uploadFile(md5FileUid)
}
defineExpose({ addFile })
// 计算md5 用于秒传 前端算好md5 后端通过md5查 查上传文件的file已经在服务器上有了 就不做上传动作 直接进行秒传
// 当用户需要上传一个文件时，系统会首先计算文件的哈希值（比如MD5），然后将这个哈希值与服务器端已有的文件列表进行比对。
// 如果服务器已经存在具有相同哈希值的文件，就说明这个文件已经上传过，那么就不需要再次上传整个文件，而是可以直接使用已存在的文件，从而实现“秒传”。
// 秒传---主要为了看到上传的效果
const computeMd5 = (fileItem) => {
  let file = fileItem.file
  // 文件切片 多个浏览器兼容
  let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
  // ceil 向上取整
  // 总共切几片 chunkSize：一个分片5MB
  let chunks = Math.ceil(file.size / chunkSize)
  // 当前处理的切片索引
  let currentChunk = 0
  // 计算文件的 MD5 哈希值
  let spark = new SparkMD5.ArrayBuffer()
  // 读取文件内容
  let fileReader = new FileReader()

  let loadNext = () => {
    // 当前要处理的分片在文件中的起始位置。currentChunk：当前处理的分片索引 chunkSize：每个分片的大小
    let start = currentChunk * chunkSize
    // 当前要处理的分片在文件中的结束位置
    // 如果当前分片的结束位置超过了文件总大小，则将结束位置设置为文件总大小，否则设置为当前分片的起始位置加上分片大小。
    let end = start + chunkSize >= file.size ? file.size : start + chunkSize
    // 读取文件分片的内容并以 ArrayBuffer 格式进行处理
    // 用 blobSlice 方法从文件中切割出指定范围的分片数据
    fileReader.readAsArrayBuffer(blobSlice.call(file, start, end))
  }
  loadNext()
  return new Promise((resolve, reject) => {
    let resultFile = getFileByUid(file.uid)
    // 监听文件读取成功时的事件
    fileReader.onload = (e) => {
      // 将当前读取的文件片段内容追加到一个名为 spark 的对象
      spark.append(e.target.result)
      // 增加当前处理的文件片段数
      currentChunk++
      // 如果当前处理片段未达到总片段数
      if (currentChunk < chunks) {
        // console.log(`第${file.name}分片解析完成，开始第${currentChunk + 1}`)
        // 更新 MD5 计算进度、加载下一片段
        let percent = Math.floor((currentChunk / chunks) * 100)
        resultFile.md5Progress = percent
        loadNext()
      } 
      // 已处理完所有片段 进行最后的 MD5 计算和状态更新操作
      else {
        // 生成最终的 MD5 值
        let md5 = spark.end()
        // 销毁 spark 对象
        spark.destroy()
        // 将 MD5 计算进度更新为 100%
        resultFile.md5Progress = 100
        // 更新文件对象的状态为上传中
        resultFile.status = STATUS.uploading.value
        resultFile.md5 = md5
        resolve(fileItem.uid)
      }
    }
    // 监听文件读取失败时的事件
    fileReader.onerror = () => {
      resultFile.md5Progress = -1
      resultFile.status = STATUS.fail.value
      resolve(fileItem.uid)
    }
  }).catch((error) => {
    return null
  })
}
// 获取文件
// uid是element-ui在文件上传时生成的唯一标志位
// 根据给定的 uid 在文件列表 fileList.value 中查找对应的文件对象
const getFileByUid = (uid) => {
  let file = fileList.value.find((item) => {
    return item.file.uid == uid
  })
  return file
}
const emit = defineEmits(['uploadCallback'])
 // chunkIndex 第几片开始
  const uploadFile = async (uid, chunkIndex) => {
  chunkIndex = chunkIndex ? chunkIndex : 0
  // 分片上传
  let currentFile = getFileByUid(uid)
  const file = currentFile.file
  const fileSize = currentFile.totalSize
  const chunks = Math.ceil(fileSize / chunkSize)
  for (let i = chunkIndex; i < chunks; i++) {
    let delIndex = delList.value.indexOf(uid)
    if (delIndex != -1) {
      delList.value.splice(delIndex, 1)
      break
    }
    // 状态会改变，需要重新获取
    currentFile = getFileByUid(uid)
    if (currentFile.pause) {
      break
    }
    let start = i * chunkSize
    let end = start + chunkSize >= fileSize ? fileSize : start + chunkSize
    let chunkFile = file.slice(start, end)
    // debugger
    let updtateResult = await proxy.Request({
      url: api.uploadFile,
      showLoading: false,
      dataType: 'file',
      params: {
        file: chunkFile,
        fileName: file.name,
        fileMd5: currentFile.md5,
        chunkIndex: i,
        chunks: chunks,
        fileId: currentFile.fileId,
        filePid: currentFile.filePid,
      },
      showError: false,
      errorCallback: (errorMsg) => {
        currentFile.status = STATUS.fail.value
        currentFile.errorMsg = errorMsg
      },
      uploadProgressCallback: (e) => {
        let loaded = e.loaded
        if (loaded > fileSize) {
          loaded = fileSize
        }
        currentFile.uploadSize = i * chunkSize + loaded
        currentFile.uploadProgress = Math.floor((currentFile.uploadSize / fileSize) * 100)
      },
    })
    if (updtateResult == null) {
      break
    }
    // 后端生成唯一id
    currentFile.fileId = updtateResult.data.fileId
    currentFile.status = STATUS[updtateResult.data.status].value
    if (
      updtateResult.data.status == STATUS.upload_seconds.value ||
      updtateResult.data.status == STATUS.upload_finish.value
    ) {
      currentFile.uploadProgress = 100
      emit('uploadCallback')
      break
    }
  }
}
</script>

<style lang="scss" scoped>
.uploader-panel {
  .uploader-title {
    border-bottom: 1px solid #ddd;
    line-height: 40px;
    padding: 0 10px;
    font-size: 15px;
    .tips {
      font-size: 13px;
      color: rgb(169, 169, 169);
    }
  }
  .file-list {
    overflow: auto;
    padding: 10px 0px;
    min-height: calc(100vh / 2);
    max-height: calc(100vh - 120px);
    .file-item {
      position: relative;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 3px 10px;
      background-color: #fff;
      border-bottom: 1px solid #ddd;
    }
    .file-item:nth-child(even) {
      background-color: #fcf8f4;
    }
    .upload-panel {
      flex: 1;
      .file-name {
        color: rgb(64, 62, 62);
      }
      .upload-status {
        display: flex;
        align-items: center;
        margin-top: 5px;
      }
      .iconfont {
        margin-right: 3px;
      }
      .status {
        color: red;
        font-size: 13px;
      }
      .upload-info {
        margin-left: 5px;
        font-size: 12px;
        color: rgb(112, 111, 111);
      }
    }
    .progress {
      height: 10px;
    }
  }
  .op {
    width: 100px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    .op-btn {
      .btn-item {
        cursor: pointer;
      }
      .del,
      .clean {
        margin-left: 5px;
      }
    }
  }
}
</style>
