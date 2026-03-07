<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ErtimeCMC - გაუმჯობესებული ვერსია</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

<style>
    :root {
        --blue: #001f3f;
        --red: #8b0000;
        --bg: #f8f9fa;
        --light: #ffffff;
    }
    * { box-sizing: border-box; }
    body {
        font-family: system-ui, -apple-system, sans-serif;
        margin: 12px;
        background: var(--bg);
        color: #222;
        -webkit-tap-highlight-color: transparent;
    }
    .container {
        max-width: 100%;
        margin: 0 auto;
        background: var(--light);
        padding: 16px;
        border-radius: 12px;
        box-shadow: 0 2px 12px rgba(0,0,0,0.08);
    }

    .top-grid {
        display: grid;
        grid-template-columns: 1fr;
        gap: 16px;
        margin-bottom: 20px;
    }
    @media (min-width: 640px) {
        .top-grid { grid-template-columns: 1fr 1fr; }
    }

    .box {
        background: #fff;
        padding: 16px;
        border-radius: 10px;
        border: 1px solid #e0e0e0;
    }

    .controls-bar {
        display: flex;
        flex-direction: column;
        gap: 12px;
        background: #f0f2f5;
        padding: 12px;
        border-radius: 10px;
        margin-bottom: 16px;
    }
    @media (min-width: 500px) {
        .controls-bar { flex-direction: row; justify-content: space-between; align-items: center; }
    }

    .controls-bar > div {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
    }

    input, select, button {
        padding: 10px 12px;
        border: 1px solid #ccc;
        border-radius: 8px;
        font-size: 15px;
        min-height: 44px;       /* touch-friendly */
        touch-action: manipulation;
    }
    button.btn {
        background: var(--blue);
        color: white;
        border: none;
        font-weight: 600;
        cursor: pointer;
    }
    button.btn-green {
        background: #28a745;
        font-size: 13px;
        padding: 6px 10px;
        min-height: 32px;
    }

    table {
        width: 100%;
        border-collapse: collapse;
        background: white;
        font-size: 14px;
    }
    th, td {
        border: 1px solid #ddd;
        padding: 8px;
        text-align: center;
        min-width: 60px;
    }
    th {
        background: var(--blue);
        color: white;
        position: sticky;
        top: 0;
        z-index: 10;
    }
    .weekend { background-color: #fff5f5; }
    .real-col {
        background-color: #f0f8ff;
        cursor: pointer;
        font-weight: 600;
    }
    .has-note { background-color: #fffacd !important; }

    /* Responsive table wrapper */
    .table-wrapper {
        overflow-x: auto;
        -webkit-overflow-scrolling: touch;
    }

    #modal {
        display: none;
        position: fixed;
        inset: 0;
        background: rgba(0,0,0,0.6);
        z-index: 1000;
        align-items: center;
        justify-content: center;
        padding: 16px;
    }
    .modal-content {
        background: white;
        padding: 20px;
        border-radius: 12px;
        width: 100%;
        max-width: 460px;
        max-height: 90vh;
        overflow-y: auto;
    }
    textarea {
        width: 100%;
        height: 100px;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 8px;
        resize: vertical;
        font-size: 15px;
    }

    @media print {
        .no-print, .top-grid, .controls-bar, #modal { display: none !important; }
        body { margin: 0; background: white; }
        .container { box-shadow: none; padding: 0; }
        table { font-size: 11px; }
    }
</style>
</head>
<body>

<div class="container">

    <div class="top-grid no-print">
        <div class="box">
            <h3>📍 რეალური საათები</h3>
            <input type="text" id="rName" placeholder="ექთანი" list="nList">
            <datalist id="nList"></datalist>
            <select id="rDay"></select>
            <select id="rHrs">
                <option value="0">0</option>
                <option value="8">8</option>
                <option value="16">16</option>
                <option value="24">24</option>
            </select>
            <button class="btn" onclick="saveReal()">შენახვა</button>
        </div>

        <div class="box">
            <h3>🔍 ძებნა</h3>
            <input type="text" id="sName" placeholder="სახელი">
            <button class="btn" onclick="search()">ძებნა</button>
            <div id="sOut" style="margin-top:12px; color:var(--blue); font-weight:600;"></div>
        </div>
    </div>

    <div class="controls-bar no-print">
        <div>
            <input type="text" id="nurseInp" placeholder="ახალი ექთანი..." style="flex:1;">
            <button class="btn" onclick="addNurse()">+ დამატება</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2026" min="2020" max="2035" style="width:90px;" onchange="load()">
            <button class="btn" style="background:#555;" onclick="window.print()">ბეჭდვა</button>
        </div>
    </div>

    <div class="table-wrapper" id="tableBox">
        <p style="text-align:center; padding:60px 20px; color:#666;">
            დაამატეთ ერთი ან რამდენიმე ექთანი ზემოთ, რომ დაიწყოთ.
        </p>
    </div>

</div>

<div id="modal" onclick="event.target===this && closeModal()">
    <div class="modal-content">
        <h3 id="modalTitle">დღის ჩანაწერი</h3>
        <div id="expDiv">
            <label><b>🛠 კლინიკური გამოცდილება / შემთხვევები:</b></label>
            <textarea id="noteExp" placeholder="რა იყო საინტერესო ან რთული დღეს..."></textarea>
        </div>
        <label><b>📖 დღიური / პირადი შენიშვნები:</b></label>
        <textarea id="noteDaily" placeholder="როგორ გრძნობ თავს? რა გინდა გააკეთო მომავალში..."></textarea>
        <button class="btn" onclick="saveNote()" style="width:100%; margin:12px 0 4px;">შენახვა</button>
        <button onclick="closeModal()" style="width:100%; background:#6c757d; color:white;">დახურვა</button>
    </div>
</div>

<script>
// ────────────────────────────────────────────────
const firebaseConfig = {
    apiKey: "AIzaSyB3roORgMRtg6mmAyH3rUQmzmyAfc_ud6U",
    authDomain: "ertimecmc.firebaseapp.com",
    databaseURL: "https://ertimecmc-default-rtdb.europe-west1.firebasedatabase.app",
    projectId: "ertimecmc",
    storageBucket: "ertimecmc.firebasestorage.app",
    messagingSenderId: "164048857022",
    appId: "1:164048857022:web:359061cf694057bc16bbef"
};

firebase.initializeApp(firebaseConfig);
const db = firebase.database();

let nurses = [], scheduleData = {}, activeCell = null;
const months = ["იანვარი","თებერვალი","მარტი","აპრილი","მაისი","ივნისი","ივლისი","აგვისტო","სექტემბერი","ოქტომბერი","ნოემბერი","დეკემბერი"];

// ────────────────────────────────────────────────
db.ref().on('value', snap => {
    const val = snap.val() || {};
    nurses = val.nurses || [];
    scheduleData = val.scheduleData || {};
    load();
});

window.onload = () => {
    const mSel = document.getElementById('mSel');
    months.forEach((m,i) => mSel.innerHTML += `<option value="${i}">${m}</option>`);
    mSel.value = new Date().getMonth();
    document.getElementById('ySel').value = new Date().getFullYear();
    load();
};

// ────────────────────────────────────────────────
function addNurse() {
    const name = document.getElementById('nurseInp').value.trim();
    if (!name) return alert("შეიყვანეთ სახელი");
    if (nurses.includes(name)) return alert("ეს სახელი უკვე არსებობს");
    
    nurses.push(name);
    load();
    db.ref('nurses').set(nurses).then(() => {
        document.getElementById('nurseInp').value = '';
    }).catch(err => alert("შეცდომა: " + err.message));
}

function load() {
    const m = parseInt(document.getElementById('mSel').value);
    const y = parseInt(document.getElementById('ySel').value);
    const daysInMonth = new Date(y, m+1, 0).getDate();

    if (nurses.length === 0) {
        document.getElementById('tableBox').innerHTML = 
            '<p style="text-align:center; padding:60px 20px; color:#666;">ექთნების სია ცარიელია. დაამატეთ ზემოთ.</p>';
        return;
    }

    let h = `<table><thead><tr><th rowspan="2">რიცხვი</th>`;
    nurses.forEach(n => {
        h += `<th colspan="2" style="position:relative;">
                ${n} <span class="no-print" style="cursor:pointer;font-size:0.9em;" onclick="delNurse('${n}')">×</span><br>
                <button class="btn btn-green no-print auto-fill-btn" 
                        data-nurse="${n}"
                        onclick="autoFillByFirst('${n}')">წლის შევსება</button>
              </th>`;
    });
    h += `</tr><tr>`;
    nurses.forEach(() => h += `<th>გეგმა</th><th>რეალური</th>`);
    h += `</tr></thead><tbody>`;

    for (let d = 1; d <= daysInMonth; d++) {
        const isWeekend = [0,6].includes(new Date(y,m,d).getDay());
        h += `<tr class="${isWeekend?'weekend':''}"><td><b>${d}</b></td>`;
        nurses.forEach(n => {
            const pVal = (scheduleData[`${y}-${m}-${n}-p`]?.[d]) || 0;
            const rVal = (scheduleData[`${y}-${m}-${n}-r`]?.[d]) || 0;
            const hasNote = !!(scheduleData[`${y}-${m}-${n}-notes`]?.[d]);

            h += `<td>
                    <select onchange="upd('${n}',${d},this.value,'p')">
                        <option value="0" ${pVal==0?'selected':''}>-</option>
                        <option value="8"  ${pVal==8?'selected':''}>8</option>
                        <option value="16" ${pVal==16?'selected':''}>16</option>
                        <option value="24" ${pVal==24?'selected':''}>24</option>
                    </select>
                  </td>
                  <td class="real-col ${hasNote?'has-note':''}" 
                      onclick="openJournal('${n}',${d},${pVal},${rVal})">
                    ${rVal || (hasNote ? '📝' : '—')}
                  </td>`;
        });
        h += `</tr>`;
    }
    document.getElementById('tableBox').innerHTML = `<div class="table-wrapper">${h}</tbody></table></div>`;
    updateDLists(daysInMonth);
    updateAutoFillTooltips();
}

function updateAutoFillTooltips() {
    document.querySelectorAll('.auto-fill-btn').forEach(btn => {
        const nurse = btn.dataset.nurse;
        const m = document.getElementById('mSel').value;
        const y = document.getElementById('ySel').value;
        const key = `${y}-${m}-${nurse}-p`;
        const days = Object.keys(scheduleData[key] || {}).filter(d=>scheduleData[key][d]>0).sort((a,b)=>a-b);
        if (days.length === 0) {
            btn.title = "ჯერ არ არის არჩეული არც ერთი სამუშაო დღე გეგმაში";
            btn.textContent = "წლის შევსება";
            btn.disabled = true;
        } else {
            const startDay = days[0];
            const hrs = scheduleData[key][startDay];
            btn.title = `შეავსებს ${hrs}-საათიანი ცვლილებით ${startDay} რიცხვიდან ყოველ 4 დღეში ერთხელ მთელი წლის განმავლობაში`;
            btn.textContent = `შევსება (${hrs}ს) ${startDay}-დან`;
            btn.disabled = false;
        }
    });
}

function upd(n, d, v, t) {
    const m = document.getElementById('mSel').value;
    const y = document.getElementById('ySel').value;
    db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v)).then(() => {
        if (t === 'p') updateAutoFillTooltips();
    });
}

function saveReal() {
    const n = document.getElementById('rName').value.trim();
    const d = document.getElementById('rDay').value;
    const h = parseInt(document.getElementById('rHrs').value);
    const m = document.getElementById('mSel').value;
    const y = document.getElementById('ySel').value;

    if (!n || !d) return alert("აირჩიეთ ექთანი და დღე");
    db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(h);
}

function openJournal(nurse, day, plan, real) {
    activeCell = { nurse, day, m: document.getElementById('mSel').value, y: document.getElementById('ySel').value };
    document.getElementById('modalTitle').textContent = `${nurse} — ${day} ${months[activeCell.m]}`;
    document.getElementById('expDiv').style.display = (plan > 0 || real > 0) ? 'block' : 'none';

    const notePath = `${activeCell.y}-${activeCell.m}-${nurse}-notes`;
    const notes = (scheduleData[notePath]?.[day]) || {exp:"", daily:""};
    document.getElementById('noteExp').value   = notes.exp   || "";
    document.getElementById('noteDaily').value = notes.daily || "";

    document.getElementById('modal').style.display = 'flex';
}

function saveNote() {
    const { nurse, day, m, y } = activeCell;
    const exp   = document.getElementById('noteExp').value;
    const daily = document.getElementById('noteDaily').value;
    db.ref(`scheduleData/${y}-${m}-${nurse}-notes/${day}`).set({ exp, daily })
      .then(closeModal);
}

function closeModal() {
    document.getElementById('modal').style.display = 'none';
}

function updateDLists(daysInMonth) {
    let opt = "";
    for(let i=1; i<=daysInMonth; i++) opt += `<option value="${i}">${i}</option>`;
    document.getElementById('rDay').innerHTML = opt;

    let list = "";
    nurses.forEach(x => list += `<option value="${x}">`);
    document.getElementById('nList').innerHTML = list;
}

function delNurse(n) {
    if (!confirm(`ნამდვილად გსურთ ${n}-ის წაშლა?`)) return;
    nurses = nurses.filter(x => x !== n);
    db.ref('nurses').set(nurses);
}

async function autoFillByFirst(nurseName) {
    const m = parseInt(document.getElementById('mSel').value);
    const y = parseInt(document.getElementById('ySel').value);
    const pKey = `${y}-${m}-${nurseName}-p`;
    
    const filledDays = Object.keys(scheduleData[pKey] || {})
                           .filter(d => scheduleData[pKey][d] > 0)
                           .map(Number)
                           .sort((a,b)=>a-b);
    
    if (filledDays.length === 0) return alert("ჯერ მონიშნეთ თუნდაც ერთი სამუშაო დღე გეგმაში!");

    const startDay = filledDays[0];
    const hours = scheduleData[pKey][startDay];

    if (!confirm(`გსურთ ${hours}-საათიანი გრაფიკის გაგრძელება ${startDay} რიცხვიდან მთელი ${y} წლის განმავლობაში (ყოველ 4 დღეში)?`)) {
        return;
    }

    let updates = {};
    let curr = new Date(y, m, startDay);
    while (curr.getFullYear() === y) {
        const key = `scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${nurseName}-p/${curr.getDate()}`;
        updates[key] = hours;
        curr.setDate(curr.getDate() + 4);
    }
    await db.ref().update(updates);
    alert(`გრაფიკი შეივსო ${hours} საათით ${startDay}-დან!`);
}

function search() {
    const name = document.getElementById('sName').value.trim();
    if (!name) return;
    const m = document.getElementById('mSel').value;
    const y = document.getElementById('ySel').value;
    const days = Object.keys(scheduleData[`${y}-${m}-${name}-p`] || {})
                       .filter(d => scheduleData[`${y}-${m}-${name}-p`][d] > 0)
                       .sort((a,b)=>a-b);
    document.getElementById('sOut').textContent = 
        days.length ? `${name} → ${days.join(', ')}` : `${name} - ამ თვეში გეგმა არ აქვს`;
}
</script>
</body>
</html>
