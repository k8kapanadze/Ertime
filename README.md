<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ექთნების Live სისტემა</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --gray: #d1d8e0; --red: #8b0000; --white: #ffffff; }
        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1500px; margin: auto; background: var(--white); padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        .top-bar { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 25px; gap: 10px; flex-wrap: wrap; }
        .btn { padding: 10px 15px; border: none; border-radius: 6px; cursor: pointer; color: white; font-weight: bold; transition: 0.2s; background: var(--blue); }
        .btn-red { background: var(--red); }
        .btn:hover { opacity: 0.8; }
        
        .table-wrap { overflow-x: auto; border: 1px solid #ddd; border-radius: 8px; }
        table { width: 100%; border-collapse: collapse; font-size: 14px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
        th { background: #f8f9fa; position: sticky; top: 0; }
        
        .nurse-name { text-align: left; font-weight: bold; width: 280px; display: flex; align-items: center; gap: 10px; }
        .real-row { background-color: var(--gray); font-size: 12px; }
        .total-red { color: var(--red); font-weight: bold; }

        #menu { display: none; position: absolute; background: white; border: 1px solid #ccc; box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 100; border-radius: 4px; }
        #menu button { display: block; width: 100%; padding: 10px; border: none; background: none; text-align: left; cursor: pointer; }

        .forms { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 30px; }
        .box { padding: 20px; background: #fafafa; border-radius: 8px; border: 1px solid #eee; }
        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }

        @media print { .top-bar, .forms, #menu, input[type="checkbox"] { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-bar">
        <div>
            <button class="btn btn-red" onclick="bulkAction('fill')">ავტომატური შევსება (მე-4 დღე)</button>
            <button class="btn btn-red" onclick="bulkAction('clear')">მონიშნულების გასუფთავება</button>
            <input type="text" id="nurseInp" placeholder="ახალი ექთანი">
            <button class="btn" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 80px;" onchange="load()">
            <button class="btn" style="background:#555" onclick="window.print()">ბეჭდვა</button>
        </div>
    </div>

    <div class="table-wrap" id="tableBox"></div>

    <div class="forms">
        <div class="box">
            <h3>საათის დაფიქსირება (რეალური)</h3>
            <input type="text" id="rName" placeholder="ექთნის სახელი" list="nList">
            <datalist id="nList"></datalist>
            <select id="rDay"></select>
            <select id="rHrs"><option value="8">8</option><option value="16">16</option><option value="24">24</option><option value="0">0</option></select>
            <button class="btn" onclick="saveReal()">შენახვა</button>
        </div>
        <div class="box">
            <h3>ძებნა</h3>
            <input type="text" id="sName" placeholder="სახელი">
            <input type="number" id="sDay" placeholder="რიცხვი">
            <button class="btn" onclick="search()">ძებნა</button>
            <div id="sOut" style="margin-top:10px; font-weight:bold;"></div>
        </div>
    </div>
</div>

<div id="menu">
    <button onclick="editNurse()">რედაქტირება</button>
    <button onclick="delNurse()" style="color:red">წაშლა</button>
</div>

<script>
    // ჩასვი შენი FirebaseConfig აქ
    const firebaseConfig = {
        apiKey: "YOUR_API_KEY",
        authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
        databaseURL: "YOUR_DATABASE_URL",
        projectId: "YOUR_PROJECT_ID",
        storageBucket: "YOUR_PROJECT_ID.appspot.com",
        messagingSenderId: "YOUR_SENDER_ID",
        appId: "YOUR_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    let nurses = [], scheduleData = {}, activeN = null;
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
    };

    function load() {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        const days = new Date(y, parseInt(m) + 1, 0).getDate();
        let h = `<table><thead><tr><th><input type="checkbox" onclick="toggleAll(this)"> ექთანი</th>`;
        for(let i=1; i<=days; i++) h += `<th>${i}</th>`;
        h += `<th>ჯამი</th></tr></thead><tbody>`;

        nurses.forEach(n => {
            const kb = `${y}-${m}-${n}`;
            let pt = 0, rt = 0;
            h += `<tr><td class="nurse-name">
                <input type="checkbox" class="nurse-check" value="${n}">
                <span>${n}</span>
                <span style="cursor:pointer;color:#999" onclick="openM(event,'${n}')">⋮</span>
            </td>`;
            for(let i=1; i<=days; i++){
                const v = (scheduleData[kb+'-p'] && scheduleData[kb+'-p'][i]) || 0; pt += v;
                h += `<td><select onchange="upd('${n}',${i},this.value,'p')">
                    <option value="0" ${v==0?'selected':''}>-</option>
                    <option value="8" ${v==8?'selected':''}>8</option>
                    <option value="16" ${v==16?'selected':''}>16</option>
                    <option value="24" ${v==24?'selected':''}>24</option>
                </select></td>`;
            }
            h += `<td>${pt}</td></tr><tr class="real-row"><td>რეალური</td>`;
            for(let i=1; i<=days; i++){
                const v = (scheduleData[kb+'-r'] && scheduleData[kb+'-r'][i]) || 0; rt += v;
                h += `<td>${v||''}</td>`;
            }
            h += `<td class="total-red">${rt}</td></tr>`;
        });
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
        updateDLists(days);
    }

    function toggleAll(src) {
        document.querySelectorAll('.nurse-check').forEach(c => c.checked = src.checked);
    }

    function bulkAction(type) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        const days = new Date(y, parseInt(m) + 1, 0).getDate();
        const selected = Array.from(document.querySelectorAll('.nurse-check:checked')).map(c => c.value);

        if(selected.length === 0) return alert("გთხოვთ მონიშნოთ ექთნები");

        selected.forEach(n => {
            const k = `${y}-${m}-${n}-p`;
            if(type === 'clear') {
                db.ref(`scheduleData/${k}`).remove();
            } else if(type === 'fill') {
                if(!scheduleData[k]) return;
                let first = Object.keys(scheduleData[k]).sort((a,b)=>a-b).find(x => scheduleData[k][x] > 0);
                if(first) {
                    let hrs = scheduleData[k][first];
                    for(let i=parseInt(first); i<=days; i+=4) db.ref(`scheduleData/${k}/${i}`).set(hrs);
                }
            }
        });
    }

    function upd(n, d, v, t) {
        const mo = document.getElementById('mSel').value, yr = document.getElementById('ySel').value;
        db.ref(`scheduleData/${yr}-${mo}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { nurses.push(v); db.ref('nurses').set(nurses); document.getElementById('nurseInp').value=''; }
    }

    function openM(e, n) { e.stopPropagation(); activeN = n; const m = document.getElementById('menu'); m.style.display='block'; m.style.left=e.pageX+'px'; m.style.top=e.pageY+'px'; }
    document.onclick = () => document.getElementById('menu').style.display='none';

    function delNurse() { if(confirm('წავშალოთ სისტემიდან?')) { nurses = nurses.filter(x => x !== activeN); db.ref('nurses').set(nurses); } }

    function saveReal() {
        const n = document.getElementById('rName').value, d = document.getElementById('rDay').value, h = document.getElementById('rHrs').value;
        const mo = document.getElementById('mSel').value, yr = document.getElementById('ySel').value;
        if(nurses.includes(n)) db.ref(`scheduleData/${yr}-${mo}-${n}-r/${d}`).set(parseInt(h));
    }

    function updateDLists(d) {
        let o = ""; for(let i=1; i<=d; i++) o += `<option value="${i}">${i}</option>`;
        document.getElementById('rDay').innerHTML = o;
        let nl = ""; nurses.forEach(x => nl += `<option value="${x}">`);
        document.getElementById('nList').innerHTML = nl;
    }

    function search() {
        const sn = document.getElementById('sName').value, sd = document.getElementById('sDay').value;
        const mo = document.getElementById('mSel').value, yr = document.getElementById('ySel').value;
        const out = document.getElementById('sOut');
        if(sn) {
            const k = `${yr}-${mo}-${sn}-p`;
            const d = scheduleData[k] ? Object.keys(scheduleData[k]).filter(x => scheduleData[k][x]>0) : [];
            out.innerHTML = d.length ? `მორიგეობს: ${d.join(', ')}` : "ვერ მოიძებნა";
        } else if(sd) {
            let f = nurses.filter(n => scheduleData[`${yr}-${mo}-${n}-p`] && scheduleData[`${yr}-${mo}-${n}-p`][sd] > 0);
            out.innerHTML = f.length ? `მორიგეობენ: ${f.join(', ')}` : "არავინ მორიგეობს";
        }
    }
</script>
</body>
</html>
