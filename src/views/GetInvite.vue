<template>
  <div class="flex justify-center items-center h-screen">
    <div class="text-lg text-gray-600">正在驗證共享連結...</div>
  </div>
</template>

<script setup>
import { useRouter, useRoute } from "vue-router";
import axios from "axios";

const router = useRouter();
const route = useRoute();

const token = route.params.token;

const checkShareToken = async () => {
  console.log("正在驗證分享 token:", token);
  const loginToken = localStorage.getItem("token");
  // 未登入直接導向登入頁
  if (!loginToken) {
    alert("請先登入再使用連結");
    router.replace("/login");
    return;
  }
  try {
    const url = `${import.meta.env.VITE_API_URL}/api/tripShares/check/${token}`;
    console.log("前端實際打的 URL:", url);

    const res = await axios.get(url, {
      headers: { Authorization: `Bearer ${loginToken}` },
      withCredentials: true,
    });
    console.log("API 回應成功:", res.data);
    const tripId = res.data.tripId;
    router.replace({ path: "/schedule", query: { openTripId: tripId } });
  } catch (err) {
    console.error("API 呼叫失敗:", err.response?.data || err.message);
    if (err.response?.status === 403) {
      alert("此分享連結已過期");
    } else {
      alert("此分享連結無效或已過期");
    }
    router.replace("/schedule");
  }
};

checkShareToken();
</script>
