/* UniBudget ULIMA PRO
   Funciones:
   - guardar en localStorage
   - CRUD transacciones
   - filtros por fecha/categoría/tipo
   - gráfico (Chart.js)
   - presupuesto mensual + alert
   - export CSV, descargar
   - conversor PEN/USD (tipo fijo)
   - sugerencias automáticas
   - modo claro/oscuro (toggle)
*/

const STORAGE_KEY = "unibudget_ulima_pro_v1";
const BUDGET_KEY = "unibudget_ulima_budget_v1";
const NAME_KEY = "unibudget_ulima_name_v1";
const DEFAULT_RATE = 0.27; // ejemplo: 1 PEN = 0.27 USD (puedes ajustar)

/* State */
let txs = [];
let chart = null;
let budget = Number(localStorage.getItem(BUDGET_KEY)) || 0;

/* Elements */
const el = {
  txList: document.getElementById("txList"),
  totalIncome: document.getElementById("totalIncome"),
  totalExpense: document.getElementById("totalExpense"),
  balance: document.getElementById("balance"),
  txCount: document.getElementById("txCount"),
  pieCanvas: document.getElementById("pieChart"),
  chartLegend: document.getElementById("chartLegend"),
  budgetFill: document.getElementById("budgetFill"),
  budgetMeta: document.getElementById("budgetMeta"),
  toast: document.getElementById("toast")
};

/* Init */
document.addEventListener("DOMContentLoaded", () => {
  // set defaults
  document.getElementById("date").value = today();
  document.getElementById("from").value = monthStart();
  document.getElementById("to").value = today();
  document.getElementById("studentName").value = localStorage.getItem(NAME_KEY) || "";

  // load txs
  load();

  // attach events
  document.getElementById("txForm").addEventListener("submit", onSubmit);
  document.getElementById("btnClear").addEventListener("click", () => {
    document.getElementById("txForm").reset();
    document.getElementById("date").value = today();
  });
  document.getElementById("btnExport").addEventListener("click", exportCSV);
  document.getElementById("search").addEventListener("input", render);
  document.getElementById("applyFilters").addEventListener("click", render);
  document.getElementById("setBudget").addEventListener("click", setBudgetFromInput);
  document.getElementById("budgetInput").value = budget || "";
  document.getElementById("convBtn").addEventListener("click", handleConvert);
  document.getElementById("resetData").addEventListener("click", resetAll);
  document.getElementById("suggestBtn").addEventListener("click", showSuggestions);
  document.getElementById("downloadCSV").addEventListener("click", exportCSV);
  document.getElementById("studentName").addEventListener("change", (e)=> {
    localStorage.setItem(NAME_KEY, e.target.value || "");
  });

  document.getElementById("themeToggle").addEventListener("click", toggleTheme);

  // initial render
  render();
  renderChart();
  updateBudgetUI();
});

/* Storage */
function load(){
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    txs = raw ? JSON.parse(raw) : sampleData();
  } catch(e){
    console.error(e);
    txs = sampleData();
  }
}
function save(){
  localStorage.setItem(STORAGE_KEY, JSON.stringify(txs));
}

/* Sample */
function sampleData(){
  return [
    { id: uid(), date: today(-2), category: "Cafetería ULIMA", amount: 18.5, type: "gasto", note: "Almuerzo" },
    { id: uid(), date: today(-5), category: "Transporte", amount: 4.0, type: "gasto", note: "Pasaje" },
    { id: uid(), date: today(-10), category: "Materiales / Libros", amount: 120.0, type: "gasto", note: "Libro" },
    { id: uid(), date: today(-12), category: "Ingreso", amount: 600.0, type: "ingreso", note: "Mesada" }
  ];
}

/* CRUD */
function onSubmit(e){
  e.preventDefault();
  const id = document.getElementById("txForm").dataset.editId || uid();
  const data = {
    id,
    date: document.getElementById("date").value,
    category: document.getElementById("category").value,
    type: document.getElementById("type").value,
    amount: parseFloat(document.getElementById("amount").value),
    note: document.getElementById("note").value || ""
  };
  if(!data.date || !data.amount || data.amount <= 0) {
    showToast("Fecha y monto válidos son obligatorios");
    return;
  }

  const editIndex = txs.findIndex(t => t.id === id);
  if(editIndex >= 0) {
    txs[editIndex] = data;
    showToast("Registro actualizado");
  } else {
    txs.unshift(data);
    showToast("Registro agregado");
  }
  save();
  document.getElementById("txForm").reset();
  document.getElementById("date").value = today();
  delete document.getElementById("txForm").dataset.editId;

  render();
  renderChart();
  updateBudgetUI();
}

/* Render list */
function render(){
  const query = (document.getElementById("search").value || "").toLowerCase().trim();
  const from = document.getElementById("from").value;
  const to = document.getElementById("to").value;
  const view = document.getElementById("viewFilter").value;

  const filtered = txs.filter(t => {
    if(view !== "all" && t.type !== view) return false;
    if(query && !((t.note||"").toLowerCase().includes(query) || (t.category||"").toLowerCase().includes(query))) return false;
    if(from && t.date < from) return false;
    if(to && t.date > to) return false;
    return true;
  });

  el.txList.innerHTML = "";
  if(filtered.length === 0) {
    el.txList.innerHTML = "<li class='tx-item'><div class='tx-left'><div class='tx-meta'>No hay registros</div></div></li>";
  } else {
    filtered.forEach(t => {
      const li = document.createElement("li");
      li.className = "tx-item";

      const left = document.createElement("div");
      left.className = "tx-left";
      const icon = document.createElement("div");
      icon.className = "icon-circle";
      icon.textContent = initials(t.category);
      const meta = document.createElement("div");
      meta.innerHTML = `<div style="font-weight:700">${t.note || t.category}</div><div class="tx-meta">${t.category} • ${t.date}</div>`;
      left.appendChild(icon);
      left.appendChild(meta);

      const right = document.createElement("div");
      right.style.textAlign = "right";
      const amount = document.createElement("div");
      amount.className = "tx-amount " + (t.type === "ingreso" ? "income" : "expense");
      amount.textContent = (t.type === "gasto" ? "-S/ " : "+S/ ") + Number(t.amount).toFixed(2);

      const actions = document.createElement("div");
      actions.className = "tx-actions";
      const editBtn = document.createElement("button");
      editBtn.className = "btn-ghost";
      editBtn.textContent = "Editar";
      editBtn.onclick = ()=> openEdit(t.id);
      const delBtn = document.createElement("button");
      delBtn.className = "btn-ghost";
      delBtn.style.color = "var(--danger)";
      delBtn.textContent = "Eliminar";
      delBtn.onclick = ()=> { if(confirm("¿Eliminar registro?")) { removeTx(t.id); } };

      actions.appendChild(editBtn);
      actions.appendChild(delBtn);

      right.appendChild(amount);
      right.appendChild(actions);

      li.appendChild(left);
      li.appendChild(right);
      el.txList.appendChild(li);
    });
  }

  el.txCount.textContent = `${filtered.length} registros`;
  renderSummary();
}

/* Edit / remove */
function openEdit(id) {
  const t = txs.find(x => x.id === id);
  if(!t) return;
  document.getElementById("date").value = t.date;
  document.getElementById("category").value = t.category;
  document.getElementById("type").value = t.type;
  document.getElementById("amount").value = t.amount;
  document.getElementById("note").value = t.note;
  document.getElementById("txForm").dataset.editId = id;
  window.scrollTo({top:0,behavior:"smooth"});
}
function removeTx(id) {
  txs = txs.filter(x => x.id !== id);
  save();
  render();
  renderChart();
  updateBudgetUI();
  showToast("Registro eliminado");
}

/* Summary */
function renderSummary(){
  const incomes = txs.filter(t=>t.type==="ingreso").reduce((s,x)=>s+Number(x.amount),0);
  const expenses = txs.filter(t=>t.type==="gasto").reduce((s,x)=>s+Number(x.amount),0);
  const bal = incomes - expenses;
  el.totalIncome.textContent = `S/ ${incomes.toFixed(2)}`;
  el.totalExpense.textContent = `S/ ${expenses.toFixed(2)}`;
  el.balance.textContent = `S/ ${bal.toFixed(2)}`;
}

/* Chart */
function renderChart(){
  // data: only gastos by category
  const byCat = {};
  txs.forEach(t=>{
    if(t.type==="gasto"){
      byCat[t.category] = (byCat[t.category]||0) + Number(t.amount);
    }
  });
  const labels = Object.keys(byCat);
  const data = Object.values(byCat);

  if(chart) { chart.destroy(); chart = null; }

  if(labels.length === 0){
    el.pieCanvas.style.display = "none";
    el.chartLegend.innerHTML = "<div class='small-muted'>No hay datos de gastos</div>";
    return;
  } else {
    el.pieCanvas.style.display = "block";
  }

  const ctx = el.pieCanvas.getContext("2d");
  chart = new Chart(ctx, {
    type: "doughnut",
    data: {
      labels,
      datasets: [{
        data,
        backgroundColor: [
          "#f47c20","#ff9a42","#ffc28a","#ffd9b2","#cfe8ff","#9fd0ff","#b9f7e4","#ffdede"
        ],
        borderColor: "#fff",
        borderWidth: 1
      }]
    },
    options: {
      plugins: { legend: { display:false } },
      maintainAspectRatio: false
    }
  });

  // legend
  el.chartLegend.innerHTML = labels.map((l,i)=>`<div style="display:flex;align-items:center;gap:8px;margin:6px 0"><span style="width:12px;height:12px;background:${chart.data.datasets[0].backgroundColor[i%8]};display:inline-block"></span>${l} — S/ ${data[i].toFixed(2)}</div>`).join("");
}

/* Budget */
function setBudgetFromInput(){
  const val = Number(document.getElementById("budgetInput").value);
  if(isNaN(val) || val < 0){ showToast("Presupuesto inválido"); return; }
  budget = val;
  localStorage.setItem(BUDGET_KEY, String(budget));
  updateBudgetUI();
  showToast("Presupuesto guardado");
}
function updateBudgetUI(){
  if(!budget || budget <= 0){
    el.budgetMeta.textContent = "Sin presupuesto establecido";
    el.budgetFill.style.width = "0%";
    return;
  }
  const spent = txs.filter(t=>t.type==="gasto").reduce((s,x)=>s+Number(x.amount),0);
  const pct = Math.min(100, (spent / budget) * 100);
  el.budgetFill.style.width = pct + "%";
  el.budgetMeta.textContent = `Gastado S/ ${spent.toFixed(2)} / S/ ${budget.toFixed(2)} (${pct.toFixed(0)}%)`;
  if(pct >= 100) showToast("⚠️ Has superado tu presupuesto mensual", 6000);
  else if(pct >= 80) showToast("⚠️ Estás cerca de tu presupuesto (>=80%)", 3500);
}

/* Export CSV */
function exportCSV(){
  if(!txs.length){ showToast("No hay datos para exportar"); return; }
  const header = "id,date,category,type,amount,note\n";
  const rows = txs.map(t => `${t.id},${t.date},${escapeCSV(t.category)},${t.type},${t.amount},"${escapeCSV(t.note||"")}"`);
  const csv = header + rows.join("\n");
  const blob = new Blob([csv], {type:"text/csv;charset=utf-8;"});
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `unibudget_ulima_export_${new Date().toISOString().slice(0,10)}.csv`;
  a.click();
  URL.revokeObjectURL(url);
}

/* Convert PEN <-> USD */
function handleConvert(){
  const val = parseFloat(document.getElementById("convValue").value);
  const dir = document.getElementById("convDir").value;
  if(isNaN(val)){ showToast("Ingrese un valor para convertir"); return; }
  let res = 0;
  if(dir === "PENtoUSD") res = val * DEFAULT_RATE;
  else res = val / DEFAULT_RATE;
  document.getElementById("convResult").textContent = `${val} → ${res.toFixed(2)} ${dir==="PENtoUSD" ? "USD" : "PEN"}`;
}

/* Suggestions (simple heuristics) */
function showSuggestions(){
  const byCat = {};
  txs.forEach(t => { if(t.type==="gasto") byCat[t.category] = (byCat[t.category]||0) + Number(t.amount); });
  const sorted = Object.entries(byCat).sort((a,b)=>b[1]-a[1]);
  if(sorted.length === 0){ showToast("No hay gastos para analizar"); return; }
  const top = sorted[0];
  const msg = `Has gastado S/ ${top[1].toFixed(2)} en ${top[0]}. Sugerencia: revisa opciones más económicas o reducir frecuencia.`;
  showToast(msg, 7000);
}

/* Reset */
function resetAll(){
  if(!confirm("¿Borrar todos los datos guardados? Esta acción no tiene deshacer.")) return;
  localStorage.removeItem(STORAGE_KEY);
  localStorage.removeItem(BUDGET_KEY);
  localStorage.removeItem(NAME_KEY);
  txs = [];
  budget = 0;
  save();
  render();
  renderChart();
  updateBudgetUI();
  showToast("Datos reseteados");
}

/* Utils */
function uid(){ return Math.random().toString(36).slice(2,9) }
function today(offset=0){ const d=new Date(); d.setDate(d.getDate()+offset); return d.toISOString().slice(0,10) }
function monthStart(){ const d=new Date(); d.setDate(1); return d.toISOString().slice(0,10) }
function initials(str){ if(!str) return "•"; return str.split(" ").map(w=>w[0]).slice(0,2).join("").toUpperCase() }
function escapeCSV(s){ return (""+s).replace(/"/g,'""') }

/* Toast */
let toastTimer = null;
function showToast(msg, duration=2500){
  el.toast.textContent = msg;
  el.toast.classList.remove("hidden");
  if(toastTimer) clearTimeout(toastTimer);
  toastTimer = setTimeout(()=> el.toast.classList.add("hidden"), duration);
}

/* Helpers */
function renderSummary(){
  renderSummary(); // already defined earlier; ensure it's consistent (call renderSummary defined previously)
}
/* Fix: previous name collision - ensure we use correct renderSummary above */
function renderSummary(){ 
  const incomes = txs.filter(t=>t.type==="ingreso").reduce((s,x)=>s+Number(x.amount),0);
  const expenses = txs.filter(t=>t.type==="gasto").reduce((s,x)=>s+Number(x.amount),0);
  const bal = incomes - expenses;
  el.totalIncome.textContent = `S/ ${incomes.toFixed(2)}`;
  el.totalExpense.textContent = `S/ ${expenses.toFixed(2)}`;
  el.balance.textContent = `S/ ${bal.toFixed(2)}`;
}

/* Theme toggle (simple) */
function toggleTheme(){
  const root = document.documentElement;
  if(root.dataset.theme === "dark"){
    root.dataset.theme = "light";
    root.style.setProperty("--bg", "#f6f8fb");
    root.style.setProperty("--card", "#fff");
    root.style.setProperty("--text", "#111827");
  } else {
    root.dataset.theme = "dark";
    root.style.setProperty("--bg", "#0b1220");
    root.style.setProperty("--card", "#071025");
    root.style.setProperty("--text", "#e6eef8");
  }
}

/* Remove collision: initial renderSummary already used in render(); so final call */
(function finalInit(){
  // ensure initial calculations & visuals
  render();
  renderChart();
  updateBudgetUI();
})();
