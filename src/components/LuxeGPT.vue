<template>
  <div id="luxegpt-app" class="flex h-screen bg-[#000000] text-white antialiased">
    <Email v-if="showEmailModal" :guest-session-id="sessionId" @auth-success="handleAuthSuccess"
      @close-email="showEmailModal = false" />

    <aside v-if="isLoggedIn && isSidebarOpen"
      class="flex flex-col w-64 border-r border-gray-800 bg-[#0b0f13] p-4 h-full">
      <div class="flex items-center justify-between mb-6">
        <img src="/public/logo.png" alt="logo" class="h-6 w-28 object-contain">
        <button @click="toggleSidebar" class="md:hidden cursor-pointer">✕</button>
      </div>

      <button @click="startNewChat" class="mb-4 px-4 py-2 bg-[#19a8ff]/80 text-black rounded-lg cursor-pointer">New Chat</button>

      <div class="flex-grow overflow-y-auto space-y-2">
        <div v-for="chat in chatHistoryList" :key="chat.chatId" @click="selectChat(chat.chatId)"
          :class="['p-3 rounded-lg cursor-pointer', chat.chatId === activeChatId ? 'bg-[#001f30] font-semibold' : 'hover:bg-gray-800']">
          <div class="truncate text-sm">{{ chat.title || 'New Chat' }}</div>
        </div>
      </div>

      <div class="mt-4 pt-4 border-t border-gray-800">
        <div class="text-sm font-semibold text-white text-center">{{ userEmail || 'Guest' }}</div>
        <button v-if="isLoggedIn" @click="handleLogout"
          class="w-full mt-2 px-4 py-2 text-sm text-white bg-gray-300/20 rounded-lg cursor-pointer">Log Out</button>
      </div>
    </aside>

    <main class="flex-1 flex flex-col relative">
      <header class="flex items-center justify-between p-4 bg-[#0b0f13] border-b border-gray-800">
        <h1 class="text-2xl font-bold text-[#19a8ff]">
          LuxeAI Assistant
        </h1>
        <button v-if="!isLoggedIn" @click="openAuthModal(true)"
          class="px-4 py-2 text-sm border rounded-full text-[#19a8ff] cursor-pointer">
          Log In / Sign Up
        </button>
      </header>

      <div class="flex-1 overflow-hidden flex flex-col">
        <div ref="chatContainer" class="flex-1 p-6 overflow-y-auto space-y-6 bg-[#000000]">
          <div v-for="(msg, idx) in activeChatMessages" :key="idx"
            :class="['flex', msg.sender === 'user' ? 'justify-end' : 'justify-start']">
            <div
              :class="['max-w-3xl px-4 py-3 rounded-xl shadow-md', msg.sender === 'user' ? 'bg-gray-300/20' : 'bg-[#0f1316]']">
              <p class="whitespace-pre-wrap">{{ msg.content }}</p>
            </div>
          </div>

          <div v-if="!activeChatMessages.length && !isTyping" class="flex justify-center h-full items-center">
            <div class="text-center p-8 bg-[#0f1316] rounded-xl max-w-lg">
              <p class="text-xl font-semibold text-gray-200 mb-2">Hello👋 <br> Welcome to LuxeAI. How may I help?</p>
              <div class="mt-5 flex flex-wrap gap-2 justify-center">
                <button v-for="(q, i) in defaultQuestions" :key="i" @click="handleQuickQuestion(q)"
                  class="px-3 py-1.5 border rounded-full text-[#19a8ff]/80 text-sm cursor-pointer">{{ q }}</button>
              </div>
            </div>
          </div>

          <div v-if="isTyping" class="flex justify-start">
            <div class="max-w-xs px-4 py-3 bg-[#0f1316] text-gray-200 rounded-xl">
              <div class="flex space-x-1">
                <div class="w-2 h-2 rounded-full animate-bounce bg-[#19a8ff]"></div>
                <div class="w-2 h-2 rounded-full animate-bounce delay-150 bg-[#19a8ff]"></div>
                <div class="w-2 h-2 rounded-full animate-bounce delay-300 bg-[#19a8ff]"></div>
              </div>
            </div>
          </div>
        </div>

        <div class="p-6 bg-[#0b0f13] border-t border-gray-800">
          <button v-for="(q, i) in defaultQuestions" :key="i" @click="handleQuickQuestion(q)"
            class="mb-6 px-3 py-1.5 border rounded-full text-[#19a8ff]/80 text-sm cursor-pointer">{{ q }}</button>
          <div class="flex space-x-4">
            <input v-model="inputMessage" @keyup.enter="sendMessage" placeholder="Ask me anything..."
              class="flex-1 px-5 py-3 rounded-full bg-gray-200/10 text-gray-200" />
            <button @click="sendMessage" :disabled="isTyping || !inputMessage.trim()"
              class="px-6 py-3 rounded-full bg-[#19a8ff] text-black cursor-pointer">Send</button>
          </div>
          <p class="text-xs text-center text-gray-400 mt-2">&copy; Luxeveda 2025. All rights reserved</p>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from 'axios';
import Email from './Email.vue';

const BACKEND_URL = 'http://localhost:3001/api';

const sessionId = ref(localStorage.getItem('sessionId') || null);
const token = ref(localStorage.getItem('token') || null);
const userEmail = ref(localStorage.getItem('email') || null);
const isLoggedIn = ref(!!token.value);
const isSidebarOpen = ref(true);

const chatHistoryList = ref([]);
const activeChatId = ref(null);
const activeChatMessages = ref([]);
const inputMessage = ref('');
const isTyping = ref(false);
const showEmailModal = ref(false);
const chatContainer = ref(null);

const defaultQuestions = ref([
  "Tell me about your products or services",
  "Connect with us",
  "I have a question",
  "Generative AI solutions from Luxeveda"
]);

window.addEventListener('beforeunload', () => {
  if (!isLoggedIn.value) {
    localStorage.removeItem('sessionId');
    localStorage.removeItem('token');
    localStorage.removeItem('email');
  }
});

function api() {
  const headers = {};
  if (sessionId.value) headers['x-session-id'] = sessionId.value;
  if (token.value) headers['Authorization'] = `Bearer ${token.value}`;
  return axios.create({ baseURL: BACKEND_URL, headers });
}

async function ensureSession() {
  if (sessionId.value) {
    // ping server to verify
    try {
      const resp = await axios.get(`${BACKEND_URL}/session`, { headers: { 'x-session-id': sessionId.value } });
      sessionId.value = resp.data.sessionId;
      localStorage.setItem('sessionId', sessionId.value);
      userEmail.value = resp.data.email || null;
    } catch (err) {
      const resp2 = await axios.get(`${BACKEND_URL}/session`);
      sessionId.value = resp2.data.sessionId;
      localStorage.setItem('sessionId', sessionId.value);
    }
  } else {
    const resp = await axios.get(`${BACKEND_URL}/session`);
    sessionId.value = resp.data.sessionId;
    localStorage.setItem('sessionId', sessionId.value);
  }
}

onMounted(async () => {
  await ensureSession();
  await fetchChatList();
});

async function handleAuthSuccess(authData) {
  token.value = authData.token;
  userEmail.value = authData.email;
  localStorage.setItem('token', token.value);
  localStorage.setItem('email', userEmail.value);
  isLoggedIn.value = true;

  if (authData.sessionId && authData.sessionId !== sessionId.value) {
    sessionId.value = authData.sessionId;
    localStorage.setItem('sessionId', sessionId.value);
  }

  await fetchChatList();
  showEmailModal.value = false;
}

async function handleLogout() {
  try {
    localStorage.removeItem('token');
    localStorage.removeItem('email');
    token.value = null;
    userEmail.value = null;
    isLoggedIn.value = false;

    chatHistoryList.value = [];
    activeChatId.value = null;
    activeChatMessages.value = [];

    const resp = await axios.get(`${BACKEND_URL}/session`);
    sessionId.value = resp.data.sessionId;
    localStorage.setItem('sessionId', sessionId.value);

    await fetchChatList();
  } catch (err) {
    console.error('logout error', err);
    chatHistoryList.value = [];
    activeChatId.value = null;
    activeChatMessages.value = [];
    localStorage.removeItem('token');
    localStorage.removeItem('email');
    token.value = null;
    userEmail.value = null;
    isLoggedIn.value = false;
  }
}


function openAuthModal() { showEmailModal.value = true; }

function toggleSidebar() { isSidebarOpen.value = !isSidebarOpen.value; }

async function fetchChatList() {
  try {
    const resp = await api().get('/chat/list');
    chatHistoryList.value = resp.data || [];
    if (chatHistoryList.value.length > 0) {
      selectChat(chatHistoryList.value[0].chatId);
    } else {
      activeChatId.value = null;
      activeChatMessages.value = [];
    }
  } catch (err) {
    console.error('fetch chat list', err);
    chatHistoryList.value = [];
  }
}

async function selectChat(id) {
  if (!id) return;
  activeChatId.value = id;
  try {
    const resp = await api().get(`/chat/${id}`);
    activeChatMessages.value = resp.data || [];
  } catch (err) {
    console.error('selectChat', err);
    activeChatMessages.value = [{ sender: 'ai', content: 'Could not load chat.' }];
  } finally {
    nextTick(scrollToBottom);
  }
}

async function startNewChat() {
  try {
    const resp = await api().post('/chat/new');
    await fetchChatList();
    if (resp.data?.chatId) selectChat(resp.data.chatId);
  } catch (err) {
    console.error('startNewChat', err);
  }
}

async function sendMessage() {
  if (!inputMessage.value.trim() || isTyping.value) return;

  const message = inputMessage.value.trim();
  inputMessage.value = '';
  isTyping.value = true;

  activeChatMessages.value.push({ sender: 'user', content: message });
  nextTick(scrollToBottom);

  try {
    const payload = {
      chatId: activeChatId.value,
      message
    };
    const resp = await api().post('/chat', payload);
    const aiText = resp.data.aiResponse;
    activeChatMessages.value.push({ sender: 'ai', content: aiText });

    if (!activeChatId.value && resp.data.chatId) activeChatId.value = resp.data.chatId;

    await fetchChatList();
  } catch (err) {
    console.error('sendMessage', err);
    activeChatMessages.value.push({ sender: 'ai', content: err.response?.data?.message || 'API error' });
  } finally {
    isTyping.value = false;
    nextTick(scrollToBottom);
  }
}

function scrollToBottom() {
  if (chatContainer.value) {
    chatContainer.value.scrollTo({
      top: chatContainer.value.scrollHeight,
      behavior: "smooth"
    });
  }
}


async function handleQuickQuestion(q) {
  inputMessage.value = q;
  await nextTick();
  sendMessage();
}
</script>

<style scoped></style>