<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ექთნების მართვის სისტემა (Live)</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --primary-blue: #001f3f; --real-row-bg: #d1d8e0; --total-red: #8b0000; }
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        .top-bar { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 20px; gap: 15px; flex-wrap: wrap; }
        .main-controls { display: flex; gap: 10px; align-items: center; }
        .btn { padding: 10px 18px; border: none; border-radius: 4px; cursor: pointer; color: white; font-weight: bold; transition: 0.3s; }
        .btn-blue { background: var(--primary-blue); }
        .btn-green { background: #27ae60; }
        .btn-pdf { background: #2c3e50; }
        .table-container { overflow-x: auto; margin-top: 10px; }
        table { width: 100%; border-collapse: collapse; font-size: 13px; }
        th, td { border: 1px solid #ccc; text-align: center; padding: 6px; min-width: 35px; }
        th { background: #f8f9fa; }
        .nurse-cell { font-weight: bold; width: 250px; text-align: left; display: flex; align-items: center; gap: 10px; padding: 8px; }
        .menu-dots { cursor: pointer; margin-left: auto; font-size: 18px; }
        .real-row { background-color: var(--real-row-bg); }
        #contextMenu { display: none; position: absolute; background: white; border: 1px solid #ccc; box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 1000; padding: 5px 0; border-radius: 4px; }
        #contextMenu button { display: block; width: 100%; padding: 8px 15px; border: none; background: none; text-align: left; cursor: pointer; }
        .footer-forms { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 30px; border-top: 2px solid #eee; padding-top: 20px; }
        .form-box { background: #fdfdfd; padding: 15px; border-radius: 6px; border: 1px solid #ddd; }
        @media print { .top-bar, .footer-forms, #contextMenu, .menu-dots, .nurse-check, #selectAllContainer { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-bar">
        <div class="main-controls">
            <div id="selectAllContainer" style="display:flex; align-items:center; gap:5px; margin-right:10px;">
                <input type="checkbox" id="selectAll" onclick="toggleAll(this)"> <label for="selectAll">All</label>
            </div>
            <button class="btn btn-green" onclick="fillSelected()">მონიშნულების შევსება (1/4)</button>
            <input type="text" id="newNurseName" placeholder="ექთნის სახელი">
            <button class="btn btn-blue" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="monthSelect" onchange="renderTable()"></select>
            <input type="number" id="yearSelect" value="2024" style="width: 80px;" onchange="renderTable()">
            <button class="btn btn-pdf" onclick="window.print()">ბეჭდვა / PDF</button>
        </div>
    </div>

    <div class="table-container" id="tableWrapper"></div>

    <div class="footer-forms">
        <div class="form-box">
            <h3>რეალური საათი</h3>
            <input type="text" id="realNurseName" list="nurseList">
            <datalist id="nurseList"></datalist>
            <select id="realDaySelect"></select>
            <select id="realHoursSelect">
                <option value="8">8 სთ</option><option value="16">16 სთ</option><option value="24">24 სთ</option><option value="0">0 სთ</option>
            </select>
            <button class="btn btn-blue" onclick="saveRealHours()">შენახვა</button>
        </div>
        <div class="form-box">
            <h3>ძებნა</h3>
            <input type="text" id="searchNurse" placeholder="სახელი">
            <input type="number" id="searchDay" placeholder="რიცხვი">
            <button class="btn btn-blue" onclick="performSearch()">ძებნა</button>
            <div id="searchOutput" style="margin-top:10px; padding:10px; background:#e3f2fd; display:none;"></div>
        </div>
    </div>
</div>

<div id="contextMenu"><button onclick="editNurseName()">რედაქტირება</button><button onclick="deleteNurse()" style="color: red;">წაშლა</button></div>

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

    let nurses = [], scheduleData = {}, activeNurse = null;
    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    db.ref().on('value', (snap) => {
        const d = snap.val() || {};
        nurses = d.nurses || [];
        scheduleData = d.scheduleData || {};
        renderTable();
        updateNurseList();
    });

    window.onload = () => {
        const m = document.getElementById('monthSelect');
        months.forEach((name, i) => m.innerHTML += `<option value="${i}">${name}</option>`);
        m.value = new Date().getMonth();
    };

    function renderTable() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = new Date(year, parseInt(month) + 1, 0).getDate();
        
        let html = `<table><thead><tr><th>ექთანი</th>`;
        for (let i = 1; i <= days; i++) html += `<th>${i}</th>`;
        html += `<th>ჯამი</th></tr></thead><tbody>`;

        nurses.forEach(nurse => {
            const keyBase = `${year}-${month}-${nurse}`;
            let pTotal = 0, rTotal = 0;
            html += `<tr><td class="nurse-cell">
                <input type="checkbox" class="nurse-check" value="${nurse}">
                <span>${nurse}</span>
                <span class="menu-dots" onclick="showMenu(event, '${nurse}')">⋮</span>
            </td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-p'] && scheduleData[keyBase + '-p'][i]) || 0;
                pTotal += parseInt(val);
                html += `<td><select onchange="updateCell('${nurse}', ${i}, this.value, 'p')">
                    <option value="0" ${val==0?'selected':''}>-</option>
                    <option value="8" ${val==8?'selected':''}>8</option>
                    <option value="16" ${val==16?'selected':''}>16</option>
                    <option value="24" ${val==24?'selected':''}>24</option>
                </select></td>`;
            }
            html += `<td style="font-weight:bold">${pTotal}</td></tr><tr class="real-row"><td><small>რეალური</small></td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-r'] && scheduleData[keyBase + '-r'][i]) || 0;
                rTotal += parseInt(val);
                html += `<td>${val || ''}</td>`;
            }
            html += `<td style="color:var(--total-red); font-weight:bold">${rTotal}</td></tr>`;
        });
        document.getElementById('tableWrapper').innerHTML = html + `</tbody></table>`;
        updateDayDropdowns(days);
    }

    function toggleAll(source) {
        document.querySelectorAll('.nurse-check').forEach(cb => cb.checked = source.checked);
    }

    function fillSelected() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = new Date(year, parseInt(month) + 1, 0).getDate();
        const selectedNurses = Array.from(document.querySelectorAll('.nurse-check:checked')).map(cb => cb.value);

        if(selectedNurses.length === 0) return alert("გთხოვთ მონიშნოთ ექთნები!");

        selectedNurses.forEach(nurse => {
            const key = `${year}-${month}-${nurse}-p`;
            if(!scheduleData[key]) return;
            let firstDay = Object.keys(scheduleData[key]).sort((a,b)=>a-b).find(d => scheduleData[key][d] > 0);
            if(firstDay) {
                let hrs = scheduleData[key][firstDay];
                for(let i = parseInt(firstDay); i <= days; i += 4) {
                    db.ref(`scheduleData/${key}/${i}`).set(hrs);
                }
            }
        });
        alert("განრიგი შეივსო მონიშნული ექთნებისთვის!");
    }

    // --- დანარჩენი სტანდარტული ფუნქციები ---
    function updateCell(nurse, day, val, type) {
        const m = document.getElementById('monthSelect').value, y = document.getElementById('yearSelect').value;
        db.ref(`scheduleData/${y}-${m}-${nurse}-${type}/${day}`).set(parseInt(val));
    }
    function addNurse() {
        const name = document.getElementById('newNurseName').value.trim();
        if(name && !nurses.includes(name)) { nurses.push(name); db.ref('nurses').set(nurses); document.getElementById('newNurseName').value = ''; }
    }
    function deleteNurse() { if(confirm(`წავშალოთ ${activeNurse}?`)) { const up = nurses.filter(n => n !== activeNurse); db.ref('nurses').set(up); } }
    function editNurseName() { 
        const n = prompt("ახალი სახელი:", activeNurse); 
        if(n && n !== activeNurse) { nurses[nurses.indexOf(activeNurse)] = n; db.ref('nurses').set(nurses); } 
    }
    function saveRealHours() {
        const n = document.getElementById('realNurseName').value, d = document.getElementById('realDaySelect').value, h = document.getElementById('realHoursSelect').value;
        const m = document.getElementById('monthSelect').value, y = document.getElementById('yearSelect').value;
        if(nurses.includes(n)) db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(parseInt(h));
    }
    function updateDayDropdowns(days) {
        let o = ""; for(let i=1; i<=days; i++) o += `<option value="${i}">${i}</option>`;
        document.getElementById('realDaySelect').innerHTML = o;
    }
    function updateNurseList() {
        const dl = document.getElementById('nurseList'); dl.innerHTML = "";
        nurses.forEach(n => dl.innerHTML += `<option value="${n}">`);
    }
    function showMenu(e, nurse) {
        e.stopPropagation(); activeNurse = nurse; const m = document.getElementById('contextMenu');
        m.style.display = 'block'; m.style.left = e.pageX + 'px'; m.style.top = e.pageY + 'px';
    }
    document.addEventListener('click', () => document.getElementById('contextMenu').style.display = 'none');
    function performSearch() {
        const sn = document.getElementById('searchNurse').value.trim(), sd = document.getElementById('searchDay').value;
        const m = document.getElementById('monthSelect').value, y = document.getElementById('yearSelect').value, out = document.getElementById('searchOutput');
        out.style.display = 'block';
        if(sn) {
            const k = `${y}-${m}-${sn}-p`, d = scheduleData[k] ? Object.keys(scheduleData[k]).filter(x => scheduleData[k][x] > 0) : [];
            out.innerHTML = d.length ? `<strong>${sn}</strong> მორიგეობს: ${d.join(', ')}` : "ვერ მოიძებნა.";
        } else if(sd) {
            let f = nurses.filter(n => scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][sd] > 0);
            out.innerHTML = f.length ? `${sd} რიცხვში: <strong>${f.join(', ')}</strong>` : "ამ დღეს მორიგე არავინაა.";
        }
    }
</script>
</body>
</html>
