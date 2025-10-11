<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

// BOSS 資料
const bosses = ref([]);

const currentTime = ref(Date.now());
let intervalId = null;

// 載入儲存的資料
onMounted(() => {
  const saved = localStorage.getItem("bossTimers");
  if (saved) {
    const data = JSON.parse(saved);
    bosses.value = data.map((boss) => ({
      ...boss,
      lastKillTime: boss.lastKillTime ? new Date(boss.lastKillTime) : null,
    }));
  }

  // 每秒更新時間
  intervalId = setInterval(() => {
    currentTime.value = Date.now();
  }, 1000);
});

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
});

// 儲存資料
const saveData = () => {
  localStorage.setItem("bossTimers", JSON.stringify(bosses.value));
};

// 記錄擊殺時間
const recordKill = (boss) => {
  boss.lastKillTime = new Date();
  saveData();
};

// 重置計時器
const resetTimer = (boss) => {
  boss.lastKillTime = null;
  saveData();
};

// 計算剩餘時間
const getRemainingTime = (boss) => {
  if (!boss.lastKillTime) return null;

  const killTime = new Date(boss.lastKillTime).getTime();
  const respawnTime = killTime + boss.spawnTime * 1000;
  const remaining = respawnTime - currentTime.value;

  if (remaining <= 0) return 0;
  return Math.floor(remaining / 1000);
};

// 格式化時間顯示
const formatTime = (seconds) => {
  if (seconds === null) return "--:--:--";

  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;

  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(
    2,
    "0"
  )}:${String(secs).padStart(2, "0")}`;
};

// 格式化重生時間
const formatRespawnTime = (boss) => {
  if (!boss.lastKillTime) return "尚未擊殺";

  const killTime = new Date(boss.lastKillTime).getTime();
  const respawnTime = new Date(killTime + boss.spawnTime * 1000);

  return respawnTime.toLocaleString("zh-TW", {
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
    hour12: false,
  });
};

// 判斷是否已重生
const isRespawned = (boss) => {
  const remaining = getRemainingTime(boss);
  return remaining !== null && remaining === 0;
};

// 新增 BOSS
const addBoss = () => {
  const newId = Math.max(...bosses.value.map((b) => b.id), 0) + 1;
  bosses.value.push({
    id: newId,
    name: `BOSS ${newId}`,
    spawnTime: 30 * 60,
    lastKillTime: null,
    note: "",
  });
  saveData();
};

// 刪除 BOSS
const removeBoss = (bossId) => {
  bosses.value = bosses.value.filter((b) => b.id !== bossId);
  saveData();
};

// 編輯模式
const editingBoss = ref(null);
const editForm = ref({
  name: "",
  spawnTime: 30,
  note: "",
});

const startEdit = (boss) => {
  editingBoss.value = boss.id;
  editForm.value = {
    name: boss.name,
    spawnTime: boss.spawnTime / 60,
    note: boss.note,
  };
};

const saveEdit = (boss) => {
  boss.name = editForm.value.name;
  boss.spawnTime = editForm.value.spawnTime * 60;
  boss.note = editForm.value.note;
  editingBoss.value = null;
  saveData();
};

const cancelEdit = () => {
  editingBoss.value = null;
};
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <!-- 頁尾 -->
    <div class="text-center text-white/50 text-sm">
      <p>資料會自動儲存在瀏覽器本地</p>
    </div>
  </div>
</template>

<style scoped>
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}
</style>
