<template>
  <div class="add-comment flex flex-wrap items-center gap-2 mt-4">
    <input
      v-model="input"
      placeholder="寫下你的留言..."
      rows="1"
      class="comment-input flex-1 min-w-0 p-2 border border-gray-300 rounded-xl focus:outline-none focus:border-blue-500"
      :disabled="isSubmitting"
      @keyup.enter="emitSubmit"
    />
    <button
      @click="handleSubmit"
      :disabled="!input.trim() || isSubmitting"
      class="submit-btn whitespace-nowrap px-3 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 transition disabled:bg-gray-300"
    >
      {{ isSubmitting ? "送出中..." : "發表留言" }}
    </button>
  </div>
</template>

<script setup>
import { ref, defineEmits } from "vue";

const emit = defineEmits(["submit"]);
const input = ref("");

function emitSubmit() {
  const trimmed = input.value.trim();
  if (trimmed) {
    emit("submit", trimmed);
    input.value = "";
  }
}

const props = defineProps({
  isSubmitting: Boolean,
});

const handleSubmit = () => {
  if (!input.value.trim()) return;
  emit("submit", input.value.trim());
  input.value = ""; // 清空輸入框
};
</script>

<!-- <style scoped>
/* 你可以複製原本的 .add-comment、.comment-input、.submit-btn 樣式 */
.add-comment {
  margin-top: 16px;
}
.comment-input {
  width: 80%;
  height: 80%;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 14px;
  resize: vertical;
  font-family: inherit;
}
.comment-input:focus {
  outline: none;
  border-color: #007bff;
}
.submit-btn {
  padding: 4px 8px;
  background-color: #007bff;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.submit-btn:hover:not(:disabled) {
  background-color: #0056b3;
}
/* .submit-btn:disabled {
  background-color: #ccc;
} */
</style> -->
