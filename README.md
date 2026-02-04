<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ექთნების მართვის სისტემა (Live)</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root {
            --primary-blue: #001f3f;
            --real-row-bg: #d1d8e0;
            --total-red: #8b0000;
        }

        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        
        .top-bar { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 20px; gap: 15px; flex-wrap: wrap; }
        .main-controls { display: flex; gap: 10px; align-items: center; }
        
        .btn { padding: 10px 18px; border: none; border-radius: 4px; cursor: pointer; color: white; font-weight: bold; transition: 0.3s; }
        .btn-blue { background: var(--primary-blue); }
        .btn-pdf { background: #2c3e50; }

        .table-container { overflow-x: auto; margin-top: 10px; }
        table { width: 100%; border-collapse: collapse; font-size: 13px; }
        th, td { border: 1px solid #ccc; text-align: center; padding: 6px; min-width: 35px; }
        th { background: #f8f9fa; }
        
        .nurse-cell { 
            font-weight: bold; width: 220px; text-align: left; 
            display: flex; justify-content: space-between; align-items: center;
            padding: 8px; position: relative;
        }
        .menu-dots { cursor: pointer; padding: 0 8px; font-size: 18px; font-weight: bold; }
        .real-row { background-color: var(--real-row-bg); }
        .total-real { color: var(--total-red); font-weight: bold; }

        #contextMenu {
            display: none; position: absolute; background: white; border: 1px solid #ccc;
            box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 1000; padding: 5px 0; border-radius: 4px;
        }
        #contextMenu button {
            display: block; width: 100%; padding: 8px 15px; border: none; background: none;
            text-align: left; cursor: pointer; font-size: 14px;
        }
        #contextMenu button:hover { background: #eee; }

        .footer-forms { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 30px; border-top: 2px solid #eee; padding-top: 20px; }
        .form-box { background: #fdfdfd; padding: 15px; border-radius: 6px; border: 1px solid #ddd; }
        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 5px; }

        .search-output { margin-top: 10px; padding: 10px; background: #e3f2fd; border-left: 4px solid #2196f3; display: none; }

        /* ბეჭდვის ოპტიმიზაცია - აქრობს ღილაკებს ფურცელზე */
        @media print {
            .top-bar, .footer-forms, #contextMenu, .menu-dots { display: none !important; }
            .container { box-shadow: none; max-width: 100%; }
            body { background: white; margin: 0; }
        }
    </style>
</head>
<body>

<div class="container">
    <div class="top-bar">
        <div class="main-controls">
            <button class="btn btn-blue" onclick="autoFillAll()">ავტომატური შევსება (მე-4 დღე)</button>
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
            <h3>რეალური საათის დაფიქსირება</h3>
            <input type="text" id="realNurseName" placeholder="ექთნის სახელი" list="nurseList">
            <datalist id="nurseList"></datalist>
            <select id="realDaySelect"></select>
            <select id="realHoursSelect">
                <option value="8">8 სთ</option>
                <option value="16">16 სთ</option>
                <option value="24">24 სთ</option>
                <option value="0">0 სთ</option>
            </select>
            <button class="btn btn-blue" onclick="saveRealHours()">შენახვა</button>
        </div>

        <div class="form-box">
            <h3>ძებნა</h3>
            <div style="display: flex; gap: 5px; flex-direction: column;">
                <input type="text" id="searchNurse" placeholder="სახელი">
                <input type="number" id="searchDay" placeholder="რიცხვი">
                <button class="btn btn-blue" onclick="performSearch()">ძებნა</button>
            </div>
            <div id="searchOutput" class="search-output"></div>
        </div>
    </div>
</div>

<div id="contextMenu">
    <button onclick="editNurseName()">რედაქტირება</button>
    <button onclick="deleteNurse()" style="color: red;">წაშლა</button>
</div>

<script>
    // შენი Firebase მონაცემები
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

    let nurses = [];
    let scheduleData = {};
    let activeNurse = null;

    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    // მონაცემების წამოღება ბაზიდან რეალურ დროში
    db.ref().on('value', (snapshot) => {
        const data = snapshot.val() || {};
        nurses = data.nurses || [];
        scheduleData = data.scheduleData || {};
        renderTable();
        updateNurseList();
    });

    window.onload = () => {
        const mSelect = document.getElementById('monthSelect');
        months.forEach((m, i) => mSelect.innerHTML += `<option value="${i}">${m}</option>`);
        mSelect.value = new Date().getMonth();
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

            html += `<tr><td class="nurse-cell"><span>${nurse}</span><span class="menu-dots" onclick="showMenu(event, '${nurse}')">⋮</span></td>`;
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
            html += `<td style="font-weight:bold">${pTotal}</td></tr>`;

            html += `<tr class="real-row"><td style="text-align:left; padding-left:10px;"><small>რეალური</small></td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-r'] && scheduleData[keyBase + '-r'][i]) || 0;
                rTotal += parseInt(val);
                html += `<td>${val || ''}</td>`;
            }
            html += `<td class="total-real">${rTotal}</td></tr>`;
        });

        html += `</tbody></table>`;
        document.getElementById('tableWrapper').innerHTML = html;
        updateDayDropdowns(days);
    }

    function updateCell(nurse, day, val, type) {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const key = `${year}-${month}-${nurse}-${type}`;
        db.ref(`scheduleData/${key}/${day}`).set(parseInt(val));
    }

    function addNurse() {
        const name = document.getElementById('newNurseName').value.trim();
        if(name && !nurses.includes(name)) {
            nurses.push(name);
            db.ref('nurses').set(nurses);
            document.getElementById('newNurseName').value = '';
        }
    }

    function deleteNurse() {
        if(confirm(`წავშალოთ ${activeNurse}?`)) {
            const updated = nurses.filter(n => n !== activeNurse);
            db.ref('nurses').set(updated);
        }
    }

    function editNurseName() {
        const newName = prompt("შეიყვანეთ ახალი სახელი:", activeNurse);
        if(newName && newName !== activeNurse) {
            const idx = nurses.indexOf(activeNurse);
            nurses[idx] = newName;
            db.ref('nurses').set(nurses);
        }
    }

    function autoFillAll() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = new Date(year, parseInt(month) + 1, 0).getDate();

        nurses.forEach(nurse => {
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
    }

    function saveRealHours() {
        const nurse = document.getElementById('realNurseName').value;
        const day = document.getElementById('realDaySelect').value;
        const hrs = document.getElementById('realHoursSelect').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;

        if(!nurses.includes(nurse)) return alert("ექთანი არ არსებობს!");
        db.ref(`scheduleData/${year}-${month}-${nurse}-r/${day}`).set(parseInt(hrs));
    }

    function performSearch() {
        const sNurse = document.getElementById('searchNurse').value.trim();
        const sDay = document.getElementById('searchDay').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const output = document.getElementById('searchOutput');
        output.style.display = 'block';

        if(sNurse) {
            const key = `${year}-${month}-${sNurse}-p`;
            const days = scheduleData[key] ? Object.keys(scheduleData[key]).filter(d => scheduleData[key][d] > 0) : [];
            output.innerHTML = days.length ? `<strong>${sNurse}</strong> მორიგეობს: ${days.join(', ')}` : "ვერ მოიძებნა.";
        } else if(sDay) {
            let found = [];
            nurses.forEach(n => {
                const key = `${year}-${month}-${n}-p`;
                if(scheduleData[key] && scheduleData[key][sDay] > 0) found.push(n);
            });
            output.innerHTML = found.length ? `${sDay} რიცხვში: <strong>${found.join(', ')}</strong>` : "ამ დღეს მორიგე არავინაა.";
        }
    }

    function showMenu(e, nurse) {
        e.stopPropagation();
        activeNurse = nurse;
        const menu = document.getElementById('contextMenu');
        menu.style.display = 'block';
        menu.style.left = e.pageX + 'px';
        menu.style.top = e.pageY + 'px';
    }

    function updateDayDropdowns(days) {
        let options = "";
        for(let i=1; i<=days; i++) options += `<option value="${i}">${i}</option>`;
        document.getElementById('realDaySelect').innerHTML = options;
    }

    function updateNurseList() {
        const dl = document.getElementById('nurseList');
        dl.innerHTML = "";
        nurses.forEach(n => dl.innerHTML += `<option value="${n}">`);
    }

    document.addEventListener('click', () => document.getElementById('contextMenu').style.display = 'none');
</script>
</body>
</html>
