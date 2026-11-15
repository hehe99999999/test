<script setup>
import { onMounted, ref } from 'vue'
import { UserFilled, WarningFilled } from '@element-plus/icons-vue'

const tabs = [
  { target: 'tab-basic', label: '基本资料' },
  { target: 'tab-avatar', label: '修改头像' },
  { target: 'tab-password', label: '密码管理' },
  { target: 'tab-deactivate', label: '注销账号' },
  { target: 'tab-language', label: '语言' }
]

const activeTab = ref('tab-basic')
const gender = ref('male')
const avatarPreview = ref('/resource/头像.png')
const avatarUpload = ref(null)
const oldPwd = ref('')
const newPwd = ref('')
const confirmPwd = ref('')
const verificationCode = ref('')
const selectedLang = ref('zh')
const modalVisible = ref(false)

const toastVisible = ref(false)
const toastMessage = ref('操作提示')

// 通用 Toast 提示
function showToast(message) {
  toastMessage.value = message
  toastVisible.value = true
  setTimeout(() => {
    toastVisible.value = false
  }, 2500) // 2.5秒后自动消失
}

// Tab 切换
function selectTab(targetId) {
  if (!targetId) return
  activeTab.value = targetId
}

// 头像上传与保存同步
function chooseAvatar() {
  avatarUpload.value?.click()
}

function onAvatarChange(event) {
  const file = event.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = function (loadEvent) {
      avatarPreview.value = loadEvent.target.result
    }
    reader.readAsDataURL(file)
  }
}

function saveAvatar() {
  const newAvatarUrl = avatarPreview.value
  // 将更新通知给父窗口（保留原版 postMessage 逻辑）
  window.parent.postMessage({ type: 'AVATAR_UPDATED', avatarUrl: newAvatarUrl }, '*')
  showToast('头像保存成功！')
}

// 密码逻辑
function modifyPassword() {
  const oldPassword = oldPwd.value
  const newPassword = newPwd.value
  const confirmPassword = confirmPwd.value

  if (!oldPassword || !newPassword || !confirmPassword) return showToast('密码不能为空！')
  if (oldPassword === newPassword) return showToast('新密码不能与旧密码相同！')
  if (newPassword !== confirmPassword) return showToast('两次输入的新密码不一致！')

  showToast('密码修改成功！')
  oldPwd.value = ''
  newPwd.value = ''
  confirmPwd.value = ''
}

// 忘记密码切换
function openForgotPassword() {
  activeTab.value = 'tab-forgot-pwd'
}

function backToPassword() {
  activeTab.value = 'tab-password'
}

// 注销弹窗
function openModal() {
  modalVisible.value = true
}

function closeModal() {
  modalVisible.value = false
}

function confirmDeactivate() {
  showToast('账号已注销 (演示)')
  closeModal()
}

// 语言切换
function saveLanguage() {
  showToast(selectedLang.value === 'en' ? 'Language switched to English!' : '语言已切换！')
  activeTab.value = 'tab-basic'
}

onMounted(() => {
  // 初始化头像预览
  const savedAvatar = localStorage.getItem('userAvatar')
  if (savedAvatar) avatarPreview.value = savedAvatar
})
</script>

<template>
  <div class="account-view">
    <!-- 弹窗 -->
    <div id="toast" class="toast" :class="{ show: toastVisible }">{{ toastMessage }}</div>

    <div class="tabs-header">
      <div class="tab-title">
        <span class="title-icon"><el-icon><UserFilled /></el-icon></span>
        账号管理
      </div>
      <ul id="tabs-nav" class="tabs-nav">
        <li
          v-for="tab in tabs"
          :key="tab.target"
          :data-target="tab.target"
          :class="{ active: activeTab === tab.target }"
          @click="selectTab(tab.target)"
        >
          {{ tab.label }}
        </li>
      </ul>
    </div>

    <div id="tab-basic" class="tab-content" :class="{ active: activeTab === 'tab-basic' }">
      <div class="form-row"><label>姓名</label><span>贺梓涵</span></div>
      <div class="form-row"><label>id</label><span>301692714</span></div>
      <div class="form-row">
        <label>性别</label>
        <el-radio-group v-model="gender" class="radio-group">
          <el-radio value="male">男</el-radio>
          <el-radio value="female">女</el-radio>
        </el-radio-group>
      </div>
      <div class="form-row">
        <label>手机号</label>
        <span>
          177****0053
          <a href="javascript:void(0)" class="modify-link">修改</a>
        </span>
      </div>
      <div class="form-row">
        <label>单位</label>
        <a href="javascript:void(0)" class="modify-link">+ 添加单位</a>
      </div>

      <div class="unit-list">
        <div class="unit-item">
          <div class="unit-name">重庆理工大学</div>
          <div class="unit-desc">学号/工号:12523020119</div>
        </div>
      </div>
    </div>

    <div id="tab-avatar" class="tab-content" :class="{ active: activeTab === 'tab-avatar' }">
      <div class="avatar-container">
        <img id="avatar-preview" :src="avatarPreview" alt="头像预览">
        <input
          ref="avatarUpload"
          id="avatar-upload"
          type="file"
          accept="image/*"
          style="display: none;"
          @change="onAvatarChange"
        >
        <button class="btn-primary" @click="chooseAvatar">选择图片</button>
        <button id="btn-save-avatar" class="btn-primary" @click="saveAvatar">保存头像</button>
      </div>
    </div>

    <div id="tab-password" class="tab-content" :class="{ active: activeTab === 'tab-password' }">
      <div class="pwd-form">
        <div class="input-group">
          <label>旧密码：</label>
          <el-input v-model="oldPwd" type="password" placeholder="请输入旧密码" />
        </div>
        <div class="input-group">
          <label>新密码：</label>
          <el-input v-model="newPwd" type="password" placeholder="请输入新密码" />
        </div>
        <div class="input-group">
          <label>确认新密码：</label>
          <el-input v-model="confirmPwd" type="password" placeholder="请再次输入新密码" />
        </div>
        <div class="pwd-actions">
          <button id="btn-modify-pwd" class="btn-primary" @click="modifyPassword">确认修改</button>
          <a href="javascript:void(0)" id="btn-forgot-pwd" @click="openForgotPassword">忘记密码？</a>
        </div>
      </div>
    </div>

    <!-- 忘记密码页 -->
    <div id="tab-forgot-pwd" class="tab-content" :class="{ active: activeTab === 'tab-forgot-pwd' }">
      <div class="pwd-form">
        <h3>找回密码</h3>
        <div class="input-group">
          <label>手机号：</label>
          <input type="text" value="177****0053" disabled>
        </div>
        <div class="input-group">
          <label>验证码：</label>
          <input v-model="verificationCode" type="text" placeholder="模拟验证码">
        </div>
        <div class="pwd-actions">
          <button class="btn-primary">重置密码</button>
          <button id="btn-back-pwd" class="btn-secondary" @click="backToPassword">返回</button>
        </div>
      </div>
    </div>

    <div id="tab-deactivate" class="tab-content" :class="{ active: activeTab === 'tab-deactivate' }">
      <div class="danger-zone">
        <h3><el-icon><WarningFilled /></el-icon> 危险操作</h3>
        <p>注销账号后，您的所有数据将被清空且无法恢复，请谨慎操作。</p>
        <button id="btn-deactivate" class="btn-danger" @click="openModal">注销账号</button>
      </div>
    </div>

    <div id="tab-language" class="tab-content" :class="{ active: activeTab === 'tab-language' }">
      <div class="language-selector">
        <h3>选择语言</h3>
        <el-radio-group v-model="selectedLang" class="language-options">
          <el-radio value="zh" class="lang-option">简体中文</el-radio>
          <el-radio value="en" class="lang-option">English</el-radio>
        </el-radio-group>
        <button id="btn-save-lang" class="btn-primary" style="margin-top: 20px;" @click="saveLanguage">
          保存并切换
        </button>
      </div>
    </div>

    <!-- 注销二次确认弹窗 -->
    <el-dialog
      v-model="modalVisible"
      title="二次确认"
      width="400px"
      align-center
      append-to-body
      class="deactivate-dialog"
    >
      <p class="dialog-message">确定要注销当前账号吗？此操作不可逆！</p>
      <template #footer>
        <div class="modal-actions">
          <button class="btn-secondary" @click="closeModal">取消</button>
          <button class="btn-danger" @click="confirmDeactivate">确定注销</button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>
