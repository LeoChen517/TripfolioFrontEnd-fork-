<template>
  <div
    :class="
      isFromTracker
        ? 'p-6 w-full max-w-3xl mx-auto'
        : 'px-0 py-6 w-full grid lg:grid-cols-1 max-w-7xl mx-auto gap-6'
    "
  >
    <!-- 左側：行程總覽＋每日行程 -->
    <div class="w-full max-w-lg mx-auto">
      <!-- 行程總覽卡片 -->
      <TripOverview
        v-if="trip"
        :trip="trip"
        :cover-timestamp="coverTimestamp"
        :role="role"
        :is-from-tracker="isFromTracker"
        @back-to-list="goBack"
        @update-cover="updateCover"
        @update-title="updateTitle"
        @update-notes="updateNotes"
        @update-dates="updateDates"
      />
      <!-- 日期切換按鈕 -->

      <div
        v-if="trip && trip.days?.length"
        class="mt-1 overflow-x-auto whitespace-nowrap scrollbar-hide"
      >
        <div class="inline-flex gap-2 px-2">
          <button
            v-for="(day, index) in trip.days"
            :key="day"
            @click="changeDay(index)"
            class="px-5 py-1 rounded-full text-sm border"
            :class="
              currentDayIndex === index
                ? 'bg-[#828282] text-white border-[#828282]'
                : 'bg-white text-gray-700 border-gray-300 '
            "
          >
            第{{ index + 1 }}天
          </button>
        </div>
      </div>
      <!-- 每日計畫 -->
      <DailyPlan
        v-if="trip && currentDayIndex !== null"
        :selected-trip="trip"
        :day-index="currentDayIndex"
        :itinerary-places="itineraryRef?.itineraryPlaces || []"
        :role="role"
        class="mt-6 w-full"
        ref="dailyPlanRef"
      />
    </div>

    <!-- 右側：景點清單 -->
    <div class="lg:col-span-1 w-[80%] mx-auto">
      <!-- <Itinerary
        v-if="tripLoaded && trip?.id && currentDayIndex !== null"
        :trip-id="trip.id"
        :selected-date="trip.days[currentDayIndex].date"
        :default-image="'/images/default.jpg'"
        @refresh="handleItineraryRefresh"
        ref="itineraryRef"
      /> -->
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from "vue";
import { useRoute, useRouter } from "vue-router";
import axios from "axios";
import TripOverview from "../components/TripOverview.vue";
import DailyPlan from "../components/DailyPlan.vue";
import Itinerary from "../components/Itinerary.vue";
import { generateDaysArray } from "../stores/tripStore";

const props = defineProps({
  tripId: [String, Number],
});

const emit = defineEmits(["back"]);
const router = useRouter();
const trip = ref(null);
const tripLoaded = ref(false);
const currentDayIndex = ref(0);
const coverTimestamp = ref(Date.now());
const itineraryRef = ref(null);
const dailyPlanRef = ref(null);
const role = ref("viewer");
const resolvedTripId = props.tripId ?? route.params.id;

const token = localStorage.getItem("token");

//會員動態頁面用
const route = useRoute();
const isFromTracker = computed(() => route.query.from === "tracker");

//返回行程列表
const goBack = () => {
  emit("back");
};

//取得行程資料
const fetchTrip = async () => {
  try {
    const res = await axios.get(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${props.tripId}`,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    const tripData = res.data;
    tripData.days = generateDaysArray(tripData);
    trip.value = tripData;
    tripLoaded.value = true;
  } catch (err) {
    alert("無法載入行程");
    router.push("/schedule");
  }
};

const fetchRole = async () => {
  try {
    const res = await axios.get(
      `${import.meta.env.VITE_API_URL}/api/tripShares/getPermission/${resolvedTripId}`,
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    role.value = res.data.role;
    emit("role-changed", role.value);
    console.log("✅ 取得角色:", role.value);
  } catch {
    role.value = "viewer";
    emit("role-changed", role.value);
  }
};

// ✅ 當 tripId 變更時重新取得資料
watch(
  () => props.tripId,
  async () => {
    await fetchTrip();
    await fetchRole();
    dailyPlanRef.value?.refresh();
  },
);

//更新送到後端
const updateCover = async (blob) => {
  const formData = new FormData();
  formData.append("cover", blob);

  try {
    const res = await axios.put(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${props.tripId}`,
      formData,
      { headers: { Authorization: `Bearer ${token}` } },
    );
    if (res.data.coverURL) {
      trip.value.coverURL = res.data.coverURL;
      coverTimestamp.value = Date.now();
    }
    alert("封面已成功更新！");
  } catch (err) {
    alert("封面更新失敗");
  }
};

const updateTitle = async (newTitle) => {
  try {
    await axios.put(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${props.tripId}`,
      { title: newTitle },
      { headers: { Authorization: `Bearer ${token}` } },
    );
    trip.value.title = newTitle;
  } catch (err) {
    alert("標題更新失敗");
  }
};

const updateDates = async ({ startDate, endDate }) => {
  try {
    await axios.patch(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${props.tripId}`,
      { startDate, endDate },
      { headers: { Authorization: `Bearer ${token}` } },
    );

    await fetchTrip();

    if (!trip.value.days.length) {
      currentDayIndex.value = null;
    } else if (
      currentDayIndex.value === null ||
      currentDayIndex.value >= trip.value.days.length
    ) {
      currentDayIndex.value = 0;
    }
  } catch (err) {
    alert("更新日期失敗");
  }
};

const updateNotes = async (newNotes) => {
  try {
    await axios.put(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${props.tripId}`,
      { description: newNotes },
      { headers: { Authorization: `Bearer ${token}` } },
    );
    trip.value.description = newNotes;
  } catch (err) {
    alert("筆記更新失敗");
  }
};

function handleItineraryRefresh() {
  fetchTrip();
  refreshDailyPlan();
}

watch(trip, (newTrip) => {
  if (Array.isArray(newTrip?.days) && newTrip.days.length > 0) {
    if (
      currentDayIndex.value === null ||
      currentDayIndex.value >= newTrip.days.length
    ) {
      currentDayIndex.value = 0;
    }
  }
});

function changeDay(index) {
  currentDayIndex.value = index;
  emit("day-changed", index); // 通知父層 Travel.vue
}

function refreshDailyPlan() {
  // 呼叫 DailyPlan 的 refresh 方法
  dailyPlanRef.value?.refresh?.();
}

// ✅ 初始化
onMounted(async () => {
  await fetchTrip();
  await fetchRole();
  dailyPlanRef.value?.refresh();
});

// 給父層或地圖強制刷新 DailyPlan 用
defineExpose({
  refreshDailyPlan,
  dailyPlanRef,
  fetchTrip,
});
</script>
