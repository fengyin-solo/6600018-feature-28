<template>
  <div class="flex h-screen">
    <!-- Left: Document list -->
    <div class="w-64 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800">
      <h1 class="text-lg font-bold text-amber-400">古籍 OCR 标注平台</h1>

      <div>
        <label class="block bg-amber-500 text-black text-center py-2 rounded cursor-pointer hover:bg-amber-400 text-sm font-medium">
          上传古籍图片
          <input type="file" accept="image/*" @change="onUpload" class="hidden" />
        </label>
      </div>

      <button @click="store.loadMockDocument()" class="bg-gray-800 py-2 rounded text-sm hover:bg-gray-700">
        加载示例文档
      </button>

      <!-- Search -->
      <div>
        <input v-model="store.searchQuery" @input="store.searchInDocuments(store.searchQuery)"
          placeholder="全文检索..." class="w-full bg-gray-800 rounded px-3 py-2 text-sm" />
        <div v-if="store.searchResults.length" class="mt-1 space-y-1">
          <div v-for="r in store.searchResults" :key="r.id" class="bg-gray-800 rounded p-1 text-xs">
            {{ r.text }} <span class="text-gray-500">{{ (r.confidence * 100).toFixed(0) }}%</span>
          </div>
        </div>
      </div>

      <!-- Document list -->
      <div class="flex-1 overflow-y-auto space-y-1">
        <div v-for="d in store.documents" :key="d.id" @click="store.currentDoc = d"
          class="bg-gray-800 rounded p-2 cursor-pointer text-sm"
          :class="store.currentDoc?.id === d.id ? 'ring-1 ring-amber-500' : ''">
          {{ d.name }}
          <div class="text-xs text-gray-500">{{ d.results.length }} 行识别</div>
        </div>
      </div>

      <!-- Export -->
      <button @click="doExport" class="bg-green-700 py-2 rounded text-sm hover:bg-green-600">
        导出 TEI/XML
      </button>
    </div>

    <!-- Center: Image + OCR overlay -->
    <div class="flex-1 relative bg-gray-950 overflow-hidden">
      <ImageCanvas v-if="store.currentDoc" />
      <div v-else class="flex items-center justify-center h-full text-gray-600">
        请上传古籍图片或加载示例文档
      </div>
    </div>

    <!-- Right: OCR results & annotations -->
    <div class="w-80 bg-gray-900 p-4 flex flex-col gap-3 border-l border-gray-800 overflow-y-auto">
      <h3 class="text-amber-300 font-bold text-sm">OCR 识别结果</h3>
      <div v-if="store.currentDoc" class="space-y-2">
        <div v-for="r in store.currentDoc.results" :key="r.id"
          class="bg-gray-800 rounded p-2 text-sm">
          <div class="flex justify-between">
            <span class="text-white font-medium">{{ r.text }}</span>
            <span class="text-xs px-2 py-0.5 rounded"
              :class="r.confidence > 0.9 ? 'bg-green-900 text-green-400' : 'bg-yellow-900 text-yellow-400'">
              {{ (r.confidence * 100).toFixed(0) }}%
            </span>
          </div>
          <div class="text-xs text-gray-400 mt-1">
            简体: {{ store.convertVariant(r.text) }}
          </div>
          <input v-model="r.corrected" placeholder="人工校正..."
            class="w-full bg-gray-700 rounded px-2 py-1 text-xs mt-1" />
        </div>
      </div>

      <h3 class="text-amber-300 font-bold text-sm mt-4">标注列表</h3>
      <div v-if="store.currentDoc" class="space-y-1">
        <div v-for="a in store.currentDoc.annotations" :key="a.id"
          class="bg-gray-800 rounded p-2 text-xs flex justify-between items-center">
          <span>[{{ a.type }}] {{ a.label }}: {{ a.content }}</span>
          <button
            v-if="pendingDeleteId !== a.id"
            @click="pendingDeleteId = a.id"
            class="text-red-400 hover:underline ml-2 shrink-0"
          >删除</button>
          <span v-else class="flex items-center gap-1 ml-2 shrink-0">
            <button @click="confirmDelete(a.id)" class="text-red-400 hover:underline">确认</button>
            <button @click="pendingDeleteId = ''" class="text-gray-400 hover:underline">取消</button>
          </span>
        </div>
        <div v-if="!store.currentDoc.annotations.length" class="text-gray-600 text-xs">
          在图片上拖拽框选区域添加标注
        </div>
      </div>
    </div>

    <Transition name="snackbar">
      <div v-if="showUndoToast"
        class="fixed bottom-6 left-1/2 -translate-x-1/2 bg-gray-800 border border-gray-700 rounded-lg px-4 py-3 flex items-center gap-3 shadow-lg z-50">
        <span class="text-sm text-gray-300">已删除标注「{{ store.recentlyDeleted?.label }}」</span>
        <button @click="handleUndo" class="text-amber-400 text-sm font-medium hover:underline">撤销</button>
      </div>
    </Transition>
  </div>
</template>

<script setup lang="ts">
import { ref, watch } from 'vue'
import { useOcrStore } from './store/ocr'
import ImageCanvas from './components/ImageCanvas.vue'

const store = useOcrStore()
const pendingDeleteId = ref('')
const showUndoToast = ref(false)
let undoTimer: ReturnType<typeof setTimeout> | null = null

function confirmDelete(id: string) {
  store.removeAnnotation(id)
  pendingDeleteId.value = ''
  showUndoToast.value = true
  if (undoTimer) clearTimeout(undoTimer)
  undoTimer = setTimeout(() => {
    showUndoToast.value = false
    store.clearRecentlyDeleted()
  }, 5000)
}

function handleUndo() {
  store.undoRemoveAnnotation()
  showUndoToast.value = false
  if (undoTimer) clearTimeout(undoTimer)
}

watch(() => store.currentDoc, () => {
  pendingDeleteId.value = ''
  showUndoToast.value = false
  if (undoTimer) clearTimeout(undoTimer)
})

function onUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) store.uploadAndOCR(file)
}

function doExport() {
  const tei = store.exportTEI()
  if (!tei) return
  const blob = new Blob([tei], { type: 'application/xml' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `${store.currentDoc?.name || 'export'}.xml`
  a.click()
  URL.revokeObjectURL(url)
}
</script>

<style scoped>
.snackbar-enter-active,
.snackbar-leave-active {
  transition: opacity 0.3s ease, transform 0.3s ease;
}
.snackbar-enter-from,
.snackbar-leave-to {
  opacity: 0;
  transform: translate(-50%, 12px);
}
.snackbar-enter-to,
.snackbar-leave-from {
  opacity: 1;
  transform: translate(-50%, 0);
}
</style>
