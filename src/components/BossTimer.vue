<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

// 導入 BOSS 配置
import {
  bossConfig,
  getLootsByBossId,
  getAllLoots,
} from "@/config/bossConfig.js";

// 導入 Heroicons
import {
  TrashIcon,
  PlusCircleIcon,
  PlusIcon,
  XMarkIcon,
  MagnifyingGlassIcon,
  ClockIcon,
  MapPinIcon,
  SparklesIcon,
  ShareIcon,
  DocumentTextIcon,
  CheckBadgeIcon,
  FireIcon,
  BoltIcon,
} from "@heroicons/vue/24/outline";

// ===== 清除記錄(本地端所有記錄) =====

const clearRecordsLocal = () => {
  if (confirm("確定要清除所有記錄嗎？")) {
    localStorage.removeItem("killRecords");
    killRecords.value = [];
  }
};

// ===== 擊殺記錄列表 =====
const killRecords = ref([]);

// 排序後的記錄列表（按最早重生時間排序）
const sortedKillRecords = computed(() => {
  return [...killRecords.value].sort((a, b) => {
    const timeA = new Date(a.respawnTimeMin).getTime();
    const timeB = new Date(b.respawnTimeMin).getTime();
    return timeA - timeB; // 由早到晚排序
  });
});

// 按狀態分組的記錄
const groupedRecords = computed(() => {
  const respawned = []; // 已重生
  const respawning = []; // 可能已重生
  const waiting = []; // 即將重生

  sortedKillRecords.value.forEach((record) => {
    const status = getRecordCountdown(record).status;
    if (status === "respawned") {
      respawned.push(record);
    } else if (status === "respawning") {
      respawning.push(record);
    } else if (status === "waiting") {
      waiting.push(record);
    }
  });

  return { respawned, respawning, waiting };
});

// 新增一筆紀錄的彈窗
const isAddRecordModalOpen = ref(false);
const addRecordForm = ref({
  boss: "",
  channel: "",
  killTime: "",
  killLocation: "",
  loot: "",
});

// 自定義下拉選單狀態
const isDropdownOpen = ref(false);
const dropdownSearch = ref("");

// 根據選擇的 BOSS 判斷是否需要顯示地點欄位
const selectedBossConfig = computed(() => {
  if (!addRecordForm.value.boss) return null;
  return bossConfig.find((b) => b.id === addRecordForm.value.boss);
});

// 過濾後的 BOSS 列表（根據搜尋）
const filteredBosses = computed(() => {
  if (!dropdownSearch.value) return bossConfig;
  const search = dropdownSearch.value.toLowerCase();
  return bossConfig.filter(
    (boss) =>
      boss.name.toLowerCase().includes(search) ||
      boss.description.toLowerCase().includes(search)
  );
});

// 選擇 BOSS
const selectBoss = (bossId) => {
  addRecordForm.value.boss = bossId;
  isDropdownOpen.value = false;
  dropdownSearch.value = "";
};

// 取得 BOSS 圖片 URL
const getBossImage = (bossId) => {
  try {
    return new URL(`../assets/images/${bossId}.png`, import.meta.url).href;
  } catch (e) {
    console.error(`無法載入圖片: ${bossId}.png`, e);
    return ""; // 返回空字串，讓圖片載入失敗時不顯示
  }
};

const openAddRecordModal = () => {
  if (!addRecordForm.value.boss) {
    alert("請先選擇一個 BOSS");
    return;
  }

  isAddRecordModalOpen.value = true;
  // 获取当前时间并格式化为 HH:mm 格式（input type="time" 需要的格式）
  const now = new Date();
  const hours = String(now.getHours()).padStart(2, "0");
  const minutes = String(now.getMinutes()).padStart(2, "0");
  const timeString = `${hours}:${minutes}`;

  // 保留已选择的 boss，只更新其他字段
  addRecordForm.value = {
    boss: addRecordForm.value.boss, // 保留已选择的 boss
    channel: "",
    killTime: timeString,
    killLocation: "",
    loot: "",
  };
};

const closeAddRecordModal = () => {
  isAddRecordModalOpen.value = false;
};

const addRecord = () => {
  if (
    !addRecordForm.value.boss ||
    !addRecordForm.value.channel ||
    !addRecordForm.value.killTime
  ) {
    alert("請填寫必填欄位");
    return;
  }

  const bossInfo = bossConfig.find((b) => b.id === addRecordForm.value.boss);

  // 將時間字串轉換為完整的日期時間
  const [hours, minutes] = addRecordForm.value.killTime.split(":");
  const killDateTime = new Date();
  killDateTime.setHours(parseInt(hours), parseInt(minutes), 0, 0);

  // 計算重生時間區間（最早和最晚）
  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );

  // 檢查是否已存在相同 BOSS 和頻道的記錄
  const existingRecordIndex = killRecords.value.findIndex(
    (record) =>
      record.bossId === addRecordForm.value.boss &&
      record.channel === addRecordForm.value.channel
  );

  const recordData = {
    bossId: addRecordForm.value.boss,
    bossName: bossInfo.name,
    channel: addRecordForm.value.channel,
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: addRecordForm.value.killLocation,
    loot: addRecordForm.value.loot,
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
  };

  if (existingRecordIndex !== -1) {
    // 如果存在，更新該記錄（保留原有的 id）
    killRecords.value[existingRecordIndex] = {
      ...recordData,
      id: killRecords.value[existingRecordIndex].id,
    };
  } else {
    // 如果不存在，新增記錄
    const newRecord = {
      ...recordData,
      id: Date.now(),
    };
    killRecords.value.unshift(newRecord); // 新記錄放在最前面
  }

  saveRecords();

  isAddRecordModalOpen.value = false;
  addRecordForm.value = {
    boss: addRecordForm.value.boss,
    channel: "",
    killTime: "",
    killLocation: "",
    loot: "",
  };
};

// 成功擊殺 新增紀錄
const reAddRecord = (record) => {
  const bossInfo = bossConfig.find((b) => b.id === record.bossId);

  if (!bossInfo) {
    alert("找不到 BOSS 資料");
    return;
  }

  // 獲取當前時間
  const killDateTime = new Date();

  // 計算重生時間區間（最早和最晚）
  const respawnDateTimeMin = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMin * 60 * 1000
  );
  const respawnDateTimeMax = new Date(
    killDateTime.getTime() + bossInfo.respawnMinutesMax * 60 * 1000
  );

  // 檢查是否已存在相同 BOSS 和頻道的記錄
  const existingRecordIndex = killRecords.value.findIndex(
    (r) => r.bossId === record.bossId && r.channel === record.channel
  );

  const recordData = {
    bossId: record.bossId,
    bossName: record.bossName,
    channel: record.channel,
    killTime: killDateTime.toISOString(),
    respawnTimeMin: respawnDateTimeMin.toISOString(),
    respawnTimeMax: respawnDateTimeMax.toISOString(),
    killLocation: record.killLocation || "",
    loot: record.loot || "",
    respawnMinutesMin: bossInfo.respawnMinutesMin,
    respawnMinutesMax: bossInfo.respawnMinutesMax,
  };

  if (existingRecordIndex !== -1) {
    // 如果存在，更新該記錄（保留原有的 id）
    killRecords.value[existingRecordIndex] = {
      ...recordData,
      id: killRecords.value[existingRecordIndex].id,
    };
  } else {
    // 如果不存在，新增記錄
    const newRecord = {
      ...recordData,
      id: Date.now(),
    };
    killRecords.value.unshift(newRecord); // 新記錄放在最前面
  }

  saveRecords();
};

// BOSS 資料（保留原有的，暫時不刪除以免影響其他功能）
const bosses = ref([]);

const currentTime = ref(Date.now());
let intervalId = null;

// 儲存記錄
const saveRecords = () => {
  localStorage.setItem("killRecords", JSON.stringify(killRecords.value));
};

// 載入記錄
const loadRecords = () => {
  const saved = localStorage.getItem("killRecords");
  if (saved) {
    killRecords.value = JSON.parse(saved);
  }
};

// 點擊外部關閉下拉選單
const closeDropdownOnClickOutside = (event) => {
  const dropdown = document.querySelector(".relative.w-96");
  if (dropdown && !dropdown.contains(event.target)) {
    isDropdownOpen.value = false;
  }
};

// 載入儲存的資料
onMounted(() => {
  loadRecords();

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

  // 監聽點擊事件來關閉下拉選單
  document.addEventListener("click", closeDropdownOnClickOutside);
});

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
  // 清除事件監聽器
  document.removeEventListener("click", closeDropdownOnClickOutside);
});

// 儲存資料（保留原有的）
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

// ===== 記錄列表相關功能 =====

// 刪除記錄
const deleteRecord = (recordId) => {
  //   if (confirm("確定要刪除這筆記錄嗎？")) {
  //   }
  killRecords.value = killRecords.value.filter((r) => r.id !== recordId);
  saveRecords();
};

// 格式化顯示時間
const formatDisplayTime = (isoString) => {
  const date = new Date(isoString);
  const now = new Date();

  // 判斷是否為當天
  const isToday =
    date.getFullYear() === now.getFullYear() &&
    date.getMonth() === now.getMonth() &&
    date.getDate() === now.getDate();

  // 如果是當天，只顯示時間
  if (isToday) {
    return date.toLocaleString("zh-TW", {
      hour: "2-digit",
      minute: "2-digit",
      hour12: false,
    });
  }

  // 如果不是當天，顯示日期和時間
  return date.toLocaleString("zh-TW", {
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  });
};

// 計算記錄的倒數計時
const getRecordCountdown = (record) => {
  const now = currentTime.value;
  const respawnMin = new Date(record.respawnTimeMin).getTime();
  const respawnMax = new Date(record.respawnTimeMax).getTime();

  // 如果還沒到最早重生時間
  if (now < respawnMin) {
    const remaining = Math.floor((respawnMin - now) / 1000);
    return {
      status: "waiting",
      text: "即將重生",
      timeText: formatCountdownTime(remaining),
      remaining,
      percentage: 0,
    };
  }

  // 如果在重生時間範圍內
  if (now >= respawnMin && now < respawnMax) {
    const remaining = Math.floor((respawnMax - now) / 1000);
    const totalDuration = respawnMax - respawnMin;
    const elapsed = now - respawnMin;
    const percentage = Math.min(
      100,
      Math.max(0, (elapsed / totalDuration) * 100)
    );

    return {
      status: "respawning",
      text: "可能已重生",
      timeText: formatCountdownTime(remaining),
      remaining,
      percentage: Math.round(percentage),
    };
  }

  // 如果已過最晚重生時間
  return {
    status: "respawned",
    text: "已重生",
    timeText: "00:00:00",
    remaining: 0,
    percentage: 100,
  };
};

// 格式化倒數時間
const formatCountdownTime = (seconds) => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;

  return `${String(hours).padStart(2, "0")}:${String(minutes).padStart(
    2,
    "0"
  )}:${String(secs).padStart(2, "0")}`;
};
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-6xl">
    <!-- 標題 & 功能按鈕 (清除、分享、使用說明) -->
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-2xl font-bold text-white">2魚 の Artale 打王小工具</h1>
      <div class="flex space-x-4">
        <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
          @click="clearRecordsLocal"
        >
          <TrashIcon class="w-4 h-4" />
          清除所有紀錄
        </button>
        <!-- <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
        >
          <ShareIcon class="w-4 h-4" />
          分享
        </button> -->
        <!-- <button
          class="bg-gray-800 hover:bg-gray-700 text-white px-4 py-2 rounded-md flex items-center gap-2 transition"
        >
          <DocumentTextIcon class="w-4 h-4" />
          使用說明
        </button> -->
      </div>
    </div>

    <!-- 
    * 新增一筆紀錄
      選擇一個BOSS 新增按鈕
      彈窗填寫資料：頻道、擊殺時間(預設當下 可調整)、擊殺地點(特定 BOSS 才要)、收穫的寶物(後續再加)
    -->
    <div
      class="flex flex-col items-end fixed right-0 bottom-0 gap-2 bg-gray-950 bg-opacity-50 backdrop-blur-sm p-3 w-full max-w-[480px] ml-auto z-20 rounded-tl-lg"
    >
      <div class="flex justify-end gap-2 w-full">
        <!-- 自定義下拉選單 -->
        <div class="relative w-full">
          <!-- 選單按鈕 -->
          <button
            @click="isDropdownOpen = !isDropdownOpen"
            class="w-full bg-gray-700 text-white px-4 py-2 rounded-md flex items-center justify-between hover:bg-gray-600 transition"
          >
            <span v-if="!selectedBossConfig" class="text-gray-400"
              >選擇一個BOSS</span
            >
            <span v-else class="flex items-center gap-2">
              <div class="w-8 h-8 flex-shrink-0">
                <img
                  :src="getBossImage(selectedBossConfig.id)"
                  :alt="selectedBossConfig.name"
                  class="w-full h-full object-contain"
                />
              </div>
              <span>{{ selectedBossConfig.name }}</span>
              <span class="text-xs text-gray-400"
                >({{ selectedBossConfig.description }})</span
              >
            </span>
            <svg
              class="w-5 h-5 transition-transform"
              :class="{ 'rotate-180': isDropdownOpen }"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M19 9l-7 7-7-7"
              ></path>
            </svg>
          </button>

          <!-- 下拉選單內容 -->
          <div
            v-show="isDropdownOpen"
            class="absolute z-10 w-full bottom-full mb-2 bg-gray-800 border border-gray-600 rounded-lg shadow-xl max-h-96 overflow-hidden"
          >
            <!-- 搜尋框 -->
            <div class="p-2 border-b border-gray-600">
              <div class="relative">
                <MagnifyingGlassIcon
                  class="w-4 h-4 absolute left-3 top-1/2 transform -translate-y-1/2 text-gray-400"
                />
                <input
                  v-model="dropdownSearch"
                  type="text"
                  placeholder="搜尋 BOSS 名稱..."
                  class="w-full bg-gray-700 text-white pl-9 pr-3 py-2 rounded-md text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
                  @click.stop
                />
              </div>
            </div>

            <!-- BOSS 列表 -->
            <div class="overflow-y-auto max-h-80">
              <button
                v-for="boss in filteredBosses"
                :key="boss.id"
                @click="selectBoss(boss.id)"
                class="w-full px-4 py-3 flex items-center gap-3 hover:bg-gray-700 transition text-left"
                :class="{ 'bg-gray-700': addRecordForm.boss === boss.id }"
              >
                <div class="w-12 h-12 flex-shrink-0">
                  <img
                    :src="getBossImage(boss.id)"
                    :alt="boss.name"
                    class="w-full h-full object-contain"
                  />
                </div>
                <div class="flex-1 min-w-0">
                  <div class="text-white font-medium">{{ boss.name }}</div>
                  <div class="text-xs text-gray-400">
                    {{ boss.description }}
                  </div>
                  <div
                    class="text-xs text-gray-500 truncate"
                    v-if="boss.location.length > 0"
                  >
                    📍 {{ boss.location[0] }}
                  </div>
                </div>
              </button>
              <div
                v-if="filteredBosses.length === 0"
                class="px-4 py-8 text-center text-gray-500"
              >
                找不到符合的 BOSS
              </div>
            </div>
          </div>
        </div>

        <button
          class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-md transition flex items-center gap-2"
          @click="openAddRecordModal"
        >
          <PlusIcon class="w-5 h-5" />
        </button>
      </div>

      <!-- 新增一筆紀錄的彈窗 -->
      <div
        class="w-full"
        v-if="isAddRecordModalOpen"
        @click.self="closeAddRecordModal"
      >
        <div class="bg-gray-800 p-6 rounded-lg w-full">
          <h3 class="text-xl font-bold text-white mb-4">
            新增擊殺記錄 - {{ selectedBossConfig?.name }}
          </h3>

          <div class="flex flex-col space-y-4">
            <div>
              <label for="channel" class="text-white block mb-2">
                頻道 <span class="text-red-600">*</span>
              </label>
              <input
                type="number"
                min="1"
                max="9999"
                id="channel"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 1111"
                v-model="addRecordForm.channel"
              />
            </div>

            <div>
              <label for="killTime" class="text-white block mb-2">
                擊殺時間 <span class="text-red-600">*</span>
              </label>
              <input
                type="time"
                id="killTime"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                v-model="addRecordForm.killTime"
              />
            </div>

            <!-- <div v-if="selectedBossConfig?.needLocation">
              <label for="killLocation" class="text-white block mb-2">
                擊殺地點
              </label>
              <input
                type="text"
                id="killLocation"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 北方洞穴"
                v-model="addRecordForm.killLocation"
              />
            </div> -->

            <!-- <div>
              <label for="loot" class="text-white block mb-2">收穫的寶物</label>
              <input
                type="text"
                id="loot"
                class="bg-gray-700 text-white px-4 py-2 rounded-md w-full"
                placeholder="例如: 紫裝、材料"
                v-model="addRecordForm.loot"
              />
            </div> -->

            <div class="flex gap-2 mt-4">
              <button
                class="flex-1 bg-gray-600 hover:bg-gray-700 text-white px-4 py-2 rounded-md transition flex items-center justify-center gap-2"
                @click="closeAddRecordModal"
              >
                取消
              </button>
              <button
                :disabled="
                  !(
                    addRecordForm.channel &&
                    addRecordForm.killTime &&
                    addRecordForm.boss
                  )
                "
                class="flex-1 bg-blue-600 hover:bg-blue-700 disabled:bg-gray-500 disabled:cursor-not-allowed disabled:opacity-50 text-white px-4 py-2 rounded-md transition flex items-center justify-center gap-2"
                @click="addRecord"
              >
                新增
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 記錄列表 -->
    <div class="border border-gray-700 rounded-xl p-4">
      <!-- 空狀態 -->
      <div
        v-if="killRecords.length === 0"
        class="bg-gray-800 rounded-lg p-8 text-center"
      >
        <p class="text-gray-400 text-lg">尚無記錄</p>
        <p class="text-gray-500 text-sm mt-2">選擇一個 BOSS 開始記錄吧！</p>
      </div>

      <!-- 分組顯示 -->
      <div v-else class="space-y-4">
        <!-- 已重生區塊 -->
        <div
          v-if="groupedRecords.respawned.length > 0"
          class="border border-gray-700 rounded-xl p-4"
        >
          <div class="flex items-center gap-2 mb-3">
            <div class="w-3 h-3 rounded-full bg-sky-500"></div>
            <h3 class="text-lg font-bold text-sky-400">
              已重生 ({{ groupedRecords.respawned.length }})
            </h3>
          </div>
          <div class="space-y-4">
            <div
              v-for="record in groupedRecords.respawned"
              :key="record.id"
              class="bg-gray-800 rounded-lg px-4 py-1 hover:bg-gray-750 transition border-l-4 border-sky-500"
            >
              <div class="flex items-center gap-4">
                <!-- BOSS 圖片 -->
                <div class="w-8 h-8 flex-shrink-0">
                  <img
                    :src="getBossImage(record.bossId)"
                    :alt="record.bossName"
                    class="pic-auto"
                  />
                </div>

                <!-- 主要資訊 -->
                <div class="flex-1 flex justify-between items-center min-w-0">
                  <!-- BOSS 名稱 & 頻道 -->
                  <div class="flex items-center gap-2">
                    <h3 class="text-md text-gray-200">
                      {{ record.bossName }}
                    </h3>
                    <span class="text-gray-400">|</span>
                    <span class="text-gray-400 text-sm font-medium">
                      頻道 {{ record.channel }}
                    </span>
                  </div>

                  <!-- 時間資訊 -->
                  <div class="text-sm">
                    <div class="flex items-center gap-1">
                      <ClockIcon class="w-3.5 h-3.5 text-blue-400" />
                      <span class="text-gray-200">
                        {{ formatDisplayTime(record.respawnTimeMin) }} ~
                        {{ formatDisplayTime(record.respawnTimeMax) }}
                      </span>
                    </div>
                  </div>
                </div>

                <!-- 倒數計時 & 操作按鈕 -->
                <div class="flex items-center gap-3 flex-shrink-0">
                  <!-- 倒數計時 -->
                  <div
                    class="text-center px-4 py-1 rounded-lg"
                    :class="{
                      'bg-yellow-900/50 border border-yellow-500':
                        getRecordCountdown(record).status === 'waiting',
                      'bg-green-900/50 border border-green-500 animate-pulse':
                        getRecordCountdown(record).status === 'respawning',
                      'bg-sky-900/50 border border-sky-500':
                        getRecordCountdown(record).status === 'respawned',
                    }"
                  >
                    <div
                      class="text-xs font-semibold"
                      :class="{
                        'text-yellow-400':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-400':
                          getRecordCountdown(record).status === 'respawning',
                        'text-sky-400':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ formatDisplayTime(record.respawnTimeMax) }}
                    </div>
                    <div
                      class="text-xs"
                      :class="{
                        'text-yellow-300':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-300':
                          getRecordCountdown(record).status === 'respawning',
                        'text-sky-300':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ getRecordCountdown(record).text }}
                    </div>
                  </div>

                  <!-- 成功擊殺 重新計時 -->
                  <button
                    @click="reAddRecord(record)"
                    class="border border-green-600 hover:bg-green-700 hover:text-white text-green-600 p-2 rounded text-sm transition"
                    title="重新計時"
                  >
                    <CheckBadgeIcon class="w-5 h-5" />
                  </button>

                  <!-- 刪除按鈕 -->
                  <button
                    @click="deleteRecord(record.id)"
                    class="border border-red-600 hover:bg-red-700 hover:text-white text-red-600 p-2 rounded text-sm transition"
                    title="刪除記錄"
                  >
                    <TrashIcon class="w-5 h-5" />
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 可能已重生區塊 -->
        <div
          v-if="groupedRecords.respawning.length > 0"
          class="border border-gray-700 rounded-xl p-4"
        >
          <div class="flex items-center gap-2 mb-3">
            <div class="w-3 h-3 rounded-full bg-green-500 animate-pulse"></div>
            <h3 class="text-lg font-bold text-green-400">
              可能已重生 ({{ groupedRecords.respawning.length }})
            </h3>
          </div>
          <div class="space-y-4">
            <div
              v-for="record in groupedRecords.respawning"
              :key="record.id"
              class="bg-gray-800 rounded-lg px-4 py-1 hover:bg-gray-750 transition border-l-4 border-green-500"
            >
              <div class="flex items-center gap-4">
                <!-- BOSS 圖片 -->
                <div class="w-8 h-8 flex-shrink-0">
                  <img
                    :src="getBossImage(record.bossId)"
                    :alt="record.bossName"
                    class="pic-auto"
                  />
                </div>

                <!-- 主要資訊 -->
                <div class="flex-1 flex flex-col min-w-0">
                  <div class="flex items-center gap-3 justify-between mb-1">
                    <!-- BOSS 名稱 & 頻道 -->
                    <div class="flex items-center gap-2">
                      <h3 class="text-md text-gray-200">
                        {{ record.bossName }}
                      </h3>
                      <span class="text-gray-400">|</span>
                      <span class="text-gray-400 text-sm font-medium">
                        頻道 {{ record.channel }}
                      </span>
                    </div>

                    <!-- 時間資訊 -->
                    <div class="text-sm">
                      <div class="flex items-center gap-1">
                        <ClockIcon class="w-3.5 h-3.5 text-blue-400" />
                        <span class="text-gray-200">
                          {{ formatDisplayTime(record.respawnTimeMin) }} ~
                          {{ formatDisplayTime(record.respawnTimeMax) }}
                        </span>
                      </div>
                    </div>
                  </div>

                  <!-- 重生進度條 -->
                  <div class="mb-2">
                    <div class="flex gap-1 w-full items-center">
                      <div
                        class="w-full bg-gray-700 rounded-full h-2.5 overflow-hidden"
                      >
                        <div
                          class="h-full bg-gradient-to-r from-yellow-500 via-green-500 to-green-400 transition-all duration-1000 ease-out shadow-lg"
                          :style="{
                            width: getRecordCountdown(record).percentage + '%',
                          }"
                        ></div>
                      </div>
                      <span class="text-xs font-semibold text-green-400">
                        {{ getRecordCountdown(record).percentage }}%
                      </span>
                    </div>
                  </div>
                </div>

                <!-- 倒數計時 & 操作按鈕 -->
                <div class="flex items-center gap-2 flex-shrink-0">
                  <!-- 倒數計時 -->
                  <div
                    class="text-center px-4 py-1 rounded-lg"
                    :class="{
                      'bg-yellow-900/50 border border-yellow-500':
                        getRecordCountdown(record).status === 'waiting',
                      'bg-green-900/50 border border-green-500 animate-pulse':
                        getRecordCountdown(record).status === 'respawning',
                      'bg-red-900/50 border border-red-500':
                        getRecordCountdown(record).status === 'respawned',
                    }"
                  >
                    <div
                      class="text-xs font-semibold"
                      :class="{
                        'text-yellow-400':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-400':
                          getRecordCountdown(record).status === 'respawning',
                        'text-red-400':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ getRecordCountdown(record).text }}
                    </div>
                    <div
                      class="text-sm font-mono font-bold"
                      :class="{
                        'text-yellow-300':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-300':
                          getRecordCountdown(record).status === 'respawning',
                        'text-red-300':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ getRecordCountdown(record).timeText }}
                    </div>
                  </div>

                  <!-- 成功擊殺 重新計時 -->
                  <button
                    @click="reAddRecord(record)"
                    class="border border-green-600 hover:bg-green-700 hover:text-white text-green-600 p-2 rounded text-sm transition"
                    title="重新計時"
                  >
                    <CheckBadgeIcon class="w-5 h-5" />
                  </button>

                  <!-- 刪除按鈕 -->
                  <button
                    @click="deleteRecord(record.id)"
                    class="border border-red-600 hover:bg-red-700 hover:text-white text-red-600 p-2 rounded text-sm transition"
                    title="刪除記錄"
                  >
                    <TrashIcon class="w-5 h-5" />
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 即將重生區塊 -->
        <div
          v-if="groupedRecords.waiting.length > 0"
          class="border border-gray-700 rounded-xl p-4"
        >
          <div class="flex items-center gap-2 mb-3">
            <div class="w-3 h-3 rounded-full bg-yellow-500"></div>
            <h3 class="text-lg font-bold text-yellow-400">
              即將重生 ({{ groupedRecords.waiting.length }})
            </h3>
          </div>
          <div class="space-y-4">
            <div
              v-for="record in groupedRecords.waiting"
              :key="record.id"
              class="bg-gray-800 rounded-lg px-4 py-1 hover:bg-gray-750 transition border-l-4 border-yellow-500"
            >
              <div class="flex items-center gap-4">
                <!-- BOSS 圖片 -->
                <div class="w-8 h-8 flex-shrink-0">
                  <img
                    :src="getBossImage(record.bossId)"
                    :alt="record.bossName"
                    class="pic-auto"
                  />
                </div>

                <!-- 主要資訊 -->
                <div class="flex-1 flex justify-between items-center min-w-0">
                  <!-- BOSS 名稱 & 頻道 -->
                  <div class="flex items-center gap-2">
                    <h3 class="text-md text-gray-200">
                      {{ record.bossName }}
                    </h3>
                    <span class="text-gray-400">|</span>
                    <span class="text-gray-400 text-sm font-medium">
                      頻道 {{ record.channel }}
                    </span>
                  </div>

                  <!-- 時間資訊 -->
                  <div class="text-sm">
                    <div class="flex items-center gap-1">
                      <ClockIcon class="w-3.5 h-3.5 text-blue-400" />
                      <span class="text-gray-200">
                        {{ formatDisplayTime(record.respawnTimeMin) }} ~
                        {{ formatDisplayTime(record.respawnTimeMax) }}
                      </span>
                    </div>
                  </div>
                </div>

                <!-- 倒數計時 & 操作按鈕 -->
                <div class="flex items-center gap-2 flex-shrink-0">
                  <!-- 倒數計時 -->
                  <div
                    class="text-center px-4 py-1 rounded-lg"
                    :class="{
                      'bg-yellow-900/50 border border-yellow-500':
                        getRecordCountdown(record).status === 'waiting',
                      'bg-green-900/50 border border-green-500 animate-pulse':
                        getRecordCountdown(record).status === 'respawning',
                      'bg-red-900/50 border border-red-500':
                        getRecordCountdown(record).status === 'respawned',
                    }"
                  >
                    <div
                      class="text-xs font-semibold"
                      :class="{
                        'text-yellow-400':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-400':
                          getRecordCountdown(record).status === 'respawning',
                        'text-red-400':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ getRecordCountdown(record).text }}
                    </div>
                    <div
                      class="text-sm font-mono font-bold"
                      :class="{
                        'text-yellow-300':
                          getRecordCountdown(record).status === 'waiting',
                        'text-green-300':
                          getRecordCountdown(record).status === 'respawning',
                        'text-red-300':
                          getRecordCountdown(record).status === 'respawned',
                      }"
                    >
                      {{ getRecordCountdown(record).timeText }}
                    </div>
                  </div>

                  <!-- 刪除按鈕 -->
                  <button
                    @click="deleteRecord(record.id)"
                    class="border border-red-600 hover:bg-red-700 hover:text-white text-red-600 p-2 rounded text-sm transition"
                    title="刪除記錄"
                  >
                    <TrashIcon class="w-5 h-5" />
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 頁尾 -->
    <div class="text-center text-white/50 text-sm min-h-[88px] py-4">
      <p>資料會自動儲存在瀏覽器本地</p>
    </div>
  </div>
</template>

<style scoped>
.pic-auto {
  width: 100%;
  height: 100%;
  vertical-align: middle;
  object-fit: cover;
}
@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.8;
  }
}

.animate-pulse {
  animation: pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}
</style>
