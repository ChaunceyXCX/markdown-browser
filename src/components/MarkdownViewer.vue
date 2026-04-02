<script setup lang="ts">
import { ref, computed } from 'vue'
import { marked, Renderer } from 'marked'
import { markedHighlight } from 'marked-highlight'
import hljs from 'highlight.js'

// Create custom renderer for code blocks
const renderer = new Renderer()

renderer.code = function(token: any) {
  const text = token.text
  const language = token.lang || 'text'
  let highlighted = text
  
  if (language && hljs.getLanguage(language)) {
    try {
      highlighted = hljs.highlight(text, { language }).value
    } catch (e) {
      console.error(e)
    }
  } else {
    highlighted = hljs.highlightAuto(text).value
  }
  
  // Encode code for data attribute
  const encodedCode = encodeURIComponent(text)
  
  return `<div class="code-block-wrapper">
    <button class="code-copy-btn" onclick="copyCodeBlock(this, '${encodedCode}')">
      <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
        <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
      </svg>
      <span class="copy-text">复制</span>
    </button>
    <pre><code class="hljs language-${language}">${highlighted}</code></pre>
  </div>`
}

// Configure marked with custom renderer and highlight.js
marked.use(
  markedHighlight({
    langPrefix: 'hljs language-',
    highlight(code: string, lang: string) {
      if (lang && hljs.getLanguage(lang)) {
        try {
          return hljs.highlight(code, { language: lang }).value
        } catch (e) {
          console.error(e)
        }
      }
      return hljs.highlightAuto(code).value
    }
  })
)

marked.setOptions({ renderer })

const markdownContent = ref('')
const isPreview = ref(false)
const copySuccess = ref(false)

const renderedHtml = computed(() => {
  if (!markdownContent.value) return ''
  return marked(markdownContent.value)
})

function togglePreview() {
  isPreview.value = !isPreview.value
}

function clearContent() {
  markdownContent.value = ''
  isPreview.value = false
}

async function copyContent() {
  try {
    await navigator.clipboard.writeText(markdownContent.value)
    copySuccess.value = true
    setTimeout(() => {
      copySuccess.value = false
    }, 2000)
  } catch (err) {
    console.error('Failed to copy:', err)
  }
}

// Expose copy function globally for code block buttons
if (typeof window !== 'undefined') {
  (window as any).copyCodeBlock = async function(btn: HTMLElement, encodedCode: string) {
    const code = decodeURIComponent(encodedCode)
    try {
      await navigator.clipboard.writeText(code)
      const copyText = btn.querySelector('.copy-text') as HTMLElement
      if (copyText) {
        copyText.textContent = '已复制'
      }
      btn.classList.add('success')
      setTimeout(() => {
        if (copyText) {
          copyText.textContent = '复制'
        }
        btn.classList.remove('success')
      }, 2000)
    } catch (err) {
      console.error('Failed to copy code:', err)
    }
  }
}
</script>

<template>
  <div class="markdown-browser">
    <div class="toolbar">
      <button @click="togglePreview" :disabled="!markdownContent">
        {{ isPreview ? '编辑' : '预览' }}
      </button>
      <button @click="clearContent" :disabled="!markdownContent" class="clear-btn">
        清空
      </button>
    </div>

    <div class="editor-container" :class="{ 'has-content': markdownContent }">
      <textarea
        v-if="!isPreview"
        v-model="markdownContent"
        placeholder="在此粘贴 Markdown 代码..."
        class="markdown-input"
      ></textarea>
      <div v-else class="preview-wrapper">
        <button class="copy-btn" @click="copyContent" :class="{ success: copySuccess }">
          <svg v-if="!copySuccess" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect>
            <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path>
          </svg>
          <svg v-else xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <polyline points="20 6 9 17 4 12"></polyline>
          </svg>
          {{ copySuccess ? '已复制' : '复制' }}
        </button>
        <div class="markdown-content" v-html="renderedHtml"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.markdown-browser {
  max-width: 900px;
  margin: 0 auto;
  padding: 20px;
}

.toolbar {
  margin-bottom: 20px;
  display: flex;
  gap: 10px;
}

.toolbar button {
  padding: 10px 20px;
  background: #42b883;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.2s;
}

.toolbar button:hover:not(:disabled) {
  background: #33a06f;
}

.toolbar button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.toolbar .clear-btn {
  background: #666;
}

.toolbar .clear-btn:hover:not(:disabled) {
  background: #444;
}

.editor-container {
  border: 2px dashed #ddd;
  border-radius: 8px;
  min-height: 500px;
  transition: all 0.3s;
}

.editor-container.has-content {
  border-style: solid;
  border-color: #eee;
}

.markdown-input {
  width: 100%;
  min-height: 500px;
  padding: 20px;
  border: none;
  resize: vertical;
  font-family: 'Fira Code', 'Consolas', monospace;
  font-size: 14px;
  line-height: 1.6;
  background: #fafafa;
  border-radius: 8px;
  outline: none;
}

.markdown-input:focus {
  background: #fff;
  box-shadow: inset 0 0 0 2px #42b883;
}

.markdown-input::placeholder {
  color: #999;
}

.markdown-content {
  padding: 20px;
  line-height: 1.6;
  min-height: 500px;
  background: #fff;
  border-radius: 8px;
}

.preview-wrapper {
  position: relative;
}

.copy-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: #333;
  color: #fff;
  border: 1px solid #333;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
  z-index: 10;
}

.copy-btn:hover {
  background: #555;
}

.copy-btn.success {
  background: #42b883;
  border-color: #42b883;
}

.markdown-content :deep(h1) {
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.markdown-content :deep(h2) {
  border-bottom: 1px solid #eee;
  padding-bottom: 8px;
}

.markdown-content :deep(code) {
  background: #f5f5f5;
  padding: 2px 6px;
  border-radius: 3px;
  font-family: 'Fira Code', monospace;
}

.markdown-content :deep(pre) {
  background: #282c34;
  color: #abb2bf;
  padding: 15px;
  border-radius: 6px;
  overflow-x: auto;
}

.markdown-content :deep(.code-block-wrapper) {
  position: relative;
  margin: 16px 0;
}

.markdown-content :deep(.code-block-wrapper pre) {
  margin: 0;
  padding-top: 35px;
}

.markdown-content :deep(.code-copy-btn) {
  position: absolute;
  top: 8px;
  right: 8px;
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  background: rgba(255, 255, 255, 0.1);
  color: #abb2bf;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  transition: all 0.2s;
  z-index: 10;
}

.markdown-content :deep(.code-copy-btn:hover) {
  background: rgba(255, 255, 255, 0.2);
  color: #fff;
}

.markdown-content :deep(.code-copy-btn.success) {
  background: #42b883;
  border-color: #42b883;
  color: #fff;
}

.markdown-content :deep(pre code) {
  background: none;
  color: inherit;
  padding: 0;
}

.markdown-content :deep(blockquote) {
  border-left: 4px solid #42b883;
  margin: 0;
  padding-left: 15px;
  color: #666;
}

.markdown-content :deep(img) {
  max-width: 100%;
}

.markdown-content :deep(table) {
  border-collapse: collapse;
  width: 100%;
}

.markdown-content :deep(th),
.markdown-content :deep(td) {
  border: 1px solid #ddd;
  padding: 8px;
}

.markdown-content :deep(tr:nth-child(even)) {
  background: #f9f9f9;
}
</style>