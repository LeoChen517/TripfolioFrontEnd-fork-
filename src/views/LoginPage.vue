<template>
  <div class="homepage-bg min-h-screen relative">
    <main
      class="w-1/2 py-5 absolute top-100 left-1/2 -translate-1/2 rounded-2xl bg-black/45 shadow-2xl min-h-[400px] min-w-[400px]"
    >
      <h4 class="text-lg font-bold mb-10 text-center text-white">登入</h4>

      <div
        v-if="showError"
        class="w-[300px] mx-auto gap-2 bg-red-100 text-red-800 border border-red-200 px-4 py-3 rounded-md mb-4"
      >
        <font-awesome-icon
          icon="exclamation-triangle"
          class="mr-2 text-red-500 text-lg"
        />
        <span class="text-sm font-medium mx-auto">
          {{ errorMessage }}
        </span>
      </div>

      <!-- 登入表單 -->
      <form
        v-if="!isLoggedIn"
        @submit.prevent="login"
        class="space-y-6 flex flex-col justify-center items-center"
      >
        <div>
          <input
            v-model="email"
            type="email"
            id="email"
            placeholder="請輸入電子郵件"
            required
            class="mb-2 block w-[300px] text-white rounded-md border border-gray-300 shadow-sm p-2"
          />
        </div>

        <div>
          <input
            v-model="password"
            type="password"
            id="password"
            placeholder="請輸入密碼"
            required
            class="mb-5 block w-[300px] text-white rounded-md border border-gray-300 shadow-sm p-2"
          />
        </div>

        <div class="flex justify-center gap-6 mt-4">
          <!-- Google 登入按鈕 -->
          <button
            type="button"
            @click="handleGoogleLogin"
            class="p-[10px] border border-[#fff] rounded-full text-[14px] flex items-center justify-center gap-2 hover:bg-white/30 cursor-pointer"
          >
            <img
              src="https://www.google.com/favicon.ico"
              alt="Google"
              class="w-5 h-5"
            />
            <span class="text-white px-0.5">使用 Google 登入</span>
          </button>

          <button
            type="submit"
            class="w-[100px] bg-black/50 text-white py-2 rounded-full transition mx-auto cursor-pointer hover:bg-white/30"
          >
            登入
          </button>
        </div>

        <RouterLink to="/signup">
          <button class="text-[#4d4d4d] py-2 transition hover:text-[#ffffff]">
            還沒有帳號？註冊
          </button>
        </RouterLink>
      </form>

      <!-- 登入成功畫面 -->
      <div v-else class="mt-6 bg-blue-50 text-center rounded-md shadow-md p-4">
        <p class="text-blue-700 font-semibold mb-4">您已成功登入！</p>
        <button
          @click="logout"
          class="bg-red-500 text-white py-2 px-4 rounded-md shadow-sm hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-red-500 transition duration-150 ease-in-out"
        >
          <font-awesome-icon
            icon="exclamation-triangle"
            class="mr-2 mt-0.5 text-red-600"
          />
          <span class="text-sm">
            {{ errorMessage }}
          </span>
        </button>
      </div>

      <div v-if="isLoggedIn" class="mt-6">
        <p class="text-blue-600 font-semibold mb-4">您已登入</p>
        <button
          @click="logout"
          class="bg-blue-200 text-black py-2 px-4 rounded transition"
        >
          登出
        </button>
      </div>
    </main>
  </div>
</template>

<script setup>
import { initializeAuth } from "../composable/authUtils";
import { ref, onMounted } from "vue";
import { useRouter } from "vue-router";
import axios from "axios";

const router = useRouter();
const TOKEN_NAME = "token";
const email = ref("");
const password = ref("");
const isLoggedIn = ref(false);
const showError = ref(false);
const errorMessage = ref("");
const API_BASE_URL = `${import.meta.env.VITE_API_URL}/api/login/`;

const clearText = () => {
  email.value = "";
  password.value = "";
};

const login = async () => {
  if (email.value === "" || password.value === "") {
    showError.value = true;
    errorMessage.value = "請輸入 Email 和密碼";
    return;
  }

  try {
    const res = await axios.post(API_BASE_URL, {
      email: email.value,
      password: password.value,
    });

    const token = res.data.token;
    localStorage.setItem(TOKEN_NAME, token);
    axios.defaults.headers.common["Authorization"] = `Bearer ${token}`;

    isLoggedIn.value = true;
    showError.value = false;
    clearText();

    window.dispatchEvent(new Event("login-status-changed"));
    router.push("/");
  } catch (err) {
    showError.value = true;
    const backendErrors = err.response?.data?.errors;
    if (backendErrors && backendErrors.length > 0) {
      errorMessage.value = backendErrors[0];
      console.error(err);
    } else {
      errorMessage.value = "登入失敗，請檢查郵件與密碼";
      console.error(err);
    }
  }
};

const handleGoogleLogin = () => {
  window.location.href = `${import.meta.env.VITE_API_URL}/auth/google`;
};

const logout = async () => {
  const token = localStorage.getItem(TOKEN_NAME);
  if (!token) return;

  localStorage.removeItem(TOKEN_NAME);
  localStorage.removeItem("memberId");
  isLoggedIn.value = false;
  clearText();

  window.dispatchEvent(new Event("login-status-changed"));
  router.push("/login");
};

onMounted(() => {
  initializeAuth(router);

  const urlParams = new URLSearchParams(window.location.search);
  const error = urlParams.get("error");
  const details = urlParams.get("details");

  if (error) {
    showError.value = true;
    errorMessage.value = decodeURIComponent(details || "登入失敗，請稍後再試");
    return;
  }

  const token = localStorage.getItem(TOKEN_NAME);
  if (!token) {
    isLoggedIn.value = false;
    return;
  }

  try {
    const payload = JSON.parse(atob(token.split(".")[1]));
    const exp = payload.exp;
    const now = Math.floor(Date.now() / 1000);

    if (exp > now) {
      isLoggedIn.value = true;
      axios.defaults.headers.common["Authorization"] = `Bearer ${token}`;
    } else {
      localStorage.removeItem(TOKEN_NAME);
      isLoggedIn.value = false;
    }
  } catch (err) {
    localStorage.removeItem(TOKEN_NAME);
    isLoggedIn.value = false;
  }
});
</script>
