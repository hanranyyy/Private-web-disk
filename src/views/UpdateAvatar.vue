<template>
  <div>
    <Dialog
      :show="dialogConfig.show"
      :title="dialogConfig.title"
      :buttons="dialogConfig.buttons"
      width="400px"
      :showCancel="false"
      @close="dialogConfig.show = false"
    >
      <el-form :model="formData" ref="formDataRef" label-width="80px" @submit.prevent>
        <el-form-item label="昵称">
          {{ formData.nickName }}
        </el-form-item>
        <el-form-item label="头像">
          <AvatarUpload v-model="formData.avatar"></AvatarUpload>
        </el-form-item>
      </el-form>
    </Dialog>
  </div>
</template>

<script setup>
import AvatarUpload from '../components/AvatarUpload.vue'
import { ref, reactive, getCurrentInstance } from 'vue'
import { useRoute, useRouter } from 'vue-router'
const { proxy } = getCurrentInstance()
const route = useRoute()
const router = useRouter()
const api = {
  updateUserAvatar: '/updateUserAvatar',
}
const formData = ref({})
const formDataRef = ref()
const show = (data) => {
  // 将一个对象 data 的内容浅拷贝到 formData.value 中
  formData.value = Object.assign({}, data)
  formData.value.avatar = { userId: data.userId, qqAvatar: data.avatar }
  // 显示对话框
  dialogConfig.value.show = true
}
defineExpose({ show })
const dialogConfig = ref({
  show: false,
  title: '修改头像',
  buttons: [
    {
      type: 'primary',
      text: '确定',
      click: (e) => {
        submitForm()
      },
    },
  ],
})
const emit = defineEmits()
const submitForm = async () => {
  // 如果不是个文件
  if (!(formData.value.avatar instanceof File)) {
    dialogConfig.value.show = false
    return
  }
  let result = await proxy.Request({
    url: api.updateUserAvatar,
    params: {
      avatar: formData.value.avatar,
    },
  })
  if (!result) {
    return
  }
  // 隐藏对话框
  dialogConfig.value.show = false
  // 获取'userInfo 的 cookie 信息
  const cookieUserInfo = proxy.VueCookies.get('userInfo')
  // 删除 cookieUserInfo 对象中的 avatar 属性
  delete cookieUserInfo.avatar
  // 将更新后的 cookieUserInfo 对象重新设置到名为 'userInfo' 的 cookie 中，并设置过期时间为 0，即立即过期，相当于删除该 cookie。
  proxy.VueCookies.set('userInfo', cookieUserInfo, 0)
  emit('updateAvatar')
}
</script>

<style lang="scss" scoped></style>
