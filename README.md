<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ErtimeCMC - სტაბილური ვერსია</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --red: #8b0000; --bg: #f4f7f6; }
        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: var(--bg); color: #333; }
        .container { max-width: 1400px; margin: auto; background: #fff; padding: 25px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.1); }
        
        /* ზედა პანელი */
        .top-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 25px; }
        .box { padding: 20px; background: #fdfdfd; border-radius: 10px; border: 1px solid #e0e0e0; box-shadow: inset 0 0 5px rgba(0,0,0,0.02); }
        h3 { margin-top: 0; color: var(--blue); border-bottom: 2px solid #eee; padding-bottom: 8px; }

        /* მართვის ზოლი */
        .controls-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; background: #f0f0f0; padding: 15px; border-radius: 8px; flex-wrap: wrap; gap: 10px; }
        
        /* ცხრილი */
        .table-wrapper { overflow-x: auto; border: 1px solid #ddd; border-radius: 8px; }
        table { width: 100%; border-collapse: collapse; background: white; min-width: 800px; }
        th, td { border: 1px solid #ccc; padding: 10px; text-align: center; }
        th { background: var(--blue); color: white; font-size: 14px; position: sticky; top: 0; }
        
        .weekend { background-color: #fff5f5 !important; }
        .real-col { background-color: #f8fbff; cursor: pointer; transition: 0.2s; font-weight: bold; border-left: 2px solid #e3f2fd; }
        .real-col:hover { background-color: #e3f2fd; }
        .has-note { background-color: #fff9db !important; position: relative; }
        .has-note::after { content: '📝'; position: absolute; top: 2px; right: 2px; font-size: 10px; }

        /* ღილაკები და ინპუტები */
        .btn { padding: 10px 18px; border: none; border-radius: 6px; cursor: pointer; color: white; font-weight: bold; background: var(--blue); transition: 0.3s; }
        .btn:hover { opacity: 0.8; }
        .btn-green { background: #28a745; font-size: 11px; padding: 5px 10px; margin-top: 5px; }
        input, select { padding: 10px; border: 1px solid #ccc; border-radius: 6px; font-size: 14px; }

        /* მოდალური ფანჯარა */
        #modal { display:none; position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:white; padding:30px; box-shadow:0 0 40px rgba(0,0,0,0.6); z-index:1000; border-radius:15px; width:480px; }
        .modal-overlay { display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.7); z-index:999; }
        textarea { width:100%; height:110px; margin: 10px 0; padding:12px; border: 1px solid #ddd; border-radius: 8px; box-sizing: border-box; resize: none; font-size: 14px; }
        
        @media print { .no-print, .top-grid, .box, .btn-green { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-grid no-print">
        <div class="box">
            <h3>📍 რეალური საათის დაფიქსირება</h3>
            <div style="display: flex; gap: 10px; flex-wrap: wrap;">
                <input type="text" id="rName" placeholder="ექთნის სახელი" list="nList" style="flex: 1;">
                <datalist id="nList"></datalist>
                <select id="rDay" style="width: 80px;"></select>
                <select id="rHrs" style="width: 80px;">
                    <option value="24">24</option>
                    <option value="16">16</option>
                    <option value="8">8</option>
                    <option value="0">0</option>
                </select>
                <button class="btn" onclick="saveReal()">შენახვა</button>
            </div>
        </div>
        <div class="box">
            <h3>🔍 ძებნა</h3>
            <div style="display: flex; gap: 10px;">
                <input type="text" id="sName" placeholder="შეიყვანეთ სახელი" style="flex: 1;">
                <button class="btn" onclick="search()">ძებნა</button>
            </div>
            <div id="sOut" style="margin-top:12px; font-weight:bold; color: var(--blue);"></div>
        </div>
    </div>

    <div class="controls-bar no-print">
        <div>
            <input type="text" id="nurseInp" placeholder="ახალი ექთნის სახელი">
            <button class="btn" onclick="addNurse()">+ დამატება</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2026" style="width: 90px;" onchange="load()">
            <button class="btn" style="background:#666" onclick="window.print()">🖨 ბეჭდვა</button>
        </div>
    </div>

    <div id="tableBox" class="table-wrapper">
        <p style="padding: 20px; text-align: center;">ცხრილი მზადდება...</p>
    </div>
</div>

<div class="modal-overlay" id="overlay" onclick="closeModal()"></div>
<div id="modal">
    <h3 id="modalTitle" style="margin-bottom: 15px;">დღის ჩანაწერი</h3>
    <div id="expDiv">
        <label>🩺 <b>სამუშაო გამოცდილება:</b></label>
        <textarea id="noteExp" placeholder="ჩაწერეთ კლინიკური შემთხვევები..."></textarea>
    </div>
    <label>📓 <b>დღიური / პირადი:</b></label>
    <textarea id="noteDaily" placeholder="როგორ ჩაიარა დღემ?"></textarea>
    <button class="btn" onclick="saveNote()" style="width:100%; padding:15px; margin-top:10px;">💾 შენახვა</button>
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

    // მონაცემების მოსმენა
    db.ref().on('value', snap => {
        const val = snap.val() || {};
        nurses = val.nurses || [];
        scheduleData = val.scheduleData || {};
        load();
    }, error => console.error("Firebase error:", error));

    window.onload = () => {
        const mSel = document.getElementById('mSel');
        months.forEach((m, i) => mSel.innerHTML += `<option value="${i}">${m}</option>`);
        mSel.value = new Date().getMonth();
        document.getElementById('ySel').value = new Date().getFullYear();
        load(); // პირველადი დახატვა
    };

    function load() {
        const m = parseInt(document.getElementById('mSel').value) || 0;
        const y = parseInt(document.getElementById('ySel').value) || 2026;
        const daysInMonth = new Date(y, m + 1, 0).getDate();
        
        let html = `<table><thead><tr><th rowspan="2" style="width:60px">რიცხვი</th>`;
        
        if (nurses.length === 0) {
            html += `<th>ჯერ დაამატეთ ექთანი</th></tr></thead><tbody><tr><td colspan="2">ექთნების სია ცარიელია</td></tr>`;
        } else {
            nurses.forEach(n => {
                html += `<th colspan="2">
                    ${n} <span class="no-print" style="cursor:pointer; color:#ff9999" onclick="delNurse('${n}')"> (წაშლა)</span><br>
                    <button class="btn btn-green no-print" onclick="autoFillByFirst('${n}')">წლის შევსება</button>
                </th>`;
            });
            html += `</tr><tr>`;
            nurses.forEach(() => html += `<th>გეგმა</th><th>რეალური</th>`);
            html += `</tr></thead><tbody>`;

            for (let d = 1; d <= daysInMonth; d++) {
                const dateObj = new Date(y, m, d);
                const isWknd = [0, 6].includes(dateObj.getDay());
                html += `<tr class="${isWknd ? 'weekend' : ''}"><td><b>${d}</b></td>`;
                
                nurses.forEach(n => {
                    const pPath = `${y}-${m}-${n}-p`;
                    const rPath = `${y}-${m}-${n}-r`;
                    const nPath = `${y}-${m}-${n}-notes`;
                    
                    const pVal = (scheduleData[pPath] && scheduleData[pPath][d]) || 0;
                    const rVal = (scheduleData[rPath] && scheduleData[rPath][d]) || 0;
                    const note = (scheduleData[nPath] && scheduleData[nPath][d]);

                    html += `<td>
                        <select onchange="upd('${n}',${d},this.value,'p')">
                            <option value="0" ${pVal==0?'selected':''}>-</option>
                            <option value="8" ${pVal==8?'selected':''}>8</option>
                            <option value="16" ${pVal==16?'selected':''}>16</option>
                            <option value="24" ${pVal==24?'selected':''}>24</option>
                        </select>
                    </td>
                    <td class="real-col ${note ? 'has-note' : ''}" onclick="openJournal('${n}',${d},${pVal},${rVal})">
                        ${rVal > 0 ? rVal : (note ? '...' : '-')}
                    </td>`;
                });
                html += `</tr>`;
            }
        }
        
        document.getElementById('tableBox').innerHTML = html + `</tbody></table>`;
        updateDLists(daysInMonth);
    }

    function saveReal() {
        const n = document.getElementById('rName').value, d = document.getElementById('rDay').value, h = document.getElementById('rHrs').value;
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        if(!n) return alert("აირჩიეთ ექთანი!");
        db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(parseInt(h));
    }

    function upd(n, d, v, t) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function openJournal(nurse, day, plan, real) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        activeCell = { nurse, day, m, y };
        document.getElementById('modalTitle').innerText = `${nurse} (${day} ${months[m]})`;
        
        // გამოცდილების ველი ჩანს თუ გეგმაში ან რეალურში საათებია
        document.getElementById('expDiv').style.display = (plan > 0 || real > 0) ? 'block' : 'none';
        
        const notePath = `${y}-${m}-${nurse}-notes`;
        const notes = (scheduleData[notePath] && scheduleData[notePath][day]) || {exp:"", daily:""};
        
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
        let opt = ""; for(let i=1; i<=d; i++) opt += `<option value="${i}">${i}</option>`;
        document.getElementById('rDay').innerHTML = opt;
        let list = ""; nurses.forEach(x => list += `<option value="${x}">`);
        document.getElementById('nList').innerHTML = list;
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) {
            nurses.push(v);
            db.ref('nurses').set(nurses);
            document.getElementById('nurseInp').value = '';
        }
    }

    function delNurse(n) {
        if(confirm(`წავშალოთ ${n}?`)) {
            nurses = nurses.filter(x => x !== n);
            db.ref('nurses').set(nurses);
        }
    }

    async function autoFillByFirst(nurseName) {
        const m = parseInt(document.getElementById('mSel').value), y = parseInt(document.getElementById('ySel').value);
        const pKey = `${y}-${m}-${nurseName}-p`;
        let days = Object.keys(scheduleData[pKey] || {}).filter(d => scheduleData[pKey][d] > 0).sort((a,b)=>a-b);
        if(days.length === 0) return alert("ჯერ მონიშნეთ პირველი მორიგეობა (გეგმაში)!");
        
        const firstDay = parseInt(days[0]), hrs = scheduleData[pKey][firstDay];
        if(!confirm(`შევავსო ყოველი მე-4 დღე (${hrs} სთ) წლის ბოლომდე?`)) return;

        let curr = new Date(y, m, firstDay), updates = {};
        while(curr.getFullYear() == y) {
            updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${nurseName}-p/${curr.getDate()}`] = hrs;
            curr.setDate(curr.getDate() + 4);
        }
        await db.ref().update(updates);
        alert("გრაფიკი შეივსო!");
    }

    function closeModal() {
        document.getElementById('modal').style.display = 'none';
        document.getElementById('overlay').style.display = 'none';
    }

    function search() {
        const sn = document.getElementById('sName').value, m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        if(!sn) return;
        const pKey = `${y}-${m}-${sn}-p`;
        const days = scheduleData[pKey] ? Object.keys(scheduleData[pKey]).filter(d => scheduleData[pKey][d] > 0) : [];
        document.getElementById('sOut').innerHTML = days.length ? `${sn}-ს მორიგეობები: ${days.join(', ')}` : "მორიგეობა ვერ მოიძებნა.";
    }
</script>
</body>
</html>
