<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'
import { ArrowDown, DArrowLeft } from '@element-plus/icons-vue'
import AccountView from '../components/AccountView.vue'
import PlaceholderPage from '../components/PlaceholderPage.vue'
import { menuItems } from '../data/menu'

const DEFAULT_AVATAR = '/resource/头像.png'

const activeMenu = ref('首页')
const breadcrumbVisible = ref(false)
const breadcrumbText = ref('首页')
const collapsed = ref(false)
const avatar = ref(DEFAULT_AVATAR)
const placeholderTitle = ref('首页')
const showAccount = ref(false)

/**
 * 原版通过 iframe.contentWindow.document.write 写入，这里改为 Vue 的响应式渲染
 */
function loadPlaceholder(title) {
  placeholderTitle.value = title
}

function toggleSidebar() {
  collapsed.value = !collapsed.value
}

// 切换到账号管理页面
function switchToAccount() {
  // 移除侧边栏高亮
  activeMenu.value = ''
  breadcrumbVisible.value = true
  breadcrumbText.value = '账号管理'
  showAccount.value = true
}

// 点击侧边栏切换页面
function changeTab(item) {
  activeMenu.value = item.title
  breadcrumbVisible.value = true
  breadcrumbText.value = item.title

}

// 监听账号页头像更新消息（保留原版 postMessage 通信方式）
function handleMessage(event) {
  if (event.data?.type === 'AVATAR_UPDATED') {
    const newAvatar = event.data.avatarUrl
    avatar.value = newAvatar
    localStorage.setItem('userAvatar', newAvatar)
  }
}

onMounted(() => {
  // 初始化头像
  const savedAvatar = localStorage.getItem('userAvatar')
  if (savedAvatar) {
    avatar.value = savedAvatar
  }

  // 默认加载首页
  loadPlaceholder('首页')
  window.addEventListener('message', handleMessage)
})

onBeforeUnmount(() => {
  window.removeEventListener('message', handleMessage)
})
</script>

<template>
  <div class="app-shell">
    <header class="top-bar">
      <div class="logo">重庆理工大学(学生)</div>
      <div class="top-right">
        <span class="invite">输入邀请码</span>
        <!-- 点击头像进入账号管理 -->
        <div class="user-info" @click="switchToAccount">
          <img :src="avatar" id="top-avatar" class="avatar-img" alt="头像">
          <span>贺梓涵</span>
          <el-icon class="user-chevron"><ArrowDown /></el-icon>
        </div>
      </div>
    </header>

    <div class="main-container">
      <!-- 侧边栏 -->
      <aside id="sidebar" class="sidebar" :class="{ collapsed }">
        <div class="user-profile">
          <img :src="avatar" id="side-avatar" class="avatar-large" alt="头像">
          <div class="username">贺梓涵</div>
        </div>

        <ul id="menu-list" class="menu-list">
          <li
            v-for="item in menuItems"
            :key="item.title"
            :data-title="item.title"
            :class="{ active: activeMenu === item.title }"
            @click="changeTab(item)"
          >
            <img :src="item.icon" :alt="item.title">
            <span>{{ item.title }}</span>
          </li>
        </ul>

        <div class="collapse-btn" @click="toggleSidebar">
          <el-icon id="collapse-icon"><DArrowLeft /></el-icon>
        </div>
      </aside>

      <!-- 右侧内容区 -->
      <main class="content-area">
        <!-- 面包屑导航 -->
        <div
          v-show="breadcrumbVisible"
          id="breadcrumb"
          class="breadcrumb"
        >
          {{ breadcrumbText }}
        </div>

        <div class="content-body">
          <!-- 原 iframe 页面改为 Vue 组件 -->
          <AccountView v-if="showAccount" />
          <PlaceholderPage v-else :title="placeholderTitle" />
        </div>
      </main>
    </div>
  </div>
</template>
