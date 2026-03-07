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
        .container { max-width: 1200px; margin: auto; background: #fff; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        /* ცხრილის სტილი */
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ccc; padding: 8px; text-align: center; }
        th { background: var(--blue); color: white; position: sticky; top: 0; }
        .weekend { background-color: #fff0f0 !important; }
        .real-cell { background-color: #f9f9f9; cursor: pointer; font-weight: bold; color: var(--blue); }
        .real-cell:hover { background-color: #e3f2fd; }
        .has-note { background-color: #fff3cd !important; }

        /* მართვის პანელი */
        .controls-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; background: #eee; padding: 15px; border-radius: 8px; gap: 10px; flex-wrap: wrap; }
        .btn { padding: 10px 15px; border: none; border-radius: 6px; cursor: pointer; color: white; font-weight: bold; background: var(--blue); }
        .btn-red { background: var(--red); }
        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }

        /* ჟურნალის ფანჯარა (Modal) */
        #modal { display:none; position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:white; padding:25px; box-shadow:0 0 20px rgba(0,0,0,0.5); z-index:1000; border-radius:12px; width:400px; }
        #modal h3 { margin-top:0; color: var(--blue); }
        .modal-overlay { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.6); z-index:999; }
        textarea { width:100%; height:80px; margin-top:10px; margin-bottom:10px; padding:10px; border:1px solid #ddd; border-radius:5px; resize:none; }
        
        @media print { .no-print { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="controls-bar no-print">
        <div>
            <input type="text" id="nurseInp" placeholder="ექთნის სახელი">
            <button class="btn" onclick="addNurse()">დამატება</button>
            <button class="btn btn-red" onclick="bulkFillYear()">წლიური ავტო-შევსება (1/4)</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 80px;" onchange="load()">
            <button class="btn" style="background:#555" onclick="window.print()">ბეჭდვა</button>
        </div>
    </div>

    <div id="tableBox">ცარიელია...</div>
</div>

<div class="modal-overlay" id="overlay" onclick="closeModal()"></div>
<div id="modal">
    <h3 id="modalTitle">დღის ჩანაწერი</h3>
    <div id="expDiv">
        <label><b>სამუშაო გამოცდილება:</b></label>
        <textarea id="noteExp" placeholder="აღწერეთ შემთხვევა..."></textarea>
    </div>
    <label><b>დღიური / პირადი:</b></label>
    <textarea id="noteDaily" placeholder="ჩაწერეთ შენიშვნა..."></textarea>
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
        document.getElementById('ySel').value = new Date().getFullYear();
        load();
    };

    function load() {
        const m = parseInt(document.getElementById('mSel').value);
        const y = parseInt(document.getElementById('ySel').value);
        const daysInMonth = new Date(y, m + 1, 0).getDate();
        
        let h = `<table><thead><tr><th rowspan="2">რიცხვი</th>`;
        nurses.forEach(n => h += `<th colspan="2">${n} <span class="no-print" style="cursor:pointer" onclick="delNurse('${n}')">❌</span></th>`);
        h += `</tr><tr>`;
        nurses.forEach(() => h += `<th>გეგმა</th><th>რეალური</th>`);
        h += `</tr></thead><tbody>`;

        for (let d = 1; d <= daysInMonth; d++) {
            const isWknd = [0, 6].includes(new Date(y, m, d).getDay());
            h += `<tr class="${isWknd ? 'weekend' : ''}"><td><b>${d}</b></td>`;
            
            nurses.forEach(n => {
                const pVal = (scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][d]) || 0;
                const rVal = (scheduleData[`${y}-${m}-${n}-r`] && scheduleData[`${y}-${m}-${n}-r`][d]) || 0;
                const hasNote = (scheduleData[`${y}-${m}-${n}-notes`] && scheduleData[`${y}-${m}-${n}-notes`][d]);

                h += `<td>
                    <select onchange="upd('${n}',${d},this.value,'p')">
                        <option value="0" ${pVal==0?'selected':''}>-</option>
                        <option value="8" ${pVal==8?'selected':''}>8</option>
                        <option value="16" ${pVal==16?'selected':''}>16</option>
                        <option value="24" ${pVal==24?'selected':''}>24</option>
                    </select>
                </td>
                <td class="real-cell ${hasNote ? 'has-note' : ''}" onclick="openJournal('${n}',${d},${pVal})">
                    ${rVal || '-'}
                </td>`;
            });
            h += `</tr>`;
        }
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
    }

    function upd(n, d, v, t) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function openJournal(nurse, day, plan) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        activeCell = { nurse, day, m, y };
        
        document.getElementById('modalTitle').innerText = `${nurse} - ${day} ${months[m]}`;
        
        // თუ მორიგეა (plan > 0), ვაჩვენებთ გამოცდილების ველს
        document.getElementById('expDiv').style.display = (plan > 0) ? 'block' : 'none';
        
        // მონაცემების ამოღება
        const existing = (scheduleData[`${y}-${m}-${nurse}-notes`] && scheduleData[`${y}-${m}-${nurse}-notes`][day]) || {exp:"", daily:""};
        document.getElementById('noteExp').value = existing.exp || "";
        document.getElementById('noteDaily').value = existing.daily || "";

        document.getElementById('modal').style.display = 'block';
        document.getElementById('overlay').style.display = 'block';
    }

    function saveNote() {
        const { nurse, day, m, y } = activeCell;
        const exp = document.getElementById('noteExp').value;
        const daily = document.getElementById('noteDaily').value;
        
        db.ref(`scheduleData/${y}-${m}-${nurse}-notes/${day}`).set({ exp, daily });
        
        // თუ რეალურში არაფერი წერია, ავტომატურად ჩავწეროთ გეგმიური საათი
        const pVal = (scheduleData[`${y}-${m}-${nurse}-p`] && scheduleData[`${y}-${m}-${nurse}-p`][day]) || 0;
        db.ref(`scheduleData/${y}-${m}-${nurse}-r/${day}`).set(pVal);
        
        closeModal();
    }

    function closeModal() {
        document.getElementById('modal').style.display = 'none';
        document.getElementById('overlay').style.display = 'none';
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { 
            nurses.push(v); 
            db.ref('nurses').set(nurses); 
            document.getElementById('nurseInp').value=''; 
        }
    }

    function delNurse(n) { if(confirm('წავშალოთ ' + n + '?')) { nurses = nurses.filter(x => x !== n); db.ref('nurses').set(nurses); } }

    async function bulkFillYear() {
        const n = prompt("ჩაწერეთ სახელი:");
        if(!nurses.includes(n)) return alert("ვერ მოიძებნა");
        const startDay = parseInt(prompt("საწყისი რიცხვი:", "1"));
        const yr = document.getElementById('ySel').value;
        
        let curr = new Date(yr, document.getElementById('mSel').value, startDay);
        let updates = {};
        while(curr.getFullYear() == yr) {
            updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${n}-p/${curr.getDate()}`] = 24;
            curr.setDate(curr.getDate() + 4);
        }
        await db.ref().update(updates);
        alert("შეივსო წლის ბოლომდე!");
    }
</script>
</body>
</html>
