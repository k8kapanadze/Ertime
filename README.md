<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ErtimeCMC - Live სისტემა</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --red: #8b0000; --bg: #f4f7f6; }
        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: var(--bg); }
        .container { max-width: 1300px; margin: auto; background: #fff; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        /* ზედა პანელი */
        .top-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
        .box { padding: 15px; background: #fafafa; border-radius: 8px; border: 1px solid #ddd; }
        
        .controls-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; background: #eee; padding: 12px; border-radius: 8px; }
        
        /* ცხრილი */
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ccc; padding: 8px; text-align: center; }
        th { background: var(--blue); color: white; position: sticky; top: 0; z-index: 10; font-size: 14px; }
        
        .weekend { background-color: #fff0f0 !important; }
        .real-col { background-color: #f9f9f9; cursor: pointer; font-weight: bold; color: var(--blue); }
        .has-note { background-color: #fff3cd !important; border-left: 3px solid #ff9800 !important; }

        .btn { padding: 8px 12px; border: none; border-radius: 5px; cursor: pointer; color: white; font-weight: bold; background: var(--blue); }
        .btn-red { background: var(--red); }
        input, select { padding: 6px; border: 1px solid #ccc; border-radius: 4px; }

        /* ჟურნალის ფანჯარა */
        #modal { display:none; position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:white; padding:25px; box-shadow:0 0 30px rgba(0,0,0,0.5); z-index:1000; border-radius:12px; width:450px; }
        .modal-overlay { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.7); z-index:999; }
        textarea { width:100%; height:100px; margin: 10px 0; padding:10px; border:1px solid #ddd; border-radius:5px; box-sizing: border-box; resize: none; }
        
        @media print { .no-print, .top-grid { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-grid no-print">
        <div class="box">
            <h3 style="margin-top:0">საათის დაფიქსირება (რეალური)</h3>
            <input type="text" id="rName" placeholder="ექთნის სახელი" list="nList">
            <datalist id="nList"></datalist>
            <select id="rDay"></select>
            <select id="rHrs">
                <option value="24">24</option>
                <option value="16">16</option>
                <option value="8">8</option>
                <option value="0">0</option>
            </select>
            <button class="btn" onclick="saveReal()">შენახვა</button>
        </div>
        <div class="box">
            <h3 style="margin-top:0">ძებნა</h3>
            <input type="text" id="sName" placeholder="სახელი">
            <button class="btn" onclick="search()">ძებნა</button>
            <div id="sOut" style="margin-top:10px; font-weight:bold; color: var(--blue);"></div>
        </div>
    </div>

    <div class="controls-bar no-print">
        <div>
            <input type="text" id="nurseInp" placeholder="ახალი ექთანი">
            <button class="btn" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 80px;" onchange="load()">
            <button class="btn" style="background:#555" onclick="window.print()">ბეჭდვა</button>
        </div>
    </div>

    <div id="tableBox" style="overflow-x: auto;">ჩაიტვირთება...</div>
</div>

<div class="modal-overlay" id="overlay" onclick="closeModal()"></div>
<div id="modal">
    <h3 id="modalTitle" style="margin:0; color:var(--blue)">მორიგეობის ჟურნალი</h3>
    <hr>
    <div id="expDiv">
        <label>🛠 <b>სამუშაო გამოცდილება:</b></label>
        <textarea id="noteExp" placeholder="აღწერეთ შემთხვევები..."></textarea>
    </div>
    <label>📖 <b>დღიური / პირადი შენიშვნა:</b></label>
    <textarea id="noteDaily" placeholder="როგორ ჩაიარა დღემ?"></textarea>
    <button class="btn" onclick="saveNote()" style="width:100%; padding:12px; margin-top:10px">შენახვა</button>
</div>

<script>
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
    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    db.ref().on('value', snap => {
        const val = snap.val() || {};
        nurses = val.nurses || [];
        scheduleData = val.scheduleData || {};
        load();
    });

    window.onload = () => {
        const mSel = document.getElementById('mSel');
        months.forEach((m, i) => mSel.innerHTML += `<option value="${i}">${m}</option>`);
        mSel.value = new Date().getMonth();
        document.getElementById('ySel').value = new Date().getFullYear();
    };

    function load() {
        const m = parseInt(document.getElementById('mSel').value), y = parseInt(document.getElementById('ySel').value);
        const days = new Date(y, m + 1, 0).getDate();
        
        let h = `<table><thead><tr><th rowspan="2" style="width:40px">რიცხვი</th>`;
        nurses.forEach(n => {
            h += `<th colspan="2">
                    ${n} <span class="no-print" style="cursor:pointer" onclick="delNurse('${n}')">🗑</span><br>
                    <button class="btn no-print" style="background:#28a745; font-size:10px; padding:2px 5px" onclick="autoFillByFirst('${n}')">წლის შევსება</button>
                  </th>`;
        });
        h += `</tr><tr>`;
        nurses.forEach(() => h += `<th>გეგმა</th><th>რეალური</th>`);
        h += `</tr></thead><tbody>`;

        for (let d = 1; d <= days; d++) {
            const isWknd = [0, 6].includes(new Date(y, m, d).getDay());
            h += `<tr class="${isWknd ? 'weekend' : ''}"><td><b>${d}</b></td>`;
            nurses.forEach(n => {
                const pVal = (scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][d]) || 0;
                const rVal = (scheduleData[`${y}-${m}-${n}-r`] && scheduleData[`${y}-${m}-${n}-r`][d]) || 0;
                const note = (scheduleData[`${y}-${m}-${n}-notes`] && scheduleData[`${y}-${m}-${n}-notes`][d]);

                h += `<td>
                        <select onchange="upd('${n}',${d},this.value,'p')">
                            <option value="0" ${pVal==0?'selected':''}>-</option>
                            <option value="8" ${pVal==8?'selected':''}>8</option>
                            <option value="16" ${pVal==16?'selected':''}>16</option>
                            <option value="24" ${pVal==24?'selected':''}>24</option>
                        </select>
                      </td>
                      <td class="real-col ${note ? 'has-note' : ''}" onclick="openJournal('${n}',${d},${pVal},${rVal})">
                        ${rVal || (note ? '📝' : '-')}
                      </td>`;
            });
            h += `</tr>`;
        }
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
        updateDLists(days);
    }

    // ზედა პანელიდან რეალური საათის შენახვა
    function saveReal() {
        const n = document.getElementById('rName').value, d = document.getElementById('rDay').value, h = document.getElementById('rHrs').value;
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        if(n && d) db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(parseInt(h));
    }

    function upd(n, d, v, t) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function openJournal(nurse, day, plan, real) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        activeCell = { nurse, day, m, y };
        document.getElementById('modalTitle').innerText = `${nurse} (${day} ${months[m]})`;
        
        // გამოცდილება გამოჩნდება თუ გეგმაში ან რეალურში საათები ფიქსირდება
        document.getElementById('expDiv').style.display = (plan > 0 || real > 0) ? 'block' : 'none';
        
        const notes = (scheduleData[`${y}-${m}-${nurse}-notes`] && scheduleData[`${y}-${m}-${nurse}-notes`][day]) || {exp:"", daily:""};
        document.getElementById('noteExp').value = notes.exp || "";
        document.getElementById('noteDaily').value = notes.daily || "";
        
        document.getElementById('modal').style.display = 'block';
        document.getElementById('overlay').style.display = 'block';
    }

    function saveNote() {
        const { nurse, day, m, y } = activeCell;
        const exp = document.getElementById('noteExp').value;
        const daily = document.getElementById('noteDaily').value;
        db.ref(`scheduleData/${y}-${m}-${nurse}-notes/${day}`).set({ exp, daily });
        closeModal();
    }

    function updateDLists(d) {
        let o = ""; for(let i=1; i<=d; i++) o += `<option value="${i}">${i}</option>`;
        document.getElementById('rDay').innerHTML = o;
        let nl = ""; nurses.forEach(x => nl += `<option value="${x}">`);
        document.getElementById('nList').innerHTML = nl;
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { nurses.push(v); db.ref('nurses').set(nurses); document.getElementById('nurseInp').value=''; }
    }

    function delNurse(n) { if(confirm('წავშალოთ ' + n + '?')) { nurses = nurses.filter(x => x !== n); db.ref('nurses').set(nurses); } }

    async function autoFillByFirst(nurseName) {
        const m = parseInt(document.getElementById('mSel').value), y = parseInt(document.getElementById('ySel').value);
        const pKey = `${y}-${m}-${nurseName}-p`;
        let days = Object.keys(scheduleData[pKey] || {}).filter(d => scheduleData[pKey][d] > 0).sort((a,b)=>a-b);
        if(days.length === 0) return alert("ჯერ მონიშნეთ პირველი მორიგეობა!");
        const firstDay = parseInt(days[0]), hours = scheduleData[pKey][firstDay];
        let curr = new Date(y, m, firstDay), updates = {};
        while(curr.getFullYear() == y) {
            updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${nurseName}-p/${curr.getDate()}`] = hours;
            curr.setDate(curr.getDate() + 4);
        }
        await db.ref().update(updates);
        alert("შეივსო წლის ბოლომდე!");
    }

    function closeModal() { document.getElementById('modal').style.display = 'none'; document.getElementById('overlay').style.display = 'none'; }
</script>
</body>
</html>

