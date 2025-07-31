<template>
  <div
    v-if="isOpen"
    class="fixed inset-0 z-50 bg-black/50 flex items-center justify-center px-4"
  >
    <div
      class="navbar-style w-full max-w-[500px] rounded-2xl shadow-xl p-6 sm:p-6"
    >
      <div class="flex justify-between items-center mb-4">
        <h2 class="text-xl font-semibold text-white">分享行程</h2>
        <button
          @click="emit('close')"
          class="text-white hover:text-gray-400 text-lg cursor-pointer"
        >
          ✕
        </button>
      </div>

      <!-- 只有行程擁有者可以看到的產生分享區塊 -->
      <div v-if="isOwner">
        <!-- 權限選擇 -->
        <div class="mb-4">
          <label class="block mb-1 text-sm text-white font-medium"
            >權限設定</label
          >
          <select
            v-model="selectedPermission"
            class="w-full border border-gray-300 rounded-md px-3 py-2 text-black shadow-sm cursor-pointer"
          >
            <option value="viewer">僅可檢視</option>
            <option value="editor">可編輯</option>
          </select>
        </div>

        <button
          @click="generateShareLink"
          class="bg-white/30 hover:bg-black/20 text-white px-4 py-2 rounded-md w-full font-medium cursor-pointer"
          :disabled="isLoading"
        >
          產生分享連結
        </button>

        <!-- 分享結果區塊 -->
        <div v-if="shareUrl" class="mt-6">
          <label class="block mb-1 text-sm text-white font-medium"
            >分享連結</label
          >
          <div class="flex items-center gap-2">
            <input
              :value="shareUrl"
              readonly
              class="flex-1 border border-gray-300 rounded-md px-3 py-2 text-sm text-white shadow-sm"
            />
            <button
              @click="copyToClipboard"
              class="text-sm text-white hover:text-white/20 whitespace-nowrap cursor-pointer"
            >
              複製
            </button>
          </div>

          <div class="mt-4 flex justify-center">
            <qrcode-vue :value="shareUrl" :size="160" />
          </div>

          <div class="text-sm text-white mt-2 text-center">
            到期時間：{{ formattedExpire }}
          </div>
        </div>
      </div>

      <!-- 共享者清單區塊 -->
      <div v-if="sharedUsers.length" class="mt-6">
        <h3 class="text-white font-semibold text-base mb-2">
          目前共享的使用者
        </h3>
        <div class="space-y-2">
          <div
            v-for="user in sharedUsers"
            :key="user.id"
            class="flex items-center justify-between border border-gray-300 rounded-md px-3 py-2 shadow-sm"
          >
            <div>
              <div class="font-medium text-white">{{ user.name }}</div>
              <div class="text-sm text-gray-400">{{ user.email }}</div>
            </div>

            <div class="flex items-center gap-2">
              <span
                v-if="user.role === 'owner'"
                class="text-sm text-gray-200 font-semibold"
                >建立者</span
              >

              <template v-else>
                <select
                  v-if="isOwner"
                  v-model="user.role"
                  @change="updatePermission(user.id, user.role)"
                  class="text-sm border border-gray-300 rounded px-2 py-1 text-black"
                >
                  <option value="viewer">檢視者</option>
                  <option value="editor">編輯者</option>
                </select>

                <span v-else class="text-sm text-black">{{
                  formatRole(user.role)
                }}</span>

                <button
                  v-if="isOwner"
                  @click="removeUser(user.id)"
                  class="text-red-500 hover:text-red-700 text-base"
                >
                  🗑
                </button>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from "vue";
import axios from "axios";
import QrcodeVue from "qrcode.vue";

const props = defineProps({
  tripId: Number,
  isOpen: Boolean,
  role: {
    type: String, // 🔒 權限控制：接收 role
  },
});
const emit = defineEmits(["close"]);
const canEdit = computed(
  () => props.role === "editor" || props.role === "owner",
);

const selectedPermission = ref("viewer");
const shareUrl = ref("");
const expiresAt = ref("");
const isLoading = ref(false);
const sharedUsers = ref([]);
const isOwner = ref(false);

const generateShareLink = async () => {
  const token = localStorage.getItem("token");
  console.log("發送的 Token:", token);
  try {
    isLoading.value = true;
    const res = await axios.post(
      `${import.meta.env.VITE_API_URL}/api/tripShares/create/${props.tripId}`,
      {
        permission: selectedPermission.value,
      },
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    const data = {
      permission: selectedPermission.value, // 檢查 permission
    };
    console.log("發送的資料:", data);
    shareUrl.value = res.data.shareUrl;
    expiresAt.value = res.data.expiresAt;
  } catch (err) {
    alert("建立分享連結失敗");
    console.error(err);
  } finally {
    isLoading.value = false;
  }
};

const fetchSharedUsers = async () => {
  const token = localStorage.getItem("token");
  try {
    const res = await axios.get(
      `${import.meta.env.VITE_API_URL}/api/tripShares/list/${props.tripId}`,
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    sharedUsers.value = res.data.data;
    isOwner.value = res.data.isOwner;
  } catch (err) {
    console.error("取得共享者清單失敗", err);
  }
};

const updatePermission = async (targetUserId, newRole) => {
  const token = localStorage.getItem("token");
  try {
    await axios.patch(
      `${import.meta.env.VITE_API_URL}/api/tripShares/permission`,
      {
        targetUserId,
        tripId: props.tripId,
        newRole,
      },
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true,
      },
    );
    alert("權限已更新");
  } catch (err) {
    alert("權限更新失敗");
    console.error(err);
  }
};

const removeUser = async (targetUserId) => {
  if (!confirm("確定要取消共享這位使用者嗎？")) return;
  const token = localStorage.getItem("token");
  try {
    await axios.delete(
      `${import.meta.env.VITE_API_URL}/api/tripShares/remove/${targetUserId}/${props.tripId}`,
      {
        headers: { Authorization: `Bearer ${token}` },
        withCredentials: true, // 放在 config 裡
      },
    );
    sharedUsers.value = sharedUsers.value.filter((u) => u.id !== targetUserId);
    alert("已取消共享");
  } catch (err) {
    alert("取消共享失敗");
    console.error(err);
  }
};

const formatRole = (role) => {
  switch (role) {
    case "viewer":
      return "檢視者";
    case "editor":
      return "編輯者";
  }
};

// 開啟 Modal 時觸發
watch(
  () => props.isOpen,
  (open) => {
    if (open) fetchSharedUsers();
  },
);

const copyToClipboard = () => {
  navigator.clipboard.writeText(shareUrl.value);
  alert("已複製分享連結");
};

const formattedExpire = computed(() =>
  new Date(expiresAt.value).toLocaleString("zh-TW"),
);
</script>
