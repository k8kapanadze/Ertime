
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ექთნების Live სისტემა - ErtimeCMC</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --gray: #f8f9fa; --red: #8b0000; --white: #ffffff; --border: #dee2e6; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 20px; background: #f0f2f5; color: #333; }
        .container { max-width: 900px; margin: auto; background: var(--white); padding: 25px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); }
        
        .header-box { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid var(--blue); padding-bottom: 15px; }
        .controls { display: flex; gap: 10px; flex-wrap: wrap; background: #fdfdfd; padding: 15px; border-radius: 8px; border: 1px solid #eee; margin-bottom: 20px; }
        
        table { width: 100%; border-collapse: collapse; background: white; border-radius: 8px; overflow: hidden; }
        th, td { border: 1px solid var(--border); padding: 10px; text-align: center; }
        th { background: var(--blue); color: white; position: sticky; top: 0; }
        
        .day-row:nth-child(even) { background-color: #fcfcfc; }
        .weekend { background-color: #fff0f0 !important; }
        .today { border: 2px solid var(--red) !important; font-weight: bold; }
        
        .btn { padding: 8px 16px; border: none; border-radius: 5px; cursor: pointer; font-weight: 600; transition: 0.3s; }
        .btn-blue { background: var(--blue); color: white; }
        .btn-red { background: var(--red); color: white; }
        .btn:hover { opacity: 0.8; }

        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; outline: none; }
        .nurse-header { font-size: 1.1em; color: var(--blue); }
        
        @media print { .controls, .btn, .no-print { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="header-box">
        <h2 style="margin:0">📅 მორიგეობის განრიგი</h2>
        <div class="no-print">
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 80px;" onchange="load()">
        </div>
    </div>

    <div class="controls no-print">
        <div style="flex:1">
            <input type="text" id="nurseInp" placeholder="ექთნის სახელი">
            <button class="btn btn-blue" onclick="addNurse()">დამატება</button>
        </div>
        <button class="btn btn-red" onclick="bulkFillYear()">წლიური ავტო-შევსება (1/4)</button>
        <button class="btn" style="background:#555; color:white" onclick="window.print()">ბეჭდვა</button>
    </div>

    <div id="tableBox">
        </div>
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

    let nurses = [], scheduleData = {};
    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    db.ref().on('value', snap => {
        const d = snap.val() || {};
        nurses = d.nurses || [];
        scheduleData = d.scheduleData || {};
        load();
    });

    window.onload = () => {
        const sel = document.getElementById('mSel');
        months.forEach((m, i) => sel.innerHTML += `<option value="${i}">${m}</option>`);
        sel.value = new Date().getMonth();
        document.getElementById('ySel').value = new Date().getFullYear();
    };

    function load() {
        const m = parseInt(document.getElementById('mSel').value);
        const y = parseInt(document.getElementById('ySel').value);
        const daysInMonth = new Date(y, m + 1, 0).getDate();
        
        let h = `<table><thead><tr><th>თარიღი</th>`;
        nurses.forEach(n => h += `<th class="nurse-header">${n} <span style="cursor:pointer; font-size:10px" onclick="delNurse('${n}')">❌</span></th>`);
        h += `</tr></thead><tbody>`;

        for(let d = 1; d <= daysInMonth; d++) {
            const dateObj = new Date(y, m, d);
            const isWeekend = dateObj.getDay() === 0 || dateObj.getDay() === 6;
            h += `<tr class="${isWeekend ? 'weekend' : ''}">
                <td style="font-weight:bold">${d} ${months[m].substring(0,3)}</td>`;
            
            nurses.forEach(n => {
                const val = (scheduleData[`${y}-${m}-${n}`] && scheduleData[`${y}-${m}-${n}`][d]) || 0;
                h += `<td>
                    <select onchange="upd('${n}',${d},this.value)">
                        <option value="0" ${val==0?'selected':''}>-</option>
                        <option value="8" ${val==8?'selected':''}>8</option>
                        <option value="16" ${val==16?'selected':''}>16</option>
                        <option value="24" ${val==24?'selected':''}>24</option>
                    </select>
                </td>`;
            });
            h += `</tr>`;
        }
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
    }

    function upd(n, d, v) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}/${d}`).set(parseInt(v));
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
        const n = prompt("რომელი ექთნისთვის შევავსოთ? (ჩაწერეთ სახელი ზუსტად)");
        if(!nurses.includes(n)) return alert("ექთანი ვერ მოიძებნა");
        
        const startDay = prompt("რომელი რიცხვიდან დავიწყოთ (მიმდინარე თვეში)?", "1");
        const hrs = prompt("რამდენ საათიანი მორიგეობა? (8, 16, ან 24)", "24");
        
        if(!startDay || !hrs) return;

        const startM = parseInt(document.getElementById('mSel').value);
        const startY = parseInt(document.getElementById('ySel').value);
        
        // საწყისი თარიღის ობიექტი
        let currentDate = new Date(startY, startM, parseInt(startDay));
        const endYear = startY; // ავსებს მიმდინარე წლის ბოლომდე

        if(!confirm(`ნამდვილად გსურთ ${n}-სთვის წლის ბოლომდე გრაფიკის შევსება?`)) return;

        let updates = {};
        while(currentDate.getFullYear() === endYear) {
            let y = currentDate.getFullYear();
            let m = currentDate.getMonth();
            let d = currentDate.getDate();
            
            updates[`scheduleData/${y}-${m}-${n}/${d}`] = parseInt(hrs);
            
            // გადავდივართ 4 დღით წინ
            currentDate.setDate(currentDate.getDate() + 4);
        }

        await db.ref().update(updates);
        alert("წლიური გრაფიკი განახლდა!");
    }
</script>
</body>
</html>
