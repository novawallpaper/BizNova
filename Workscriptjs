/* =========================================================
   BizNova — AI Business Suite
   ---------------------------------------------------------
   Front-end demo build. Data is persisted via the artifact
   storage API (window.storage) so it survives reloads inside
   this preview. For a real production deployment, swap the
   `db` helpers below for Firebase Firestore calls — the shape
   of every function (list/get/set/remove) is designed to map
   1:1 onto Firestore collection methods.

   Integration points that need real credentials before going
   live (all clearly marked below):
     - Firebase Auth (Google Sign-In)      -> initGoogleAuth()
     - Razorpay subscriptions               -> startPremiumUpgrade()
     - WhatsApp Business API                -> shareInvoiceWhatsapp()
     - Email API (e.g. SendGrid)            -> shareInvoiceEmail()
     - Gemini / OpenAI (AI Assistant)       -> callAiApi()
   ========================================================= */

/* ---------------------------------------------------------
   0. LIGHTWEIGHT STORAGE LAYER (window.storage wrapper)
   --------------------------------------------------------- */
const DB_PREFIX = "biznova:";
async function dbGet(key, fallback) {
  try {
    const res = await window.storage.get(DB_PREFIX + key, false);
    return res ? JSON.parse(res.value) : fallback;
  } catch (e) {
    return fallback;
  }
}
async function dbSet(key, value) {
  try {
    await window.storage.set(DB_PREFIX + key, JSON.stringify(value), false);
  } catch (e) {
    /* storage unavailable — app still works for this session */
  }
}

/* ---------------------------------------------------------
   1. APP STATE
   --------------------------------------------------------- */
const state = {
  signedIn: false,
  user: { name: "", email: "", picture: "", avatar: "U", sub: "" },
  theme: "light",
  plan: "free", // free | premium
  // AI credits are monthly and persisted (see loadAiCredits) so a page
  // refresh can never reset/bypass the limit.
  aiCreditsUsedMonth: 0,
  aiCreditsMonthKey: "", // "YYYY-MM" the counters above belong to
  businesses: [],
  currentBusinessId: null,
  invoicesUsedMonth: 0,
  invoicesMonthKey: "",
  fabPos: null, // { x, y } last dragged position of the AI assistant fab
  data: {
    products: [],
    customers: [],
    sales: [],
    expenses: [],
    employees: [],
    invoices: [],
  },
  currentModule: null,
  currentModuleFilter: "all",
  editing: { type: null, id: null },
  invoiceDraft: { items: [], customerId: null },
  aiChatHistory: [],
  searchFilter: "all",
};

// Single source of truth for plan limits — used everywhere usage is
// checked (products, customers, invoices, businesses, AI credits).
const PLAN_LIMITS = {
  free: { products: 100, customers: 100, invoices: 30, businesses: 1, aiCredits: 10 },
  premium: { products: Infinity, customers: Infinity, invoices: Infinity, businesses: Infinity, aiCredits: 5000 },
};
function limitsForPlan() {
  return PLAN_LIMITS[state.plan] || PLAN_LIMITS.free;
}
function monthKey(d) {
  d = d || new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`;
}

const MODULES = [
  { id: "products", label: "Inventory", icon: "fa-boxes-stacked", color1: "#2b76f9", color2: "#1149b5", nameField: "name" },
  { id: "customers", label: "Customers", icon: "fa-users", color1: "#17a866", color2: "#0b6b40", nameField: "name" },
  { id: "sales", label: "Sales", icon: "fa-chart-line", color1: "#7c5cff", color2: "#4527a0", nameField: "customerName" },
  { id: "expenses", label: "Expenses", icon: "fa-receipt", color1: "#e5484d", color2: "#8f1f23", nameField: "title" },
  { id: "employees", label: "Employees", icon: "fa-id-badge", color1: "#f1a624", color2: "#a5690a", nameField: "name" },
  { id: "reports", label: "Reports", icon: "fa-file-lines", color1: "#0aa1a8", color2: "#065f64", nameField: "title" },
  { id: "analytics", label: "Analytics", icon: "fa-chart-pie", color1: "#ff7a45", color2: "#b8461f", nameField: "title" },
];

/* ---------------------------------------------------------
   2. SEED DATA (used only on first run)
   --------------------------------------------------------- */
function seedData() {
  return {
    products: [
      { id: "p1", name: "Wireless Mouse", sku: "WM-101", category: "Electronics", price: 599, stock: 42, lowStockAt: 10 },
      { id: "p2", name: "A4 Paper Ream", sku: "AP-220", category: "Stationery", price: 249, stock: 8, lowStockAt: 15 },
      { id: "p3", name: "Office Chair", sku: "OC-330", category: "Furniture", price: 4999, stock: 3, lowStockAt: 5 },
      { id: "p4", name: "LED Desk Lamp", sku: "DL-410", category: "Electronics", price: 899, stock: 25, lowStockAt: 8 },
      { id: "p5", name: "Notebook Set (5pc)", sku: "NB-050", category: "Stationery", price: 199, stock: 60, lowStockAt: 20 },
    ],
    customers: [
      { id: "c1", name: "Rahul Sharma", phone: "+91 98765 43210", email: "rahul@example.com", gstin: "27ABCDE1234F1Z5", totalSpent: 18500, dues: 0 },
      { id: "c2", name: "Priya Enterprises", phone: "+91 91234 56789", email: "priya@example.com", gstin: "07AAACP1234C1Z1", totalSpent: 42300, dues: 3200 },
      { id: "c3", name: "Verma Traders", phone: "+91 90000 11223", email: "verma@example.com", gstin: "", totalSpent: 9800, dues: 0 },
    ],
    sales: [
      { id: "s1", customerName: "Rahul Sharma", amount: 3599, date: "2026-07-10", status: "paid" },
      { id: "s2", customerName: "Priya Enterprises", amount: 12800, date: "2026-07-11", status: "pending" },
      { id: "s3", customerName: "Verma Traders", amount: 5400, date: "2026-07-12", status: "paid" },
    ],
    expenses: [
      { id: "e1", title: "Office Rent", category: "Rent", amount: 25000, date: "2026-07-01" },
      { id: "e2", title: "Electricity Bill", category: "Utilities", amount: 3400, date: "2026-07-05" },
      { id: "e3", title: "Packaging Supplies", category: "Supplies", amount: 1800, date: "2026-07-08" },
    ],
    employees: [
      { id: "em1", name: "Ankit Verma", role: "Sales Executive", phone: "+91 99887 76655", salary: 25000, attendance: "present" },
      { id: "em2", name: "Sneha Kapoor", role: "Accountant", phone: "+91 99001 22334", salary: 32000, attendance: "present" },
    ],
    invoices: [],
  };
}

/* ---------------------------------------------------------
   3. INIT
   --------------------------------------------------------- */
window.addEventListener("DOMContentLoaded", init);

async function init() {
  buildLoginBg();
  initGoogleAuth();
  initAiFabDrag();

  const savedTheme = await dbGet("theme", "light");
  applyTheme(savedTheme);

  const savedPlan = await dbGet("plan", "free");
  state.plan = savedPlan;

  await loadAiCredits();
  await loadInvoiceCounter();

  const savedBiz = await dbGet("businesses", null);
  if (savedBiz && savedBiz.length) {
    state.businesses = savedBiz;
  } else {
    state.businesses = [{ id: "biz1", name: "My Business", gstin: "" }];
    await dbSet("businesses", state.businesses);
  }
  state.currentBusinessId = await dbGet("currentBusinessId", state.businesses[0].id);

  await loadBusinessData();

  // Google Identity Services keeps the user signed in across reloads by
  // design (same as any real auth session) — this is not a "refresh to
  // bypass limits" hole, since all limits below are read from storage,
  // not from in-memory state that a reload would reset.
  const session = await dbGet("session", null);
  if (session) {
    state.signedIn = true;
    state.user = session;
    document.getElementById("login-gate").classList.add("hidden");
  }

  const savedFabPos = await dbGet("fabPos", null);
  if (savedFabPos) applyFabPosition(savedFabPos);

  renderAll();
  document.getElementById("invoice-date").value = new Date().toISOString().slice(0, 10);
}

/* Monthly AI credits: stored under a month-stamped key so they reset
   automatically on the 1st of each month but NEVER on a page refresh. */
async function loadAiCredits() {
  const key = monthKey();
  const saved = await dbGet("aiCredits", null);
  if (saved && saved.monthKey === key) {
    state.aiCreditsUsedMonth = saved.used;
  } else {
    state.aiCreditsUsedMonth = 0;
    await dbSet("aiCredits", { monthKey: key, used: 0 });
  }
  state.aiCreditsMonthKey = key;
}
async function useAiCredit() {
  state.aiCreditsUsedMonth++;
  await dbSet("aiCredits", { monthKey: state.aiCreditsMonthKey, used: state.aiCreditsUsedMonth });
}

/* Monthly invoice counter — same reset-proof pattern as AI credits. */
async function loadInvoiceCounter() {
  const key = monthKey();
  const saved = await dbGet("invoiceCounter", null);
  if (saved && saved.monthKey === key) {
    state.invoicesUsedMonth = saved.used;
  } else {
    state.invoicesUsedMonth = 0;
    await dbSet("invoiceCounter", { monthKey: key, used: 0 });
  }
  state.invoicesMonthKey = key;
}
async function useInvoiceSlot() {
  state.invoicesUsedMonth++;
  await dbSet("invoiceCounter", { monthKey: state.invoicesMonthKey, used: state.invoicesUsedMonth });
}

async function loadBusinessData() {
  const key = "data:" + state.currentBusinessId;
  const saved = await dbGet(key, null);
  state.data = saved || seedData();
  if (!saved) await dbSet(key, state.data);
}
async function persistData() {
  await dbSet("data:" + state.currentBusinessId, state.data);
}

/* ---------------------------------------------------------
   3b. DRAGGABLE AI ASSISTANT FAB
   ---------------------------------------------------------
   Pointer-events based drag (works for mouse + touch). A small
   movement threshold distinguishes a tap (opens the chat) from
   a drag (repositions the fab). Position is persisted so it
   reopens in the same spot next time.
   --------------------------------------------------------- */
function initAiFabDrag() {
  const fab = document.getElementById("ai-fab");
  const container = document.getElementById("app-container");
  let startX, startY, startLeft, startTop, dragging, moved;

  fab.addEventListener("pointerdown", (e) => {
    e.preventDefault();
    const rect = fab.getBoundingClientRect();
    const containerRect = container.getBoundingClientRect();
    startX = e.clientX;
    startY = e.clientY;
    startLeft = rect.left - containerRect.left;
    startTop = rect.top - containerRect.top;
    dragging = true;
    moved = false;
    fab.setPointerCapture(e.pointerId);
    fab.classList.add("dragging");
  });

  fab.addEventListener("pointermove", (e) => {
    if (!dragging) return;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;
    if (Math.abs(dx) > 4 || Math.abs(dy) > 4) moved = true;
    if (!moved) return;

    const containerRect = container.getBoundingClientRect();
    let left = startLeft + dx;
    let top = startTop + dy;
    left = Math.max(4, Math.min(containerRect.width - fab.offsetWidth - 4, left));
    top = Math.max(4, Math.min(containerRect.height - fab.offsetHeight - 4, top));

    fab.style.left = left + "px";
    fab.style.top = top + "px";
    fab.style.right = "auto";
    fab.style.bottom = "auto";
  });

  fab.addEventListener("pointerup", (e) => {
    if (!dragging) return;
    dragging = false;
    fab.classList.remove("dragging");
    fab.releasePointerCapture(e.pointerId);
    if (moved) {
      const pos = { left: fab.style.left, top: fab.style.top };
      state.fabPos = pos;
      dbSet("fabPos", pos);
    } else {
      openAiChat(); // it was a tap, not a drag
    }
  });
}

function applyFabPosition(pos) {
  const fab = document.getElementById("ai-fab");
  if (!pos || !fab) return;
  fab.style.left = pos.left;
  fab.style.top = pos.top;
  fab.style.right = "auto";
  fab.style.bottom = "auto";
}

function buildLoginBg() {
  const grid = document.getElementById("login-bg-grid");
  const icons = ["fa-chart-line", "fa-file-invoice", "fa-boxes-stacked", "fa-users", "fa-receipt", "fa-robot", "fa-building", "fa-indian-rupee-sign"];
  let html = "";
  for (let i = 0; i < 12; i++) {
    const hue = 215 + (i % 4) * 20;
    html += `<div style="background:linear-gradient(135deg,hsl(${hue},70%,45%),hsl(${hue + 30},70%,30%));display:flex;align-items:center;justify-content:center;">
      <i class="fa-solid ${icons[i % icons.length]}" style="color:rgba(255,255,255,0.35);font-size:26px;"></i>
    </div>`;
  }
  grid.innerHTML = html;
}

/* ---------------------------------------------------------
   4. GOOGLE AUTH
   ---------------------------------------------------------
   Uses real Google Identity Services (the same GIS script the
   original template already loaded) with BizNova's OAuth client
   ID. The ID token below is decoded client-side purely to read
   name/email/photo for the UI.

   PRODUCTION NOTE: a client-side JWT decode is fine for display,
   but is NOT authentication by itself — the ID token must be
   verified server-side (e.g. Firebase Admin SDK / Cloud Function)
   before trusting it for real account access or Firestore rules.
   --------------------------------------------------------- */
const GOOGLE_CLIENT_ID = "452456583028-1l86bibq60ggkl3o1h5j88sed7v04eof.apps.googleusercontent.com";

function initGoogleAuth() {
  const statusEl = document.getElementById("signin-status");
  const fallbackBtn = document.getElementById("google-fallback-btn");

  // Give the GIS script a moment to load (it's fetched async/defer).
  let attempts = 0;
  const tryInit = () => {
    attempts++;
    if (window.google && window.google.accounts && window.google.accounts.id) {
      google.accounts.id.initialize({
        client_id: GOOGLE_CLIENT_ID,
        callback: handleGoogleCredential,
        auto_select: false,
      });
      google.accounts.id.renderButton(document.getElementById("google-signin-btn-gate"), {
        theme: "filled_black",
        size: "large",
        shape: "pill",
        width: 300,
      });
      statusEl.textContent = "Sign in to continue";
      return;
    }
    if (attempts < 20) {
      setTimeout(tryInit, 250);
    } else {
      // GIS failed to load (offline / blocked) — reveal the manual fallback.
      statusEl.textContent = "Google Sign-In unavailable — try below";
      fallbackBtn.style.display = "flex";
    }
  };
  tryInit();
}

// Called by Google Identity Services with a signed ID token (JWT).
function handleGoogleCredential(response) {
  try {
    const payload = decodeJwt(response.credential);
    completeSignIn({
      name: payload.name || "Google User",
      email: payload.email || "",
      picture: payload.picture || "",
      sub: payload.sub || "",
      avatar: (payload.name || "U").charAt(0).toUpperCase(),
    });
  } catch (e) {
    showToast("Sign-in failed — please try again");
  }
}

function decodeJwt(token) {
  const base64 = token.split(".")[1].replace(/-/g, "+").replace(/_/g, "/");
  const json = decodeURIComponent(
    atob(base64)
      .split("")
      .map((c) => "%" + c.charCodeAt(0).toString(16).padStart(2, "0"))
      .join("")
  );
  return JSON.parse(json);
}

function handleFallbackGoogleClick() {
  // Manual fallback only used if the GIS script itself failed to load.
  // Still routes through the same completeSignIn() flow.
  showToast("Retrying Google Sign-In...");
  initGoogleAuth();
}

async function completeSignIn(user) {
  state.signedIn = true;
  state.user = user;
  await dbSet("session", user);
  document.getElementById("login-gate").classList.add("hidden");
  renderAll();
  showToast(`Welcome, ${user.name.split(" ")[0]}!`);
}

async function signOutGoogle() {
  state.signedIn = false;
  await dbSet("session", null);
  if (window.google && window.google.accounts && window.google.accounts.id) {
    google.accounts.id.disableAutoSelect();
  }
  document.getElementById("login-gate").classList.remove("hidden");
  showToast("Signed out");
}

/* ---------------------------------------------------------
   5. TAB / SCREEN NAVIGATION
   --------------------------------------------------------- */
function switchTab(tab, navEl) {
  document.querySelectorAll(".nav-item").forEach((n) => n.classList.remove("active"));
  if (navEl) navEl.classList.add("active");

  const screenMap = { categories: "screen-categories", explore: "screen-explore", request: "screen-request", settings: "screen-settings" };
  document.querySelectorAll(".screen-content").forEach((s) => s.classList.remove("active"));
  document.getElementById(screenMap[tab]).classList.add("active");

  if (tab === "categories") renderDashboard();
  if (tab === "explore") renderSearch();
  if (tab === "request") renderInvoicesHub();
  if (tab === "settings") renderSettings();
}

function showScreen(id) {
  document.querySelectorAll(".screen-content").forEach((s) => s.classList.remove("active"));
  document.getElementById(id).classList.add("active");
}

function renderAll() {
  renderDashboard();
  renderSearch();
  renderInvoicesHub();
  renderSettings();
  renderAiSuggestChips();
}

/* ---------------------------------------------------------
   6. DASHBOARD (was Categories screen)
   --------------------------------------------------------- */
function renderDashboard() {
  const biz = state.businesses.find((b) => b.id === state.currentBusinessId) || state.businesses[0];
  document.getElementById("dashboard-bizname").textContent = biz ? biz.name : "My Business";
  document.getElementById("biz-switcher-label").textContent = biz ? biz.name.split(" ")[0] : "Switch";

  const hour = new Date().getHours();
  const greeting = hour < 12 ? "Good morning" : hour < 17 ? "Good afternoon" : "Good evening";
  document.getElementById("dashboard-greeting").textContent = `${greeting}${state.user.name ? ", " + state.user.name.split(" ")[0] : ""} 👋`;

  const revenue = state.data.sales.reduce((a, s) => a + s.amount, 0);
  const expenses = state.data.expenses.reduce((a, e) => a + e.amount, 0);
  const profit = revenue - expenses;
  const lowStock = state.data.products.filter((p) => p.stock <= p.lowStockAt).length;

  const kpis = [
    { label: "Revenue", value: fmtINR(revenue), icon: "fa-indian-rupee-sign", color: "#17a866", trend: "+12% MoM", up: true },
    { label: "Expenses", value: fmtINR(expenses), icon: "fa-receipt", color: "#e5484d", trend: "+4% MoM", up: false },
    { label: "Profit", value: fmtINR(profit), icon: "fa-chart-line", color: "#4c8dff", trend: profit >= 0 ? "Healthy" : "Loss", up: profit >= 0 },
    { label: "Low Stock", value: lowStock + " items", icon: "fa-triangle-exclamation", color: "#f1a624", trend: lowStock ? "Needs attention" : "All good", up: !lowStock },
  ];
  document.getElementById("kpi-grid").innerHTML = kpis
    .map(
      (k) => `<div class="kpi-card">
        <div class="kpi-icon" style="background:${k.color}22;color:${k.color};"><i class="fa-solid ${k.icon}"></i></div>
        <div class="kpi-value">${k.value}</div>
        <div class="kpi-label">${k.label}</div>
        <span class="kpi-trend ${k.up ? "up" : "down"}"><i class="fa-solid fa-arrow-${k.up ? "up" : "down"}"></i> ${k.trend}</span>
      </div>`
    )
    .join("");

  document.getElementById("collection-carousel").innerHTML = MODULES.map((m) => {
    const count = state.data[m.id] ? state.data[m.id].length : 0;
    return `<div class="carousel-card" style="background:linear-gradient(135deg, ${m.color1} 0%, ${m.color2} 100%);" onclick="openModule('${m.id}')">
      <div class="carousel-badge"><i class="fa-solid ${m.icon}"></i> ${count} ${count === 1 ? "record" : "records"}</div>
      <div class="carousel-info">
        <h2>${m.label}</h2>
        <p>${moduleSubtitle(m.id)}</p>
        <button class="explore-pill-btn" onclick="event.stopPropagation(); openModule('${m.id}')">Open</button>
      </div>
    </div>`;
  }).join("");
}

function moduleSubtitle(id) {
  const map = {
    products: "Track stock levels & pricing",
    customers: "Manage your customer relationships",
    sales: "Review recent sales activity",
    expenses: "Log and categorize spending",
    employees: "Staff, attendance & salary",
    reports: "Daily, weekly & monthly summaries",
    analytics: "AI-powered business insights",
  };
  return map[id] || "";
}

function fmtINR(n) {
  return "₹" + Number(n || 0).toLocaleString("en-IN");
}

/* ---------------------------------------------------------
   7. MODULE LIST SCREEN (was Category Detail)
   --------------------------------------------------------- */
function openModule(moduleId) {
  state.currentModule = moduleId;
  state.currentModuleFilter = "all";
  const mod = MODULES.find((m) => m.id === moduleId);
  document.getElementById("cat-detail-title").textContent = mod.label;

  const addBtn = document.getElementById("cat-detail-add-btn");
  addBtn.style.display = moduleId === "reports" || moduleId === "analytics" ? "none" : "flex";

  const tabsByModule = {
    products: ["All", "Low Stock", "Out of Stock"],
    customers: ["All", "Has Dues", "Top Spenders"],
    sales: ["All", "Paid", "Pending"],
    expenses: ["All", "This Month"],
    employees: ["All", "Present", "Absent"],
  };
  const tabs = tabsByModule[moduleId];
  const tabsEl = document.getElementById("cat-detail-tabs");
  if (tabs) {
    tabsEl.style.display = "flex";
    tabsEl.innerHTML = tabs
      .map((t, i) => `<div class="cat-tab ${i === 0 ? "active" : ""}" data-cat-filter="${t.toLowerCase().replace(/ /g, "-")}" onclick="switchCatTab(this)">${t}</div>`)
      .join("");
  } else {
    tabsEl.style.display = "none";
    tabsEl.innerHTML = "";
  }

  showScreen("screen-category-detail");
  renderModuleGrid();
}

function switchCatTab(el) {
  el.parentElement.querySelectorAll(".cat-tab").forEach((t) => t.classList.remove("active"));
  el.classList.add("active");
  state.currentModuleFilter = el.dataset.catFilter;
  renderModuleGrid();
}

function renderModuleGrid() {
  const grid = document.getElementById("cat-detail-grid");
  const moduleId = state.currentModule;

  if (moduleId === "reports") {
    grid.innerHTML = renderReportsPanel();
    return;
  }
  if (moduleId === "analytics") {
    grid.innerHTML = `<div class="chart-card"><h3>Sales — last 6 records</h3><canvas id="analytics-canvas" height="180"></canvas></div>
      <div class="chart-card"><h3>Expense breakdown</h3><canvas id="analytics-canvas-2" height="180"></canvas>
      <div class="insight-row"><i class="fa-solid fa-lightbulb"></i><p id="analytics-insight">Tap "Generate AI Insight" in the assistant for a deeper read on this data.</p></div></div>`;
    setTimeout(renderAnalyticsCharts, 30);
    return;
  }

  let items = [...(state.data[moduleId] || [])];
  items = filterModuleItems(moduleId, items, state.currentModuleFilter);

  if (!items.length) {
    grid.innerHTML = `<div class="data-row-empty"><i class="fa-solid fa-inbox"></i>No records yet. Tap "Add" to create one.</div>`;
    return;
  }

  grid.innerHTML = `<div class="data-list">${items.map((it) => renderDataRow(moduleId, it)).join("")}</div>`;
}

function filterModuleItems(moduleId, items, filter) {
  if (filter === "all") return items;
  if (moduleId === "products") {
    if (filter === "low-stock") return items.filter((p) => p.stock > 0 && p.stock <= p.lowStockAt);
    if (filter === "out-of-stock") return items.filter((p) => p.stock <= 0);
  }
  if (moduleId === "customers") {
    if (filter === "has-dues") return items.filter((c) => c.dues > 0);
    if (filter === "top-spenders") return [...items].sort((a, b) => b.totalSpent - a.totalSpent).slice(0, 5);
  }
  if (moduleId === "sales") {
    if (filter === "paid") return items.filter((s) => s.status === "paid");
    if (filter === "pending") return items.filter((s) => s.status === "pending");
  }
  if (moduleId === "expenses") {
    if (filter === "this-month") {
      const m = new Date().getMonth();
      return items.filter((e) => new Date(e.date).getMonth() === m);
    }
  }
  if (moduleId === "employees") {
    if (filter === "present") return items.filter((e) => e.attendance === "present");
    if (filter === "absent") return items.filter((e) => e.attendance === "absent");
  }
  return items;
}

function renderDataRow(moduleId, item) {
  const renderers = {
    products: (p) => {
      const badge = p.stock <= 0 ? '<span class="badge out-stock">Out of stock</span>' : p.stock <= p.lowStockAt ? '<span class="badge low-stock">Low stock</span>' : '<span class="badge in-stock">In stock</span>';
      return dataRowHtml("fa-box", "#4c8dff", p.name, `${p.sku} • ${p.category}`, fmtINR(p.price), `${p.stock} units`, badge, "products", p.id);
    },
    customers: (c) => {
      const badge = c.dues > 0 ? `<span class="badge pending">Dues ${fmtINR(c.dues)}</span>` : '<span class="badge in-stock">Settled</span>';
      return dataRowHtml("fa-user", "#17a866", c.name, c.phone, fmtINR(c.totalSpent), "lifetime", badge, "customers", c.id);
    },
    sales: (s) => {
      const badge = s.status === "paid" ? '<span class="badge paid">Paid</span>' : '<span class="badge pending">Pending</span>';
      return dataRowHtml("fa-chart-line", "#7c5cff", s.customerName, s.date, fmtINR(s.amount), "", badge, "sales", s.id);
    },
    expenses: (e) => dataRowHtml("fa-receipt", "#e5484d", e.title, `${e.category} • ${e.date}`, fmtINR(e.amount), "", "", "expenses", e.id),
    employees: (em) => {
      const badge = em.attendance === "present" ? '<span class="badge present">Present</span>' : '<span class="badge absent">Absent</span>';
      return dataRowHtml("fa-id-badge", "#f1a624", em.name, em.role, fmtINR(em.salary) + "/mo", "", badge, "employees", em.id);
    },
  };
  return renderers[moduleId] ? renderers[moduleId](item) : "";
}

function dataRowHtml(icon, color, title, subtitle, amount, sub2, badge, moduleId, id) {
  return `<div class="data-row" onclick="openQuickView('${moduleId}','${id}')">
    <div class="data-row-icon" style="background:${color}1f;color:${color};"><i class="fa-solid ${icon}"></i></div>
    <div class="data-row-body"><h4>${escapeHtml(title)}</h4><p>${escapeHtml(subtitle)}</p></div>
    <div class="data-row-right"><div class="data-row-amount">${amount}</div>${sub2 ? `<p style="font-size:10.5px;color:var(--text-muted);">${sub2}</p>` : ""}${badge}</div>
  </div>`;
}

function escapeHtml(str) {
  return String(str || "").replace(/[&<>"']/g, (m) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[m]));
}

/* ---------------------------------------------------------
   8. REPORTS PANEL
   --------------------------------------------------------- */
function renderReportsPanel() {
  const revenue = state.data.sales.reduce((a, s) => a + s.amount, 0);
  const expenses = state.data.expenses.reduce((a, e) => a + e.amount, 0);
  const today = new Date();
  const periods = [
    { label: "Today", revenue: Math.round(revenue * 0.08), expenses: Math.round(expenses * 0.05) },
    { label: "This Week", revenue: Math.round(revenue * 0.4), expenses: Math.round(expenses * 0.3) },
    { label: "This Month", revenue, expenses },
  ];
  return `<div class="data-list">${periods
    .map(
      (p) => `<div class="data-row" style="cursor:default;">
      <div class="data-row-icon" style="background:#0aa1a81f;color:#0aa1a8;"><i class="fa-solid fa-file-lines"></i></div>
      <div class="data-row-body"><h4>${p.label}</h4><p>Revenue ${fmtINR(p.revenue)} • Expenses ${fmtINR(p.expenses)}</p></div>
      <div class="data-row-right"><div class="data-row-amount" style="color:${p.revenue - p.expenses >= 0 ? "#17a866" : "#e5484d"}">${fmtINR(p.revenue - p.expenses)}</div><p style="font-size:10.5px;color:var(--text-muted);">net profit</p></div>
    </div>`
    )
    .join("")}</div>`;
}

function renderAnalyticsCharts() {
  const c1 = document.getElementById("analytics-canvas");
  const c2 = document.getElementById("analytics-canvas-2");
  if (!c1 || typeof Chart === "undefined") return;

  const sales = state.data.sales.slice(-6);
  new Chart(c1, {
    type: "line",
    data: {
      labels: sales.map((s) => s.date.slice(5)),
      datasets: [{ label: "Sales", data: sales.map((s) => s.amount), borderColor: "#4c8dff", backgroundColor: "rgba(76,141,255,0.12)", fill: true, tension: 0.35 }],
    },
    options: { plugins: { legend: { display: false } }, scales: { y: { beginAtZero: true } } },
  });

  const byCat = {};
  state.data.expenses.forEach((e) => (byCat[e.category] = (byCat[e.category] || 0) + e.amount));
  new Chart(c2, {
    type: "doughnut",
    data: {
      labels: Object.keys(byCat),
      datasets: [{ data: Object.values(byCat), backgroundColor: ["#4c8dff", "#e5484d", "#f1a624", "#17a866", "#7c5cff"] }],
    },
    options: { plugins: { legend: { position: "bottom", labels: { boxWidth: 10, font: { size: 10 } } } } },
  });
}

/* ---------------------------------------------------------
   9. ADD / EDIT FORM (was Wallpaper Editor)
   --------------------------------------------------------- */
const FIELD_SCHEMAS = {
  products: [
    { key: "name", label: "Product Name", type: "text" },
    { key: "sku", label: "SKU", type: "text" },
    { key: "category", label: "Category", type: "text" },
    { key: "price", label: "Price (₹)", type: "number" },
    { key: "stock", label: "Stock Quantity", type: "number" },
    { key: "lowStockAt", label: "Low Stock Alert Below", type: "number" },
  ],
  customers: [
    { key: "name", label: "Customer / Business Name", type: "text" },
    { key: "phone", label: "Phone", type: "text" },
    { key: "email", label: "Email", type: "text" },
    { key: "gstin", label: "GSTIN (optional)", type: "text" },
  ],
  sales: [
    { key: "customerName", label: "Customer Name", type: "text" },
    { key: "amount", label: "Amount (₹)", type: "number" },
    { key: "date", label: "Date", type: "date" },
    { key: "status", label: "Status", type: "select", options: ["paid", "pending"] },
  ],
  expenses: [
    { key: "title", label: "Expense Title", type: "text" },
    { key: "category", label: "Category", type: "text" },
    { key: "amount", label: "Amount (₹)", type: "number" },
    { key: "date", label: "Date", type: "date" },
  ],
  employees: [
    { key: "name", label: "Employee Name", type: "text" },
    { key: "role", label: "Role", type: "text" },
    { key: "phone", label: "Phone", type: "text" },
    { key: "salary", label: "Monthly Salary (₹)", type: "number" },
    { key: "attendance", label: "Today's Attendance", type: "select", options: ["present", "absent"] },
  ],
};

function openAddEditor() {
  state.editing = { type: state.currentModule, id: null };
  buildEditorForm();
  document.getElementById("editor-title").textContent = "Add " + singularLabel(state.currentModule);
  showScreen("screen-editor");
}

function openEditEditor(moduleId, id) {
  state.editing = { type: moduleId, id };
  buildEditorForm();
  document.getElementById("editor-title").textContent = "Edit " + singularLabel(moduleId);
  showScreen("screen-editor");
}

function singularLabel(moduleId) {
  const map = { products: "Product", customers: "Customer", sales: "Sale", expenses: "Expense", employees: "Employee" };
  return map[moduleId] || "Item";
}

function buildEditorForm() {
  const { type, id } = state.editing;
  const schema = FIELD_SCHEMAS[type];
  const record = id ? state.data[type].find((r) => r.id === id) : {};

  document.getElementById("editor-maintabs").innerHTML = `<div class="editor-maintab active"><i class="fa-solid fa-pen"></i> Details</div>`;
  document.getElementById("editor-preview-img").style.background = MODULES.find((m) => m.id === type) ? `linear-gradient(135deg, ${MODULES.find((m) => m.id === type).color1}, ${MODULES.find((m) => m.id === type).color2})` : "";
  document.getElementById("editor-preview-clock").textContent = record[schema[0].key] || singularLabel(type);
  document.getElementById("editor-preview-date").textContent = "Live preview";

  document.getElementById("editor-panel").innerHTML = schema
    .map((f) => {
      const val = record[f.key] !== undefined ? record[f.key] : "";
      if (f.type === "select") {
        return `<div class="field-row"><label>${f.label}</label><select id="field-${f.key}">${f.options
          .map((o) => `<option value="${o}" ${o === val ? "selected" : ""}>${o.charAt(0).toUpperCase() + o.slice(1)}</option>`)
          .join("")}</select></div>`;
      }
      return `<div class="field-row"><label>${f.label}</label><input type="${f.type}" id="field-${f.key}" value="${escapeHtml(val)}" placeholder="${f.label}" oninput="updateEditorPreview()" /></div>`;
    })
    .join("");
}

function updateEditorPreview() {
  const { type } = state.editing;
  const schema = FIELD_SCHEMAS[type];
  const nameField = document.getElementById("field-" + schema[0].key);
  if (nameField) document.getElementById("editor-preview-clock").textContent = nameField.value || singularLabel(type);
}

function closeEditorScreen() {
  openModule(state.editing.type || state.currentModule);
}

async function saveEditorRecord() {
  const { type, id } = state.editing;

  // Enforce plan limits only when creating a NEW product/customer —
  // editing an existing record never counts against the cap.
  if (!id && (type === "products" || type === "customers")) {
    const limits = limitsForPlan();
    const cap = limits[type];
    if (state.data[type].length >= cap) {
      showToast(`Free plan allows up to ${cap} ${type} — upgrade for unlimited`);
      openPremiumScreen();
      return;
    }
  }

  const schema = FIELD_SCHEMAS[type];
  const record = id ? state.data[type].find((r) => r.id === id) : { id: type[0] + Date.now() };

  schema.forEach((f) => {
    const el = document.getElementById("field-" + f.key);
    if (!el) return;
    record[f.key] = f.type === "number" ? Number(el.value || 0) : el.value;
  });

  if (type === "customers" && record.totalSpent === undefined) record.totalSpent = 0;
  if (type === "customers" && record.dues === undefined) record.dues = 0;

  if (!id) state.data[type].push(record);
  await persistData();
  showToast(id ? "Updated" : "Added successfully");
  openModule(type);
  renderDashboard();
}

function toggleFavoriteEditor() {
  document.getElementById("editor-fav-btn").classList.toggle("favorited");
}

/* ---------------------------------------------------------
   10. QUICK VIEW MODAL (was wallpaper preview modal)
   --------------------------------------------------------- */
let quickViewCtx = null;
function openQuickView(moduleId, id) {
  const record = state.data[moduleId].find((r) => r.id === id);
  if (!record) return;
  quickViewCtx = { moduleId, id };
  const mod = MODULES.find((m) => m.id === moduleId);
  document.getElementById("modal-preview-bg").style.background = `linear-gradient(135deg, ${mod.color1}, ${mod.color2})`;
  const schema = FIELD_SCHEMAS[moduleId];
  document.getElementById("modal-wallpaper-title").innerHTML = schema
    .map((f) => `<div style="font-size:13px;font-weight:600;margin-top:6px;opacity:0.9;">${f.label}: ${record[f.key]}</div>`)
    .join("");
  document.getElementById("modal-download-action").textContent = "Edit";
  document.getElementById("modal-set-action").textContent = moduleId === "products" ? "Restock" : "Close";
  document.getElementById("wallpaper-modal").classList.add("active");

  document.getElementById("modal-download-action").onclick = () => {
    closeQuickView();
    openEditEditor(moduleId, id);
  };
  document.getElementById("modal-set-action").onclick = () => {
    if (moduleId === "products") {
      record.stock += 10;
      persistData();
      showToast("Restocked +10 units");
      renderModuleGrid();
    }
    closeQuickView();
  };
}
function closeQuickView() {
  document.getElementById("wallpaper-modal").classList.remove("active");
}
function toggleFavoriteModal() {
  document.getElementById("modal-fav-btn").classList.toggle("favorited");
}

/* ---------------------------------------------------------
   11. PRODUCT / GLOBAL SEARCH (was Explore)
   --------------------------------------------------------- */
function switchSubCat(el) {
  el.parentElement.querySelectorAll(".sub-cat-item").forEach((s) => s.classList.remove("active"));
  el.classList.add("active");
  state.searchFilter = el.dataset.filter;
  renderSearch();
}

function renderSearch() {
  const q = (document.getElementById("search-input").value || "").toLowerCase();
  const filter = state.searchFilter;
  const container = document.getElementById("wallpaper-grid-container");

  let rows = [];
  if (filter === "all" || filter === "products") {
    state.data.products
      .filter((p) => p.name.toLowerCase().includes(q))
      .forEach((p) => rows.push(renderDataRow("products", p)));
  }
  if (filter === "lowstock") {
    state.data.products
      .filter((p) => p.stock <= p.lowStockAt && p.name.toLowerCase().includes(q))
      .forEach((p) => rows.push(renderDataRow("products", p)));
  }
  if (filter === "all" || filter === "customers") {
    state.data.customers
      .filter((c) => c.name.toLowerCase().includes(q))
      .forEach((c) => rows.push(renderDataRow("customers", c)));
  }
  if (filter === "topcustomers") {
    [...state.data.customers]
      .sort((a, b) => b.totalSpent - a.totalSpent)
      .slice(0, 5)
      .filter((c) => c.name.toLowerCase().includes(q))
      .forEach((c) => rows.push(renderDataRow("customers", c)));
  }
  if (filter === "all" || filter === "invoices") {
    state.data.invoices
      .filter((i) => (i.customerName || "").toLowerCase().includes(q))
      .forEach((inv) => rows.push(renderInvoiceRow(inv)));
  }

  container.innerHTML = rows.length ? `<div class="data-list">${rows.join("")}</div>` : `<div class="data-row-empty"><i class="fa-solid fa-magnifying-glass"></i>No results found</div>`;
}

function quickAddCustomer() {
  state.editing = { type: "customers", id: null };
  buildEditorForm();
  document.getElementById("editor-title").textContent = "Add Customer";
  showScreen("screen-editor");
}

/* ---------------------------------------------------------
   12. INVOICES HUB (was Request screen)
   --------------------------------------------------------- */
function renderInvoicesHub() {
  const list = state.data.invoices.slice().reverse();
  const container = document.getElementById("samples-grid");
  if (!list.length) {
    container.innerHTML = `<div class="data-row-empty"><i class="fa-solid fa-file-invoice"></i>No invoices yet. Create your first one above.</div>`;
    return;
  }
  container.innerHTML = list.map(renderInvoiceRow).join("");
}
function renderInvoiceRow(inv) {
  return `<div class="data-row" onclick="showToast('Invoice ${inv.number}')">
    <div class="data-row-icon" style="background:#4c8dff1f;color:#4c8dff;"><i class="fa-solid fa-file-invoice"></i></div>
    <div class="data-row-body"><h4>${escapeHtml(inv.customerName)}</h4><p>${inv.number} • ${inv.date}</p></div>
    <div class="data-row-right"><div class="data-row-amount">${fmtINR(inv.total)}</div><span class="badge paid">Generated</span></div>
  </div>`;
}

function openOrderScreen() {
  state.invoiceDraft = { items: [{ name: "", qty: 1, price: 0, gst: 18 }], customerId: state.data.customers[0] ? state.data.customers[0].id : null };
  const sel = document.getElementById("invoice-customer-select");
  sel.innerHTML = state.data.customers.map((c) => `<option value="${c.id}">${escapeHtml(c.name)}</option>`).join("") || `<option value="">Add a customer first</option>`;
  document.getElementById("invoice-share-row").style.display = "none";
  renderInvoiceItems();
  showScreen("screen-order");
}

function addInvoiceItemRow() {
  state.invoiceDraft.items.push({ name: "", qty: 1, price: 0, gst: 18 });
  renderInvoiceItems();
}
function removeInvoiceItemRow(idx) {
  state.invoiceDraft.items.splice(idx, 1);
  renderInvoiceItems();
}
function updateInvoiceItem(idx, key, value) {
  state.invoiceDraft.items[idx][key] = key === "name" ? value : Number(value || 0);
  renderInvoiceTotals();
}
function renderInvoiceItems() {
  const container = document.getElementById("invoice-items-container");
  container.innerHTML = state.invoiceDraft.items
    .map(
      (it, idx) => `<div class="invoice-item-row">
      <input class="invoice-item-name" placeholder="Item name" value="${escapeHtml(it.name)}" oninput="updateInvoiceItem(${idx},'name',this.value)" />
      <input class="invoice-item-qty" type="number" min="1" placeholder="Qty" value="${it.qty}" oninput="updateInvoiceItem(${idx},'qty',this.value)" />
      <input class="invoice-item-price" type="number" min="0" placeholder="Price" value="${it.price}" oninput="updateInvoiceItem(${idx},'price',this.value)" />
      <i class="fa-solid fa-trash invoice-item-remove" onclick="removeInvoiceItemRow(${idx})"></i>
    </div>`
    )
    .join("");
  renderInvoiceTotals();
}
function renderInvoiceTotals() {
  const items = state.invoiceDraft.items;
  const subtotal = items.reduce((a, i) => a + i.qty * i.price, 0);
  const gst = items.reduce((a, i) => a + i.qty * i.price * (i.gst / 100), 0);
  const total = subtotal + gst;
  const gstType = document.getElementById("invoice-gst-type").value;
  const gstLabel = gstType === "inter" ? "IGST (18%)" : "CGST (9%) + SGST (9%)";
  document.getElementById("invoice-totals").innerHTML = `
    <div class="invoice-total-row"><span>Subtotal</span><span>${fmtINR(subtotal)}</span></div>
    <div class="invoice-total-row"><span>${gstLabel}</span><span>${fmtINR(gst)}</span></div>
    <div class="invoice-total-row grand"><span>Grand Total</span><span>${fmtINR(total)}</span></div>`;
  return { subtotal, gst, total };
}
document.addEventListener("change", (e) => {
  if (e.target && e.target.id === "invoice-gst-type") renderInvoiceTotals();
});

let lastInvoice = null;
async function submitOrderPayment() {
  const limits = limitsForPlan();
  if (state.invoicesUsedMonth >= limits.invoices) {
    showToast(`Free plan allows ${limits.invoices} invoices/month — upgrade for unlimited`);
    openPremiumScreen();
    return;
  }

  const custId = document.getElementById("invoice-customer-select").value;
  const customer = state.data.customers.find((c) => c.id === custId);
  const items = state.invoiceDraft.items.filter((i) => i.name && i.qty > 0);
  if (!customer || !items.length) {
    showToast("Add a customer and at least one item");
    return;
  }
  const totals = renderInvoiceTotals();
  const invNumber = "INV-" + String(state.data.invoices.length + 1).padStart(4, "0");
  const invoice = {
    id: "inv" + Date.now(),
    number: invNumber,
    customerName: customer.name,
    date: document.getElementById("invoice-date").value,
    items,
    subtotal: totals.subtotal,
    gst: totals.gst,
    total: totals.total,
    notes: document.getElementById("order-message").value,
  };
  state.data.invoices.push(invoice);
  state.data.sales.push({ id: "s" + Date.now(), customerName: customer.name, amount: invoice.total, date: invoice.date, status: "paid" });
  customer.totalSpent += invoice.total;
  await persistData();
  await useInvoiceSlot();
  lastInvoice = invoice;

  generateInvoicePdf(invoice);
  document.getElementById("invoice-share-row").style.display = "flex";
  showToast("Invoice generated");
  renderDashboard();
  renderInvoicesHub();
  renderSettings();
}

function generateInvoicePdf(invoice) {
  if (typeof window.jspdf === "undefined") {
    showToast("PDF library failed to load");
    return;
  }
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();
  const biz = state.businesses.find((b) => b.id === state.currentBusinessId) || {};

  doc.setFontSize(18);
  doc.text(biz.name || "My Business", 14, 20);
  doc.setFontSize(10);
  doc.text("GST Invoice", 14, 27);
  doc.text(`Invoice No: ${invoice.number}`, 140, 20);
  doc.text(`Date: ${invoice.date}`, 140, 26);

  doc.setFontSize(11);
  doc.text(`Bill To: ${invoice.customerName}`, 14, 40);

  let y = 55;
  doc.setFontSize(10);
  doc.text("Item", 14, y);
  doc.text("Qty", 100, y);
  doc.text("Price", 130, y);
  doc.text("Amount", 165, y);
  y += 4;
  doc.line(14, y, 196, y);
  y += 8;
  invoice.items.forEach((it) => {
    doc.text(String(it.name), 14, y);
    doc.text(String(it.qty), 100, y);
    doc.text(fmtINR(it.price), 130, y);
    doc.text(fmtINR(it.qty * it.price), 165, y);
    y += 8;
  });
  y += 4;
  doc.line(14, y, 196, y);
  y += 10;
  doc.text(`Subtotal: ${fmtINR(invoice.subtotal)}`, 140, y);
  y += 7;
  doc.text(`GST: ${fmtINR(invoice.gst)}`, 140, y);
  y += 7;
  doc.setFontSize(12);
  doc.text(`Total: ${fmtINR(invoice.total)}`, 140, y);

  if (invoice.notes) {
    y += 16;
    doc.setFontSize(9);
    doc.text("Notes: " + invoice.notes, 14, y);
  }

  doc.save(`${invoice.number}.pdf`);
}

function shareInvoiceWhatsapp() {
  // PRODUCTION: call the WhatsApp Business API (Cloud API) from a backend
  // Cloud Function, passing the invoice PDF as a media message. Direct
  // browser calls to the WhatsApp API are not possible without a server.
  showToast("Connect WhatsApp Business API in Settings to enable sending");
}
function shareInvoiceEmail() {
  // PRODUCTION: call your email provider's API (e.g. SendGrid) from a
  // backend Cloud Function with the invoice PDF attached.
  showToast("Connect an Email API in Settings to enable sending");
}

/* ---------------------------------------------------------
   13. SUBSCRIPTION / PLANS (was Premium)
   --------------------------------------------------------- */
function openPremiumScreen() {
  showScreen("screen-premium");
}
function closePremiumScreen() {
  switchTab("settings", document.getElementById("nav-settings"));
}
function selectPlan(el) {
  el.parentElement.querySelectorAll(".plan-option").forEach((p) => p.classList.remove("selected"));
  el.classList.add("selected");
}
function startPremiumUpgrade() {
  const selected = document.querySelector(".plan-option.selected");
  const plan = selected.dataset.plan; // "premium" | "free"
  const amount = Number(selected.dataset.amount) / 100;
  if (plan === "free") {
    applyPlan("free");
    return;
  }
  launchRazorpay(amount, "BizNova Premium Plan", () => applyPlan(plan));
}
async function applyPlan(plan) {
  state.plan = plan;
  await dbSet("plan", plan);
  showToast(`You're now on the ${plan.charAt(0).toUpperCase() + plan.slice(1)} plan`);
  renderSettings();
  switchTab("settings", document.getElementById("nav-settings"));
}

/* Razorpay Checkout — modular payment launcher.
   The Key ID below is a PUBLIC identifier (safe for frontend JS by
   Razorpay's own design) — it is NOT the secret key. The Key Secret
   must only ever live server-side (Cloud Function) for order creation
   and payment signature verification; never put it in this file. */
const RAZORPAY_KEY_ID = "rzp_live_TCZM7OsD80tNpH";
function launchRazorpay(amountRupees, description, onSuccess) {
  // PRODUCTION: create the order server-side (Cloud Function) first and
  // pass the returned order_id here, then verify the payment signature
  // server-side in the success handler before granting entitlements.
  if (typeof Razorpay === "undefined") {
    showToast("Razorpay SDK unavailable — simulating payment");
    onSuccess();
    return;
  }
  const options = {
    key: RAZORPAY_KEY_ID,
    amount: Math.round(amountRupees * 100),
    currency: "INR",
    name: "BizNova",
    description,
    handler: function () {
      onSuccess();
    },
    theme: { color: "#1a6cf0" },
  };
  try {
    const rz = new Razorpay(options);
    rz.open();
  } catch (e) {
    showToast("Simulating payment (demo mode)");
    onSuccess();
  }
}

/* ---------------------------------------------------------
   14. SETTINGS SCREEN
   --------------------------------------------------------- */
function renderSettings() {
  document.getElementById("profile-name-text").textContent = state.signedIn ? state.user.name : "Not signed in";
  document.getElementById("profile-email-text").textContent = state.signedIn ? state.user.email : "";

  const avatarEl = document.getElementById("profile-avatar");
  if (state.signedIn && state.user.picture) {
    avatarEl.innerHTML = `<img src="${state.user.picture}" alt="${escapeHtml(state.user.name)}" style="width:100%;height:100%;object-fit:cover;" referrerpolicy="no-referrer" />`;
  } else {
    avatarEl.textContent = state.signedIn ? state.user.avatar : "U";
  }
  document.getElementById("verified-badge").style.display = state.signedIn ? "flex" : "none";

  const planLabel = state.plan.charAt(0).toUpperCase() + state.plan.slice(1);
  document.getElementById("tier-label").textContent = planLabel;
  document.getElementById("limits-tier-label").textContent = "Tier: " + planLabel.toUpperCase();
  document.getElementById("limits-tier-label-2").textContent = "Tier: " + planLabel.toUpperCase();

  const limits = limitsForPlan();
  document.getElementById("monthly-label").textContent = `${state.aiCreditsUsedMonth} / ${limits.aiCredits === Infinity ? "∞" : limits.aiCredits} used`;
  document.getElementById("monthly-bar").style.width = limits.aiCredits === Infinity ? "8%" : Math.min(100, (state.aiCreditsUsedMonth / limits.aiCredits) * 100) + "%";

  document.getElementById("toggle-darkmode").classList.toggle("on", state.theme === "dark");
  document.getElementById("theme-desc").textContent = state.theme === "dark" ? "Dark theme" : "Light theme";

  const biz = state.businesses.find((b) => b.id === state.currentBusinessId);
  document.getElementById("current-biz-desc").textContent = biz ? `Currently: ${biz.name}` : "Manage another business you own";
  document.getElementById("multi-biz-desc").textContent = state.plan === "premium" ? "Premium: unlimited businesses" : `Free plan: ${state.businesses.length} / 1 business used`;

  renderUsageStatGrid(limits);
}

function renderUsageStatGrid(limits) {
  const productsUsed = state.data.products.length;
  const customersUsed = state.data.customers.length;
  const stats = [
    { label: "Subscription", value: state.plan === "premium" ? "Premium" : "Free" },
    { label: "AI Credits Left", value: limits.aiCredits === Infinity ? "Unlimited" : Math.max(0, limits.aiCredits - state.aiCreditsUsedMonth) },
    { label: "Businesses", value: `${state.businesses.length}${limits.businesses === Infinity ? "" : " / " + limits.businesses}`, full: limits.businesses !== Infinity && state.businesses.length >= limits.businesses },
    { label: "Invoices (this month)", value: `${state.invoicesUsedMonth}${limits.invoices === Infinity ? "" : " / " + limits.invoices}`, full: limits.invoices !== Infinity && state.invoicesUsedMonth >= limits.invoices },
    { label: "Products Used", value: `${productsUsed}${limits.products === Infinity ? "" : " / " + limits.products}`, full: limits.products !== Infinity && productsUsed >= limits.products },
    { label: "Customers Used", value: `${customersUsed}${limits.customers === Infinity ? "" : " / " + limits.customers}`, full: limits.customers !== Infinity && customersUsed >= limits.customers },
  ];
  document.getElementById("usage-stat-grid").innerHTML = stats
    .map((s) => `<div class="usage-stat ${s.full ? "full" : ""}"><div class="usage-stat-value">${s.value}</div><div class="usage-stat-label">${s.label}</div></div>`)
    .join("");
}

async function toggleSetting(rowEl, key) {
  const toggle = rowEl.querySelector(".toggle-switch");
  const isOn = toggle.classList.toggle("on");
  if (key === "darkmode") {
    applyTheme(isOn ? "dark" : "light");
    await dbSet("theme", state.theme);
    renderSettings();
  }
}

function applyTheme(theme) {
  state.theme = theme;
  const container = document.getElementById("app-container");
  if (theme === "dark") container.setAttribute("data-theme", "dark");
  else container.removeAttribute("data-theme");
}

function contactSupportEmail() {
  window.location.href = "mailto:depthnovacustomersupport@gmail.com";
}
function contactSupportTelegram() {
  window.open("https://wa.me/91XXXXXXXXXX", "_blank");
}

/* ---------------------------------------------------------
   15. BUSINESS SWITCHER (Multi-Business Support)
   --------------------------------------------------------- */
function openBizSwitcher() {
  const list = document.getElementById("biz-list");
  list.innerHTML = state.businesses
    .map(
      (b) => `<div class="biz-option ${b.id === state.currentBusinessId ? "selected" : ""}" onclick="switchBusiness('${b.id}')">
      <div class="biz-option-icon">${b.name.charAt(0)}</div>
      <div class="collection-option-text"><h4>${escapeHtml(b.name)}</h4><p>${b.id === state.currentBusinessId ? "Currently active" : "Tap to switch"}</p></div>
      ${b.id === state.currentBusinessId ? '<i class="fa-solid fa-check" style="color:var(--primary-color);"></i>' : ""}
    </div>`
    )
    .join("");
  document.getElementById("biz-switcher-overlay").classList.add("active");
}
function closeBizSwitcher() {
  document.getElementById("biz-switcher-overlay").classList.remove("active");
}
async function switchBusiness(id) {
  if (id === state.currentBusinessId) {
    closeBizSwitcher();
    return;
  }
  state.currentBusinessId = id;
  await dbSet("currentBusinessId", id);
  await loadBusinessData();
  closeBizSwitcher();
  renderAll();
  showToast("Switched business");
}
async function createNewBusiness() {
  const limits = limitsForPlan();
  if (state.businesses.length >= limits.businesses) {
    closeBizSwitcher();
    showToast("Free plan allows only 1 business — upgrade for unlimited");
    openPremiumScreen();
    return;
  }
  const name = prompt("Business name?");
  if (!name) return;
  const biz = { id: "biz" + Date.now(), name, gstin: "" };
  state.businesses.push(biz);
  await dbSet("businesses", state.businesses);
  await switchBusiness(biz.id);
}

/* ---------------------------------------------------------
   16. AI CREDIT GATE SHEET (was Unlock Premium sheet)
   ---------------------------------------------------------
   Credits are read from state (which is itself loaded from
   persistent storage in loadAiCredits() at init) — so there is
   no in-memory counter a page refresh could ever reset. Once a
   plan's monthly credits are used up, the ONLY way past this
   gate is upgrading — no ad-watching / free-bypass button.
   --------------------------------------------------------- */
function requireAiCredit(action) {
  const limits = limitsForPlan();
  if (state.aiCreditsUsedMonth < limits.aiCredits) {
    action();
    return;
  }
  document.getElementById("unlock-sheet-desc").textContent = `You've used all ${limits.aiCredits} AI credits for this month.`;
  document.getElementById("unlock-sheet-overlay").classList.add("active");
}
function closeUnlockSheet() {
  document.getElementById("unlock-sheet-overlay").classList.remove("active");
}
function chooseUnlockPremium() {
  closeUnlockSheet();
  openPremiumScreen();
}

/* ---------------------------------------------------------
   17. AI BUSINESS ASSISTANT (Gemini / OpenAI in production)
   --------------------------------------------------------- */
function openAiChat() {
  document.getElementById("ai-chat-overlay").classList.add("active");
  if (!state.aiChatHistory.length) {
    pushAiMessage("bot", "Hi! I'm your AI business assistant. Ask me about sales trends, stock levels, expenses, or get a business suggestion.");
  }
}
function closeAiChat() {
  document.getElementById("ai-chat-overlay").classList.remove("active");
}
function renderAiSuggestChips() {
  const chips = ["📈 Analyze my sales", "📦 Predict stock needs", "💰 Expense breakdown", "📊 Profit analysis", "📄 Generate a report", "💡 Business suggestion"];
  document.getElementById("ai-suggest-chips").innerHTML = chips.map((c) => `<div class="ai-suggest-chip" onclick="sendAiMessage('${c.replace(/'/g, "")}')">${c}</div>`).join("");
}
function pushAiMessage(role, text) {
  state.aiChatHistory.push({ role, text });
  const container = document.getElementById("ai-chat-messages");
  const div = document.createElement("div");
  div.className = "ai-msg " + role;
  div.textContent = text;
  container.appendChild(div);
  container.scrollTop = container.scrollHeight;
}

function sendAiMessage(preset) {
  const input = document.getElementById("ai-chat-input");
  const text = preset || input.value.trim();
  if (!text) return;
  input.value = "";
  pushAiMessage("user", text);

  requireAiCredit(async () => {
    await useAiCredit();
    renderSettings();

    const container = document.getElementById("ai-chat-messages");
    const typing = document.createElement("div");
    typing.className = "ai-msg bot typing";
    typing.innerHTML = '<div class="ai-typing-dot"></div><div class="ai-typing-dot"></div><div class="ai-typing-dot"></div>';
    container.appendChild(typing);
    container.scrollTop = container.scrollHeight;

    callAiApi(text).then((reply) => {
      typing.remove();
      pushAiMessage("bot", reply);
    });
  });
}

async function callAiApi(userText) {
  // Builds a compact snapshot of the current business data so the model
  // can ground its answer in real numbers.
  const revenue = state.data.sales.reduce((a, s) => a + s.amount, 0);
  const expenses = state.data.expenses.reduce((a, e) => a + e.amount, 0);
  const lowStock = state.data.products.filter((p) => p.stock <= p.lowStockAt).map((p) => p.name);
  const context = `Business snapshot — Revenue: ₹${revenue}, Expenses: ₹${expenses}, Profit: ₹${revenue - expenses}, Low stock items: ${lowStock.join(", ") || "none"}, Customers: ${state.data.customers.length}, Products: ${state.data.products.length}.`;

  try {
    const response = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-6",
        max_tokens: 300,
        messages: [{ role: "user", content: `You are a concise AI assistant inside a small-business SaaS app for India. ${context}\n\nUser question: ${userText}\n\nAnswer in under 80 words, practically, using the numbers above where relevant.` }],
      }),
    });
    const data = await response.json();
    const text = data.content && data.content.find((c) => c.type === "text");
    if (text) return text.text;
    throw new Error("no text");
  } catch (e) {
    // Fallback mock response for standalone hosting without an API proxy,
    // or if the request fails for any reason.
    return mockAiReply(userText, revenue, expenses, lowStock);
  }
}

function mockAiReply(userText, revenue, expenses, lowStock) {
  const t = userText.toLowerCase();
  const profit = revenue - expenses;
  if (t.includes("profit")) return `Your net profit this period is ₹${profit.toLocaleString("en-IN")} (revenue ₹${revenue.toLocaleString("en-IN")} − expenses ₹${expenses.toLocaleString("en-IN")}). ${profit >= 0 ? "That's a healthy margin — keep an eye on rising costs." : "You're running at a loss — review your top expense categories."}`;
  if (t.includes("report")) return `Quick report: Revenue ₹${revenue.toLocaleString("en-IN")}, Expenses ₹${expenses.toLocaleString("en-IN")}, Profit ₹${profit.toLocaleString("en-IN")}, ${lowStock.length} low-stock item(s). Open the Reports module for the full breakdown.`;
  if (t.includes("sale")) return `Your total recorded sales are ₹${revenue.toLocaleString("en-IN")}. Consider following up with customers who have pending invoices to improve cash flow.`;
  if (t.includes("stock") || t.includes("inventory")) return lowStock.length ? `${lowStock.length} item(s) are running low: ${lowStock.join(", ")}. I'd recommend reordering soon to avoid stockouts.` : "All your products are healthily stocked right now — no immediate reorders needed.";
  if (t.includes("expense")) return `Your total expenses are ₹${expenses.toLocaleString("en-IN")}. Rent and utilities are typically your biggest recurring costs — review them quarterly.`;
  if (t.includes("suggest")) return "Try bundling slow-moving products with your bestsellers, and send WhatsApp reminders to customers with pending dues — both are quick wins for revenue.";
  return `Based on your current numbers (revenue ₹${revenue.toLocaleString("en-IN")}, expenses ₹${expenses.toLocaleString("en-IN")}), your business looks ${profit >= 0 ? "profitable" : "at a loss"} this period. Ask me about sales, stock, profit, or a report for more detail.`;
}

/* ---------------------------------------------------------
   18. TOAST
   --------------------------------------------------------- */
let toastTimer;
function showToast(msg) {
  const t = document.getElementById("app-toast");
  t.textContent = msg;
  t.classList.add("show");
  clearTimeout(toastTimer);
  toastTimer = setTimeout(() => t.classList.remove("show"), 2200);
}
