<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="utf-8" />
  <title>Excel Importer & Differenze (stable)</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
  <style>
    .diff-modified-old { text-decoration: line-through; color:#b91c1c; margin-right:6px; display:inline-block; }
    .diff-modified-new { color:#047857; font-weight:600; display:inline-block; }
    .days-green { color:#047857; font-weight:600; }
    .days-yellow { color:#b45309; font-weight:600; }
    .days-red { color:#b91c1c; font-weight:600; }
    table th.sticky, table td.sticky { position: sticky; left: 0; z-index: 10; background: white; }
    .border { border: 1px solid #e5e7eb; }
    .small-muted { color: #6b7280; font-size: 0.9rem; }

    /* Modal styles */
    .modal-backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.4); display: none; align-items: center; justify-content: center; z-index: 60; }
    .modal { background: white; border-radius: 8px; width: 520px; max-width: calc(100% - 32px); box-shadow: 0 6px 18px rgba(0,0,0,0.2); padding: 18px; }
    .modal h3 { font-weight: 700; margin-bottom: 12px; }
    .modal label { display: block; margin-bottom: 6px; font-size: 0.95rem; }
    .modal .row { display:flex; gap:12px; align-items:center; margin-bottom:10px; }
    .modal .tiny { font-size:0.85rem; color:#6b7280; }
  </style>
</head>
<body class="bg-gray-50 text-gray-900">
  <div class="max-w-7xl mx-auto p-6 space-y-6">
    <div class="flex justify-between items-center">
      <h1 class="text-xl font-bold">Gestione file Excel</h1>
      <div class="space-x-2">
        <label class="bg-blue-600 text-white px-4 py-2 rounded cursor-pointer">
          <input id="fileInput" type="file" accept=".xls,.xlsx" class="hidden" />
          Carica file
        </label>
        <button id="settingsBtn" class="bg-gray-600 text-white px-3 py-2 rounded">⚙️</button>
      </div>
    </div>

    <div class="flex border-b">
      <button id="tabContentBtn" class="px-4 py-2 border-b-2 border-blue-600 text-blue-600">Contenuto</button>
      <button id="tabDiffBtn" class="px-4 py-2 border-b-2 border-transparent">Differenze</button>
    </div>

    <!-- Barra di ricerca e filtri (globale) -->
    <div class="bg-white p-4 rounded shadow flex flex-wrap gap-3 items-end">
      <div style="flex:1; min-width:220px;">
        <label class="small-muted">Ricerca (tutti i campi)</label>
        <input id="globalSearch" type="search" placeholder="Cerca... (testo parziale, case-insensitive)" class="w-full border px-2 py-1 rounded" />
      </div>
      <div>
        <label class="small-muted">Dal</label>
        <input id="dateFrom" type="date" class="border px-2 py-1 rounded" />
      </div>
      <div>
        <label class="small-muted">Al</label>
        <input id="dateTo" type="date" class="border px-2 py-1 rounded" />
      </div>
      <div class="ml-auto flex gap-2">
        <button id="clearFilters" class="px-3 py-2 border rounded">Pulisci filtri</button>
        <button id="applyFilters" class="px-3 py-2 bg-blue-600 text-white rounded">Applica</button>
      </div>
    </div>

    <!-- Contenuto -->
    <div id="contentMode" class="space-y-4">
      <div class="flex items-center space-x-4">
        <label class="flex items-center gap-2"><input id="showSelected" type="checkbox"> Mostra solo selezionati</label>
        <button id="exportPdfBtn" class="bg-green-600 text-white px-4 py-2 rounded">Esporta PDF</button>
        <button id="exportExcelBtn" class="bg-red-600 text-white px-4 py-2 rounded">Esporta Excel</button>
      </div>
      <div id="contentTableContainer" class="bg-white rounded shadow overflow-auto min-h-[120px] p-4"></div>
    </div>

    <!-- Differenze -->
    <div id="diffMode" class="space-y-4 hidden">
      <div class="bg-white p-4 rounded shadow">
        <div class="flex justify-between items-center mb-2">
          <h2 class="font-semibold">Cronologia versioni</h2>
          <div>
            <button id="clearHistory" class="text-red-600 mr-4">Cancella cronologia</button>
            <button id="refreshHistory" class="text-gray-600">Aggiorna</button>
          </div>
        </div>
        <ul id="historyList" class="space-y-1 small-muted"></ul>
      </div>

      <div class="bg-white p-4 rounded shadow">
        <h2 class="font-semibold mb-2">Risultati del confronto</h2>
        <div id="diffTabs" class="flex space-x-4 mb-2">
          <button data-type="new" class="diff-tab px-3 py-1 text-sm text-blue-600 border-b-2 border-blue-600">Nuove righe</button>
          <button data-type="modified" class="diff-tab px-3 py-1 text-sm text-gray-600">Righe modificate</button>
          <button data-type="deleted" class="diff-tab px-3 py-1 text-sm text-gray-600">Righe eliminate</button>
        </div>
        <div id="diffTableContainer" class="overflow-auto min-h-[120px] p-2"></div>
      </div>
    </div>

  </div>

  <!-- Settings Modal (soglie giorni) -->
  <div id="modalBackdrop" class="modal-backdrop" aria-hidden="true">
    <div class="modal" role="dialog" aria-modal="true" aria-labelledby="modalTitle">
      <h3 id="modalTitle">Impostazioni soglie giorni</h3>

      <div class="row">
        <div style="flex:1">
          <label for="greenInput">Soglia verde (giorni)</label>
          <input id="greenInput" type="number" min="0" class="w-full border px-2 py-1 rounded" />
          <div class="tiny">Valori minori della soglia verde mostreranno lo stato verde.</div>
        </div>
        <div style="flex:1">
          <label for="yellowInput">Soglia gialla (giorni)</label>
          <input id="yellowInput" type="number" min="0" class="w-full border px-2 py-1 rounded" />
          <div class="tiny">Valori tra verde e giallo mostreranno giallo; sopra mostreranno rosso.</div>
        </div>
      </div>

      <div class="flex justify-end gap-3 mt-4">
        <button id="modalCancel" class="px-3 py-2 rounded border">Annulla</button>
        <button id="modalSave" class="px-3 py-2 rounded bg-blue-600 text-white">Salva</button>
      </div>
    </div>
  </div>

<script>
(function(){
  // ---------- stato ----------
  let versions = JSON.parse(localStorage.getItem("excel_versions") || "[]"); // each {name,date,rows}
  let diffs = { new: [], modified: [], deleted: [] };
  let activeDiffTab = "new";
  let currentRows = [];
  let selectedIds = new Set();
  let thresholds = JSON.parse(localStorage.getItem("giorni_soglie") || '{"green":7,"yellow":14}');

  // ---------- filtri globali ----------
  let searchQuery = "";
  let dateFilterFrom = null; // Date or null
  let dateFilterTo = null;   // Date or null

  // ---------- utils ----------
  function saveVersions(){ localStorage.setItem("excel_versions", JSON.stringify(versions)); }
  function saveThresholds(){ localStorage.setItem("giorni_soglie", JSON.stringify(thresholds)); }
  function generateId(){ return Date.now().toString(36) + "_" + Math.random().toString(36).slice(2,8); }
  function normalizeKeyName(k){ return (k||"").toString().toLowerCase().replace(/[^a-z0-9]/g,""); }
  function isDateKey(k){
    if(!k) return false;
    const nk = normalizeKeyName(k);
    const isDate = nk.includes("data") || k === "DATA ST" || k === "Data ST" || nk === "datast";
    return isDate;
  }
  function isHiddenKey(k){
    if(!k) return false;
    const nk = normalizeKeyName(k);
    return nk === "doc" || nk === "doc?" || nk === "datafatt" || nk === "fatt";
  }

  // date parsing/format
  function tryParseDate(v){
    if(v === null || v === undefined || v === "") return null;
    if(v instanceof Date && !isNaN(v.getTime())) return v;
    if(typeof v === "number"){
      // excel numeric date
      const epoch = new Date(Date.UTC(1899,11,30));
      return new Date(epoch.getTime() + Math.round(v) * 86400000);
    }
    if(typeof v === "string"){
      // ISO or other parsable
      const p = Date.parse(v);
      if(!isNaN(p)) return new Date(p);
      // dd/mm/yyyy or d/m/yy
      const m = v.match(/^(\d{1,2})[\/\-](\d{1,2})[\/\-](\d{2,4})$/);
      if(m){ const day = parseInt(m[1],10), mon = parseInt(m[2],10)-1, yr = parseInt(m[3],10); return new Date(yr,mon,day); }
    }
    return null;
  }
  function isoDate(d){ if(!d) return ""; const yyyy = d.getFullYear(); const mm = String(d.getMonth()+1).padStart(2,"0"); const dd = String(d.getDate()).padStart(2,"0"); return `${yyyy}-${mm}-${dd}`; }
  function displayDate(d){ if(!d) return ""; return d.toLocaleDateString("it-IT",{day:"2-digit",month:"2-digit",year:"numeric"}); }

  function normalizeForCompare(v, key){
    if(v === null || v === undefined) return "";
    if(isDateKey(key)){ const d = tryParseDate(v); return d ? isoDate(d) : String(v).trim(); }
    if(typeof v === "number") return String(v);
    if(typeof v === "string") return v.trim();
    return String(v).trim();
  }
  function formatForDisplay(v, key){
    if(v === null || v === undefined) return "";
    if(isDateKey(key)){ const d = tryParseDate(v); return d ? displayDate(d) : String(v); }
    return String(v);
  }

  function escapeHtml(s){ return String(s||"").replace(/[&<>"']/g, m => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":"&#39;"}[m])); }

  // ensure __id on all rows in versions
  function ensureIdsOnVersions(){
    versions.forEach(ver => {
      if(!Array.isArray(ver.rows)) ver.rows = [];
      ver.rows.forEach(r => { if(!r.__id) r.__id = generateId(); });
    });
    saveVersions();
  }

  // try to detect a stable identifier key among columns
  function detectIdKey(allKeys){
    const candidates = ["odl","id","numero","num","n","codice","code","ref"];
    const keysNorm = allKeys.map(k => ({k, n: normalizeKeyName(k)}));
    for(const c of candidates){
      const found = keysNorm.find(x => x.n === c || x.n.includes(c));
      if(found) return found.k;
    }
    // fallback: first key
    return allKeys[0];
  }

  // ---------- matching di ricerca / filtri ----------
  function rowMatchesSearch(row, q){
    if(!q) return true;
    const lower = q.toLowerCase();
    for(const k in row){
      if(k === "__id") continue;
      const v = row[k];
      if(v === null || v === undefined) continue;
      const s = (typeof v === "object") ? JSON.stringify(v) : String(v);
      if(s.toLowerCase().includes(lower)) return true;
    }
    return false;
  }

  function rowHasDateInRange(row, from, to){
    // if no date filters applied, pass
    if(!from && !to) return true;
    // find any date-like field in row (uses isDateKey heuristic) or any parsable date value
    for(const k in row){
      if(k === "__id") continue;
      const v = row[k];
      const d = tryParseDate(v);
      if(!d) continue;
      // normalize day boundaries
      const dd = new Date(d.getFullYear(), d.getMonth(), d.getDate());
      if(from && dd < from) continue;
      if(to && dd > to) continue;
      return true; // at least one date is inside range
    }
    // no date in range found
    return false;
  }

  function rowMatchesAllFilters(row){
    return rowMatchesSearch(row, searchQuery) && rowHasDateInRange(row, dateFilterFrom, dateFilterTo);
  }

  // ---------- rendering contenuto ----------
  function findDataStKey(keys){
    return keys.find(k => normalizeKeyName(k).includes("datast")) || null;
  }
  function isTotalRow(row){
    return Object.values(row).some(v => typeof v === "string" && v.trim().toLowerCase().startsWith("totale"));
  }
  function computeGiorniAttesa(val){
    const d = tryParseDate(val);
    if(!d) return null;
    const today = new Date(); today.setHours(0,0,0,0);
    d.setHours(0,0,0,0);
    return Math.floor((today - d) / 86400000);
  }
  function giornoClass(days){
    if(days === null) return "";
    if(days < thresholds.green) return "days-green";
    if(days < thresholds.yellow) return "days-yellow";
    return "days-red";
  }

  function getColorForClient(name){
    name = String(name||"");
    let hash = 0;
    for(let i=0;i<name.length;i++){ hash = name.charCodeAt(i) + ((hash<<5)-hash); hash |= 0; }
    const hue = Math.abs(hash) % 360;
    return `hsl(${hue},70%,85%)`;
  }

  function sortRows(rows, dataStKey){
    return [...rows].sort((a,b) => {
      const ca = (a["Cliente"]||"").toString().localeCompare((b["Cliente"]||"").toString());
      if(ca !== 0) return ca;
      const da = computeGiorniAttesa(a[dataStKey]) || 0;
      const db = computeGiorniAttesa(b[dataStKey]) || 0;
      return db - da;
    });
  }

  function renderTable(rows, keys, dataStKey, showSelect = true){
    let html = "<table class='min-w-full text-sm border-collapse'><thead><tr>";
    if(showSelect) html += `<th class="px-2 py-1 border">Sel</th>`;
    keys.forEach(k => html += `<th class="px-2 py-1 border ${k==="Cliente"?"sticky":""}">${escapeHtml(k)}</th>`);
    html += "</tr></thead><tbody>";
    rows.forEach(r => {
      const color = getColorForClient(r["Cliente"]||"");
      html += `<tr style="background-color:${color};">`;
      if(showSelect) html += `<td class="px-2 py-1 border"><input type="checkbox" class="row-select" data-id="${r.__id}"></td>`;
      keys.forEach(k => {
        if(k === "Giorni di attesa"){
          const d = computeGiorniAttesa(r[dataStKey]);
          html += `<td class="px-2 py-1 border ${giornoClass(d)}">${d!==null?d:""}</td>`;
        } else {
          html += `<td class="px-2 py-1 border">${escapeHtml(formatForDisplay(r[k],k))}</td>`;
        }
      });
      html += "</tr>";
    });
    html += "</tbody></table>";
    return html;
  }

  function renderContent(rows){
    const cont = document.getElementById("contentTableContainer");
    cont.innerHTML = "";
    if(!Array.isArray(rows) || !rows.length){
      // leave empty initially
      return;
    }
    // apply filters and remove totals
    let dataRows = rows.filter(r => !isTotalRow(r)).filter(r => rowMatchesAllFilters(r));
    // apply show-only-selected
    if(document.getElementById("showSelected")?.checked){
      dataRows = dataRows.filter(r => selectedIds.has(String(r.__id)));
    }
    if(!dataRows.length){
      cont.innerHTML = "<div class='p-4 text-gray-600'>Nessun dato</div>";
      return;
    }
    const origKeys = Object.keys(rows[0]||{});
    const dataStKey = findDataStKey(origKeys);
    let keys = origKeys.filter(k => k !== "__id" && !isHiddenKey(k) && !["Doc?","Data","Data Fatt","Fatt"].includes(k));
    if(keys.includes("Cliente")){ keys.splice(keys.indexOf("Cliente"),1); keys.unshift("Cliente"); }
    if(dataStKey && !keys.includes("Giorni di attesa")) keys.push("Giorni di attesa");
    const sorted = sortRows(dataRows, dataStKey);
    cont.innerHTML = renderTable(sorted, keys, dataStKey, true);
    // set checkbox states
    cont.querySelectorAll('.row-select').forEach(inp => {
      const id = inp.getAttribute('data-id');
      inp.checked = selectedIds.has(String(id));
    });
  }

  // delegate checkbox changes inside content table
  document.getElementById("contentTableContainer").addEventListener("change", function(e){
    const t = e.target;
    if(t && t.matches && t.matches(".row-select")){
      const id = String(t.getAttribute("data-id"));
      if(t.checked) selectedIds.add(id); else selectedIds.delete(id);
      if(document.getElementById("showSelected").checked) renderContent(currentRows);
    }
  });

  // ---------- compute diffs (exclude any key containing 'data' from change detection) ----------
  function computeDiffs(iOld, iNew){
    const oldRows = versions[iOld]?.rows || [];
    const newRows = versions[iNew]?.rows || [];
    const allKeys = [...new Set(oldRows.concat(newRows).flatMap(r => Object.keys(r) || []))];
    const keyId = detectIdKey(allKeys);

    const oldMap = {}, newMap = {};
    oldRows.forEach(r => { oldMap[String(r[keyId] !== undefined ? r[keyId] : "")] = r; });
    newRows.forEach(r => { newMap[String(r[keyId] !== undefined ? r[keyId] : "")] = r; });

    diffs = { new: [], modified: [], deleted: [] };

    // new / deleted
    for(const k in newMap) if(!oldMap[k]) diffs.new.push(newMap[k]);
    for(const k in oldMap) if(!newMap[k]) diffs.deleted.push(oldMap[k]);

    // modified
    for(const k in newMap){
      if(!oldMap[k]) continue;
      const o = oldMap[k], n = newMap[k];
      const fields = [...new Set([...Object.keys(o||{}), ...Object.keys(n||{})])];
      const changedFields = [];
      for(const f of fields){
        if(f === "__id") continue;
        const isDateField = isDateKey(f);
        const isHiddenField = isHiddenKey(f);
        if(isDateField) continue; // we ignore all date-like fields in the change detection
        if(isHiddenField) continue; // ignore hidden docs
        const ov = normalizeForCompare(o[f], f);
        const nv = normalizeForCompare(n[f], f);
        if(ov !== nv){
          changedFields.push({ campo: f, old_normalized: ov, new_normalized: nv, old_display: formatForDisplay(o[f],f), new_display: formatForDisplay(n[f],f) });
        }
      }
      if(changedFields.length){
        diffs.modified.push({ old: o, new: n, changedFields });
      }
    }
    renderDiffs();
  }

  function renderDiffs(){
    const cont = document.getElementById("diffTableContainer");
    cont.innerHTML = "";
    if(activeDiffTab === "modified"){
      const visible = diffs.modified.filter(pair => {
        // consider a modified pair visible if either old or new matches global filters
        return rowMatchesAllFilters(pair.old) || rowMatchesAllFilters(pair.new);
      });
      if(!visible.length){ cont.textContent = "Nessun risultato"; return; }
      const sampleNew = visible[0].new || {};
      let keys = Object.keys(sampleNew).filter(k => k !== "__id");
      if(keys.includes("Cliente")){ keys.splice(keys.indexOf("Cliente"),1); keys.unshift("Cliente"); }
      const dataStKey = findDataStKey(keys);
      if(dataStKey && !keys.includes("Giorni di attesa")) keys.push("Giorni di attesa");

      let html = "<table class='min-w-full text-sm'><thead><tr>";
      keys.forEach(k => html += `<th class="px-2 py-1 border ${k==="Cliente"?"sticky":""}">${escapeHtml(k)}</th>`);
      html += "</tr></thead><tbody>";

      visible.forEach(pair => {
        const o = pair.old, n = pair.new;
        const color = getColorForClient(n["Cliente"] || "");
        html += `<tr style="background-color:${color};">`;
        keys.forEach(k => {
          if(k === "Giorni di attesa"){
            const d = computeGiorniAttesa(n[findDataStKey(Object.keys(n)||[])]||"");
            html += `<td class="px-2 py-1 border ${giornoClass(d)}">${d!==null?d:""}</td>`;
            return;
          }
          if(isDateKey(k)){
            html += `<td class="px-2 py-1 border">${escapeHtml(formatForDisplay(n[k],k))}</td>`;
            return;
          }
          const ov_display = formatForDisplay(o[k], k);
          const nv_display = formatForDisplay(n[k], k);
          const ov_norm = normalizeForCompare(o[k], k);
          const nv_norm = normalizeForCompare(n[k], k);
          if(ov_norm !== nv_norm){
            html += `<td class="px-2 py-1 border"><span class="diff-modified-old">${escapeHtml(ov_display)}</span><span class="diff-modified-new">${escapeHtml(nv_display)}</span></td>`;
          } else {
            html += `<td class="px-2 py-1 border">${escapeHtml(nv_display)}</td>`;
          }
        });
        html += `</tr>`;
      });

      html += "</tbody></table>";
      cont.innerHTML = html;
      return;
    }

    // new / deleted simple table
    let rows = (activeDiffTab === "new") ? diffs.new : diffs.deleted;
    // apply filters (remove totals, then filter)
    const filteredRows = rows.filter(r => !isTotalRow(r)).filter(r => rowMatchesAllFilters(r));
    if(!filteredRows.length){ cont.textContent = "Nessun risultato"; return; }
    let keys = Object.keys(filteredRows[0]||{}).filter(k => k !== "__id");
    if(keys.includes("Cliente")){ keys.splice(keys.indexOf("Cliente"),1); keys.unshift("Cliente"); }
    const dataStKey = findDataStKey(keys);
    if(dataStKey && !keys.includes("Giorni di attesa")) keys.push("Giorni di attesa");
    const sorted = sortRows(filteredRows, dataStKey);
    cont.innerHTML = renderTable(sorted, keys, dataStKey, false);
  }

  // ---------- history UI ----------
  function renderHistory(){
    const list = document.getElementById("historyList");
    list.innerHTML = "";
    versions.forEach((v,i) => {
      const li = document.createElement("li");
      li.className = "flex justify-between p-2 border rounded small-muted";
      li.innerHTML = `<span>${escapeHtml(v.name)} — ${escapeHtml(v.date)}</span>
        <div>
          <button onclick="viewVersion(${i})" class="text-blue-600 mr-2">👁</button>
          <button onclick="compareWithLatest(${i})" class="text-green-600 mr-2">⇄</button>
          <button onclick="deleteVersion(${i})" class="text-red-600">🗑</button>
        </div>`;
      list.appendChild(li);
    });
  }

  window.viewVersion = function(i){
    currentRows = versions[i]?.rows || [];
    renderContent(currentRows);
    showTab("content");
  };
  window.deleteVersion = function(i){
    versions.splice(i,1);
    saveVersions();
    ensureIdsOnVersions();
    renderHistory();
  };
  window.compareWithLatest = function(i){
    if(versions.length >= 2) computeDiffs(i, versions.length - 1);
    showTab("diff");
  };

  // ---------- tabs & events ----------
  function showTab(tab){
    document.getElementById("contentMode").classList.add("hidden");
    document.getElementById("diffMode").classList.add("hidden");
    if(tab === "content") document.getElementById("contentMode").classList.remove("hidden");
    else document.getElementById("diffMode").classList.remove("hidden");
  }
  document.getElementById("tabContentBtn").onclick = () => showTab("content");
  document.getElementById("tabDiffBtn").onclick = () => { renderHistory(); showTab("diff"); };

  document.querySelectorAll(".diff-tab").forEach(b => b.onclick = function(){
    document.querySelectorAll(".diff-tab").forEach(x=>x.classList.remove("text-blue-600","border-b-2","border-blue-600"));
    this.classList.add("text-blue-600","border-b-2","border-blue-600");
    activeDiffTab = this.dataset.type;
    renderDiffs();
  });

  document.getElementById("refreshHistory").onclick = () => renderHistory();
  document.getElementById("clearHistory").onclick = () => { versions = []; localStorage.removeItem("excel_versions"); renderHistory(); };

  // ---------- settings modal behaviour ----------
  const modalBackdrop = document.getElementById("modalBackdrop");
  const settingsBtn = document.getElementById("settingsBtn");
  const modalSave = document.getElementById("modalSave");
  const modalCancel = document.getElementById("modalCancel");
  const greenInput = document.getElementById("greenInput");
  const yellowInput = document.getElementById("yellowInput");

  function openSettingsModal(){
    // populate inputs with current thresholds
    greenInput.value = Number.isFinite(thresholds.green) ? thresholds.green : "";
    yellowInput.value = Number.isFinite(thresholds.yellow) ? thresholds.yellow : "";
    modalBackdrop.style.display = "flex";
    modalBackdrop.setAttribute("aria-hidden", "false");
    setTimeout(() => greenInput.focus(), 50);
  }
  function closeSettingsModal(){
    modalBackdrop.style.display = "none";
    modalBackdrop.setAttribute("aria-hidden", "true");
  }

  settingsBtn.addEventListener("click", openSettingsModal);
  modalCancel.addEventListener("click", closeSettingsModal);
  modalBackdrop.addEventListener("click", function(e){
    if(e.target === modalBackdrop) closeSettingsModal();
  });
  document.addEventListener("keydown", function(e){
    if(e.key === "Escape" && modalBackdrop.style.display === "flex") closeSettingsModal();
  });

  modalSave.addEventListener("click", function(){
    const g = parseInt(greenInput.value, 10);
    const y = parseInt(yellowInput.value, 10);
    if(isNaN(g) || isNaN(y)){
      alert("Inserisci numeri validi per entrambe le soglie.");
      return;
    }
    if(g < 0 || y < 0){
      alert("Le soglie devono essere numeri >= 0.");
      return;
    }
    if(g >= y){
      alert("La soglia verde deve essere minore della soglia gialla.");
      return;
    }
    thresholds.green = g;
    thresholds.yellow = y;
    saveThresholds();
    // ri-render per applicare nuovi colori
    renderContent(currentRows);
    renderDiffs();
    closeSettingsModal();
  });

  // ---------- file load ----------
  document.getElementById("fileInput").onchange = async function(e){
    const f = e.target.files[0]; if(!f) return;
    const data = new Uint8Array(await f.arrayBuffer());
    const wb = XLSX.read(data, { type: "array", cellDates: true });
    const ws = wb.Sheets[wb.SheetNames[0]];
    const rows = XLSX.utils.sheet_to_json(ws, { defval: "" });
    // ensure each row has __id stable within this version
    rows.forEach(r => { if(!r.__id) r.__id = generateId(); });
    const ver = { name: f.name, date: new Date().toLocaleString(), rows };
    versions.push(ver);
    ensureIdsOnVersions();
    saveVersions();
    currentRows = rows;
    renderHistory();
    renderContent(currentRows);
    if(versions.length >= 2) computeDiffs(versions.length - 2, versions.length - 1);
    // reset file input so same file can be reloaded if needed
    e.target.value = "";
  };

  // showSelected toggle
  document.getElementById("showSelected").onchange = () => renderContent(currentRows);

  // export PDF landscape
  document.getElementById("exportPdfBtn").onclick = function(){
    const element = document.getElementById("contentTableContainer");
    if(!element) return;
    const opt = {
      margin: 6,
      filename: 'tabella_contenuto.pdf',
      image: { type: 'jpeg', quality: 0.98 },
      html2canvas: { scale: 2, useCORS: true },
      jsPDF: { unit: 'mm', format: 'a4', orientation: 'landscape' }
    };
    html2pdf().set(opt).from(element).save();
  };

  // ---------- esporta selezionati in Excel ----------
  document.getElementById("exportExcelBtn").onclick = function(){
    const selected = currentRows.filter(r => selectedIds.has(String(r.__id)));
    if (!selected.length) {
      alert("Nessun record selezionato.");
      return;
    }
    const toExport = selected.map(r => {
      const copy = {};
      Object.keys(r).forEach(k => { if(k !== "__id") copy[k] = r[k]; });
      return copy;
    });
    const ws = XLSX.utils.json_to_sheet(toExport);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, "Selezionati");
    XLSX.writeFile(wb, "selected_records.xlsx");
  };

  // ---------- filtri: UI bindings ----------
  const inputSearch = document.getElementById("globalSearch");
  const inputFrom = document.getElementById("dateFrom");
  const inputTo = document.getElementById("dateTo");
  const btnApply = document.getElementById("applyFilters");
  const btnClear = document.getElementById("clearFilters");

  function parseDateInputValue(v){
    if(!v) return null;
    const p = tryParseDate(v);
    if(!p) return null;
    // normalize to start of day
    return new Date(p.getFullYear(), p.getMonth(), p.getDate());
  }

  function applyFilterState(){
    searchQuery = (inputSearch.value || "").trim();
    dateFilterFrom = parseDateInputValue(inputFrom.value);
    // dateTo should include whole day -> set to end of day
    const toRaw = parseDateInputValue(inputTo.value);
    if(toRaw) dateFilterTo = new Date(toRaw.getFullYear(), toRaw.getMonth(), toRaw.getDate(), 23,59,59,999);
    else dateFilterTo = null;
    // re-render current view
    renderContent(currentRows);
    renderDiffs();
  }

  btnApply.addEventListener("click", applyFilterState);
  // make search reactive (live search)
  inputSearch.addEventListener("input", function(){ applyFilterState(); });
  inputFrom.addEventListener("change", applyFilterState);
  inputTo.addEventListener("change", applyFilterState);
  btnClear.addEventListener("click", function(){
    inputSearch.value = "";
    inputFrom.value = "";
    inputTo.value = "";
    searchQuery = "";
    dateFilterFrom = null;
    dateFilterTo = null;
    renderContent(currentRows);
    renderDiffs();
  });

  // ---------- init ----------
  // Ensure thresholds from storage are numbers and valid; fallback to defaults if invalid
  try{
    thresholds.green = Number.isFinite(Number(thresholds.green)) ? Number(thresholds.green) : 7;
    thresholds.yellow = Number.isFinite(Number(thresholds.yellow)) ? Number(thresholds.yellow) : 14;
    if(!(thresholds.green < thresholds.yellow)){ thresholds.green = 7; thresholds.yellow = 14; saveThresholds(); }
  }catch(e){
    thresholds = { green:7, yellow:14 };
    saveThresholds();
  }

  ensureIdsOnVersions();
  renderHistory();
  // page initially empty (user will upload o view history)
})();
</script>
</body>
</html>
