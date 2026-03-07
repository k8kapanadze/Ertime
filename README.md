
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ErtimeCMC - ექთნების Live სისტემა</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

    <style>
        :root { --blue: #001f3f; --light-blue: #e7eff6; --red: #8b0000; --white: #ffffff; --border: #ccc; }
        body { font-family: 'Segoe UI', sans-serif; margin: 15px; background: #f4f7f6; }
        .container { max-width: 1200px; margin: auto; background: var(--white); padding: 20px; border-radius: 10px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        
        .top-panel { display: grid; grid-template-columns: 2fr 1fr; gap: 15px; margin-bottom: 20px; }
        .box { padding: 15px; border: 1px solid var(--border); border-radius: 8px; background: #fafafa; }
        
        .controls { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; padding: 10px; background: #eee; border-radius: 5px; }
        
        table { width: 100%; border-collapse: collapse; background: white; font-size: 14px; }
        th, td { border: 1px solid var(--border); padding: 8px; text-align: center; }
        th { background: var(--blue); color: white; position: sticky; top: 0; }
        
        .nurse-col-header { background: #003366; }
        .real-col { background-color: var(--light-blue); }
        .weekend { background-color: #ffeaea !important; }
        
        .btn { padding: 8px 12px; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; color: white; background: var(--blue); }
        .btn-red { background: var(--red); }
        input, select { padding: 6px; border: 1px solid #ccc; border-radius: 4px; }

        @media print { .top-panel, .controls, .no-print { display: none !important; } }
    </style>
</head>
<body>

<div class="container">
    <div class="top-panel">
        <div class="box">
            <h3>🔍 ძებნა და მართვა</h3>
            <input type="text" id="nurseInp" placeholder="ახალი ექთანი">
            <button class="btn" onclick="addNurse()">დამატება</button>
            <button class="btn btn-red" onclick="bulkFillYear()">📅 წლიური ავტო-შევსება (1/4)</button>
        </div>
        <div class="box">
            <h3>📅 პერიოდი</h3>
            <select id="mSel" onchange="load()"></select>
            <input type="number" id="ySel" value="2024" style="width: 70px;" onchange="load()">
        </div>
    </div>

    <div class="controls">
        <div id="summary">მონიშნეთ თვე და წელი</div>
        <button class="btn" style="background:#444" onclick="window.print()">ბეჭდვა</button>
    </div>

    <div id="tableBox" style="overflow-x: auto;">
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
        
        let h = `<table><thead><tr>
            <th rowspan="2">რიცხვი</th>`;
        
        // ექთნების სვეტები (გეგმიური და რეალური)
        nurses.forEach(n => {
            h += `<th colspan="2" class="nurse-col-header">${n} <span class="no-print" style="cursor:pointer" onclick="delNurse('${n}')"> (❌)</span></th>`;
        });
        h += `</tr><tr>`;
        nurses.forEach(() => h += `<th>გეგმა</th><th class="real-col">რეალური</th>`);
        h += `</tr></thead><tbody>`;

        for(let d = 1; d <= days; d++) {
            const isWknd = [0, 6].includes(new Date(y, m, d).getDay());
            h += `<tr class="${isWknd ? 'weekend' : ''}">
                <td style="font-weight:bold">${d}</td>`;
            
            nurses.forEach(n => {
                const plan = (scheduleData[`${y}-${m}-${n}-p`] && scheduleData[`${y}-${m}-${n}-p`][d]) || 0;
                const real = (scheduleData[`${y}-${m}-${n}-r`] && scheduleData[`${y}-${m}-${n}-r`][d]) || 0;
                
                h += `<td>
                    <select onchange="upd('${n}',${d},this.value,'p')">
                        <option value="0" ${plan==0?'selected':''}>-</option>
                        <option value="8" ${plan==8?'selected':''}>8</option>
                        <option value="16" ${plan==16?'selected':''}>16</option>
                        <option value="24" ${plan==24?'selected':''}>24</option>
                    </select>
                </td>
                <td class="real-col">
                    <select onchange="upd('${n}',${d},this.value,'r')">
                        <option value="0" ${real==0?'selected':''}>-</option>
                        <option value="8" ${real==8?'selected':''}>8</option>
                        <option value="16" ${real==16?'selected':''}>16</option>
                        <option value="24" ${real==24?'selected':''}>24</option>
                    </select>
                </td>`;
            });
            h += `</tr>`;
        }
        
        // ჯამების სტრიქონი
        h += `<tr style="background:#eee; font-weight:bold"><td>ჯამი:</td>`;
        nurses.forEach(n => {
            let sumP = 0, sumR = 0;
            if(scheduleData[`${y}-${m}-${n}-p`]) Object.values(scheduleData[`${y}-${m}-${n}-p`]).forEach(v => sumP += v);
            if(scheduleData[`${y}-${m}-${n}-r`]) Object.values(scheduleData[`${y}-${m}-${n}-r`]).forEach(v => sumR += v);
            h += `<td>${sumP}</td><td class="real-col">${sumR}</td>`;
        });

        document.getElementById('tableBox').innerHTML = h + `</tr></tbody></table>`;
    }

    function upd(n, d, v, type) {
        const m = document.getElementById('mSel').value, y = document.getElementById('ySel').value;
        db.ref(`scheduleData/${y}-${m}-${n}-${type}/${d}`).set(parseInt(v));
    }

    function addNurse() {
        const v = document.getElementById('nurseInp').value.trim();
        if(v && !nurses.includes(v)) { 
            nurses.push(v); 
            db.ref('nurses').set(nurses); 
            document.getElementById('nurseInp').value=''; 
        }
    }

    function delNurse(n) { if(confirm('წავშალოთ ' + n + '?')) { 
        nurses = nurses.filter(x => x !== n); 
        db.ref('nurses').set(nurses); 
    } }

    async function bulkFillYear() {
        const n = prompt("ჩაწერეთ ექთნის სახელი:");
        if(!nurses.includes(n)) return alert("ექთანი ვერ მოიძებნა");
        
        const startDay = parseInt(prompt("რომელი რიცხვიდან დავიწყოთ (ამ თვეში)?", "1"));
        const hrs = parseInt(prompt("საათები (8, 16, 24):", "24"));
        
        if(!startDay || !hrs) return;

        const startM = parseInt(document.getElementById('mSel').value);
        const startY = parseInt(document.getElementById('ySel').value);
        
        let curr = new Date(startY, startM, startDay);
        let updates = {};

        if(!confirm(`${n}-სთვის შეივსება გრაფიკი წლის ბოლომდე 1/4 პრინციპით. გავაგრძელოთ?`)) return;

        // ციკლი წლის ბოლომდე
        while(curr.getFullYear() === startY) {
            let y = curr.getFullYear(), m = curr.getMonth(), d = curr.getDate();
            updates[`scheduleData/${y}-${m}-${n}-p/${d}`] = hrs;
            curr.setDate(curr.getDate() + 4);
        }

        await db.ref().update(updates);
        alert("წლიური გრაფიკი წარმატებით შეივსო!");
    }
</script>
</body>
</html>
