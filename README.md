
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ექთნების Live სისტემა - ErtimeCMC</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --gray: #f4f7f6; --red: #8b0000; --white: #ffffff; --border: #ddd; }
        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: var(--gray); }
        .container { max-width: 1200px; margin: auto; background: var(--white); padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        .top-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
        .box { padding: 15px; background: #fafafa; border-radius: 8px; border: 1px solid var(--border); }
        
        .controls-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; background: #eee; padding: 15px; border-radius: 8px; gap: 10px; }
        
        .btn { padding: 10px 15px; border: none; border-radius: 6px; cursor: pointer; color: white; font-weight: bold; background: var(--blue); }
        .btn-red { background: var(--red); }
        
        .table-wrap { overflow-x: auto; border: 1px solid var(--border); border-radius: 8px; background: white; }
        table { width: 100%; border-collapse: collapse; font-size: 14px; }
        th, td { border: 1px solid var(--border); padding: 10px; text-align: center; }
        th { background: #f1f1f1; color: var(--blue); position: sticky; top: 0; }
        
        .weekend { background-color: #fff0f0 !important; }
        .real-col { background-color: #f9f9f9; }
        .sum-row { background: #eee; font-weight: bold; }

        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
        
        @media print { .top-grid, .controls-bar, .no-print { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-grid">
        <div class="box">
            <h3 style="margin-top:0">საათის დაფიქსირება (რეალური)</h3>
            <input type="text" id="rName" placeholder="ექთნის სახელი" list="nList">
            <datalist id="nList"></datalist>
            <select id="rDay"></select>
            <select id="rHrs"><option value="8">8</option><option value="16">16</option><option value="24">24</option><option value="0">0</option></select>
            <button class="btn" onclick="saveReal()">შენახვა</button>
        </div>
        <div class="box">
            <h3 style="margin-top:0">ძებნა</h3>
            <input type="text" id="sName" placeholder="სახელი">
            <input type="number" id="sDay" placeholder="რიცხვი">
            <button class="btn" onclick="search()">ძებნა</button>
            <div id="sOut" style="margin-top:10px; font-weight:bold; color: var(--blue);"></div>
        </div>
    </div>

    <div class="controls-bar">
        <div>
            <button class="btn btn-red" onclick="bulkFillYear()">წლიური ავტო-შევსება (1/4)</button>
            <input type="text" id="nurseInp" placeholder="ახალი ექთანი">
            <button class="btn" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 80px;" onchange="load()">
            <button class="btn" style="background:#555" onclick="window.print()">ბეჭდვა</button>
        </div>
    </div>

    <div class="table-wrap" id="tableBox">
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
        const days = new Date(y, m + 1, 0).getDate();
        
        let h = `<table><thead><tr><th rowspan="2">რიცხვი</th>`;
        nurses.forEach(n => {
            h += `<th colspan="2">${n} <button class="no-print" onclick="delNurse('${n}')" style="color:red; border:none; background:none; cursor:pointer">×</button></th>`;
        });
        h += `</tr><tr>`;
        nurses.forEach(() => h += `<th>გეგმა</th><th class="real-col">რეალური</th>`);
        h += `</tr></thead><tbody>`;

        for(let d = 1; d <= days; d++) {
            const isWknd = [0, 6].includes(new Date(y, m, d).getDay());
            h += `<tr class="${isWknd ? 'weekend' : ''}"><td>${d}</td>`;
            nurses.forEach(n => {
                const pVal = (scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][d]) || 0;
                const rVal = (scheduleData[`${y}-${m}-${n}-r`] && scheduleData[`${y}-${m}-${n}-r`][d]) || 0;
                h += `<td>
                    <select onchange="upd('${n}',${d},this.value,'p')">
                        <option value="0" ${pVal==0?'selected':''}>-</option>
                        <option value="8" ${pVal==8?'selected':''}>8</option>
                        <option value="16" ${pVal==16?'selected':''}>16</option>
                        <option value="24" ${pVal==24?'selected':''}>24</option>
                    </select>
                </td>
                <td class="real-col">${rVal || ''}</td>`;
            });
            h += `</tr>`;
        }

        // ჯამების გამოთვლა
        h += `<tr class="sum-row"><td>ჯამი</td>`;
        nurses.forEach(n => {
            let sP = 0, sR = 0;
            if(scheduleData[`${y}-${m}-${n}-p`]) Object.values(scheduleData[`${y}-${m}-${n}-p`]).forEach(v => sP += v);
            if(scheduleData[`${y}-${m}-${n}-r`]) Object.values(scheduleData[`${y}-${m}-${n}-r`]).forEach(v => sR += v);
            h += `<td>${sP}</td><td class="real-col">${sR}</td>`;
        });

        document.getElementById('tableBox').innerHTML = h + `</tr></tbody></table>`;
        updateDLists(days);
    }

    function upd(n, d, v, t) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function saveReal() {
        const n = document.getElementById('rName').value, d = document.getElementById('rDay').value, h = document.getElementById('rHrs').value;
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        if(n && d) db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(parseInt(h));
    }

    async function bulkFillYear() {
        const n = prompt("ჩაწერეთ ექთნის სახელი ზუსტად:");
        if(!nurses.includes(n)) return alert("ექთანი ვერ მოიძებნა!");
        const startDay = parseInt(prompt("რომელი რიცხვიდან დავიწყოთ (მიმდინარე თვეში)?", "1"));
        const hrs = parseInt(prompt("რამდენი საათი? (8, 16, 24)", "24"));
        
        if(!confirm(`დავაგენერო ${n}-სთვის გრაფიკი წლის ბოლომდე ყოველ მე-4 დღეს?`)) return;

        let curr = new Date(document.getElementById('ySel').value, document.getElementById('mSel').value, startDay);
        const endYear = curr.getFullYear();
        let updates = {};

        while(curr.getFullYear() === endYear) {
            updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${n}-p/${curr.getDate()}`] = hrs;
            curr.setDate(curr.getDate() + 4);
        }
        await db.ref().update(updates);
        alert("გრაფიკი წარმატებით შეივსო წლის ბოლომდე!");
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { nurses.push(v); db.ref('nurses').set(nurses); document.getElementById('nurseInp').value=''; }
    }

    function delNurse(n) { if(confirm('წავშალოთ ' + n + '?')) { nurses = nurses.filter(x => x !== n); db.ref('nurses').set(nurses); } }

    function updateDLists(d) {
        let o = ""; for(let i=1; i<=d; i++) o += `<option value="${i}">${i}</option>`;
        document.getElementById('rDay').innerHTML = o;
        let nl = ""; nurses.forEach(x => nl += `<option value="${x}">`);
        document.getElementById('nList').innerHTML = nl;
    }

    function search() {
        const sn = document.getElementById('sName').value, sd = document.getElementById('sDay').value;
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        const out = document.getElementById('sOut');
        out.innerHTML = "";
        if(sn) {
            const k = `${y}-${m}-${sn}-p`;
            const days = scheduleData[k] ? Object.keys(scheduleData[k]).filter(x => scheduleData[k][x]>0) : [];
            out.innerHTML = days.length ? `${sn} მორიგეობს: ${days.join(', ')}` : "ვერ მოიძებნა";
        } else if(sd) {
            let f = nurses.filter(n => scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][sd] > 0);
            out.innerHTML = f.length ? `${sd} რიცხვში არიან: ${f.join(', ')}` : "არავინ მორიგეობს";
        }
    }
</script>
</body>
</html>
