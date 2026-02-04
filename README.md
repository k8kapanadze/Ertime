<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ექთნების Live სისტემა - ErtimeCMC</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --gray: #d1d8e0; --red: #8b0000; --white: #ffffff; }
        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1550px; margin: auto; background: var(--white); padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        /* ზედა ბლოკები: საათები და ძებნა */
        .top-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
        .box { padding: 15px; background: #fafafa; border-radius: 8px; border: 1px solid #ddd; }
        
        /* მართვის პანელი */
        .controls-bar { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; background: #eee; padding: 15px; border-radius: 8px; gap: 10px; }
        
        .btn { padding: 10px 15px; border: none; border-radius: 6px; cursor: pointer; color: white; font-weight: bold; background: var(--blue); }
        .btn-red { background: var(--red); }
        
        /* ცხრილის სტილი */
        .table-wrap { overflow-x: auto; border: 1px solid #ddd; border-radius: 8px; background: white; min-height: 200px; }
        table { width: 100%; border-collapse: collapse; font-size: 13px; }
        th, td { border: 1px solid #ddd; padding: 6px; text-align: center; }
        th { background: #f1f1f1; color: var(--blue); }
        
        .nurse-name { text-align: left; font-weight: bold; width: 250px; display: flex; align-items: center; gap: 8px; padding-left: 10px; }
        .real-row { background-color: var(--gray); }
        .total-cell { font-weight: bold; color: var(--red); }

        #menu { display: none; position: absolute; background: white; border: 1px solid #ccc; box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 100; border-radius: 4px; }
        #menu button { display: block; width: 100%; padding: 10px; border: none; background: none; text-align: left; cursor: pointer; }

        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
        
        @media print { .top-grid, .controls-bar, #menu, input[type="checkbox"] { display: none !important; } }
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

    <div class="table-wrap" id="tableBox">
        <p style="text-align:center; padding: 20px;">იტვირთება მონაცემები ერტიმე-დან...</p>
    </div>
</div>

<div id="menu">
    <button onclick="delNurse()" style="color:red">წაშლა</button>
</div>

<script>
    // შენი პირადი Firebase კონფიგურაცია
    const firebaseConfig = {
        apiKey: "AIzaSyB3roORgMRtg6mmAyH3rUQmzmyAfc_ud6U",
        authDomain: "ertimecmc.firebaseapp.com",
        databaseURL: "https://ertimecmc-default-rtdb.europe-west1.firebasedatabase.app",
        projectId: "ertimecmc",
        storageBucket: "ertimecmc.firebasestorage.app",
        messagingSenderId: "164048857022",
        appId: "1:164048857022:web:359061cf694057bc16bbef"
    };

    // ინიციალიზაცია
    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();

    let nurses = [], scheduleData = {}, activeN = null;
    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    // ბაზიდან მონაცემების "მოსმენა" რეალურ დროში
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
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        const days = new Date(y, parseInt(m) + 1, 0).getDate();
        
        let h = `<table><thead><tr><th><input type="checkbox" onclick="toggleAll(this)"> ექთანი</th>`;
        for(let i=1; i<=days; i++) h += `<th>${i}</th>`;
        h += `<th>ჯამი</th></tr></thead><tbody>`;

        if(nurses.length === 0) h += `<tr><td colspan="${days+2}">ექთნების სია ცარიელია. დაამატეთ ახალი ექთანი.</td></tr>`;

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
            h += `<td style="font-weight:bold">${pt}</td></tr><tr class="real-row"><td>რეალური</td>`;
            for(let i=1; i<=days; i++){
                const v = (scheduleData[kb+'-r'] && scheduleData[kb+'-r'][i]) || 0; rt += v;
                h += `<td>${v||''}</td>`;
            }
            h += `<td class="total-cell">${rt}</td></tr>`;
        });
        document.getElementById('tableBox').innerHTML = h + `</tbody></table>`;
        updateDLists(days);
    }

    function toggleAll(src) { document.querySelectorAll('.nurse-check').forEach(c => c.checked = src.checked); }

    function upd(n, d, v, t) {
        const mo = document.getElementById('mSel').value, yr = document.getElementById('ySel').value;
        db.ref(`scheduleData/${yr}-${mo}-${n}-${t}/${d}`).set(parseInt(v));
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { 
            nurses.push(v); 
            db.ref('nurses').set(nurses); 
            document.getElementById('nurseInp').value=''; 
        }
    }

    function bulkAction(type) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        const days = new Date(y, parseInt(m) + 1, 0).getDate();
        const selected = Array.from(document.querySelectorAll('.nurse-check:checked')).map(c => c.value);
        if(selected.length === 0) return alert("მონიშნეთ ექთნები ცხრილში!");

        if(type === 'clear' && !confirm("წავშალოთ მონიშნულების გრაფიკი?")) return;

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

    function saveReal() {
        const n = document.getElementById('rName').value, d = document.getElementById('rDay').value, h = document.getElementById('rHrs').value;
        const mo = document.getElementById('mSel').value, yr = document.getElementById('ySel').value;
        if(n && d) db.ref(`scheduleData/${yr}-${mo}-${n}-r/${d}`).set(parseInt(h));
    }

    function openM(e, n) { e.stopPropagation(); activeN = n; const m = document.getElementById('menu'); m.style.display='block'; m.style.left=e.pageX+'px'; m.style.top=e.pageY+'px'; }
    document.onclick = () => document.getElementById('menu').style.display='none';

    function delNurse() { if(confirm('წავშალოთ სისტემიდან?')) { nurses = nurses.filter(x => x !== activeN); db.ref('nurses').set(nurses); } }

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
        out.innerHTML = "";
        if(sn) {
            const k = `${yr}-${mo}-${sn}-p`;
            const d = scheduleData[k] ? Object.keys(scheduleData[k]).filter(x => scheduleData[k][x]>0) : [];
            out.innerHTML = d.length ? `${sn} მორიგეობს: ${d.join(', ')}` : "ვერ მოიძებნა";
        } else if(sd) {
            let f = nurses.filter(n => scheduleData[`${yr}-${mo}-${n}-p`] && scheduleData[`${yr}-${mo}-${n}-p`][sd] > 0);
            out.innerHTML = f.length ? `${sd} რიცხვში არიან: ${f.join(', ')}` : "არავინ მორიგეობს";
        }
    }
</script>
</body>
</html>
