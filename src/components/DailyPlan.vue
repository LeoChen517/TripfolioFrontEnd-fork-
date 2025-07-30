<template>
  <div v-if="selectedTrip && currentDay" class="w-full max-w-xl mx-auto">
    <h2 class="text-xl font-bold mb-4 ml-13">{{ currentDay.date }}</h2>

    <Itinerary
      ref="itineraryRef"
      :trip-id="selectedTrip.id"
      :selected-date="currentDay.date"
      :role="role"
      class="hidden"
      @refresh="refresh"
    />

    <draggable
      :list="itinerarySpots"
      item-key="id"
      ghost-class="solo-card-style"
      animation="200"
      @end="canEdit ? updateOrder() : alert('您沒有調整順序的權限')"
      :disabled="!canEdit"
    >
      <template #item="{ element: p, index }">
        <div class="">
          <li
            class="navbar-style mb-4 list-none flex justify-between rounded-2xl w-[80%] mx-auto relative items-stretch"
          >
            <div class="w-1/2 p-3">
              <p
                class="number bg-[#6a7282] w-6 text-center rounded-2xl text-amber-50"
              >
                {{ index + 1 }}
              </p>
              <h3 class="block text-white text-l mb-1.5">{{ p.name }}</h3>
              <div class="flex flex-col items-start text-white text-xs">
                <p
                  v-if="!p.editingTime"
                  class="cursor-pointer pb-2"
                  @click="canEdit ? startEditing(p) : alert('您沒有編輯權限')"
                >
                  {{ formatTime(p.arrivalHour, p.arrivalMinute) }}抵達
                </p>
                <div v-else class="flex flex-col gap-1">
                  <div class="flex gap-1 items-center">
                    <select
                      v-model="p.arrivalHourTemp"
                      class="appearance-none outline-0"
                      :disabled="!canEdit"
                    >
                      <option v-for="h in 24" :key="h" :value="h - 1">
                        {{ (h - 1).toString().padStart(2, "0") }}
                      </option>
                    </select>
                    :
                    <select
                      v-model="p.arrivalMinuteTemp"
                      class="appearance-none outline-0"
                      :disabled="!canEdit"
                    >
                      <option v-for="m in [0, 15, 30, 45]" :key="m" :value="m">
                        {{ m.toString().padStart(2, "0") }}
                      </option>
                    </select>
                    抵達
                  </div>

                  <div class="flex gap-2 mt-1">
                    <button
                      @click="
                        canEdit ? confirmTime(p) : alert('您沒有編輯權限')
                      "
                      class="text-green-300"
                      :disabled="!canEdit"
                    >
                      更改
                    </button>
                    <button @click="cancelEditing(p)" class="text-red-300">
                      取消
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <img
              :src="p.photo"
              class="w-1/2 rounded-tr-lg rounded-br-lg object-cover"
            />

            <div class="relative">
              <button
                v-if="canEdit"
                @click.stop="toggleMenu(index)"
                class="button-list absolute right-0"
              >
                <font-awesome-icon
                  icon="ellipsis-h"
                  class="p-1 text-white bg-[#878787] rounded-full cursor-pointer absolute right-2 top-2"
                />
              </button>
              <ul
                v-if="openMenuIndex === index && canEdit"
                class="absolute right-0 mt-12 bg-white shadow rounded"
              >
                <li>
                  <button
                    @click="removePlace(p)"
                    class="px-3 py-1 bg-white text-black rounded hover:bg-gray-100 whitespace-nowrap"
                  >
                    🗑️ 移除
                  </button>
                </li>
              </ul>
            </div>
          </li>
          <!-- 每兩個景點之間顯示交通 -->
          <TrafficBetween
            v-if="index < itinerarySpots.length - 1"
            :itinerary-id="selectedTrip.id"
            :from-place-id="p.id"
            :to-place-id="itinerarySpots[index + 1].id"
            :origin="{ lat: p.lat, lng: p.lng }"
            :destination="{
              lat: itinerarySpots[index + 1].lat,
              lng: itinerarySpots[index + 1].lng,
            }"
            :traffic-data="
              trafficMap[p.id + '-' + itinerarySpots[index + 1].id] || null
            "
            :role="role"
            @traffic-updated="refresh"
            class="mx-auto"
          />
        </div>
      </template>
    </draggable>

    <div v-if="itinerarySpots.length === 0" class="text-gray-400 mb-2">
      尚未加入任何景點
    </div>
  </div>

  <div v-else class="text-center text-gray-500 py-10">
    <p>請從右側邊欄選擇一個旅程和日期來查看每日計畫。</p>
  </div>
</template>

<script setup>
import { ref, computed, toRefs, onMounted, watch } from "vue";
import TrafficBetween from "./TrafficBetween.vue";
import draggable from "vuedraggable";
import Itinerary from "./Itinerary.vue";
import axios from "axios";

const API_URL = import.meta.env.VITE_API_URL;

const itineraryRef = ref(null);

const props = defineProps({
  selectedTrip: Object,
  dayIndex: Number,
  role: {
    type: String, // 🔒 權限控制：接收 role
  },
});

const { selectedTrip, dayIndex } = toRefs(props);

// 🔒 權限控制：計算是否可編輯
const canEdit = computed(
  () => props.role === "editor" || props.role === "owner",
);

const openMenuIndex = ref(null);

const trafficMap = ref({});

// 暴露方法給父元件使用
defineExpose({
  refresh,
});

//取得目前日期
const currentDay = computed(() => {
  return selectedTrip.value?.days?.[dayIndex.value] || null;
});

const itinerarySpots = ref([]);

//更新景點資料
async function refresh() {
  if (!selectedTrip.value?.id || !currentDay.value?.date) {
    return;
  }

  try {
    // 直接向 API 請求最新資料
    const res = await axios.get(`${API_URL}/api/itinerary/places`, {
      params: {
        itineraryId: selectedTrip.value.id,
        date: currentDay.value.date,
      },
    });
    console.log("更新景點資料：", res.data.places);

    // 直接更新 DailyPlan 的資料
    itinerarySpots.value = res.data.places
      .filter((p) => p.date === currentDay.value.date)
      .sort((a, b) => a.arrivalHour - b.arrivalHour);

    trafficMap.value = itineraryRef.value?.trafficMap || {}; //交通
  } catch (error) {
    console.error("載入行程失敗", error);
  }
}

// 元件掛載時載入資料
onMounted(() => {
  refresh();
});

// 監聽日期變化時重新載入
watch(
  () => currentDay.value?.date,
  () => {
    refresh();
  },
);

// 監聽變化時重新載入

watch(
  () => selectedTrip.value,
  () => {
    refresh();
  },
  { deep: true },
);

function toggleMenu(index) {
  openMenuIndex.value = openMenuIndex.value === index ? null : index;
}

function formatTime(hour, minute) {
  return `${String(hour ?? 0).padStart(2, "0")}:${String(minute ?? 0).padStart(2, "0")}`;
}

//呼叫子層
function startEditing(p) {
  if (!canEdit.value) {
    alert("您沒有編輯權限");
    return;
  }
  itineraryRef.value?.startEditing(p);
}

function cancelEditing(p) {
  if (!canEdit.value) {
    alert("您沒有編輯權限");
    return;
  }
  itineraryRef.value?.cancelEditing(p);
}

function confirmTime(p) {
  if (!canEdit.value) {
    alert("您沒有編輯權限");
    return;
  }
  itineraryRef.value?.confirmTime(p);
}

function removePlace(p) {
  if (!canEdit.value) {
    alert("您沒有刪除權限");
    return;
  }
  itineraryRef.value?.removePlace(p);
}

//更新排序
function updateOrder() {
  if (!canEdit.value) {
    alert("您沒有調整順序的權限");
    return;
  }
  const newOrder = itinerarySpots.value.map((p, i) => ({
    id: p.id,
    placeOrder: i + 1,
  }));
  axios
    .put(`${API_URL}/api/itinerary/places/reorder`, { places: newOrder })
    .then(() => {
      console.log("排序更新成功");
    })
    .catch(() => {
      alert("排序更新失敗");
    });
}
</script>
