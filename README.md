
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>NurseFlow Elite - 4 Columns</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700;800&display=swap" rel="stylesheet">
    <style>
        :root {
            --glass: rgba(255, 255, 255, 0.88);
            --glass-border: rgba(255, 255, 255, 0.5);
            --accent: #3b82f6;
            --success: #10b981;
            --bg-gradient: linear-gradient(135deg, #f1f5f9 0%, #e2e8f0 100%);
            --shadow: 0 10px 30px rgba(0, 0, 0, 0.04);
        }

        * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Inter', sans-serif; }
        
        body { 
            background: var(--bg-gradient);
            background-attachment: fixed;
            min-height: 100vh;
            padding: 10px 0;
            color: #1e293b;
        }

        .container { width: 100%; max-width: 600px; margin: 0 auto; padding: 0 8px; }

        /* Glass Cards */
        .glass-card {
            background: var(--glass);
            backdrop-filter: blur(15px);
            -webkit-backdrop-filter: blur(15px);
            border: 1px solid var(--glass-border);
            border-radius: 20px;
            box-shadow: var(--shadow);
            padding: 20px;
            margin-bottom: 15px;
        }

        .section-title {
            font-size: 0.65rem;
            font-weight: 800;
            color: #94a3b8;
            text-transform: uppercase;
            letter-spacing: 1.5px;
            margin-bottom: 12px;
            display: block;
        }

        .input-group { display: flex; flex-direction: column; gap: 4px; margin-bottom: 12px; }
        label { font-size: 0.75rem; font-weight: 700; color: #64748b; margin-left: 4px; }

        input, select {
            background: rgba(255, 255, 255, 0.7);
            border: 1px solid var(--glass-border);
            border-radius: 12px;
            padding: 12px;
            font-size: 1rem;
            outline: none;
            color: #334155;
            width: 100%;
        }

        .control-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }

        /* Buttons 3D Style */
        .btn {
            border-radius: 12px;
            border: none;
            padding: 14px;
            font-weight: 700;
            cursor: pointer;
            transition: 0.2s;
            font-size: 0.9rem;
            box-shadow: 0 4px 0 rgba(0, 0, 0, 0.1);
            width: 100%;
            margin-bottom: 10px;
        }
        .btn:active { transform: translateY(2px); box-shadow: none; }
        
        .btn-primary { background: #0f172a; color: white; }
        .btn-secondary { background: white; color: #1e293b; border: 1px solid var(--glass-border); }
        .btn-danger { background: #fee2e2; color: #ef4444; border: 1px solid #fecaca; box-shadow: none; }

        /* Full Width Table Card */
        .table-card { padding: 0; overflow: hidden; border-radius: 20px; width: 100%; }
        table { width: 100%; border-collapse: collapse; background: var(--glass); table-layout: fixed; }
        
        th { 
            background: #1e293b; 
            color: white; 
            padding: 12px 4px; 
            font-size: 0.65rem; 
            font-weight: 700; 
            text-transform: uppercase;
        }
        
        td { padding: 10px 2px; text-align: center; border-bottom: 1px solid rgba(0,0,0,0.03); vertical-align: middle; }

        .day-cell { font-weight: 800; font-size: 0.9rem; color: #475569; }

        .plan-select {
            width: 90%; 
            padding: 8px 2px; 
            border-radius: 10px; 
            border: 1px solid var(--glass-border);
            background: #fff; 
            font-weight: 800; 
            text-align: center;
            font-size: 0.95rem;
        }

        .real-display {
            background: rgba(59, 130, 246, 0.1);
            color: var(--accent);
            font-weight: 800;
            border-radius: 10px;
            padding: 8px 0;
            display: inline-block;
            width: 90%;
            font-size: 0.95rem;
        }

        /* 3D Journal Button */
        .journal-btn {
            background: #f1f5f9;
            color: #64748b;
            font-size: 0.75rem;
            font-weight: 700;
            padding: 8px 0;
            border-radius: 10px;
            border: 1px solid var(--glass-border);
            cursor: pointer;
            width: 90%;
            display: inline-block;
            box-shadow: 0 2px 0 rgba(0,0,0,0.05);
            transition: 0.2s;
        }
        .journal-btn.has-note {
            background: rgba(16, 185, 129, 0.1);
            color: var(--success);
            border-color: rgba(16, 185, 129, 0.2);
        }
        .journal-btn:active { transform: scale(0.95); }

        .totals-footer {
            display: flex; justify-content: space-around; padding: 15px;
            background: rgba(255,255,255,0.5); border-top: 1px solid var(--glass-border);
        }
        .total-item { text-align: center; }
        .total-label { font-size: 0.6rem; color: #94a3b8; font-weight: 800; text-transform: uppercase; }
        .total-val { display: block; font-size: 1.2rem; font-weight: 900; color: #0f172a; }

        /* Modal */
        .modal { display: none; position: fixed; inset: 0; background: rgba(15, 23, 42, 0.4); backdrop-filter: blur(10px); z-index: 1000; padding: 20px; }
        .modal-content { background: white; max-width: 450px; margin: 40px auto; padding: 25px; border-radius: 24px; box-shadow: var(--shadow); }
        textarea { width: 100%; border-radius: 14px; padding: 15px; background: #f8fafc; border: 1px solid #e2e8f0; margin: 15px 0; font-size: 1.1rem; outline: none; border: 2px solid transparent; transition: 0.3s; }
        textarea:focus { border-color: var(--accent); background: #fff; }

        @media print { .no-print { display: none !important; } .container { max-width: 100%; padding: 0; } }
    </style>
</head>
<body>

<div class="container">
    <div class="glass-card no-print">
        <span class="section-title">პარამეტრები</span>
        <div class="input-group">
            <label>ექთნის სახელი</label>
            <input type="text" id="nurseName" placeholder="სახელი და გვარი..." oninput="save()">
        </div>

        <div class="control-grid">
            <div class="input-group">
                <label>თვე</label>
                <select id="mSelect" onchange="render()"></select>
            </div>
            <div class="input-group">
                <label>წელი</label>
                <select id="ySelect" onchange="render()"></select>
            </div>
        </div>

        <div style="height: 1px; background: rgba(0,0,0,0.05); margin: 10px 0 20px 0;"></div>

        <span class="section-title">რეალური საათების ჩაწერა</span>
        <div class="control-grid">
            <div class="input-group">
                <label>რიცხვი</label>
                <input type="number" id="realDay" placeholder="1-31">
            </div>
            <div class="input-group">
                <label>საათი</label>
                <select id="realHours">
                    <option value="24">24 სთ</option>
                    <option value="16">16 სთ</option>
                    <option value="8">8 სთ</option>
                    <option value="0">0</option>
                </select>
            </div>
        </div>

        <button class="btn btn-primary" onclick="saveReal()">დაფიქსირება</button>
        <button class="btn btn-secondary" onclick="fillFuture()">წლის შევსება</button>
        <button class="btn btn-danger" onclick="clearFromNow()">გეგმის წაშლა</button>
    </div>

    <div class="glass-card table-card">
        <table>
            <thead>
                <tr>
                    <th style="width: 12%;">#</th>
                    <th style="width: 28%;">გეგმა</th>
                    <th style="width: 28%;">რეალ.</th>
                    <th style="width: 32%;">დღიური</th>
                </tr>
            </thead>
            <tbody id="bTable"></tbody>
        </table>
        
        <div class="totals-footer">
            <div class="total-item">
                <span class="total-label">გეგმა</span>
                <span id="totalPlan" class="total-val">0</span>
            </div>
            <div class="total-item">
                <span class="total-label">ნამუშევარი</span>
                <span id="totalReal" class="total-val" style="color: var(--accent);">0</span>
            </div>
        </div>
        <button class="btn btn-secondary no-print" style="border-radius:0; border:none; box-shadow:none; margin:0;" onclick="window.print()">ბეჭდვა</button>
    </div>
</div>

<div id="jModal" class="modal">
    <div class="modal-content">
        <h3 id="jTitle" style="font-weight: 800; color: #0f172a;"></h3>
        <p style="font-size: 0.9rem; color: #64748b; margin-top: 8px;">რა ისწავლეთ? / როგორ გაერთეთ?</p>
        <textarea id="jText" rows="6" placeholder="აღწერეთ დღე..."></textarea>
        <div style="display: flex; gap: 10px;">
            <button class="btn btn-secondary" style="flex:1; margin-bottom:0;" onclick="closeM()">გაუქმება</button>
            <button class="btn btn-primary" style="flex:1; margin-bottom:0;" onclick="saveJ()">შენახვა</button>
        </div>
    </div>
</div>

<script>
    let state = JSON.parse(localStorage.getItem('nurse_full_v13')) || { name: "", shifts: {} };
    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];
    let activeKey = null;

    function init() {
        const ms = document.getElementById('mSelect');
        const ys = document.getElementById('ySelect');
        months.forEach((m, i) => ms.add(new Option(m, i)));
        for(let y=2026; y<=2030; y++) ys.add(new Option(y, y));
        
        ms.value = new Date().getMonth();
        ys.value = new Date().getFullYear();
        document.getElementById('nurseName').value = state.name || "";
        render();
    }

    function render() {
        const m = parseInt(document.getElementById('mSelect').value);
        const y = parseInt(document.getElementById('ySelect').value);
        const days = new Date(y, m + 1, 0).getDate();
        
        let bHTML = "";
        let tPlan = 0, tReal = 0;

        for(let d=1; d<=days; d++) {
            const k = `d_${y}_${m}_${d}`;
            const pVal = state.shifts[k]?.plan || "";
            const rVal = state.shifts[k]?.real || "—";
            const noteText = state.shifts[k]?.note || "";
            const hasNoteClass = noteText ? "has-note" : "";
            const btnLabel = noteText ? "ნახვა" : "ჩაწერა";

            tPlan += parseInt(pVal) || 0;
            tReal += parseInt(rVal) || 0;

            bHTML += `
                <tr>
                    <td class="day-cell">${d}</td>
                    <td>
                        <select class="plan-select" onchange="upd('${k}','plan',this.value)">
                            <option value=""></option>
                            <option value="8" ${pVal=='8'?'selected':''}>8</option>
                            <option value="16" ${pVal=='16'?'selected':''}>16</option>
                            <option value="24" ${pVal=='24'?'selected':''}>24</option>
                        </select>
                    </td>
                    <td><div class="real-display">${rVal}</div></td>
                    <td>
                        <div class="journal-btn ${hasNoteClass}" onclick="openM('${k}', ${d})">
                            ${btnLabel}
                        </div>
                    </td>
                </tr>`;
        }
        document.getElementById('bTable').innerHTML = bHTML;
        document.getElementById('totalPlan').innerText = tPlan;
        document.getElementById('totalReal').innerText = tReal;
    }

    function saveReal() {
        const d = document.getElementById('realDay').value;
        const h = document.getElementById('realHours').value;
        const m = document.getElementById('mSelect').value;
        const y = document.getElementById('ySelect').value;
        if(!d || d < 1 || d > 31) return alert("მიუთითეთ სწორი რიცხვი");
        const k = `d_${y}_${m}_${d}`;
        if(!state.shifts[k]) state.shifts[k] = {};
        state.shifts[k].real = h;
        save(); render();
    }

    function clearFromNow() {
        const currentM = parseInt(document.getElementById('mSelect').value);
        const currentY = document.getElementById('ySelect').value;
        if(confirm(`${months[currentM]} თვიდან წლის ბოლომდე გეგმა წაიშლება?`)) {
            for(let m = currentM; m <= 11; m++) {
                for(let d = 1; d <= 31; d++) {
                    const k = `d_${currentY}_${m}_${d}`;
                    if(state.shifts[k]) state.shifts[k].plan = "";
                }
            }
            save(); render();
        }
    }

    function fillFuture() {
        const m = parseInt(document.getElementById('mSelect').value);
        const y = parseInt(document.getElementById('ySelect').value);
        let startD = null, hrs = 24;
        for(let d=1; d<=31; d++) {
            if(state.shifts[`d_${y}_${m}_${d}`]?.plan) {
                startD = new Date(y, m, d);
                hrs = state.shifts[`d_${y}_${m}_${d}`].plan;
                break;
            }
        }
        if(startD) {
            let c = new Date(startD);
            while(c.getFullYear() <= 2030) {
                const k = `d_${c.getFullYear()}_${c.getMonth()}_${c.getDate()}`;
                if(!state.shifts[k]) state.shifts[k] = {};
                state.shifts[k].plan = hrs;
                c.setDate(c.getDate() + 4);
            }
            save(); render();
        } else {
            alert("ჯერ მონიშნეთ ერთ-ერთი დღის გეგმა ცხრილში!");
        }
    }

    function upd(k, f, v) { if(!state.shifts[k]) state.shifts[k] = {}; state.shifts[k][f] = v; save(); render(); }
    function openM(k, d) { activeKey = k; document.getElementById('jTitle').innerText = `${d} რიცხვის დღიური`; document.getElementById('jText').value = state.shifts[k]?.note || ""; document.getElementById('jModal').style.display='block'; }
    function saveJ() { if(!state.shifts[activeKey]) state.shifts[activeKey] = {}; state.shifts[activeKey].note = document.getElementById('jText').value; save(); render(); closeM(); }
    function closeM() { document.getElementById('jModal').style.display='none'; }
    function save() { state.name = document.getElementById('nurseName').value; localStorage.setItem('nurse_full_v13', JSON.stringify(state)); }

    init();
</script>
</body>
</html>
