<template>
  <div class="homepage-bg relative">
    <div class="flex flex-col lg:flex-row h-screen">
      <!-- 左側：可放地圖或其他內容 -->
      <div
        class="h-[65%] lg:h-full lg:w-4/6 relative overflow-hidden mb-2 shadow-2xl"
      >
        <div class="w-full h-full relative overflow-hidden">
          <GoogleMapView
            ref="mapRef"
            :trip="selectedTrip"
            :selected-place="selectedPlace"
            :current-day-index="currentDayIndex"
            :daily-plan-ref="dailyPlanRef"
            :schedule-detail-ref="scheduleDetailRef"
            :default-image="defaultImage"
            :role="role"
            @place-added="handlePlaceAdded"
          />
        </div>
      </div>

      <!-- 右側：行程列表區-->
      <div class="flex-1 lg:w-2/6 h-[45%] lg:h-full overflow-y-auto">
        <div v-if="!editingTripId" class="fixed bottom-5 right-6 z-50">
          <button
            @click="handleOpenForm"
            class="animated-gradient-modern text-white text-xl px-10 py-3 rounded-full shadow-md shadow-black/40 cursor-pointer bg-gradient-trip hover:bg-gradient-trip-hover"
          >
            建立行程
          </button>
        </div>

        <!-- 行程卡片列表 -->
        <div>
          <div v-if="!editingTripId" class="rounded-2xl p-7 m-5">
            <div
              v-if="tripStore.trips.length > 0"
              class="mt-2 space-y-4 rounded-xl"
            >
              <div
                v-for="item in tripStore.trips"
                :key="item.id"
                @click="editingTripId = item.id"
                class="navbar-style rounded-2xl shadow-md shadow-black/40 cursor-pointer hover:ring-2 hover:ring-gray-400 transition z-0"
              >
                <div class="relative">
                  <!-- 根據角色顯示不同按鈕 -->
                  <button
                    v-if="tripRoles[item.id] === 'owner'"
                    @click.stop="deleteSchedule(item.id)"
                    title="刪除行程"
                    class="absolute top-2 right-3 text-gray-600 bg-white px-2 rounded-2xl hover:text-red-500 text-md"
                  >
                    刪除行程
                  </button>
                  <button
                    v-else
                    @click.stop="leaveSchedule(item.id)"
                    title="退出行程"
                    class="absolute top-2 right-3 text-gray-600 bg-white px-2 rounded-2xl hover:text-red-500 text-md"
                  >
                    退出行程
                  </button>
                  <button
                    @click.stop="openShareModal(item.id)"
                    title="共享行程"
                    class="absolute top-2 right-25 text-gray-600 bg-white px-2 rounded-2xl hover:text-blue-500 text-md"
                  >
                    共享行程
                  </button>
                </div>
                <img
                  :src="
                    item.coverURL || 'https://placehold.co/600x300?text=封面圖'
                  "
                  class="w-full h-60 object-cover rounded-tl-xl rounded-tr-xl mb-3"
                  alt="行程封面照"
                />
                <div class="px-5">
                  <h2 class="text-xl text-white font-bold mb-1">
                    {{ item.title }}
                  </h2>
                  <p class="text-white text-m">
                    {{ item.startDate }} - {{ item.endDate }}
                  </p>
                  <p class="text-white text-m mt-2">{{ item.description }}</p>
                </div>
              </div>
            </div>
            <div v-else class="text-gray-400 text-center">尚未建立任何行程</div>
          </div>

          <!-- 編輯行程 -->
          <ScheduleDetail
            v-else
            :trip-id="editingTripId"
            :selected-date="selectedTrip?.days?.[currentDayIndex]?.date"
            ref="scheduleDetailRef"
            @role-changed="handleRoleChanged"
            @back="handleCloseDetail"
            @day-changed="currentDayIndex = $event"
            class="w-[50%] lg:w-full"
          />
        </div>
      </div>

      <!-- 彈出建立行程表單 -->
      <div
        v-if="showForm"
        class="fixed inset-0 backdrop-blur-sm flex items-center justify-center z-50 overflow-auto p-4"
      >
        <div>
          <TravelSchedule @close="handleCloseForm" />
        </div>
      </div>

      <!-- 付款提醒Modal -->
      <PaymentModal
        v-if="showPayModal"
        :result="payResult"
        @close="showPayModal = false"
      />
    </div>
    <ShareTripModal
      :is-open="showShareModal"
      :trip-id="shareTripId"
      :role="role"
      @close="closeShareModal"
    />
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from "vue";
import { useRouter, useRoute } from "vue-router";
import TravelSchedule from "@/components/TravelSchedule.vue";
import axios from "axios";
import GoogleMapView from "@/views/GoogleMapView.vue";
import ScheduleDetail from "@/views/scheduleDetail.vue";
import { useTripStore } from "@/stores/tripStore";
import ShareTripModal from "@/components/ShareTripModal.vue";
import PaymentModal from "@/components/PaymentModal.vue";

const router = useRouter();
const route = useRoute();
const editingTripId = ref(null);
const tripStore = useTripStore();
const showForm = ref(false);
const isPremium = ref(false);
const showPayModal = ref(false);
const selectedPlace = ref(null);
const defaultImage = "https://picsum.photos/1000?image";
const currentDayIndex = ref(0);
const itineraryRef = ref(null);
const dailyPlanRef = ref(null);
const mapRef = ref(null);
const scheduleDetailRef = ref(null);
const showShareModal = ref(false);
const shareTripId = ref(null);
const payResult = ref(null);
const role = ref("viewer"); // 新增
const tripRoles = ref({}); // 新增：儲存每個行程的角色

function handleRoleChanged(newRole) {
  role.value = newRole;
}

const selectedTrip = computed(() => {
  return tripStore.trips.find((t) => t.id === editingTripId.value);
});

//取得會員是否為付費會員
const fetchIsPremium = async () => {
  const token = localStorage.getItem("token");

  try {
    const res = await axios.get(`${import.meta.env.VITE_API_URL}/api/profile`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    isPremium.value = res.data.isPremium;
  } catch (err) {
    // eslint-disable-next-line no-empty
  }
};

// 新增：獲取特定行程的角色
const fetchTripRole = async (tripId) => {
  const token = localStorage.getItem("token");
  try {
    const res = await axios.get(
      `${import.meta.env.VITE_API_URL}/api/tripShares/getPermission/${tripId}`,
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    tripRoles.value[tripId] = res.data.role;
  } catch {
    tripRoles.value[tripId] = "viewer";
  }
};

// 新增：獲取所有行程的角色
const fetchAllTripRoles = async () => {
  for (const trip of tripStore.trips) {
    await fetchTripRole(trip.id);
  }
};

//首次載入取得行程
onMounted(async () => {
  await tripStore.fetchTrips();
  await fetchAllTripRoles(); // 新增：獲取所有行程的角色
  fetchIsPremium();

  if (
    route.query.linepayResult === "success" ||
    route.query.linepayResult === "fail"
  ) {
    payResult.value = route.query.linepayResult;
    showPayModal.value = true;
  }
  if (route.query.openTripId) {
    editingTripId.value = route.query.openTripId;
  }
});

watch(
  () => route.query.openTripId,
  (newId) => {
    if (newId) {
      editingTripId.value = newId;
    }
  },
);

//建立行程時檢查是否登入
const handleOpenForm = () => {
  const token = localStorage.getItem("token");
  if (!token) {
    alert("請先登入會員");
    return;
  }

  const count = tripStore.trips.length;

  if (isPremium.value || count < 1) {
    showForm.value = true;
  } else {
    showPayModal.value = true;
  }
};

const goToPay = () => {
  showPayModal.value = false;
  router.push("/payment");
};

//事件處理函式
function handlePlaceAdded() {
  if (scheduleDetailRef.value?.refreshDailyPlan) {
    scheduleDetailRef.value.refreshDailyPlan();
  }
}

function refreshDailyPlan() {
  // 呼叫 DailyPlan 的 refresh 方法
  dailyPlanRef.value?.refresh?.();
}

//表單關閉後刷新行程列表
const handleCloseForm = async () => {
  showForm.value = false;
  await tripStore.fetchTrips();
  await fetchAllTripRoles(); // 新增：重新獲取所有行程的角色
  fetchIsPremium();
};

//返回總覽同步更新
const handleCloseDetail = async () => {
  editingTripId.value = null;
  await tripStore.fetchTrips();
  await fetchAllTripRoles(); // 新增：重新獲取所有行程的角色
};

//刪除行程
const deleteSchedule = async (id) => {
  const confirmed = confirm("確定刪除這個行程嗎?");
  if (!confirmed) return;

  const token = localStorage.getItem("token");

  try {
    await axios.delete(
      `${import.meta.env.VITE_API_URL}/api/travelSchedule/${id}`,
      {
        headers: { Authorization: `Bearer ${token}` },
      },
    );

    tripStore.trips = tripStore.trips.filter((s) => s.id !== id);
    alert("刪除成功");
  } catch (err) {
    alert("刪除失敗，請稍後再試");
  }
};

const leaveSchedule = async (id) => {
  const confirmed = confirm("確定要退出這個行程嗎？");
  if (!confirmed) return;
  const token = localStorage.getItem("token");
  try {
    await axios.delete(
      `${import.meta.env.VITE_API_URL}/api/tripShares/leave/${id}`,
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    tripStore.trips = tripStore.trips.filter((s) => s.id !== id);
    alert("已退出行程");
  } catch (err) {
    alert("退出失敗，請稍後再試");
  }
};

function handlePlaceSelect(place) {
  selectedPlace.value = place;
}

function callItinerary() {
  mapRef.value?.callItinerary();
}

function openShareModal(id) {
  shareTripId.value = id;
  showShareModal.value = true;
}

function closeShareModal() {
  showShareModal.value = false;
  shareTripId.value = null;
}

// 暴露方法給父元件使用
defineExpose({
  refreshDailyPlan,
  dailyPlanRef,
});
</script>

<style scoped></style>
