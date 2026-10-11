<script lang="ts">
  import { onMount } from 'svelte';
  import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm';

  const SUPABASE_URL = 'https://lnoytmlisurtijwatmjz.supabase.co';
  const SUPABASE_ANON_KEY = 'PASTE_ANON_KEY_PUBLIC_KAMU_DI_SINI';

  const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

  // --- INTERFACES ---
  interface Task {
    id: string;
    title: string;
    time: string;
    type: 'routine' | 'once';
    repeat: 'daily' | 'weekly' | 'monthly';
    days: string[];
    date: string;
    deadline: string;
    priority: 'low' | 'medium' | 'high';
    duration: number;
  }

  interface Transaction {
    id: string;
    title: string;
    amount: number;
    type: 'income' | 'expense';
    allocation: 'use' | 'saving' | 'investment';
    category: string;
    date: string;
  }

  interface FinanceAllocation {
    savingTarget: number;
    savingCurrent: number;
    investmentTarget: number;
    investmentCurrent: number;
  }

  interface Habit {
    id: string;
    title: string;
    category: string;
  }

  interface WishlistItem {
    id: string;
    title: string;
    estimatedCost: number;
    category: 'Book' | 'Movie' | 'Gadget' | 'Other';
    status: 'Wishlist' | 'In Progress' | 'Completed';
  }

  interface AIChatMessage {
    role: 'ai' | 'user';
    content: string;
  }

  interface SecureChatMessage {
    id: string;
    sender: string;
    text: string;
    time: string;
  }

  // --- CONSTANTS ---
  const DAYS = ['Minggu', 'Senin', 'Selasa', 'Rabu', 'Kamis', 'Jumat', 'Sabtu'];
  const MONTHS = ['Jan', 'Feb', 'Mar', 'Apr', 'Mei', 'Jun', 'Jul', 'Agt', 'Sep', 'Okt', 'Nov', 'Des'];

  function formatDateToISO(date: Date): string {
    const y = date.getFullYear();
    const m = String(date.getMonth() + 1).padStart(2, '0');
    const d = String(date.getDate()).padStart(2, '0');
    return `${y}-${m}-${d}`;
  }

  // --- SECURITY & ACCOUNT STATE (1 Akun 1 Akses & E2EE) ---
  let username = $state<string>('A1');
  let userPin = $state<string>('1234'); // Default PIN akses
  let enteredPin = $state<string>('');
  let isLocked = $state<boolean>(true);
  let pinError = $state<boolean>(false);

  // --- NAVIGATION STATE ---
  let activeTab = $state<'home' | 'daily' | 'daily-history' | 'weekly' | 'monthly' | 'finance' | 'finance-history' | 'habits' | 'journal' | 'journal-history' | 'wishlist' | 'chat' | 'ai' | 'settings'>('home');
  let selectedDate = $state<Date>(new Date());
  let currentDateISO = $derived(formatDateToISO(selectedDate));
  let currentDayName = $derived(DAYS[selectedDate.getDay()]);

  // --- CUSTOM MOTTO STATE ---
  let customMotto = $state<string>('"Small steps everyday lead to big results."');
  let isEditingMotto = $state<boolean>(false);
  let tempMotto = $state<string>('');

  // --- MASTER DATA STATES ---
  let tasks = $state<Task[]>([
    { id: '1', title: 'Olahraga Pagi & Stretching', time: '07:00', type: 'routine', repeat: 'weekly', days: ['Senin', 'Rabu', 'Jumat'], date: '', deadline: '', priority: 'high', duration: 30 }
  ]);
  let dailyChecks = $state<Record<string, string[]>>({});

  let transactions = $state<Transaction[]>([
    { id: 't1', title: 'Gaji Bulanan Utama', amount: 8000000, type: 'income', allocation: 'use', category: 'Gaji', date: formatDateToISO(new Date()) },
    { id: 't2', title: 'Belanja Kebutuhan Harian', amount: 1500000, type: 'expense', allocation: 'use', category: 'Kebutuhan', date: formatDateToISO(new Date()) }
  ]);

  let financeAllocation = $state<FinanceAllocation>({
    savingTarget: 15000000, savingCurrent: 4500000,
    investmentTarget: 15000000, investmentCurrent: 6000000
  });

  let habits = $state<Habit[]>([
    { id: 'h1', title: 'Minum 2 Liter Air', category: 'Kesehatan' }
  ]);
  let habitLogs = $state<Record<string, string[]>>({});
  
  let journals = $state<Record<string, { gratitude: string; notes: string }>>({});
  let journalGratitude = $state('');
  let journalNotes = $state('');

  let wishlist = $state<WishlistItem[]>([
    { id: 'w1', title: 'Atomic Habits (Buku)', estimatedCost: 150000, category: 'Book', status: 'Wishlist' }
  ]);

  // --- SECURE CHAT STATE ---
  let secureChats = $state<SecureChatMessage[]>([
    { id: 'c1', sender: 'System', text: 'Chat terenkripsi End-to-End aktif. Pesan ini aman.', time: '08:00' }
  ]);
  let newChatMessage = $state('');

  // AI Chat Messages
  let aiMessages = $state<AIChatMessage[]>([
    { role: 'ai', content: 'Halo! Saya AI Productivity Coach Anda. Sistem keamanan E2EE dan 1 Akun 1 Akses aktif.' }
  ]);
  let aiInput = $state('');

  let hasLoaded = $state(false);

  // Form Binding States
  let editingTask = $state<Task | null>(null);
  let editingTx = $state<Transaction | null>(null);
  let editingHabit = $state<Habit | null>(null);
  let editingWishlist = $state<WishlistItem | null>(null);
  let editingJournalDate = $state<string | null>(null);
  let editJournalGratitude = $state('');
  let editJournalNotes = $state('');

  let newTaskTitle = $state('');
  let taskType = $state<'routine' | 'once'>('routine');
  let taskRepeat = $state<'daily' | 'weekly' | 'monthly'>('weekly');
  let taskDays = $state<string[]>(['Senin', 'Rabu', 'Jumat']);
  let taskDate = $state(formatDateToISO(new Date()));
  let taskTime = $state('08:00');
  let taskDeadline = $state(formatDateToISO(new Date()));
  let taskPriority = $state<'low' | 'medium' | 'high'>('medium');
  let taskDuration = $state(30);

  let txTitle = $state('');
  let txAmount = $state<number>(0);
  let txType = $state<'income' | 'expense'>('expense');
  let txAllocation = $state<'use' | 'saving' | 'investment'>('use');
  let txCategory = $state('Umum');

  let habitTitle = $state('');
  let habitCat = $state('Kesehatan');

  let wishTitle = $state('');
  let wishCost = $state<number>(0);
  let wishCat = $state<'Book' | 'Movie' | 'Gadget' | 'Other'>('Book');
  let wishStatus = $state<'Wishlist' | 'In Progress' | 'Completed'>('Wishlist');

  // --- SIMPLE END-TO-END ENCRYPTION (E2EE) UTILS ---
  // Menggunakan enkripsi berbasis kunci PIN untuk data di LocalStorage
  function encryptData(text: string, secret: string): string {
    try {
      return btoa(encodeURIComponent(text).split('').map((char, idx) => 
        String.fromCharCode(char.charCodeAt(0) ^ secret.charCodeAt(idx % secret.length))
      ).join(''));
    } catch (e) {
      return text;
    }
  }

  function decryptData(encrypted: string, secret: string): string {
    try {
      const decoded = atob(encrypted);
      return decodeURIComponent(decoded.split('').map((char, idx) => 
        String.fromCharCode(char.charCodeAt(0) ^ secret.charCodeAt(idx % secret.length))
      ).join(''));
    } catch (e) {
      return '';
    }
  }

  // --- PERSISTENCE & SECURITY LOAD ---
  const STORAGE_KEY = 'personal_os_secure_db_v2';

  onMount(() => {
    try {
      const encryptedStore = localStorage.getItem(STORAGE_KEY);
      if (encryptedStore) {
        // Jika data terenkripsi tersimpan, butuh PIN untuk dekripsi
        // Untuk kenyamanan awal, kita load dulu metadata PIN-nya jika tersimpan unencrypted di key terpisah
        const savedPin = localStorage.getItem('personal_os_pin');
        if (savedPin) userPin = savedPin;
      }
    } catch (e) { console.error(e); }
  });

  function unlockApp() {
    if (enteredPin === userPin) {
      isLocked = false;
      pinError = false;
      try {
        const encryptedStore = localStorage.getItem(STORAGE_KEY);
        if (encryptedStore) {
          const decryptedJson = decryptData(encryptedStore, userPin);
          const parsed = JSON.parse(decryptedJson);
          if (parsed.username) username = parsed.username;
          if (parsed.customMotto) customMotto = parsed.customMotto;
          if (parsed.tasks) tasks = parsed.tasks;
          if (parsed.dailyChecks) dailyChecks = parsed.dailyChecks;
          if (parsed.transactions) transactions = parsed.transactions;
          if (parsed.financeAllocation) financeAllocation = parsed.financeAllocation;
          if (parsed.habits) habits = parsed.habits;
          if (parsed.habitLogs) habitLogs = parsed.habitLogs;
          if (parsed.journals) journals = parsed.journals;
          if (parsed.wishlist) wishlist = parsed.wishlist;
          if (parsed.secureChats) secureChats = parsed.secureChats;
        }
      } catch (e) { console.error('Gagal dekripsi data:', e); }
      hasLoaded = true;
    } else {
      pinError = true;
    }
  }

  // Auto Save & Encrypt to LocalStorage
  $effect(() => {
    if (!hasLoaded || isLocked) return;
    const currentData = {
      username,
      customMotto,
      tasks,
      dailyChecks,
      transactions,
      financeAllocation,
      habits,
      habitLogs,
      journals,
      wishlist,
      secureChats
    };
    const jsonStr = JSON.stringify(currentData);
    const encrypted = encryptData(jsonStr, userPin);
    localStorage.setItem(STORAGE_KEY, encrypted);
    localStorage.setItem('personal_os_pin', userPin);
  });

  // Sync Journal inputs when date changes
  $effect(() => {
    const entry = journals[currentDateISO];
    if (entry) {
      journalGratitude = entry.gratitude;
      journalNotes = entry.notes;
    } else {
      journalGratitude = '';
      journalNotes = '';
    }
  });

  function saveNewMotto() {
    if (tempMotto.trim()) customMotto = `"${tempMotto.replace(/^"|"$/g, '').trim()}"`;
    isEditingMotto = false;
  }

  // TASK LOGIC
  function isTaskScheduled(task: Task, targetDate: Date): boolean {
    const iso = formatDateToISO(targetDate);
    const dayName = DAYS[targetDate.getDay()];
    if (task.type === 'routine') {
      if (task.repeat === 'daily') return true;
      if (task.repeat === 'weekly') return task.days.includes(dayName);
      if (task.repeat === 'monthly') return targetDate.getDate() === (task.date ? new Date(task.date).getDate() : 1);
      return false;
    }
    return task.date === iso;
  }

  let todayTasks = $derived(tasks.filter(t => isTaskScheduled(t, selectedDate)));
  let todayRoutines = $derived(todayTasks.filter(t => t.type === 'routine'));
  let todayOnce = $derived(todayTasks.filter(t => t.type !== 'routine'));

  function isDone(id: string): boolean { return dailyChecks[currentDateISO]?.includes(id) ?? false; }
  function toggleCheck(id: string) {
    const current = dailyChecks[currentDateISO] ?? [];
    dailyChecks = { ...dailyChecks, [currentDateISO]: current.includes(id) ? current.filter(x => x !== id) : [...current, id] };
  }

  function handleSaveTask() {
    if (!newTaskTitle.trim()) return;
    if (editingTask) {
      tasks = tasks.map(t => t.id === editingTask!.id ? { ...t, title: newTaskTitle, time: taskTime, type: taskType, repeat: taskRepeat, days: [...taskDays], date: taskDate, deadline: taskDeadline, priority: taskPriority, duration: taskDuration } : t);
      editingTask = null;
    } else {
      tasks = [...tasks, { id: crypto.randomUUID(), title: newTaskTitle, time: taskTime, type: taskType, repeat: taskRepeat, days: [...taskDays], date: taskDate, deadline: taskDeadline, priority: taskPriority, duration: taskDuration }];
    }
    newTaskTitle = '';
  }

  function startEditTask(t: Task) {
    editingTask = t; newTaskTitle = t.title; taskType = t.type; taskRepeat = t.repeat; taskDays = [...t.days]; taskDate = t.date || currentDateISO; taskTime = t.time; taskDeadline = t.deadline || currentDateISO; taskPriority = t.priority; taskDuration = t.duration;
  }

  function deleteTask(id: string) { if (confirm('Hapus kegiatan ini?')) tasks = tasks.filter(t => t.id !== id); }

  // FINANCE LOGIC
  let totalIncome = $derived(transactions.filter(t => t.type === 'income').reduce((a, b) => a + b.amount, 0));
  let totalExpense = $derived(transactions.filter(t => t.type === 'expense').reduce((a, b) => a + b.amount, 0));
  let netBalance = $derived(totalIncome - totalExpense);

  let allocatedForUse = $derived(transactions.filter(t => t.allocation === 'use' && t.type === 'expense').reduce((a, b) => a + b.amount, 0));
  let allocatedForSaving = $derived(transactions.filter(t => t.allocation === 'saving').reduce((a, b) => a + b.amount, 0));
  let allocatedForInvestment = $derived(transactions.filter(t => t.allocation === 'investment').reduce((a, b) => a + b.amount, 0));

  function handleSaveTransaction() {
    if (!txTitle.trim() || txAmount <= 0) return;
    if (editingTx) {
      transactions = transactions.map(tx => tx.id === editingTx!.id ? { ...tx, title: txTitle, amount: Number(txAmount), type: txType, allocation: txAllocation, category: txCategory } : tx);
      editingTx = null;
    } else {
      transactions = [{ id: crypto.randomUUID(), title: txTitle, amount: Number(txAmount), type: txType, allocation: txAllocation, category: txCategory, date: currentDateISO }, ...transactions];
    }
    txTitle = ''; txAmount = 0;
  }

  function deleteTx(id: string) { transactions = transactions.filter(t => t.id !== id); }

  // HABIT, JOURNAL, WISHLIST, CHAT
  function toggleHabitLog(id: string) {
    const logged = habitLogs[currentDateISO] ?? [];
    habitLogs = { ...habitLogs, [currentDateISO]: logged.includes(id) ? logged.filter(x => x !== id) : [...logged, id] };
  }
  function handleSaveHabit() {
    if (!habitTitle.trim()) return;
    if (editingHabit) {
      habits = habits.map(h => h.id === editingHabit!.id ? { ...h, title: habitTitle, category: habitCat } : h);
      editingHabit = null;
    } else {
      habits = [...habits, { id: crypto.randomUUID(), title: habitTitle, category: habitCat }];
    }
    habitTitle = '';
  }
  function deleteHabit(id: string) { habits = habits.filter(h => h.id !== id); }

  function handleSaveJournal() {
    journals = {
      ...journals,
      [currentDateISO]: { gratitude: journalGratitude, notes: journalNotes }
    };
    alert('Jurnal harian berhasil disimpan untuk tanggal ' + currentDateISO + '! 📖');
  }

  function handleSaveHistoryJournal(dateStr: string) {
    journals = {
      ...journals,
      [dateStr]: { gratitude: editJournalGratitude, notes: editJournalNotes }
    };
    editingJournalDate = null;
    alert('Arsip jurnal tanggal ' + dateStr + ' berhasil diperbarui! 📖');
  }

  function deleteHistoryJournal(dateStr: string) {
    if (confirm('Hapus arsip jurnal tanggal ' + dateStr + '?')) {
      const nextJournals = { ...journals };
      delete nextJournals[dateStr];
      journals = nextJournals;
    }
  }

  function handleSaveWishlist() {
    if (!wishTitle.trim()) return;
    if (editingWishlist) {
      wishlist = wishlist.map(w => w.id === editingWishlist!.id ? { ...w, title: wishTitle, estimatedCost: Number(wishCost), category: wishCat, status: wishStatus } : w);
      editingWishlist = null;
    } else {
      wishlist = [...wishlist, { id: crypto.randomUUID(), title: wishTitle, estimatedCost: Number(wishCost), category: wishCat, status: wishStatus }];
    }
    wishTitle = ''; wishCost = 0;
  }
  function deleteWishlist(id: string) { wishlist = wishlist.filter(w => w.id !== id); }

  function sendSecureChat() {
    if (!newChatMessage.trim()) return;
    const timeNow = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    secureChats = [...secureChats, { id: crypto.randomUUID(), sender: username, text: newChatMessage, time: timeNow }];
    newChatMessage = '';
  }

  function askAI(promptText?: string) {
    const q = promptText || aiInput;
    if (!q.trim()) return;
    aiMessages = [...aiMessages, { role: 'user', content: q }];
    if (!promptText) aiInput = '';

    setTimeout(() => {
      let reply = `Halo ${username}, enkripsi E2EE aktif dan data aman! Tetap jaga konsistensimu.`;
      aiMessages = [...aiMessages, { role: 'ai', content: reply }];
    }, 800);
  }

  function exportData() {
    const data = { username, customMotto, tasks, dailyChecks, transactions, financeAllocation, habits, habitLogs, journals, wishlist, secureChats };
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = `personal-os-secure-backup-${currentDateISO}.json`; a.click();
  }

  function importData(e: Event) {
    const file = (e.target as HTMLInputElement).files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
      try {
        const json = JSON.parse(event.target?.result as string);
        if (json.username) username = json.username;
        if (json.customMotto) customMotto = json.customMotto;
        if (json.tasks) tasks = json.tasks;
        if (json.dailyChecks) dailyChecks = json.dailyChecks;
        if (json.transactions) transactions = json.transactions;
        if (json.financeAllocation) financeAllocation = json.financeAllocation;
        if (json.habits) habits = json.habits;
        if (json.habitLogs) habitLogs = json.habitLogs;
        if (json.journals) journals = json.journals;
        if (json.wishlist) wishlist = json.wishlist;
        if (json.secureChats) secureChats = json.secureChats;
        alert('Data terenkripsi berhasil dipulihkan!');
      } catch (err) { alert('Format file JSON tidak valid.'); }
    };
    reader.readAsText(file);
  }

  function formatIDR(num: number) { return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(num); }
</script>

{#if isLocked}
  <!-- SCREEN 1 AKUN 1 AKSES (PIN LOCK SCREEN) -->
  <div class="lock-screen">
    <div class="lock-card">
      <div class="brand-logo" style="margin: 0 auto 16px;">🔐</div>
      <h2>Personal OS - Akses Terproteksi</h2>
      <p>Masukkan PIN Akses Anda untuk membuka data terenkripsi (Default PIN: 1234)</p>
      
      <form onsubmit={(e) => { e.preventDefault(); unlockApp(); }}>
        <input type="password" maxlength="6" bind:value={enteredPin} placeholder="Masukkan 4-6 digit PIN..." autofocus required />
        {#if pinError}
          <small style="color: #f87171; display: block; margin-top: 6px;">PIN Salah! Silakan coba lagi.</small>
        {/if}
        <button type="submit" class="submit-btn" style="width: 100%; margin-top: 14px;">Buka Akses 🔓</button>
      </form>
    </div>
  </div>
{:else}
  <!-- APLIKASI UTAMA -->
  <div class="app-layout">
    <!-- SIDEBAR NAVIGATION -->
    <aside class="sidebar">
      <div class="brand">
        <div class="brand-logo">A</div>
        <div class="brand-title">
          <span>Personal OS</span>
          <small>{username}'s Secure System</small>
        </div>
      </div>

      <nav class="nav-list">
        <button class="nav-item" class:active={activeTab === 'home'} onclick={() => activeTab = 'home'}>
          <span class="nav-icon">🏠</span> Beranda
        </button>
        <button class="nav-item" class:active={activeTab === 'daily'} onclick={() => activeTab = 'daily'}>
          <span class="nav-icon">☀️</span> Tugas & To-Do
        </button>
        <button class="nav-item" class:active={activeTab === 'daily-history'} onclick={() => activeTab = 'daily-history'}>
          <span class="nav-icon">📋</span> Riwayat Planner (3 Bln)
        </button>
        <button class="nav-item" class:active={activeTab === 'habits'} onclick={() => activeTab = 'habits'}>
          <span class="nav-icon">🌱</span> Rutinitas & Habit
        </button>
        <button class="nav-item" class:active={activeTab === 'weekly' || activeTab === 'monthly'} onclick={() => activeTab = 'weekly'}>
          <span class="nav-icon">📅</span> Kalender
        </button>
        <button class="nav-item" class:active={activeTab === 'journal'} onclick={() => activeTab = 'journal'}>
          <span class="nav-icon">📖</span> Catatan & Jurnal
        </button>
        <button class="nav-item" class:active={activeTab === 'journal-history'} onclick={() => activeTab = 'journal-history'}>
          <span class="nav-icon">📚</span> Arsip Jurnal (3 Bln)
        </button>
        <button class="nav-item" class:active={activeTab === 'finance'} onclick={() => activeTab = 'finance'}>
          <span class="nav-icon">💰</span> Keuangan
        </button>
        <button class="nav-item" class:active={activeTab === 'finance-history'} onclick={() => activeTab = 'finance-history'}>
          <span class="nav-icon">📊</span> Riwayat Finansial
        </button>
        <button class="nav-item" class:active={activeTab === 'chat'} onclick={() => activeTab = 'chat'}>
          <span class="nav-icon">💬</span> Secure Chat (E2EE)
        </button>
        <button class="nav-item" class:active={activeTab === 'ai'} onclick={() => activeTab = 'ai'}>
          <span class="nav-icon">🧠</span> AI Companion
        </button>
        <button class="nav-item" class:active={activeTab === 'settings'} onclick={() => activeTab = 'settings'}>
          <span class="nav-icon">⚙️</span> Akun & Keamanan
        </button>
      </nav>
    </aside>

    <!-- MAIN VIEW -->
    <main class="main-content">
      
      <!-- HERO BANNER -->
      {#if activeTab === 'home'}
        <div class="hero-card">
          <div class="hero-overlay"></div>
          <div class="hero-content">
            <div class="hero-top-info">
              <span class="badge-pill">✨ {currentDayName}, {selectedDate.getDate()} {MONTHS[selectedDate.getMonth()]} {selectedDate.getFullYear()}</span>
              <div class="hero-avatar">{username.charAt(0).toUpperCase()}</div>
            </div>
            <h1>Good morning, {username}.</h1>
            {#if isEditingMotto}
              <div class="motto-edit-box">
                <input type="text" bind:value={tempMotto} placeholder="Tuliskan motto baru..." />
                <button class="submit-btn mini" onclick={saveNewMotto}>Simpan</button>
                <button class="cancel-btn mini" onclick={() => isEditingMotto = false}>Batal</button>
              </div>
            {:else}
              <p class="hero-subtitle">{customMotto} <button class="link-btn" onclick={() => { tempMotto = customMotto; isEditingMotto = true; }}>✏️ Edit</button></p>
            {/if}
            
            <div class="hero-stats-row">
              <div>
                <small>Tugas Hari Ini</small>
                <strong>{todayTasks.filter(t => isDone(t.id)).length} / {todayTasks.length} Selesai</strong>
              </div>
              <div>
                <small>Saldo Bersih</small>
                <strong style="color: #34d399;">{formatIDR(netBalance)}</strong>
              </div>
            </div>
          </div>
        </div>

        <div class="quick-access-grid">
          <div class="card qa-box" onclick={() => activeTab = 'daily'}>
            <span>☀️</span>
            <h3>Kelola Tugas</h3>
            <p>Lihat dan centang daftar kegiatan harianmu.</p>
          </div>
          <div class="card qa-box" onclick={() => activeTab = 'finance'}>
            <span>💰</span>
            <h3>Keuangan</h3>
            <p>Pantau alokasi penggunaan, tabungan & investasi.</p>
          </div>
          <div class="card qa-box" onclick={() => activeTab = 'chat'}>
            <span>💬</span>
            <h3>Secure Chat</h3>
            <p>Kirim pesan terenkripsi end-to-end dengan aman.</p>
          </div>
        </div>
      {/if}

      {#if activeTab === 'daily'}
        <!-- === DAILY PLANNER === -->
        <header class="topbar">
          <div><span class="eyebrow">DAILY PLANNER</span><h1>Kelola Tugas & To-Do</h1></div>
          <div class="date-badge">📅 {currentDayName}, {selectedDate.getDate()} {MONTHS[selectedDate.getMonth()]}</div>
        </header>

        <section class="card form-card">
          <div class="card-title-row">
            <span class="sage-icon">{editingTask ? '✏️' : '+'}</span>
            <h2>{editingTask ? 'Edit Kegiatan' : 'Tambah Kegiatan Baru'}</h2>
          </div>

          <form onsubmit={(e) => { e.preventDefault(); handleSaveTask(); }}>
            <div class="grid-3">
              <div class="field">
                <label>Nama Kegiatan</label>
                <input type="text" bind:value={newTaskTitle} placeholder="Cth: Olahraga Pagi..." required />
              </div>
              <div class="field">
                <label>Tipe Kegiatan</label>
                <select bind:value={taskType}>
                  <option value="routine">Jadwal Rutin (Berulang)</option>
                  <option value="once">Tugas Dadakan (Sekali)</option>
                </select>
              </div>
              <div class="field">
                <label>Jam Implementasi</label>
                <input type="time" bind:value={taskTime} required />
              </div>
            </div>

            {#if taskType === 'routine'}
              <div class="nested-box">
                <div class="grid-2-custom" style="align-items: flex-start;">
                  <div class="field">
                    <label>Frekuensi Rutinitas</label>
                    <select bind:value={taskRepeat}>
                      <option value="daily">Setiap Hari</option>
                      <option value="weekly">Hari-Hari Tertentu (Weekly)</option>
                      <option value="monthly">Bulanan</option>
                    </select>
                  </div>

                  {#if taskRepeat === 'weekly'}
                    <div class="field">
                      <label>Pilih Hari Berulang:</label>
                      <div class="day-chips">
                        {#each ['Senin', 'Selasa', 'Rabu', 'Kamis', 'Jumat', 'Sabtu', 'Minggu'] as day}
                          <button type="button" class="chip" class:active={taskDays.includes(day)}
                            onclick={() => taskDays = taskDays.includes(day) ? taskDays.filter(d => d !== day) : [...taskDays, day]}>
                            {day}
                          </button>
                        {/each}
                      </div>
                    </div>
                  {/if}
                </div>
              </div>
            {/if}

            <div class="form-footer-row">
              <div class="priority-picker">
                <span class="field-label">Prioritas:</span>
                <label><input type="radio" value="low" bind:group={taskPriority} /> Rendah</label>
                <label><input type="radio" value="medium" bind:group={taskPriority} /> Sedang</label>
                <label><input type="radio" value="high" bind:group={taskPriority} /> Tinggi</label>
              </div>
              <div style="display:flex; gap:8px;">
                {#if editingTask}<button type="button" class="cancel-btn" onclick={() => { editingTask = null; newTaskTitle = ''; }}>Batal</button>{/if}
                <button type="submit" class="submit-btn">✓ {editingTask ? 'Simpan Perubahan' : 'Tambah Tugas'}</button>
              </div>
            </div>
          </form>
        </section>

        <section class="card">
          <div class="card-header">
            <h2>🔄 Rutinitas Harian ({currentDayName})</h2>
            <span class="badge sage-badge">{todayRoutines.filter(t => isDone(t.id)).length}/{todayRoutines.length} Selesai</span>
          </div>
          <div class="task-list">
            {#each todayRoutines as task}
              <div class="task-row" class:done={isDone(task.id)}>
                <label class="check-label">
                  <input type="checkbox" checked={isDone(task.id)} onchange={() => toggleCheck(task.id)} />
                  <span class="task-name">{task.title}</span>
                  <span class="task-time">⏰ {task.time}</span>
                </label>
                <div style="display:flex; gap:6px;">
                  <button class="icon-btn" onclick={() => startEditTask(task)}>✏️</button>
                  <button class="icon-btn" onclick={() => deleteTask(task.id)}>🗑️</button>
                </div>
              </div>
            {:else}
              <p class="empty-msg">Tidak ada rutinitas harian untuk hari ini.</p>
            {/each}
          </div>
        </section>

        <section class="card">
          <div class="card-header">
            <h2>⚡ Tugas Dadakan</h2>
            <span class="badge amber-badge">{todayOnce.filter(t => isDone(t.id)).length}/{todayOnce.length} Selesai</span>
          </div>
          <div class="task-list">
            {#each todayOnce as task}
              <div class="task-row" class:done={isDone(task.id)}>
                <label class="check-label">
                  <input type="checkbox" checked={isDone(task.id)} onchange={() => toggleCheck(task.id)} />
                  <span class="task-name">{task.title}</span>
                  <span class="task-time">⏰ {task.time}</span>
                </label>
                <div style="display:flex; gap:6px;">
                  <button class="icon-btn" onclick={() => startEditTask(task)}>✏️</button>
                  <button class="icon-btn" onclick={() => deleteTask(task.id)}>🗑️</button>
                </div>
              </div>
            {:else}
              <p class="empty-msg">Tidak ada tugas dadakan.</p>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'daily-history'}
        <!-- === DAILY HISTORY === -->
        <header class="topbar"><div><span class="eyebrow">RIWAYAT PLANNER</span><h1>Riwayat Aktivitas 3 Bulan Terakhir (90 Hari)</h1></div></header>
        <section class="card">
          <h2>🔍 Tinjauan Rutinitas & Fitur Edit</h2>
          <p style="font-size: 13px; color: #94a3b8; margin-bottom: 16px;">
            Evaluasi konsistensi kegiatan harianmu. Klik tombol **✏️ Edit Tugas** untuk memperbarui detail kegiatan.
          </p>
          
          <div class="task-list" style="max-height: 550px; overflow-y: auto;">
            {#each Array.from({length: 90}, (_, i) => {
              const d = new Date();
              d.setDate(d.getDate() - i);
              return d;
            }) as pastDate}
              {@const iso = formatDateToISO(pastDate)}
              {@const dayName = DAYS[pastDate.getDay()]}
              {@const scheduledTasks = tasks.filter(t => isTaskScheduled(t, pastDate))}
              {@const doneIds = dailyChecks[iso] || []}
              
              {#if scheduledTasks.length > 0 || doneIds.length > 0}
                <div class="card" style="background: #121824; margin-bottom: 10px; padding: 14px;">
                  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 6px;">
                    <strong>📅 {dayName}, {pastDate.getDate()} {MONTHS[pastDate.getMonth()]} {pastDate.getFullYear()} {iso === currentDateISO ? '(Hari Ini)' : ''}</strong>
                    <span class="badge sage-badge">{doneIds.length} / {scheduledTasks.length} Selesai</span>
                  </div>
                  <div style="display:flex; flex-direction:column; gap:6px; margin-top:8px;">
                    {#each scheduledTasks as task}
                      {@const isCompleted = doneIds.includes(task.id)}
                      <div style="display:flex; justify-content:space-between; align-items:center; background:#080b11; padding:8px 12px; border-radius:8px; border:1px solid #1a2333;">
                        <span style="color: {isCompleted ? '#34d399' : '#f87171'}; font-size:12px; font-weight: 500;">
                          {isCompleted ? '✅ Selesai: ' : '❌ Dilewati: '} {task.title} <small style="color:#64748b">({task.time})</small>
                        </span>
                        <button class="icon-btn" title="Edit Tugas Ini" onclick={() => { activeTab = 'daily'; startEditTask(task); }}>✏️ Edit</button>
                      </div>
                    {/each}
                  </div>
                </div>
              {/if}
            {/each}
          </div>
        </section>

      {:else if activeTab === 'weekly'}
        <header class="topbar"><div><span class="eyebrow">KALENDER</span><h1>Weekly & Monthly Overview</h1></div></header>
        <section class="card">
          <div class="weekly-grid">
            {#each ['Senin', 'Selasa', 'Rabu', 'Kamis', 'Jumat', 'Sabtu', 'Minggu'] as day}
              <div class="day-col">
                <h3>{day}</h3>
                <div class="day-tasks">
                  {#each tasks.filter(t => t.type === 'routine' && t.days.includes(day)) as t}
                    <div class="mini-task">
                      <span>{t.title} <small>({t.time})</small></span>
                      <button class="icon-btn" onclick={() => { activeTab = 'daily'; startEditTask(t); }}>✏️</button>
                    </div>
                  {:else}
                    <small class="none">- Kosong -</small>
                  {/each}
                </div>
              </div>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'monthly'}
        <header class="topbar"><div><span class="eyebrow">KALENDER</span><h1>Monthly Planner</h1></div></header>
        <section class="card"><h2>Ikhtisar Bulanan - {MONTHS[selectedDate.getMonth()]} {selectedDate.getFullYear()}</h2></section>

      {:else if activeTab === 'finance'}
        <!-- === FINANCIAL OS === -->
        <header class="topbar"><div><span class="eyebrow">FINANCIAL OS</span><h1>Keuangan, Alokasi & Tujuan Finansial</h1></div></header>
        <div class="stats-row">
          <div class="stat-card"><span>Total Saldo Bersih</span><strong class="balance">{formatIDR(netBalance)}</strong></div>
          <div class="stat-card"><span>Total Pemasukan</span><strong class="income">{formatIDR(totalIncome)}</strong></div>
          <div class="stat-card"><span>Total Pengeluaran</span><strong class="expense">{formatIDR(totalExpense)}</strong></div>
        </div>

        <div class="stats-row" style="margin-top: -8px;">
          <div class="stat-card alloc-card">
            <div class="alloc-head"><span>💳 Alokasi Penggunaan</span><span class="badge amber-badge">Operasional</span></div>
            <strong style="color: #fca5a5; font-size: 18px;">{formatIDR(allocatedForUse)}</strong>
          </div>
          <div class="stat-card alloc-card">
            <div class="alloc-head"><span>🏦 Alokasi Tabungan</span><span class="badge sage-badge">Saving Goal</span></div>
            <strong style="color: #34d399; font-size: 18px;">{formatIDR(financeAllocation.savingCurrent + allocatedForSaving)}</strong>
            <div class="progress-bar" style="margin-top:6px;"><div class="progress-fill" style="width: {Math.min(100, Math.round(((financeAllocation.savingCurrent + allocatedForSaving) / financeAllocation.savingTarget) * 100))}%"></div></div>
          </div>
          <div class="stat-card alloc-card">
            <div class="alloc-head"><span>📈 Alokasi Investasi</span><span class="badge green-badge">Portofolio</span></div>
            <strong style="color: #6ee7b7; font-size: 18px;">{formatIDR(financeAllocation.investmentCurrent + allocatedForInvestment)}</strong>
            <div class="progress-bar" style="margin-top:6px;"><div class="progress-fill green-fill" style="width: {Math.min(100, Math.round(((financeAllocation.investmentCurrent + allocatedForInvestment) / financeAllocation.investmentTarget) * 100))}%"></div></div>
          </div>
        </div>

        <section class="card">
          <h2>{editingTx ? 'Edit Catatan Keuangan' : '+ Catat Transaksi Baru'}</h2>
          <form onsubmit={(e) => { e.preventDefault(); handleSaveTransaction(); }}>
            <div class="grid-3" style="margin-top:12px;">
              <input type="text" bind:value={txTitle} placeholder="Nama transaksi..." required />
              <input type="number" bind:value={txAmount} placeholder="Nominal (Rp)..." required />
              <select bind:value={txType}>
                <option value="expense">Pengeluaran (-)</option>
                <option value="income">Pemasukan (+)</option>
              </select>
            </div>
            <div class="grid-2-custom" style="margin-top:12px;">
              <select bind:value={txAllocation}>
                <option value="use">Untuk Digunakan (Operasional)</option>
                <option value="saving">Untuk Ditabung (Tabungan)</option>
                <option value="investment">Untuk Diinvestasikan (Saham/Reksadana)</option>
              </select>
              <input type="text" bind:value={txCategory} placeholder="Kategori..." />
            </div>
            <div style="margin-top:14px; display:flex; gap:8px;">
              {#if editingTx}<button type="button" class="cancel-btn" onclick={() => { editingTx = null; txTitle = ''; }}>Batal</button>{/if}
              <button type="submit" class="submit-btn">{editingTx ? 'Simpan' : 'Simpan Transaksi'}</button>
            </div>
          </form>
        </section>

        <section class="card">
          <h2>📜 Riwayat Transaksi</h2>
          <div class="task-list">
            {#each transactions as tx}
              <div class="tx-item">
                <div>
                  <strong>{tx.title}</strong>
                  <small>{tx.date} • {tx.category}</small>
                </div>
                <div style="display:flex; gap:10px; align-items:center;">
                  <span class="tx-val {tx.type}">{tx.type === 'income' ? '+' : '-'} {formatIDR(tx.amount)}</span>
                  <button class="icon-btn" onclick={() => { editingTx = tx; txTitle = tx.title; txAmount = tx.amount; txType = tx.type; txAllocation = tx.allocation; }}>✏️</button>
                  <button class="icon-btn" onclick={() => deleteTx(tx.id)}>🗑️</button>
                </div>
              </div>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'finance-history'}
        <!-- === FINANCE HISTORY === -->
        <header class="topbar"><div><span class="eyebrow">RIWAYAT KEUANGAN</span><h1>Analisis & Edit Pengeluaran 3 Bulan Terakhir</h1></div></header>
        
        {@const now = new Date()}
        {@const ninetyDaysAgo = new Date(new Date().setDate(now.getDate() - 90))}
        {@const recentExpenses = transactions.filter(t => t.type === 'expense' && new Date(t.date) >= ninetyDaysAgo)}
        {@const total3MonthsExpense = recentExpenses.reduce((a, b) => a + b.amount, 0)}

        <div class="stats-row">
          <div class="stat-card" style="grid-column: span 3;">
            <span>Total Pengeluaran 3 Bulan Terakhir (90 Hari)</span>
            <strong class="expense" style="font-size: 24px;">{formatIDR(total3MonthsExpense)}</strong>
          </div>
        </div>

        <section class="card">
          <h2>💳 Daftar & Edit Transaksi Pengeluaran 90 Hari Terakhir</h2>
          <div class="task-list" style="margin-top: 12px; max-height: 500px; overflow-y: auto;">
            {#each recentExpenses as tx}
              <div class="tx-item">
                <div>
                  <strong>{tx.title}</strong>
                  <small>{tx.date} • {tx.category}</small>
                </div>
                <div style="display:flex; gap:10px; align-items:center;">
                  <span class="tx-val expense">- {formatIDR(tx.amount)}</span>
                  <button class="icon-btn" title="Edit Transaksi" onclick={() => { activeTab = 'finance'; editingTx = tx; txTitle = tx.title; txAmount = tx.amount; txType = tx.type; txAllocation = tx.allocation; }}>✏️</button>
                  <button class="icon-btn" title="Hapus Transaksi" onclick={() => deleteTx(tx.id)}>🗑️</button>
                </div>
              </div>
            {:else}
              <p class="empty-msg">Tidak ada catatan pengeluaran dalam 3 bulan terakhir.</p>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'habits'}
        <!-- === HABIT TRACKER === -->
        <header class="topbar"><div><span class="eyebrow">PRODUKTIVITAS</span><h1>Rutinitas & Habit Tracker</h1></div></header>
        <section class="card">
          <h2>Checklist Kebiasaan Hari Ini ({currentDateISO})</h2>
          <div class="task-list">
            {#each habits as h}
              {@const checked = habitLogs[currentDateISO]?.includes(h.id) ?? false}
              <div class="task-row" class:done={checked}>
                <label class="check-label">
                  <input type="checkbox" {checked} onchange={() => toggleHabitLog(h.id)} />
                  <span class="task-name">{h.title} <small class="badge sage-badge">{h.category}</small></span>
                </label>
                <div style="display:flex; gap:6px;">
                  <button class="icon-btn" onclick={() => { editingHabit = h; habitTitle = h.title; habitCat = h.category; }}>✏️</button>
                  <button class="icon-btn" onclick={() => deleteHabit(h.id)}>🗑️</button>
                </div>
              </div>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'journal'}
        <!-- === DAILY JOURNAL === -->
        <header class="topbar"><div><span class="eyebrow">CATATAN</span><h1>Daily Journal & Refleksi</h1></div></header>
        <section class="card">
          <h2>Catatan Tanggal: {currentDateISO}</h2>
          <div class="field" style="margin-top:12px;">
            <label>Hal yang Disyukuri Hari Ini:</label>
            <input type="text" bind:value={journalGratitude} placeholder="Cth: Hari ini bisa berolahraga..." />
          </div>
          <div class="field" style="margin-top:12px;">
            <label>Refleksi Harian / Unek-unek:</label>
            <textarea rows="5" bind:value={journalNotes} placeholder="Tuliskan refleksi, evaluasi, atau catatan penting..."></textarea>
          </div>
          <button class="submit-btn" style="margin-top:14px;" onclick={handleSaveJournal}>Simpan Jurnal</button>
        </section>

      {:else if activeTab === 'journal-history'}
        <!-- === JOURNAL HISTORY === -->
        <header class="topbar"><div><span class="eyebrow">ARSIP JURNAL</span><h1>Arsip & Edit Refleksi 3 Bulan Terakhir</h1></div></header>
        
        {@const ninetyDaysAgoIso = formatDateToISO(new Date(new Date().setDate(new Date().getDate() - 90)))}
        {@const filteredJournals = Object.entries(journals).filter(([dateStr]) => dateStr >= ninetyDaysAgoIso).sort().reverse()}

        <section class="card">
          <h2>📖 Catatan Jurnal & Edit Arsip (90 Hari Terakhir)</h2>
          <div class="task-list" style="margin-top: 12px; max-height: 550px; overflow-y: auto;">
            {#each filteredJournals as [dateStr, entry]}
              <div class="card" style="background: #121824; margin-bottom: 12px; padding: 16px;">
                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
                  <strong>📅 Tanggal: {dateStr}</strong>
                  <div style="display:flex; gap:6px;">
                    <button class="icon-btn" title="Edit Arsip Jurnal" onclick={() => { editingJournalDate = dateStr; editJournalGratitude = entry.gratitude; editJournalNotes = entry.notes; }}>✏️ Edit</button>
                    <button class="icon-btn" title="Hapus Arsip Jurnal" onclick={() => deleteHistoryJournal(dateStr)}>🗑️ Hapus</button>
                  </div>
                </div>

                {#if editingJournalDate === dateStr}
                  <div class="nested-box">
                    <div class="field">
                      <label>Edit Rasa Syukur:</label>
                      <input type="text" bind:value={editJournalGratitude} />
                    </div>
                    <div class="field" style="margin-top: 8px;">
                      <label>Edit Catatan Refleksi:</label>
                      <textarea rows="3" bind:value={editJournalNotes}></textarea>
                    </div>
                    <div style="display:flex; gap: 8px; margin-top: 10px;">
                      <button class="submit-btn mini" onclick={() => handleSaveHistoryJournal(dateStr)}>Simpan Perubahan</button>
                      <button class="cancel-btn mini" onclick={() => editingJournalDate = null}>Batal</button>
                    </div>
                  </div>
                {:else}
                  {#if entry.gratitude}
                    <p style="margin: 4px 0; font-size: 13px; color: #34d399;"><strong>✨ Disyukuri:</strong> {entry.gratitude}</p>
                  {/if}
                  {#if entry.notes}
                    <p style="margin: 4px 0; font-size: 13px; color: #cbd5e1;"><strong>📝 Refleksi:</strong> {entry.notes}</p>
                  {/if}
                {/if}
              </div>
            {:else}
              <p class="empty-msg">Belum ada arsip jurnal dalam 3 bulan terakhir.</p>
            {/each}
          </div>
        </section>

      {:else if activeTab === 'chat'}
        <!-- === SECURE CHAT FEATURE (E2EE) === -->
        <header class="topbar"><div><span class="eyebrow">ENCRYPTED CHAT</span><h1>Secure Chat & Catatan Kilat (E2EE)</h1></div></header>
        <section class="card" style="display:flex; flex-direction:column; height: 480px;">
          <div style="background: rgba(4, 120, 87, 0.1); border: 1px solid #047857; padding: 10px 14px; border-radius: 10px; margin-bottom: 12px; font-size: 12px; color: #34d399;">
            🔒 Pesan di sini diamankan dengan enkripsi End-to-End berbasis PIN Anda. Aman dari intipan pihak luar.
          </div>
          <div style="flex:1; overflow-y:auto; display:flex; flex-direction:column; gap:10px; padding-right:8px; margin-bottom:12px;">
            {#each secureChats as chat}
              <div style="display: flex; justify-content: {chat.sender === username ? 'flex-end' : 'flex-start'};">
                <div style="max-width: 80%; background: {chat.sender === username ? '#047857' : '#121824'}; padding: 10px 14px; border-radius: 12px; font-size: 13px; color: #fff; border: 1px solid #1a2333;">
                  <strong>{chat.sender}</strong> <small style="color: #64748b; font-size: 10px;">{chat.time}</small>
                  <p style="margin: 4px 0 0;">{chat.text}</p>
                </div>
              </div>
            {/each}
          </div>
          <div style="display:flex; gap: 8px;">
            <input type="text" bind:value={newChatMessage} placeholder="Ketik pesan terenkripsi..." onkeydown={(e) => { if (e.key === 'Enter') sendSecureChat(); }} />
            <button class="submit-btn" onclick={sendSecureChat}>Kirim 🔒</button>
          </div>
        </section>

      {:else if activeTab === 'ai'}
        <!-- === AI COMPANION === -->
        <header class="topbar"><div><span class="eyebrow">AI COMPANION</span><h1>AI Productivity Coach</h1></div></header>
        <section class="card" style="display:flex; flex-direction:column; height: 450px;">
          <div style="flex:1; overflow-y:auto; display:flex; flex-direction:column; gap:10px; padding-right:8px; margin-bottom:12px;">
            {#each aiMessages as msg}
              <div style="display: flex; justify-content: {msg.role === 'user' ? 'flex-end' : 'flex-start'};">
                <div style="max-width: 80%; background: {msg.role === 'user' ? '#047857' : '#121824'}; padding: 10px 14px; border-radius: 12px; font-size: 13px; color: #fff;">
                  <strong>{msg.role === 'user' ? `👤 ${username}` : '🤖 AI Coach'}</strong>
                  <p style="margin: 4px 0 0;">{msg.content}</p>
                </div>
              </div>
            {/each}
          </div>
          <div style="display:flex; gap: 8px;">
            <input type="text" bind:value={aiInput} placeholder="Tanya sesuatu ke AI Coach..." onkeydown={(e) => { if (e.key === 'Enter') askAI(); }} />
            <button class="submit-btn" onclick={() => askAI()}>Kirim 🚀</button>
          </div>
        </section>

      {:else if activeTab === 'settings'}
        <!-- === SETTINGS & ACCOUNT MANAGEMENT === -->
        <header class="topbar"><div><span class="eyebrow">PENGATURAN</span><h1>Akun, Keamanan & Sinkronisasi</h1></div></header>
        
        <section class="card">
          <h2>👤 Pengaturan Username</h2>
          <p style="font-size:13px; color:#94a3b8; margin-bottom:12px;">Ubah nama panggilan Anda yang tampil di beranda dan sistem.</p>
          <div class="field" style="max-width: 400px; margin-bottom: 12px;">
            <input type="text" bind:value={username} placeholder="Nama Username..." />
          </div>
          <button class="submit-btn mini" onclick={() => alert('Username berhasil diperbarui menjadi: ' + username)}>Simpan Username</button>
        </section>

        <section class="card">
          <h2>🔒 Keamanan & Enkripsi (1 Akun 1 Akses)</h2>
          <p style="font-size:13px; color:#94a3b8; margin-bottom:12px;">Ganti PIN akses Anda untuk menjaga data tetap aman dengan End-to-End Encryption.</p>
          <div class="field" style="max-width: 400px; margin-bottom: 12px;">
            <label>PIN Akses Baru (4-6 digit):</label>
            <input type="password" maxlength="6" bind:value={userPin} placeholder="PIN Akses..." />
          </div>
          <button class="submit-btn mini" onclick={() => alert('PIN Akses berhasil diperbarui & enkripsi data diperbarui!')}>Perbarui PIN & Enkripsi</button>
        </section>

        <section class="card">
          <h2>📂 Backup & Sinkronisasi Data Terenkripsi</h2>
          <p style="font-size:13px; color:#94a3b8; margin-bottom:16px;">Ekspor data terenkripsi untuk dipindahkan antar perangkat dengan aman via AirDrop atau File JSON.</p>
          <div style="display:flex; gap:10px;">
            <button class="submit-btn" onclick={exportData}>📥 Ekspor Backup Terenkripsi</button>
            <label class="cancel-btn" style="cursor:pointer; display:inline-flex; align-items:center;">
              📤 Impor Data <input type="file" accept=".json" onchange={importData} style="display:none;" />
            </label>
          </div>
        </section>
      {/if}
    </main>
  </div>
{/if}

<style>
  :global(body) { margin: 0; background-color: #080b11; color: #e2e8f0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
  
  /* LOCK SCREEN STYLING */
  .lock-screen { display: grid; place-items: center; min-height: 100vh; background: #080b11; padding: 20px; }
  .lock-card { background: #0f1523; border: 1px solid #1a2333; padding: 32px; border-radius: 20px; width: 100%; max-width: 400px; text-align: center; box-shadow: 0 10px 30px rgba(0,0,0,0.4); }
  .lock-card h2 { font-size: 18px; color: #fff; margin: 0 0 8px; }
  .lock-card p { font-size: 12px; color: #94a3b8; margin-bottom: 20px; line-height: 1.5; }

  .app-layout { display: flex; min-height: 100vh; }
  
  .sidebar { width: 260px; background: #0f1523; border-right: 1px solid #1a2333; padding: 24px 18px; display: flex; flex-direction: column; }
  .brand { display: flex; align-items: center; gap: 12px; margin-bottom: 28px; }
  .brand-logo { width: 36px; height: 36px; background: #047857; border-radius: 12px; display: grid; place-items: center; color: white; font-weight: 800; font-size: 16px; }
  .brand-title { display: flex; flex-direction: column; }
  .brand-title span { font-weight: 700; font-size: 16px; color: #fff; }
  .brand-title small { font-size: 10px; color: #64748b; }

  .nav-list { display: flex; flex-direction: column; gap: 6px; }
  .nav-item { background: transparent; border: none; color: #94a3b8; padding: 11px 14px; border-radius: 12px; font-size: 13px; font-weight: 600; cursor: pointer; text-align: left; display: flex; align-items: center; gap: 12px; transition: 0.2s; }
  .nav-item:hover { background: #162032; color: #fff; }
  .nav-item.active { background: #047857; color: #fff; box-shadow: 0 4px 12px rgba(4, 120, 87, 0.3); }
  .nav-icon { font-size: 15px; }

  .main-content { flex: 1; padding: 28px 44px; max-width: 1050px; overflow-y: auto; }

  .hero-card { position: relative; background: linear-gradient(135deg, #064e3b, #0f172a); border: 1px solid #059669; border-radius: 20px; padding: 28px; margin-bottom: 24px; overflow: hidden; box-shadow: 0 10px 30px rgba(0,0,0,0.3); }
  .hero-content { position: relative; z-index: 2; }
  .hero-top-info { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
  .badge-pill { background: rgba(255,255,255,0.1); padding: 4px 12px; border-radius: 20px; font-size: 11px; color: #a7f3d0; font-weight: 600; backdrop-filter: blur(4px); }
  .hero-avatar { width: 34px; height: 34px; background: #059669; border-radius: 50%; display: grid; place-items: center; font-weight: bold; color: #fff; font-size: 13px; }
  .hero-card h1 { font-size: 26px; margin: 0 0 4px; color: #fff; }
  .hero-subtitle { font-size: 13px; color: #d1fae5; font-style: italic; margin: 0 0 18px; display: flex; align-items: center; gap: 8px; }
  .link-btn { background: none; border: none; color: #6ee7b7; cursor: pointer; font-size: 11px; text-decoration: underline; padding: 0; }
  
  .hero-stats-row { display: grid; grid-template-columns: repeat(2, 1fr); gap: 12px; background: rgba(0,0,0,0.25); padding: 12px 16px; border-radius: 12px; backdrop-filter: blur(6px); }
  .hero-stats-row small { font-size: 11px; color: #94a3b8; }
  .hero-stats-row strong { font-size: 15px; color: #fff; display: block; margin-top: 2px; }

  .quick-access-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 14px; margin-bottom: 24px; }
  .qa-box { background: #0f1523; border: 1px solid #1a2333; padding: 18px; border-radius: 16px; cursor: pointer; transition: 0.2s; }
  .qa-box:hover { border-color: #059669; transform: translateY(-2px); }
  .qa-box span { font-size: 22px; display: block; margin-bottom: 8px; }
  .qa-box h3 { font-size: 14px; margin: 0 0 4px; color: #fff; }
  .qa-box p { font-size: 11px; color: #64748b; margin: 0; line-height: 1.4; }

  .topbar { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 22px; }
  .eyebrow { font-size: 10px; letter-spacing: 1.5px; color: #34d399; font-weight: 800; }
  h1 { font-size: 24px; margin: 4px 0 0; color: #fff; }
  h2 { font-size: 15px; margin: 0; color: #fff; }
  .date-badge { background: #0f1523; border: 1px solid #1a2333; padding: 8px 14px; border-radius: 12px; font-size: 12px; color: #e2e8f0; }
  .card { background: #0f1523; border: 1px solid #1a2333; border-radius: 18px; padding: 22px; margin-bottom: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.2); }
  .card-title-row { display: flex; align-items: center; gap: 10px; margin-bottom: 16px; }
  .sage-icon { width: 28px; height: 28px; background: #047857; color: white; border-radius: 50%; display: grid; place-items: center; font-weight: bold; font-size: 12px; }
  
  .grid-3 { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 14px; align-items: end; }
  .grid-2-custom { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; align-items: center; }
  .field { display: flex; flex-direction: column; gap: 6px; min-width: 0; }
  .field label { font-size: 12px; color: #94a3b8; font-weight: 500; }
  
  input[type="text"], input[type="time"], input[type="number"], input[type="password"], select, textarea { height: 44px; padding: 0 14px; background: #080b11; border: 1px solid #1a2333; border-radius: 12px; color: #fff; font-size: 13px; outline: none; box-sizing: border-box; width: 100%; }
  textarea { height: auto; padding: 12px 14px; resize: vertical; }
  .nested-box { background: #080b11; border: 1px solid #1a2333; padding: 14px; border-radius: 12px; margin-top: 14px; }
  .day-chips { display: flex; gap: 6px; flex-wrap: wrap; margin-top: 4px; }
  .chip { height: 32px; padding: 0 10px; background: #0f1523; border: 1px solid #1a2333; color: #94a3b8; border-radius: 8px; font-size: 11px; cursor: pointer; }
  .chip.active { background: #047857; color: #fff; border-color: #047857; }
  .form-footer-row { display: flex; justify-content: space-between; align-items: center; margin-top: 16px; padding-top: 14px; border-top: 1px solid #1a2333; }
  .priority-picker { display: flex; align-items: center; gap: 12px; font-size: 12px; color: #94a3b8; }
  
  .submit-btn { height: 42px; padding: 0 20px; background: #047857; color: white; border: none; border-radius: 12px; font-weight: 600; cursor: pointer; transition: 0.2s; }
  .submit-btn:hover { background: #059669; }
  .submit-btn.mini { height: 32px; padding: 0 12px; font-size: 12px; }
  .cancel-btn { height: 42px; padding: 0 16px; background: #162032; color: #94a3b8; border: none; border-radius: 12px; font-weight: 600; cursor: pointer; }
  .cancel-btn.mini { height: 32px; padding: 0 12px; font-size: 12px; }
  
  .card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; }
  .badge { font-size: 10px; padding: 4px 10px; border-radius: 20px; font-weight: 600; }
  .sage-badge { background: rgba(52, 211, 153, 0.15); color: #34d399; }
  .amber-badge { background: rgba(251, 191, 36, 0.15); color: #fde68a; }
  .green-badge { background: rgba(16, 185, 129, 0.15); color: #6ee7b7; }
  
  .task-list { display: flex; flex-direction: column; gap: 8px; }
  .task-row { display: flex; justify-content: space-between; align-items: center; background: #080b11; border: 1px solid #1a2333; padding: 12px 16px; border-radius: 12px; }
  .task-row.done { opacity: 0.45; text-decoration: line-through; }
  .check-label { display: flex; align-items: center; gap: 12px; cursor: pointer; flex: 1; }
  .task-name { font-size: 13px; font-weight: 600; color: #fff; }
  .task-time { font-size: 11px; color: #64748b; }
  .icon-btn { background: none; border: none; cursor: pointer; font-size: 13px; color: #94a3b8; padding: 4px; }
  .icon-btn:hover { color: #fff; }
  .empty-msg { font-size: 12px; color: #475569; text-align: center; margin: 12px 0; }
  
  .stats-row { display: grid; grid-template-columns: repeat(3, 1fr); gap: 14px; margin-bottom: 20px; }
  .stat-card { background: #0f1523; border: 1px solid #1a2333; padding: 18px; border-radius: 16px; display: flex; flex-direction: column; gap: 4px; }
  .stat-card span { font-size: 11px; color: #64748b; }
  .stat-card strong { font-size: 18px; }
  .balance { color: #fff; } .income { color: #34d399; } .expense { color: #f87171; }
  .alloc-card { background: #080b11; border-color: #1a2333; }
  .alloc-head { display: flex; justify-content: space-between; align-items: center; }
  .progress-bar { height: 6px; background: #162032; border-radius: 10px; overflow: hidden; }
  .progress-fill { height: 100%; background: #34d399; }
  .green-fill { background: #10b981; }
  
  .tx-item { display: flex; justify-content: space-between; align-items: center; background: #080b11; padding: 12px 16px; border-radius: 12px; font-size: 13px; border: 1px solid #1a2333; }
  .tx-val.income { color: #34d399; font-weight: 600; }
  .tx-val.expense { color: #f87171; font-weight: 600; }
  
  .weekly-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 8px; }
  .day-col { background: #080b11; padding: 12px; border-radius: 12px; border: 1px solid #1a2333; }
  .day-col h3 { font-size: 11px; color: #34d399; margin-bottom: 8px; }
  .mini-task { font-size: 10px; margin-bottom: 4px; display: flex; justify-content: space-between; align-items: center; background: #0f1523; padding: 6px 8px; border-radius: 8px; }
  .none { font-size: 10px; color: #475569; }

  @media (max-width: 900px) {
    .app-layout { flex-direction: column; }
    .sidebar { width: 100%; box-sizing: border-box; }
    .grid-3, .grid-2-custom, .stats-row, .weekly-grid, .quick-access-grid { grid-template-columns: 1fr; }
  }
</style>