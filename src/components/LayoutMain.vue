<template>
  <a-layout class="main-layout">
    <!-- 左侧菜单 -->
    <a-layout-sider class="custom-sider">
      <div class="sider-header">
        <div class="sider-logo" title="访问 BetterGI 官网" @click="openExternalLink('https://bettergi.com/')"></div>
        <span class="repo-title">脚本仓库</span>
      </div>
      <div class="sider-menu-wrap">
        <div class="menu-content">
          <a-menu v-model:selectedKeys="selectedMenu" mode="vertical" class="custom-menu" @select="handleMenuSelect">
            <a-menu-item v-for="item in menuList" :key="item.key" class="custom-menu-item">
              <span class="menu-icon">
                <component :is="item.icon" />
              </span>
              <span class="menu-label">{{ item.label }}</span>
              <span v-if="item.count !== undefined" class="menu-count">{{ item.count }}</span>
            </a-menu-item>
          </a-menu>
        </div>
        <div class="sider-footer">
          <div class="sider-link" @click="openExternalLink('https://github.com/babalae/better-genshin-impact')">
            访问BetterGI主仓库</div>
          <div class="sider-link" @click="openExternalLink('https://github.com/babalae/bettergi-scripts-list')">
            访问BetterGI脚本仓库</div>
          <div class="sider-link" @click="openExternalLink('https://bgi.sh')">访问在线仓库</div>
        </div>
      </div>
      <!-- 最后更新时间 -->
      <div v-if="lastUpdateTime" class="last-update-time" @click="showEggModal = true" style="cursor:pointer;">
        <span>最后更新时间：</span>
        <span>{{ lastUpdateTime }}</span>
      </div>
    </a-layout-sider>

    <!-- 中间内容区域 -->
    <a-layout-sider class="script-sider">
      <div class="script-header">
        <span class="script-title">{{ currentMenuTitle }}</span>
        <div class="script-actions" style="position:relative;">
          <a-button type="primary" class="script-btn"
            :class="{ 'script-btn-active': scriptTab === 'all' }" @click="onClickShowAll">全部</a-button>
          <a-button class="script-btn" :class="{ 'script-btn-active': scriptTab === 'subscribed' }"
            @click="onClickShowSubscribed">已订阅</a-button>
          <div v-if="mode === 'web'" class="script-actions-mask">
            <span>仅本地页面可用</span>
          </div>
        </div>
      </div>
      <div class="search-section">
        <a-input v-model:value="search" :placeholder="searchPlaceholder" class="script-search" @search="handleSearch"
          allow-clear>
          <template #prefix>
            <SearchOutlined />
          </template>
        </a-input>
      </div>
      <!-- 地图追踪的树状结构 -->
      <div v-if="selectedMenu[0] === '1'" class="script-list">
        <MapTreeList ref="mapTreeRef" :search-key="search" :repo-data="repoData"
          :subscribed-paths="subscribedScriptPaths" :show-subscribed-only="scriptTab === 'subscribed'"
          @select="handleMapSelect" @leaf-count="handleLeafCount" />
      </div>
      <!-- Javascript脚本列表 -->
      <div v-else-if="selectedMenu[0] === '2'" class="script-list">
        <ScriptList :search-key="search" :repo-data="repoData" :subscribed-paths="subscribedScriptPaths"
          :show-subscribed-only="scriptTab === 'subscribed'" ref="scriptListRef" @select="handleScriptSelect"
          @script-count="handleScriptCount" />
      </div>
      <!-- 战斗策略列表 -->
      <div v-else-if="selectedMenu[0] === '3'" class="script-list">
        <CombatStrategyList :search-key="search" :repo-data="repoData" :subscribed-paths="subscribedScriptPaths"
          :show-subscribed-only="scriptTab === 'subscribed'" ref="combatStrategyRef" @select="handleScriptSelect" />
      </div>
      <!-- 七圣召唤策略列表 -->
      <div v-else-if="selectedMenu[0] === '4'" class="script-list">
        <CardStrategyList :search-key="search" :repo-data="repoData" :subscribed-paths="subscribedScriptPaths"
          :show-subscribed-only="scriptTab === 'subscribed'" ref="cardStrategyRef" @select="handleScriptSelect" />
      </div>
    </a-layout-sider>

    <!-- 右侧内容区域 -->
    <a-layout>
      <div v-if="selectedMenu[0] === '1'" class="main-right">
        <!-- 顶部操作栏 -->
        <div v-if="selectedScript" class="detail-top-bar">
          <div class="top-bar-left">
            <a-tooltip title="跳转到GitHub">
              <a-button type="text" size="small" class="action-btn" @click="jumpToGitHub">
                <LinkOutlined />
              </a-button>
            </a-tooltip>
            <a-tooltip title="评论">
              <a-button type="text" size="small" class="action-btn" @click="commentModalOpen = true">
                <MessageOutlined />
              </a-button>
            </a-tooltip>
          </div>
          <div class="top-bar-right">
            <a-tooltip title="帮助">
              <a-button type="text" size="small" class="action-btn" @click="showHelpModal = true">
                <QuestionCircleOutlined />
              </a-button>
            </a-tooltip>
          </div>
        </div>
        <MapDetail :script="selectedScript" />
      </div>
      <div v-else-if="selectedMenu[0] === '2'" class="main-right">
        <!-- 顶部操作栏 -->
        <div v-if="selectedScript" class="detail-top-bar">
          <div class="top-bar-left">
            <a-tooltip title="跳转到该脚本的GitHub仓库">
              <a-button type="text" size="small" class="action-btn" @click="jumpToGitHub">
                <LinkOutlined />
              </a-button>
            </a-tooltip>
            <a-tooltip title="评论">
              <a-button type="text" size="small" class="action-btn" @click="commentModalOpen = true">
                <MessageOutlined />
              </a-button>
            </a-tooltip>
          </div>
          <div class="top-bar-right">
            <a-tooltip title="帮助">
              <a-button type="text" size="small" class="action-btn" @click="showHelpModal = true">
                <QuestionCircleOutlined />
              </a-button>
            </a-tooltip>
          </div>
        </div>
        <ScriptDetail :script="selectedScript" />
      </div>
      <div v-else-if="selectedMenu[0] === '3' || selectedMenu[0] === '4'" class="main-right">
        <!-- 顶部操作栏 -->
        <div v-if="selectedScript" class="detail-top-bar">
          <div class="top-bar-left">
            <a-tooltip title="跳转到该脚本的GitHub仓库">
              <a-button type="text" size="small" class="action-btn" @click="jumpToGitHub">
                <LinkOutlined />
              </a-button>
            </a-tooltip>
            <a-tooltip title="评论">
              <a-button type="text" size="small" class="action-btn" @click="commentModalOpen = true">
                <MessageOutlined />
              </a-button>
            </a-tooltip>
          </div>
          <div class="top-bar-right">
            <a-tooltip title="帮助">
              <a-button type="text" size="small" class="action-btn" @click="showHelpModal = true">
                <QuestionCircleOutlined />
              </a-button>
            </a-tooltip>
          </div>
        </div>
        <StrategyDetail :script="selectedScript" />
      </div>
    </a-layout>

    <!-- 错误弹窗 -->
    <div v-if="repoError" class="repo-error-modal">
      <div class="repo-error-content">
        <div class="repo-error-icon">
          <svg width="48" height="48" viewBox="0 0 48 48" fill="none">
            <circle cx="24" cy="24" r="24" fill="#fdecea" />
            <path d="M24 14v12" stroke="#f44336" stroke-width="3" stroke-linecap="round" />
            <circle cx="24" cy="32" r="2.5" fill="#f44336" />
          </svg>
        </div>
        <div class="repo-error-title">获取仓库信息失败</div>
        <div class="repo-error-desc">请检查网络或稍后刷新页面重试</div>
        <div class="repo-error-btn-group">
          <button class="repo-error-btn" @click="getRepoJson">刷新页面</button>
          <button class="repo-help-btn" @click="showHelpModal = true">查看帮助</button>
        </div>
      </div>
    </div>

    <!-- 全局加载弹窗 -->
    <div v-if="globalLoading" class="global-loading-modal">
      <a-spin size="large" tip="仓库数据加载中，请稍候..." />
    </div>

    <!-- 评论弹窗 -->
    <a-modal v-model:open="commentModalOpen" title="评论" :footer="null" centered width="90%"
      :style="{ maxWidth: '1200px' }" @cancel="handleCommentModalCancel" class="comment-modal">
      <div class="comment-modal-content">
        <div class="comment-header">
          <div class="script-title-simple">
            {{ selectedScript?.title || '未知脚本' }}
            <template
              v-if="selectedScript?.authors && Array.isArray(selectedScript.authors) && selectedScript.authors.length">
              <span class="script-author">
                · 作者：
                <template v-for="(author, idx) in selectedScript.authors" :key="author.name">
                  <template v-if="author.link">
                    <a :href="author.link" class="author-link" target="_blank" rel="noopener noreferrer">{{ author.name
                    }}</a>
                  </template>
                  <template v-else>
                    <span>{{ author.name }}</span>
                  </template>
                  <span v-if="idx < selectedScript.authors.length - 1">，</span>
                </template>
              </span>
            </template>
            <span v-else-if="selectedScript?.author" class="script-author">· 作者：{{ selectedScript.author }}</span>
            <span v-else class="script-author">· 暂无作者信息</span>
          </div>
        </div>

        <div class="comments-container">
          <!-- 本地模式显示提示信息 -->
          <div v-if="mode === 'single'" class="local-mode-notice">
            <div class="notice-icon">
              <svg width="48" height="48" viewBox="0 0 48 48" fill="none">
                <circle cx="24" cy="24" r="24" fill="#e3f2fd" />
                <path d="M24 12v16" stroke="#2196f3" stroke-width="3" stroke-linecap="round" />
                <circle cx="24" cy="36" r="2.5" fill="#2196f3" />
              </svg>
            </div>
            <div class="notice-title">本地模式暂不支持评论功能</div>
            <div class="notice-desc">请移步在线仓库进行评论</div>
            <button class="notice-btn" @click="openExternalLink('https://bgi.sh')">访问在线仓库</button>
          </div>

          <!-- Web模式显示giscus评论 -->
          <Giscus v-else id="comments" :repo="giscusConfig.repo" :repoId="giscusConfig.repoId"
            :category="giscusConfig.category" :categoryId="giscusConfig.categoryId" :mapping="giscusConfig.mapping"
            :term="giscusTerm" :strict="giscusConfig.strict" :reactionsEnabled="giscusConfig.reactionsEnabled"
            :emitMetadata="giscusConfig.emitMetadata" :inputPosition="giscusConfig.inputPosition"
            :theme="giscusConfig.theme" :lang="giscusConfig.lang" loading="lazy" />
        </div>
      </div>
    </a-modal>

    <!-- 彩蛋弹窗 -->
    <a-modal v-model:open="showEggModal" title="小彩蛋" :footer="null" centered width="80%" :style="{ maxWidth: '900px' }"
      @cancel="showEggModal = false">
      <div class="egg-modal-content">
        <div class="egg-readme-tabs">
          <div v-if="isLoadingEggReadme" class="egg-readme-loading-indicator">
            <a-spin size="small" />
            <span>彩蛋正在拼尽全力加载…</span>
          </div>
          <div v-else-if="eggReadmeError" class="egg-readme-loading-indicator">
            <a-button type="text" size="small" @click="retryLoadEggReadme">
              <template #icon>
                <ReloadOutlined />
              </template>
            </a-button>
            <span>拼尽全力无法战胜，加载失败，请重试</span>
          </div>
        </div>
        <div class="egg-readme-content">
          <ReadmeViewer :key="eggReadmeKey"
            :path="'https://raw.githubusercontent.com/zaodonganqi/BGI-bsw-egg/main/README.md'"
            @loaded="handleEggReadmeLoaded" @error="handleEggReadmeError" />
        </div>
      </div>
    </a-modal>

    <!-- 帮助弹窗 -->
    <a-modal v-model:open="showHelpModal" title="常见问题 Q&A" :footer="null" centered width="80%"
      :style="{ maxWidth: '900px' }" @cancel="showHelpModal = false">
      <Help />
    </a-modal>

    <!-- 订阅信息获取失败弹窗 -->
    <a-modal v-model:open="subscribedErrorModalOpen" title="已订阅信息获取失败" :footer="null" centered width="400px"
      @cancel="subscribedErrorModalOpen = false">
      <div class="subscribed-error-msg">
        　　已订阅信息获取失败。<br />
        　　请检查相关设置：“BetterGI → 设置 → BetterGI Http 服务器设置”（如未找到该设置，请确保BGI已更新至最新版），确保该项已开启且端口号设置为30648。<br />
        　　完成后请关闭此页面并重启BetterGI再次尝试，仍有问题请联系作者解决。
      </div>
    </a-modal>
  </a-layout>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
import { FolderOutlined, FileOutlined, CalculatorOutlined, BulbOutlined, SearchOutlined, ExportOutlined, SettingOutlined, QuestionCircleOutlined, MessageOutlined, LinkOutlined, ReloadOutlined } from '@ant-design/icons-vue';
import Giscus from '@giscus/vue';
import MapTreeList from './lists/MapTreeList.vue';
import ScriptList from './lists/ScriptList.vue';
import CombatStrategyList from './lists/CombatStrategyList.vue';
import CardStrategyList from './lists/CardStrategyList.vue';
import ScriptDetail from './details/ScriptDetail.vue';
import StrategyDetail from './details/StrategyDetail.vue';
import MapDetail from './details/MapDetail.vue';
import ReadmeViewer from './ReadmeViewer.vue';
import Help from './Help.vue';

const mode = import.meta.env.VITE_MODE;
const selectedMenu = ref(['1']);
const search = ref('');

const menuList = ref([
  { key: '1', label: '地图追踪', icon: FolderOutlined, count: 0 },
  { key: '2', label: 'Javascript 脚本', icon: FileOutlined, count: 0 },
  { key: '3', label: '战斗策略', icon: CalculatorOutlined, count: 0 },
  { key: '4', label: '七圣召唤策略', icon: BulbOutlined, count: 0 },
]);

const repoData = ref({});
const repoError = ref(false);
const globalLoading = ref(false);
let fetchTimeoutId = null;

// 格式化时间
const formatTime = (timeString) => {
  if (!timeString) return '';
  const year = timeString.substring(0, 4);
  const month = timeString.substring(4, 6);
  const day = timeString.substring(6, 8);
  const hour = timeString.substring(8, 10);
  const minute = timeString.substring(10, 12);
  const second = timeString.substring(12, 14);

  return `${year}-${month}-${day} ${hour}:${minute}:${second}`;
};

// 计算最后更新时间
const lastUpdateTime = computed(() => {
  if (repoData.value && repoData.value.time) {
    // 区分本地与web模式，如需更改最后更新时间格式请更改此处
    if (mode === 'web') {
      return `${formatTime(repoData.value.time)}`;
    }
    return `${formatTime(repoData.value.time)}`;
  }
  return '';
});

// 清理所有readme404缓存
function clearReadme404Cache() {
  Object.keys(localStorage).forEach(key => {
    if (key.startsWith('readme404:')) {
      localStorage.removeItem(key);
    }
  });
}

async function getRepoJson() {
  clearReadme404Cache();
  repoError.value = false;
  globalLoading.value = true;
  try {
    console.log("当前模式：", mode)
    if (mode === 'web') {
      const controller = new AbortController();
      let didTimeout = false;
      if (fetchTimeoutId) clearTimeout(fetchTimeoutId);
      fetchTimeoutId = setTimeout(() => {
        didTimeout = true;
        controller.abort();
      }, 10000);
      try {
        const response = await fetch('https://raw.githubusercontent.com/babalae/bettergi-scripts-list/refs/heads/main/repo.json', {
          signal: controller.signal
        });
        clearTimeout(fetchTimeoutId);
        fetchTimeoutId = null;
        if (!response.ok) throw new Error('网络请求失败');
        repoData.value = await response.json();
      } catch (err) {
        clearTimeout(fetchTimeoutId);
        fetchTimeoutId = null;
        if (didTimeout) {
          throw new Error('请求超时');
        } else {
          throw err;
        }
      }
    } else {
      // 先获取订阅信息
      await fetchSubscribedConfig();
      const repoWebBridge = chrome.webview.hostObjects.repoWebBridge;
      const json = await repoWebBridge.GetRepoJson();
      repoData.value = typeof json === 'string' ? JSON.parse(json) : json;
    }
    console.log("json信息：", repoData.value)
    menuList.value[0].count = getMapCount(repoData.value);
    menuList.value[1].count = getJsCount(repoData.value);
    menuList.value[2].count = getCombatCount(repoData.value);
    menuList.value[3].count = getCardCount(repoData.value);
  } catch (e) {
    console.error('获取仓库信息失败', e);
    repoError.value = true;
  }
  globalLoading.value = false;
}

// 计算当前菜单标题
const currentMenuTitle = computed(() => {
  const current = menuList.value.find(item => item.key === selectedMenu.value[0]);
  return current ? current.label : '';
});

// 计算搜索框占位符
const searchPlaceholder = computed(() => {
  switch (selectedMenu.value[0]) {
    case '1':
      return '搜索资源名称、作者、tag';
    case '2':
      return '搜索脚本名称、作者、简介、tag';
    case '3':
      return '搜索战斗策略名称、作者、tag';
    case '4':
      return '搜索七圣召唤策略名称、作者、tag';
    default:
      return '输入关键词查询';
  }
});

const handleSearch = (value) => {
  search.value = value;
};

const handleMenuSelect = ({ key }) => {
  selectedMenu.value = [key];
  search.value = '';
  selectedScript.value = null;
};

const openExternalLink = (link) => {
  window.open(link, '_blank');
};

// 跳转到GitHub仓库指定位置
const jumpToGitHub = () => {
  if (!selectedScript.value) return;
  const baseUrl = 'https://github.com/babalae/bettergi-scripts-list/blob/main/repo/';
  let targetPath = '';

  // 优先使用脚本的path属性
  if (selectedScript.value.path) {
    targetPath = selectedScript.value.path;
  } else {
    // 如果没有path属性，则根据脚本名称和菜单类型构建路径
    const scriptName = selectedScript.value.name1 || selectedScript.value.name || selectedScript.value.title;
    if (!scriptName) return;

    // 根据当前选中的菜单类型构建不同的路径
    switch (selectedMenu.value[0]) {
      case '1': // 地图追踪
        targetPath = `pathing/${scriptName}`;
        break;
      case '2': // Javascript脚本
        targetPath = `js/${scriptName}`;
        break;
      case '3': // 战斗策略
        targetPath = `combat/${scriptName}`;
        break;
      case '4': // 七圣召唤策略
        targetPath = `tcg/${scriptName}`;
        break;
      default:
        targetPath = scriptName;
    }
  }

  // 清理路径，移除多余斜杠和点
  targetPath = targetPath
    .replace(/\/+/g, '/')
    .replace(/^\/|\/$/g, '');

  // 对路径进行编码，特别注意编码方括号等特殊字符
  const encodedPath = encodeURI(targetPath)
    .replace(/\[/g, '%5B')
    .replace(/\]/g, '%5D');

  const githubUrl = baseUrl + encodedPath;
  openExternalLink(githubUrl);
};

const selectedScript = ref(null);
const selectedLocation = ref(null);

const handleScriptSelect = (script) => {
  selectedScript.value = script;
};

// 地图追踪板块
const handleMapSelect = (location) => {
  selectedLocation.value = location;
  selectedScript.value = location;
};

const mapTreeRef = ref(null);
const scriptListRef = ref(null);
const combatStrategyRef = ref(null);
const cardStrategyRef = ref(null);

const handleLeafCount = (count) => {
  const index = menuList.value.findIndex(item => item.key === '1');
  if (index !== -1) {
    menuList.value[index].count = count;
  }
};

const handleScriptCount = (count) => {
  const index = menuList.value.findIndex(item => item.key === '2');
  if (index !== -1) {
    menuList.value[index].count = count;
  }
};

function getJsCount(repo) {
  const jsNode = repo.indexes.find(item => item.name === 'js');

  const hasExpandableChildren = (dataRef) => {
    if (!dataRef?.children || dataRef.children.length === 0) return false;
    return dataRef.children.some(child => child.type === 'directory');
  };

  function countLeaf(nodes) {
    if (!nodes) return 0;
    let count = 0;
    for (const node of nodes) {
      if (!hasExpandableChildren(node)) {
        count++;
      } else {
        count += countLeaf(node.children);
      }
    }
    return count;
  }
  return countLeaf(jsNode?.children);
}

function getMapCount(repo) {
  const mapNode = repo.indexes.find(item => item.name === 'pathing');

  const hasExpandableChildren = (dataRef) => {
    if (!dataRef?.children || dataRef.children.length === 0) return false;
    return dataRef.children.some(child => child.type === 'directory');
  };

  function countLeaf(nodes) {
    if (!nodes) return 0;
    let count = 0;
    for (const node of nodes) {
      if (!hasExpandableChildren(node)) {
        count++;
      } else {
        count += countLeaf(node.children);
      }
    }
    return count;
  }
  return countLeaf(mapNode?.children);
}

function getCombatCount(repo) {
  const combatNode = repo.indexes.find(item => item.name === 'combat');

  const hasExpandableChildren = (dataRef) => {
    if (!dataRef?.children || dataRef.children.length === 0) return false;
    return dataRef.children.some(child => child.type === 'directory');
  };

  function countLeaf(nodes) {
    if (!nodes) return 0;
    let count = 0;
    for (const node of nodes) {
      if (!hasExpandableChildren(node)) {
        count++;
      } else {
        count += countLeaf(node.children);
      }
    }
    return count;
  }
  return countLeaf(combatNode?.children);
}

function getCardCount(repo) {
  const cardNode = repo.indexes.find(item => item.name === 'tcg');

  function countLeaf(nodes) {
    if (!nodes) return 0;
    let count = 0;
    for (const node of nodes) {
      if (node.type === 'file') {
        count++;
      } else if (node.type === 'directory' && node.children) {
        count += countLeaf(node.children);
      }
    }
    return count;
  }

  return countLeaf(cardNode?.children);
}

const subscribedScriptPaths = ref([]);
// 订阅信息获取失败弹窗控制变量
const subscribedErrorModalOpen = ref(false);
// 订阅信息获取失败标志
const subscribedConfigError = ref(false);

// 只在初始化时获取订阅信息
async function fetchSubscribedConfig() {
  if (mode !== 'single') return;
  globalLoading.value = true;
  try {
    const response = await fetch('http://localhost:30648/api/config');
    if (!response.ok) throw new Error('网络请求失败');
    const config = await response.json();
    const paths = Array.from(new Set(config.scriptConfig?.subscribedScriptPaths || []));
    if (!paths.length) {
      subscribedConfigError.value = true;
      subscribedScriptPaths.value = [];
    } else {
      subscribedScriptPaths.value = paths;
      subscribedConfigError.value = false;
    }
  } catch (e) {
    subscribedConfigError.value = true;
    subscribedScriptPaths.value = [];
  }
}

onMounted(() => {
  getRepoJson();
});

onUnmounted(() => {
  if (fetchTimeoutId) {
    clearTimeout(fetchTimeoutId);
    fetchTimeoutId = null;
  }
});

const commentModalOpen = ref(false);

// giscus 配置
const giscusConfig = {
  repo: 'zaodonganqi/bettergi-scripts-web',
  repoId: 'R_kgDOOdJNqw',
  category: 'Q&A',
  categoryId: 'DIC_kwDOOdJNq84CsJV4',
  mapping: 'specific',
  strict: '0',
  reactionsEnabled: '1',
  emitMetadata: '0',
  inputPosition: 'top',
  theme: 'light',
  lang: 'zh-CN'
};

// 计算 giscus term，用于区分不同脚本的评论
const giscusTerm = computed(() => {
  if (!selectedScript.value) return 'default';
  return selectedScript.value.path || selectedScript.value.title || 'default';
});

// 监听评论弹窗打开
watch(commentModalOpen, (newVal) => {
  if (newVal) {
    console.log('评论弹窗打开，当前脚本:', selectedScript.value);
  }
});

const handleCommentModalCancel = () => {
  commentModalOpen.value = false;
};

const showEggModal = ref(false);

// 彩蛋 README 加载状态
const isLoadingEggReadme = ref(false);
const eggReadmeError = ref(false);
const eggReadmeKey = ref(0);

const handleEggReadmeLoaded = () => {
  isLoadingEggReadme.value = false;
  eggReadmeError.value = false;
};

const handleEggReadmeError = () => {
  console.log('彩蛋 README 加载失败');
  isLoadingEggReadme.value = false;
  eggReadmeError.value = true;
};

const retryLoadEggReadme = () => {
  console.log('重试加载彩蛋 README');
  isLoadingEggReadme.value = true;
  eggReadmeError.value = false;
  eggReadmeKey.value++;
};

// 监听彩蛋弹窗打开，重置加载状态
watch(showEggModal, (newVal) => {
  if (newVal) {
    // 重置状态
    isLoadingEggReadme.value = true;
    eggReadmeError.value = false;
    eggReadmeKey.value++;
  }
});

const showHelpModal = ref(false);

const scriptTab = ref('all');

const onClickShowAll = () => {
  if (mode === 'web') return;
  scriptTab.value = 'all';
  handleShowAll(); // 预留方法
};
// 点击“已订阅”按钮时只根据标志弹窗，不再重新请求
const onClickShowSubscribed = () => {
  if (mode === 'web') return;
  if (subscribedConfigError.value) {
    subscribedErrorModalOpen.value = true;
    return;
  }
  scriptTab.value = 'subscribed';
  handleShowSubscribed(); // 预留方法
};

function handleShowAll() {
  // 预留：切换到全部脚本
}
function handleShowSubscribed() {
  // 预留：切换到已订阅脚本
}

watch(selectedMenu, () => {
  scriptTab.value = 'all';
});

</script>

<style scoped>
.main-layout {
  height: 100vh;
  width: 100%;
  margin: 0;
  background: #fff;
  display: flex;
}

.custom-sider {
  width: 18% !important;
  min-width: 18% !important;
  max-width: 18% !important;
  background: #f7f8fa !important;
  border-right: 1px solid #e5e7eb;
  display: flex;
  flex-direction: column;
  padding: 0;
  height: 100%;
}

.sider-header {
  height: 60px;
  display: flex;
  align-items: center;
  padding: 0 16px;
  font-weight: 600;
  font-size: 18px;
  color: #222;
  border-bottom: 1px solid #ececec;
  width: 100%;
  flex-shrink: 0;
}

.sider-logo {
  width: 40px;
  height: 40px;
  background: url('@/assets/favicon.ico') no-repeat center/contain;
  margin-right: 10px;
  flex-shrink: 0;
  cursor: pointer;
  transition: opacity 0.2s;
}

.sider-logo:hover {
  opacity: 0.8;
}

.repo-title {
  font-size: 16px;
  font-weight: 600;
  color: #222;
  margin-left: 10px;
  align-items: center;
}

.sider-menu-wrap {
  padding: 8px;
  width: 100%;
  overflow-y: auto;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
  height: calc(100% - 60px - 60px);
}

.menu-content {
  overflow-y: auto;
  overflow-x: hidden;
}

.custom-menu {
  background: transparent !important;
  width: 100%;
}

.custom-menu-item {
  height: 44px !important;
  display: flex;
  align-items: center;
  font-size: 15px;
  color: #222;
  border-radius: 8px;
  margin: 0 8px;
  padding: 0 16px !important;
}

.ant-menu-item-selected {
  background: #181822 !important;
  color: #000 !important;
}

.menu-icon {
  margin-right: 10px;
  font-size: 18px;
}

.menu-label {
  flex: 1;
}

.menu-count {
  color: #b0b0b0;
  font-size: 13px;
  margin-left: 8px;
}

.sider-footer {
  padding: 24px 16px;
  color: #999;
  font-size: 14px;
  border-top: 1px solid #ececec;
  width: 100%;
  flex-shrink: 0;
}

.sider-link {
  margin-bottom: 12px;
  cursor: pointer;
}

.last-update-time {
  padding: 12px 25px;
  color: #999;
  font-size: 12px;
  border-top: 1px solid #ececec;
  width: 100%;
  flex-shrink: 0;
  height: 60px;
  display: flex;
  flex-wrap: wrap;
  align-content: center;
  gap: 0 4px;
}

.last-update-time span {
  color: #666;
  font-size: 15px;
}

.script-sider {
  width: 23% !important;
  min-width: 23% !important;
  max-width: 23% !important;
  background: #f7f8fa !important;
  border-right: 1px solid #e5e7eb;
  height: 100%;
  position: relative;
}

.script-header {
  height: 60px;
  padding: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #ececec;
}

.script-title {
  font-size: 16px;
  font-weight: 600;
  color: #222;
}

.script-actions {
  display: flex;
  gap: 8px;
}

.script-btn {
  height: 28px;
  padding: 0 12px;
  font-size: 14px;
  border-radius: 4px;
  background: #fff !important;
  color: #1677ff !important;
  border: 1px solid #1677ff !important;
  transition: background 0.2s, color 0.2s;
}

.search-section {
  height: 60px;
  padding-top: 10px;
  padding-bottom: 10px;
  padding-left: 16px;
  padding-right: 16px;
  border-bottom: 1px solid #ececec;
}

.script-search {
  width: 100%;
  height: 100%;
}

.script-list {
  position: absolute;
  top: 120px;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  overflow-x: hidden;
  overflow-y: auto;
  background: #f7f8fa;
}

.main-right {
  flex: 1;
  height: 100%;
  background: #f5f6fa;
  padding: 0;
  overflow-y: auto;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
}

.detail-top-bar {
  height: 60px;
  background: #f7f8fa;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 16px;
  flex-shrink: 0;
}

.top-bar-left {
  display: flex;
  gap: 8px;
}

.top-bar-right {
  display: flex;
  gap: 8px;
}

.action-btn {
  color: #444;
  border: none;
  padding: 2px 4px;
  border-radius: 4px;
  transition: all 0.2s;
}

.action-btn:hover {
  color: #1890ff;
  background: rgba(24, 144, 255, 0.1);
}

.main-right>div:last-child {
  flex: 1;
  overflow: hidden;
}

.strategy-tabs {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.strategy-tabs :deep(.ant-tabs-content) {
  flex: 1;
  overflow-y: auto;
}

.strategy-list {
  padding: 8px;
}

.strategy-item {
  background: #fff;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 12px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  transition: all 0.3s;
}

.strategy-item:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  transform: translateY(-1px);
}

.strategy-item-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.strategy-title {
  font-size: 16px;
  font-weight: 500;
  color: #222;
}

.strategy-tag {
  font-size: 12px;
  color: #1890ff;
  background: rgba(24, 144, 255, 0.1);
  padding: 2px 8px;
  border-radius: 4px;
}

.strategy-desc {
  font-size: 14px;
  color: #666;
  margin-bottom: 8px;
  line-height: 1.5;
}

.strategy-meta {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #999;
}

.strategy-author {
  color: #1890ff;
}

.strategy-date {
  color: #999;
}

.repo-error-modal {
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  z-index: 999;
  background: rgba(0, 0, 0, 0.18);
  display: flex;
  align-items: center;
  justify-content: center;
  animation: fadeIn 0.3s;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

.repo-error-content {
  background: #fff;
  color: #d32f2f;
  font-size: 18px;
  padding: 36px 48px 32px 48px;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.18);
  text-align: center;
  min-width: 320px;
  max-width: 90vw;
  animation: popIn 0.3s;
}

@keyframes popIn {
  from {
    transform: scale(0.9);
    opacity: 0;
  }

  to {
    transform: scale(1);
    opacity: 1;
  }
}

.repo-error-icon {
  margin-bottom: 16px;
}

.repo-error-title {
  font-size: 22px;
  font-weight: bold;
  margin-bottom: 8px;
  color: #f44336;
}

.repo-error-desc {
  font-size: 15px;
  color: #666;
  margin-bottom: 24px;
}

.repo-error-btn {
  background: linear-gradient(90deg, #ff6a6a 0%, #f44336 100%);
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 10px 32px;
  font-size: 16px;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(244, 67, 54, 0.12);
  transition: background 0.2s, box-shadow 0.2s;
}

.repo-error-btn:hover {
  background: linear-gradient(90deg, #f44336 0%, #ff6a6a 100%);
  box-shadow: 0 4px 16px rgba(244, 67, 54, 0.18);
}

.global-loading-modal {
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  z-index: 99999;
  background: rgba(255, 255, 255, 0.65);
  display: flex;
  align-items: center;
  justify-content: center;
}

.comment-modal {
  border-radius: 12px;
  overflow: hidden;
}

.comment-modal :deep(.ant-modal-content) {
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12);
}

.comment-modal :deep(.ant-modal-header) {
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-bottom: 1px solid #e5e7eb;
  padding: 20px 24px;
}

.comment-modal :deep(.ant-modal-title) {
  font-size: 18px;
  font-weight: 600;
  color: #1a1a1a;
}

.comment-modal :deep(.ant-modal-body) {
  padding: 0;
  max-height: 90vh;
  overflow: hidden;
}

.comment-modal-content {
  padding: 0;
  max-width: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  height: 100%;
}

.comment-header {
  margin: 0;
  padding: 16px 24px;
  background: #f8f9fa;
  border-bottom: 1px solid #e5e7eb;
  flex-shrink: 0;
}

.script-title-simple {
  font-size: 18px;
  font-weight: 600;
  color: #1a1a1a;
  line-height: 1.4;
  display: flex;
  align-items: center;
  gap: 8px;
}

.script-author {
  font-size: 14px;
  font-weight: 400;
  color: #666;
  opacity: 0.8;
}

.author-link {
  color: #1677ff;
  text-decoration: none;
}

.author-link:hover {
  text-decoration: underline;
}

.comments-container {
  flex: 1;
  max-height: calc(90vh - 60px);
  overflow-y: auto;
  overflow-x: hidden;
  padding: 0;
  position: relative;
  width: 100%;
}

.comments-container :deep(.giscus) {
  border: none;
  border-radius: 0;
  background: transparent;
  width: 100%;
  min-width: 100%;
}

@media (max-width: 1400px) {
  .comment-modal {
    width: 95% !important;
    margin: 10px;
  }

  .comment-modal :deep(.ant-modal-body) {
    max-height: 85vh;
  }

  .comments-container {
    max-height: calc(85vh - 60px);
  }
}

@media (max-width: 768px) {
  .comment-modal {
    width: 98% !important;
    margin: 5px;
  }

  .comment-modal :deep(.ant-modal-body) {
    max-height: 90vh;
  }

  .comments-container {
    max-height: calc(90vh - 60px);
  }

  .comment-header {
    padding: 12px 16px;
  }

  .script-title-simple {
    font-size: 16px;
  }
}

.local-mode-notice {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 60px 20px;
  text-align: center;
  height: 100%;
  min-height: 400px;
}

.notice-icon {
  margin-bottom: 24px;
}

.notice-title {
  font-size: 20px;
  font-weight: 600;
  color: #1a1a1a;
  margin-bottom: 12px;
}

.notice-desc {
  font-size: 16px;
  color: #666;
  margin-bottom: 32px;
  line-height: 1.5;
}

.notice-btn {
  background: linear-gradient(135deg, #1677ff 0%, #096dd9 100%);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 12px 32px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(24, 144, 255, 0.3);
  transition: all 0.3s ease;
}

.notice-btn:hover {
  background: linear-gradient(135deg, #096dd9 0%, #0050b3 100%);
  box-shadow: 0 6px 16px rgba(24, 144, 255, 0.4);
  transform: translateY(-1px);
}

.notice-btn:active {
  transform: translateY(0);
}

.egg-modal-content {
  max-height: 65vh;
  overflow-y: overlay;
  padding-left: 15px;
  padding-right: 15px;
  font-size: clamp(14px, 1.5vw, 16px);
  position: relative;
}

.egg-readme-tabs {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 16px;
  gap: 10px;
}


.egg-readme-loading {
  text-align: center;
  padding: 48px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.egg-readme-error {
  text-align: center;
  padding: 48px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.egg-readme-loading-indicator {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #666;
  font-size: 14px;
}

.egg-readme-content {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
}

.fade-slide-up-enter-active,
.fade-slide-up-leave-active {
  transition: all 0.3s ease;
}

.fade-slide-up-enter-from {
  opacity: 0;
  transform: translateY(20px);
}

.fade-slide-up-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}

.repo-error-btn-group {
  display: flex;
  flex-direction: row;
  gap: 16px;
  justify-content: center;
  margin-top: 8px;
}

.repo-help-btn {
  background: linear-gradient(90deg, #1677ff 0%, #40a9ff 100%);
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 10px 32px;
  font-size: 16px;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(24, 144, 255, 0.12);
  transition: background 0.2s, box-shadow 0.2s;
}

.repo-help-btn:hover {
  background: linear-gradient(90deg, #40a9ff 0%, #1677ff 100%);
  box-shadow: 0 4px 16px rgba(24, 144, 255, 0.18);
}

.script-btn-active {
  background: #1677ff !important;
  color: #fff !important;
  border: 1px solid #1677ff !important;
}

.script-actions-mask {
  position: absolute;
  left: -8px;
  top: -8px;
  width: calc(100% + 16px);
  height: calc(100% + 16px);
  background: rgba(255, 255, 255, 0.88);
  z-index: 2;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 15px;
  color: #888;
  border-radius: 10px;
  pointer-events: all;
  box-sizing: border-box;
  padding: 8px 0;
}

.subscribed-error-msg {
  font-size: 15px;
}
</style>