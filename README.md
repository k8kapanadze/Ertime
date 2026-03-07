<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>ErtimeCMC - Mobile Ready</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --bg: #f4f7f6; --white: #ffffff; }
        body { font-family: -apple-system, sans-serif; margin: 0; background: var(--bg); -webkit-tap-highlight-color: transparent; }
        
        .container { padding: 10px; padding-top: 15px; }
        
        /* მობილურზე მორგებული მართვის პანელი */
        .controls { display: flex; flex-direction: column; gap: 10px; margin-bottom: 15px; background: var(--white); padding: 15px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        .row { display: flex; gap: 8px; align-items: center; width: 100%; }
        
        /* ცხრილის კონტეინერი სკროლით */
        .table-wrapper { 
            overflow: auto; 
            max-height: 75vh; 
            border-radius: 12px; 
            border: 1px solid #ddd;
            background: var(--white);
        }
        
        table { border-collapse: separate; border-spacing: 0; width: 100%; }
        
        /* სტიკი ეფექტები მობილურისთვის */
        th, td { border: 0.5px solid #eee; padding: 12px 8px; text-align: center; min-width: 80px; }
        
        thead th { 
            position: sticky; top: 0; 
            background: var(--blue); color: white; 
            z-index: 10; font-size: 13px;
        }

        /* პირველი სვეტის (რიცხვის) გაყინვა */
        td:first-child, th:first-child { 
            position: sticky; left: 0; 
            background: #f8f9fa; z-index: 11; 
            min-width: 40px; font-weight: bold;
        }
        thead th:first-child { z-index: 12; background: var(--blue); }

        .weekend { background-color: #fff5f5 !important; }
        .real-cell { background-color: #f0f7ff; font-weight: bold; color: var(--blue); }
        .has-note::after { content: '•'; color: #ff9800; font-size: 20px; line-height: 0; vertical-align: middle; }

        /* ელემენტების ზომები თითისთვის */
        input, select, .btn { 
            height: 44px; /* Apple-ის სტანდარტი თითისთვის */
            border-radius: 8px; border: 1px solid #ccc; 
            font-size: 15px; padding: 0 10px;
            box-sizing: border-box;
        }
        .btn { 
            background: var(--blue); color: white; border: none; 
            font-weight: bold; white-space: nowrap; padding: 0 15px;
        }
        .btn-fill { 
            background: #28a745; height: 30px; font-size: 11px; 
            margin-top: 5px; width: 100%; 
        }

        /* მოდალური ფანჯარა მობილურისთვის */
        #modal { 
            display:none; position:fixed; top:50%; left:50%; 
            transform:translate(-50%, -50%); background:white; 
            padding:20px; width: 90%; max-width: 400px;
            z-index: 1000; border-radius: 15px; box-shadow: 0 0 50px rgba(0,0,0,0.3);
        }
        .overlay { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.6); z-index:999; }
        textarea { width: 100%; height: 100px; margin: 10px 0; border-radius: 8px; border: 1px solid #ddd; padding: 10px; font-family: inherit; }

        @media print { .no-print { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="controls no-print">
        <div class="row">
            <input type="text" id="nurseInp" placeholder="ექთნის სახელი" style="flex:1">
            <button class="btn" onclick="addNurse()">+</button>
        </div>
        <div class="row">
            <select id="mSel" onchange="load()" style="flex:2"></select>
            <input type="number" id="ySel" value="2026" style="flex:1" onchange="load()">
            <button class="btn" onclick="window.print()" style="background:#666">🖨</button>
        </div>
    </div>

    <div class="table-wrapper">
        <div id="tableBox">ჩაიტვირთება...</div>
    </div>
</div>

<div class="overlay" id="overlay" onclick="closeModal()"></div>
<div id="modal">
    <h3 id="modalTitle" style="margin-top:0">ჩანაწერი</h3>
    <div id="expDiv">
        <label>🛠 <b>გამოცდილება:</b></label>
        <textarea id="noteExp"></textarea>
    </div>
    <label>📖 <b>დღიური:</b></label>
    <textarea id="noteDaily"></textarea>
    <button class="btn" onclick="saveNote()" style="width:100%">შენახვა</button>
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
        load();
    };

    function load() {
        const m = parseInt(document.getElementById('mSel').value);
        const y = parseInt(document.getElementById('ySel').value);
        const days = new Date(y, m + 1, 0).getDate();
        
        let h = `<table><thead><tr><th>დღე</th>`;
        nurses.forEach(n => {
            h += `<th colspan="2">
                    ${n} <span onclick="delNurse('${n}')">🗑</span><br>
                    <button class="btn btn-fill" onclick="autoFillByFirst('${n}')">შევსება</button>
                  </th>`;
        });
        h += `</tr><tr><th>#</th>`;
        nurses.forEach(() => h += `<th>გეგმა</th><th>რეალ</th>`);
        h += `</tr></thead><tbody>`;

        for (let d = 1; d <= days; d++) {
            const isWknd = [0, 6].includes(new Date(y, m, d).getDay());
            h += `<tr class="${isWknd ? 'weekend' : ''}"><td>${d}</td>`;
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
                      <td class="real-cell ${note ? 'has-note' : ''}" onclick="openJournal('${n}',${d},${pVal})">
                        ${rVal || '-'}
                      </td>`;
            });
            h += `</tr>`;
        }
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
    }

    function upd(n, d, v, t) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        // ადგილობრივად განახლება, რომ "წლის შევსებამ" მაშინვე დაინახოს
        if(!scheduleData[`${y}-${m}-${n}-${t}`]) scheduleData[`${y}-${m}-${n}-${t}`] = {};
        scheduleData[`${y}-${m}-${n}-${t}`][d] = parseInt(v);
        db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));
    }

    async function autoFillByFirst(nurseName) {
        const m = parseInt(document.getElementById('mSel').value), y = parseInt(document.getElementById('ySel').value);
        const pKey = `${y}-${m}-${nurseName}-p`;
        
        const monthData = scheduleData[pKey] || {};
        let days = Object.keys(monthData).filter(d => monthData[d] > 0).sort((a,b)=>a-b);
        
        if(days.length === 0) return alert("ჯერ გეგმაში აირჩიეთ საათი (8, 16 ან 24) ნებისმიერ რიცხვში!");
        
        const firstDay = parseInt(days[0]), hrs = monthData[firstDay];
        if(!confirm(`შევავსო ყოველი მე-4 დღე (${hrs} სთ) წლის ბოლომდე?`)) return;

        let curr = new Date(y, m, firstDay), updates = {};
        while(curr.getFullYear() <= y) {
            updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${nurseName}-p/${curr.getDate()}`] = hrs;
            curr.setDate(curr.getDate() + 4);
        }
        await db.ref().update(updates);
    }

    function openJournal(n, d, p) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        activeCell = { n, d, m, y };
        document.getElementById('modalTitle').innerText = `${n} (${d} რიცხვი)`;
        document.getElementById('expDiv').style.display = p > 0 ? 'block' : 'none';
        const note = (scheduleData[`${y}-${m}-${n}-notes`] && scheduleData[`${y}-${m}-${n}-notes`][d]) || {exp:"", daily:""};
        document.getElementById('noteExp').value = note.exp || "";
        document.getElementById('noteDaily').value = note.daily || "";
        document.getElementById('modal').style.display = 'block';
        document.getElementById('overlay').style.display = 'block';
    }

    function saveNote() {
        const { n, d, m, y } = activeCell;
        const note = { exp: document.getElementById('noteExp').value, daily: document.getElementById('noteDaily').value };
        db.ref(`scheduleData/${y}-${m}-${n}-notes/${d}`).set(note);
        closeModal();
    }

    function closeModal() { document.getElementById('modal').style.display = 'none'; document.getElementById('overlay').style.display = 'none'; }
    function addNurse() { const v = document.getElementById('nurseInp').value.trim(); if(v) { nurses.push(v); db.ref('nurses').set(nurses); document.getElementById('nurseInp').value=''; } }
    function delNurse(n) { if(confirm('წავშალოთ ' + n + '?')) { nurses = nurses.filter(x => x !== n); db.ref('nurses').set(nurses); } }
</script>
</body>
</html>
