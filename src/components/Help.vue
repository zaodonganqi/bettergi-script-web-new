<template>
  <div class="help-modal-content">
    <div class="help-title">{{ $t('help.modalTitle') }}</div>
    <div class="author-title">{{ $t('help.author') }}</div>
    <div v-for="(item, idx) in tm('help.qa')" :key="idx" class="qa-item">
      <div class="qa-question"><span class="qa-index">{{ idx + 1 }}.</span> {{ item.q }}</div>
      <div class="qa-answer">{{ item.a }}</div>
      <div v-if="idx < tm('help.qa').length - 1" class="qa-divider"></div>
    </div>
  </div>
</template>

<script setup>
import { useI18n } from 'vue-i18n';
const { t: $t, tm } = useI18n();
</script>

<style scoped>
.help-modal-content {
  padding: 10px 16px;
  max-height: 60vh;
  overflow-y: auto;
}
.help-title {
  font-size: 30px;
  font-weight: bold;
  color: #000000;
  margin-bottom: 24px;
  letter-spacing: 2px;
}
.author-title {
  font-size: 15px;
  color: #000000;
  margin-bottom: 20px;
}
.qa-item {
  margin-bottom: 28px;
}
.qa-question {
  font-size: 20px;
  font-weight: 600;
  color: #222;
  margin-bottom: 6px;
  display: flex;
  align-items: center;
}
.qa-index {
  color: #000000;
  font-size: 18px;
  font-weight: bold;
  margin-right: 6px;
}
.qa-answer {
  font-size: 15px;
  color: #666;
  line-height: 1.7;
  padding-left: 8px;
}
.qa-divider {
  border-bottom: 1px solid #ececec;
  margin: 18px 0 0 0;
}
</style> 