```vue
<template>
  <div class="readme-viewer">
    <div v-if="readmeContent" v-html="readmeContent" class="readme-content"></div>
    <div v-else-if="desc" class="detail-desc">{{ showDescTitle ? $t('readmeViewer.descTitle') + '\n' + desc : desc }}</div>
    <div v-else-if="!isHttpUrl" class="readme-empty">{{ $t('readmeViewer.noDesc') }}</div>
  </div>
</template>

<script setup>
import { ref, watch, computed } from 'vue';
import MarkdownIt from 'markdown-it';
import markdownItAnchor from 'markdown-it-anchor';
import hljs from 'highlight.js';
import 'highlight.js/styles/github.css';
import { getWebPath, getRepoPath } from '@/utils/basePaths';
import { useI18n } from 'vue-i18n';
const { t } = useI18n();

const props = defineProps({
  path: String,
  desc: String,
  showDescTitle: {
    type: Boolean,
    default: false
  },
  forceWeb: {
    type: Boolean,
    default: false
  }
});

const emit = defineEmits(['loaded', 'error', 'hasContent']);
const readmeContent = ref('');
const isLoading = ref(false);
const loadError = ref(null);

// 判断是否为 HTTP URL
const isHttpUrl = computed(() => {
  return props.path && /^https?:\/\//i.test(props.path);
});

const md = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true,
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return '<pre class="hljs"><code>' +
          hljs.highlight(str, { language: lang, ignoreIllegals: true }).value +
          '</code></pre>';
      } catch (__) { }
    }
    return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>';
  }
}).use(markdownItAnchor, {
  level: [1, 2, 3, 4, 5, 6]
});

// 处理链接渲染逻辑
const originalLinkRender = md.renderer.rules.link_open || function (tokens, idx, options, env, self) {
  return self.renderToken(tokens, idx, options);
};
md.renderer.rules.link_open = function (tokens, idx, options, env, self) {
  const token = tokens[idx];
  const hrefIndex = token.attrIndex('href');
  if (hrefIndex >= 0) {
    const href = token.attrs[hrefIndex][1];
    const isValidHttpLink = /^https?:\/\/[\S]+$/i.test(href);
    const isRelativePath = /^\.\.?\//.test(href) || /^[^\/]*\/[^\/]+(?:\/[^\/]+)*$/i.test(href);
    const isPotentialLink = /^[a-z0-9-]+\.[a-z]{2,}\b/i.test(href);

    // 更精确的相对路径判断：排除明显不是路径的情况
    const isActuallyRelativePath = isRelativePath &&
      !href.includes('://') &&
      !href.includes('mailto:') &&
      !href.includes('tel:') &&
      !href.includes('javascript:') &&
      !/^[a-z0-9-]+\.[a-z]{2,}\b/i.test(href) && // 排除域名形式的链接
      !/\.(png|jpg|jpeg|gif|webp|svg|ico|bmp|tiff)$/i.test(href); // 排除图片文件路径

    if (isValidHttpLink) {
      token.attrPush(['target', '_blank']);
      token.attrPush(['rel', 'noopener noreferrer']);
    } else if (isActuallyRelativePath) {
      // 处理相对路径链接，转换为GitHub web界面链接
      const baseUrl = getWebPath();
      const currentPath = props.path || '';

      // 解析相对路径
      let targetPath = href;

      // 处理相对路径
      if (href.startsWith('../') || href.startsWith('./')) {
        // 标准化当前路径（移除末尾的斜杠）
        const normalizedCurrentPath = currentPath.replace(/\/$/, '');

        // 分割路径部分
        const currentParts = normalizedCurrentPath.split('/').filter(Boolean);
        const relativeParts = href.split('/').filter(part => part !== '.' && part !== '');

        // 计算需要回退的层级
        let backLevels = 0;
        for (const part of relativeParts) {
          if (part === '..') backLevels++;
          else break;
        }

        // 构建目标路径
        if (backLevels <= currentParts.length) {
          targetPath = [
            ...currentParts.slice(0, -backLevels),
            ...relativeParts.slice(backLevels)
          ].join('/');
        } else {
          // 如果回退层级超过当前路径深度，从仓库根目录开始
          targetPath = relativeParts.slice(backLevels).join('/');
        }
      } else {
        // 非标准相对路径（如直接写路径的情况）
        targetPath = currentPath ? `${currentPath}/${href}` : href;
      }
      // 移除路径中的多余斜杠和点
      targetPath = targetPath
        .replace(/\/+/g, '/')
        .replace(/^\/|\/$/g, '')
        .replace(/\/\.(?=\/|$)/g, '');

      // 判断是否为文件（有扩展名）还是目录
      const isFile = /\.[a-zA-Z0-9]+$/.test(targetPath);
      const githubUrl = baseUrl + targetPath;

      token.attrs[hrefIndex][1] = githubUrl;
      token.attrPush(['target', '_blank']);
      token.attrPush(['rel', 'noopener noreferrer']);
      token.attrPush(['class', 'internal-link']);
      token.attrPush(['title', isFile ? t('readmeViewer.clickToViewFile') : t('readmeViewer.clickToViewDir')]);
    } else if (isPotentialLink) {
      token.attrPush(['class', 'invalid-link']);
      token.attrPush(['onclick', 'return false;']);
      token.attrPush(['style', 'cursor: default;']);
      const textToken = tokens[idx + 1];
      if (textToken && textToken.type === 'text') {
        textToken.content += t('readmeViewer.invalidLinkHint');
      }
    }
  }
  return originalLinkRender(tokens, idx, options, env, self);
};

// 获取readme文件URL
function getReadmeUrl(path) {

  // 检查是否已经是外链（以 http:// 或 https:// 开头）
  if (/^https?:\/\//i.test(path)) {
    return path;
  }
  // 否则拼接 GitHub Raw URL
  return getRepoPath() + path.replace(/\\/g, '/') + '/README.md';
}

function isReadme404(path) {
  return !!localStorage.getItem('readme404:' + path);
}

// 获取并渲染readme内容
const fetchAndRenderReadme = async (path) => {
  if (!path) {
    readmeContent.value = '';
    emit('loaded');
    emit('hasContent', false);
    return;
  }
  if (isReadme404(path)) {
    readmeContent.value = '';
    emit('loaded', { status: '404' });
    emit('hasContent', false);
    return;
  }
  isLoading.value = true;
  loadError.value = null;
  readmeContent.value = '';

  let mode = import.meta.env.VITE_MODE;
  let markdown = '';
  let fetchError = null;
  if (props.forceWeb) {
    mode = 'web';
  }
  if (mode === 'single') {
    try {
      const repoWebBridge = chrome.webview.hostObjects.repoWebBridge;
      markdown = await repoWebBridge.GetFile(path.replace(/\\/g, '/') + '/README.md');
    } catch (e) {
      fetchError = e;
    }
    if (fetchError) {
      readmeContent.value = '';
      if (fetchError.name === 'AbortError') {
        loadError.value = t('readmeViewer.loadTimeout');
        emit('loaded', { status: 'error', message: t('readmeViewer.loadTimeout') });
        emit('error', t('readmeViewer.loadTimeout'));
      } else {
        loadError.value = t('readmeViewer.loadFailed');
        emit('loaded', { status: 'error', message: t('readmeViewer.loadFailed') });
        emit('error', t('readmeViewer.loadFailed'));
      }
      emit('hasContent', false);
      isLoading.value = false;
      return;
    }
    if (markdown === '404') {
      readmeContent.value = '';
      emit('loaded', { status: '404' });
      emit('hasContent', false);
      isLoading.value = false;
      return;
    }
    // 图片路径处理和渲染
    const baseImageUrl = '../../../Repos/bettergi-scripts-list-git/repo/' + path.replace(/\\/g, '/') + '/';
    markdown = markdown.replace(/!\[([^\]]*)\]\((?!https?:\/\/|data:)([^)]+)\)/gi, (alt, imgPath) => {
      let cleanPath = imgPath.trim().replace(/\\/g, '/');
      if (!cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
        const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
        cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
      }
      return `![${alt}](${baseImageUrl}${cleanPath})`;
    });
    markdown = markdown.replace(/<img\s+[^>]*src=["']([^"']+)["'][^>]*>/gi, (match, imgPath) => {
      let cleanPath = imgPath.trim().replace(/\\/g, '/');
      if (!cleanPath.startsWith('assets/') && !cleanPath.startsWith('http') && !cleanPath.startsWith('data:')) {
        const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
        cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
      }
      return match.replace(imgPath, baseImageUrl + cleanPath);
    });
    markdown = markdown.replace(/`([^`\n]+\.(?:png|jpg|jpeg|gif|webp|svg))`/gi, (imgPath) => {
      let cleanPath = imgPath.trim().replace(/\\/g, '/');
      if (!cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
        const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
        cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
      }
      return `![](${baseImageUrl}${cleanPath})`;
    });
    markdown = markdown.replace(/```\s*([^`\n]+\.(?:png|jpg|jpeg|gif|webp|svg))\s*```/gi, (imgPath) => {
      let cleanPath = imgPath.trim().replace(/\\/g, '/');
      if (!cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
        const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
        cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
      }
      return `![](${baseImageUrl}${cleanPath})`;
    });
    readmeContent.value = md.render(markdown);
    emit('loaded', { status: 'ok' });
    emit('hasContent', true);
    isLoading.value = false;
    return;
  } else {
    // 拼接完整URL
    const readmeUrl = getReadmeUrl(path);
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 6000);
    try {
      const response = await fetch(readmeUrl, { signal: controller.signal });
      if (response.ok) {
        markdown = await response.text();
      } else if (response.status === 404) {
        readmeContent.value = '';
        emit('loaded', { status: '404' });
        emit('hasContent', false);
        clearTimeout(timeoutId);
        isLoading.value = false;
        return;
      } else {
        fetchError = new Error('Load failed');
      }
    } catch (e) {
      fetchError = e;
    } finally {
      clearTimeout(timeoutId);
    }
  }
  if (fetchError) {
    readmeContent.value = '';
    if (fetchError.name === 'AbortError') {
      loadError.value = t('readmeViewer.loadTimeout');
      emit('loaded', { status: 'error', message: t('readmeViewer.loadTimeout') });
      emit('error', t('readmeViewer.loadTimeout'));
    } else {
      loadError.value = t('readmeViewer.loadFailed');
      emit('loaded', { status: 'error', message: t('readmeViewer.loadFailed') });
      emit('error', t('readmeViewer.loadFailed'));
    }
    emit('hasContent', false);
    isLoading.value = false;
    return;
  }
  // 处理图片路径，将相对路径转换为绝对路径
  let baseImageUrl;
  if (props.forceWeb) {
    baseImageUrl = path.replace(/README\.md$/i, "");
    if (!baseImageUrl.endsWith('/')) {
      baseImageUrl += '/';
    }
  } else {
    baseImageUrl = getRepoPath() + path.replace(/\\/g, '/') + '/';
  }

  // 处理markdown图片语法 ![alt](path)
  markdown = markdown.replace(/!\[[^\]]*\]\(([^)]+)\)/g, (match, imgPath) => {
    let cleanPath = imgPath.trim().replace(/\\/g, '/');
    if (!props.forceWeb && !cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
      const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
      cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
    }
    return match.replace(imgPath, baseImageUrl + cleanPath);
  });

  // 处理HTML img标签
  markdown = markdown.replace(/<img\s+[^>]*src=["']([^"']+)["'][^>]*>/gi, (match, imgPath) => {
    let cleanPath = imgPath.trim().replace(/\\/g, '/');
    if (!props.forceWeb && !cleanPath.startsWith('assets/') && !cleanPath.startsWith('http') && !cleanPath.startsWith('data:')) {
      const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
      cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
    }
    return match.replace(imgPath, baseImageUrl + cleanPath);
  });

  // 处理代码块中的图片路径
  markdown = markdown.replace(/`([^`\n]+\.(?:png|jpg|jpeg|gif|webp|svg))`/gi, (imgPath) => {
    let cleanPath = imgPath.trim().replace(/\\/g, '/');
    if (!props.forceWeb && !cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
      const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
      cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
    }
    return `![](${baseImageUrl}${cleanPath})`;
  });

  // 处理代码块中的图片路径
  markdown = markdown.replace(/```\s*([^`\n]+\.(?:png|jpg|jpeg|gif|webp|svg))\s*```/gi, (imgPath) => {
    let cleanPath = imgPath.trim().replace(/\\/g, '/');
    if (!props.forceWeb && !cleanPath.startsWith('assets/') && !cleanPath.startsWith('http')) {
      const currentDir = props.path ? props.path.replace(/\\/g, '/') : '';
      cleanPath = currentDir ? `${currentDir}/${cleanPath}` : cleanPath;
    }
    return `![](${baseImageUrl}${cleanPath})`;
  });
  readmeContent.value = md.render(markdown);
  emit('loaded', { status: 'ok' });
  emit('hasContent', true);
  isLoading.value = false;
};

// 监听路径变化
watch(
  () => props.path,
  (newPath) => {
    fetchAndRenderReadme(newPath);
  },
  { immediate: true }
);
</script>

<style scoped>
.readme-viewer {
  width: 100%;
}

.readme-content {
  color: #000;
  font-size: 15px;
  line-height: 1.8;
  word-break: break-word;
}

.readme-content :deep(h1),
.readme-content :deep(h2),
.readme-content :deep(h3),
.readme-content :deep(h4),
.readme-content :deep(h5),
.readme-content :deep(h6) {
  margin-top: 24px;
  margin-bottom: 16px;
  font-weight: 600;
}

.readme-content :deep(h1) {
  font-size: 2em;
}

.readme-content :deep(h2) {
  font-size: 1.5em;
}

.readme-content :deep(h3) {
  font-size: 1.25em;
}

.readme-content :deep(h4) {
  font-size: 1em;
}

.readme-content :deep(h5) {
  font-size: 0.875em;
}

.readme-content :deep(h6) {
  font-size: 0.85em;
}

.readme-content :deep(p) {
  margin-bottom: 16px;
}

.readme-content :deep(.invalid-link) {
  color: #ff4d4f;
  text-decoration: line-through;
  pointer-events: none;
  cursor: default;
}

.readme-content :deep(.internal-link) {
  color: #1890ff;
  text-decoration: none;
  border-bottom: 1px dashed #1890ff;
  transition: all 0.3s ease;
}

.readme-content :deep(.internal-link:hover) {
  color: #40a9ff;
  border-bottom-color: #40a9ff;
  text-decoration: none;
}

.readme-content :deep(.link-hint) {
  color: #ff4d4f !important;
  font-size: 0.85em;
  font-style: italic;
  margin-left: 4px;
  display: inline-block;
}

.readme-content :deep(.link-hint *) {
  color: #ff4d4f !important;
}

.readme-content :deep(ul),
.readme-content :deep(ol) {
  padding-left: 20px;
}

.readme-content :deep(pre) {
  background-color: #f6f8fa;
  border-radius: 6px;
  padding: 16px;
  overflow: auto;
  margin-bottom: 1.5em;
}

.readme-content :deep(pre code) {
  display: block;
  background: none;
  margin: 0;
  padding: 0;
  font-size: 1em;
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, Courier, monospace;
}

.readme-content :deep(blockquote) {
  border-left: 0.25em solid #dfe2e5;
  color: #6a737d;
  padding: 0 1em;
  margin-left: 0;
}

.readme-content :deep(table) {
  border-collapse: collapse;
  margin: 1rem 0;
  display: block;
  overflow-x: auto;
}

.readme-content :deep(tr) {
  border-top: 1px solid #c6cbd1;
}

.readme-content :deep(tr:nth-child(2n)) {
  background-color: #f6f8fa;
}

.readme-content :deep(th),
.readme-content :deep(td) {
  border: 1px solid #dfe2e5;
  padding: 0.6em 1em;
}

.readme-empty {
  color: #bbb;
  text-align: center;
  margin-top: 40px;
  font-size: 16px;
}

.detail-desc {
  padding-top: 16px;
  color: #000;
  font-size: 15px;
  white-space: pre-line;
  word-break: break-word;
}
</style>