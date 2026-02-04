
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ChronoCare ER - Advanced Scheduler</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root { --primary: #2c3e50; --plan-color: #3498db; --fact-color: #c0392b; --bg: #f4f7f6; }
        body { font-family: sans-serif; margin: 0; background: var(--bg); font-size: 12px; }
        .header { background: var(--primary); color: white; padding: 15px; text-align: center; }
        .controls { background: white; padding: 15px; display: flex; flex-wrap: wrap; gap: 10px; sticky; top: 0; z-index: 100; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        
        table { border-collapse: collapse; width: 100%; background: white; }
        th, td { border: 1px solid #ddd; padding: 4px; text-align: center; }
        th { background: #eee; position: sticky; top: 0; }
        .name-col { position: sticky; left: 0; background: white; min-width: 150px; font-weight: bold; z-index: 5; }
        
        .row-plan { background: #f0f7ff; color: var(--plan-color); font-size: 11px; }
        .row-fact { background: #2c3e50; color: white; font-weight: bold; }
        .row-fact input { background: transparent; color: white; border: none; text-align: center; width: 100%; }
        
        .total-plan { color: var(--plan-color); font-weight: bold; }
        .total-fact { color: #e74c3c; font-weight: bold; background: #ffeaea; }
        
        button { cursor: pointer; padding: 8px; border: none; border-radius: 4px; color: white; }
        .btn-blue { background: var(--plan-color); }
        .btn-red { background: var(--fact-color); }
        .btn-green { background: #27ae60; }
    </style>
</head>
<body>

<div class="header">
    <h1>🏥 ChronoCare ER System</h1>
</div>

<div class="controls">
    <label>თვე: <input type="month" id="monthPicker" onchange="initTable()"></label>
    <input type="text" id="searchName" placeholder="ძებნა სახელით..." onkeyup="filterTable()">
    <input type="number" id="searchDay" placeholder="რიცხვით..." min="1" max="31" oninput="filterTable()">
    <button class="btn-blue" onclick="autoFill4()">⚡ ყოველ მე-4 დღეს</button>
    <button class="btn-green" onclick="saveData()">💾 შენახვა</button>
    <button class="btn-red" onclick="downloadPDF()">📄 PDF ექსპორტი</button>
</div>

<div style="overflow: auto; height: 80vh;">
    <table id="schedTable">
        <thead>
            <tr id="headerRow">
                <th class="name-col">ექთნები</th>
                <th>გეგმა</th>
                <th>ფაქტი</th>
            </tr>
        </thead>
        <tbody id="tableBody"></tbody>
    </table>
</div>

<script>
    let nurses = JSON.parse(localStorage.getItem('nurseList')) || Array.from({length: 51}, (_, i) => `ექთანი ${i+1}`);
    let scheduleData = JSON.parse(localStorage.getItem('scheduleData')) || {};

    function getDaysInMonth(year, month) { return new Date(year, month + 1, 0).getDate(); }

    function initTable() {
        const picker = document.getElementById('monthPicker');
        if (!picker.value) {
            const now = new Date();
            picker.value = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
        }
        
        const [year, month] = picker.value.split('-').map(Number);
        const days = getDaysInMonth(year, month - 1);
        const headerRow = document.getElementById('headerRow');
        const body = document.getElementById('tableBody');

        // თარიღების სვეტების გაწმენდა და შექმნა
        while (headerRow.cells.length > 3) headerRow.deleteCell(1);
        for (let i = 1; i <= days; i++) {
            let th = document.createElement('th');
            th.innerText = i;
            headerRow.insertBefore(th, headerRow.cells[headerRow.cells.length - 2]);
        }

        body.innerHTML = '';
        nurses.forEach((name, nIdx) => {
            const monthKey = picker.value;
            if (!scheduleData[monthKey]) scheduleData[monthKey] = {};
            if (!scheduleData[monthKey][nIdx]) scheduleData[monthKey][nIdx] = { plan: Array(days).fill(0), fact: Array(days).fill(0) };

            // გეგმიური რიგი
            let trPlan = document.createElement('tr');
            trPlan.className = 'row-plan';
            trPlan.innerHTML = `<td rowspan="2" class="name-col">${name}</td>`;
            
            // ფაქტიური რიგი
            let trFact = document.createElement('tr');
            trFact.className = 'row-fact';

            let sumPlan = 0, sumFact = 0;

            for (let d = 0; d < days; d++) {
                let pVal = scheduleData[monthKey][nIdx].plan[d] || 0;
                let fVal = scheduleData[monthKey][nIdx].fact[d] || 0;
                sumPlan += pVal; sumFact += fVal;

                trPlan.innerHTML += `<td><select onchange="updateVal('${monthKey}', ${nIdx}, ${d}, 'plan', this.value)">
                    <option value="0" ${pVal==0?'selected':''}>-</option>
                    <option value="8" ${pVal==8?'selected':''}>8</option>
                    <option value="16" ${pVal==16?'selected':''}>16</option>
                    <option value="24" ${pVal==24?'selected':''}>24</option>
                </select></td>`;
                
                trFact.innerHTML += `<td><input type="number" value="${fVal}" onchange="updateVal('${monthKey}', ${nIdx}, ${d}, 'fact', this.value)"></td>`;
            }

            trPlan.innerHTML += `<td class="total-plan" id="tp-${nIdx}">${sumPlan}</td><td rowspan="2" class="total-fact" id="tf-${nIdx}">${sumFact}</td>`;
            body.appendChild(trPlan);
            body.appendChild(trFact);
        });
    }

    function updateVal(mKey, nIdx, dIdx, type, val) {
        scheduleData[mKey][nIdx][type][dIdx] = Number(val);
        initTable(); // გადათვლა
    }

    function autoFill4() {
        const mKey = document.getElementById('monthPicker').value;
        nurses.forEach((_, i) => {
            let start = i % 4;
            for (let d = start; d < 31; d += 4) {
                if (scheduleData[mKey][i].plan[d] === 0) scheduleData[mKey][i].plan[d] = 24;
            }
        });
        initTable();
    }

    function filterTable() {
        let name = document.getElementById('searchName').value.toLowerCase();
        let day = document.getElementById('searchDay').value;
        let rows = document.querySelectorAll('#tableBody tr');
        
        for (let i = 0; i < rows.length; i += 2) {
            let nurseName = rows[i].cells[0].innerText.toLowerCase();
            let show = nurseName.includes(name);
            rows[i].style.display = show ? '' : 'none';
            rows[i+1].style.display = show ? '' : 'none';
        }
    }

    function saveData() {
        localStorage.setItem('scheduleData', JSON.stringify(scheduleData));
        alert("შენახულია!");
    }

    async function downloadPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        doc.text(`ER Schedule: ${document.getElementById('monthPicker').value}`, 10, 10);
        doc.autoTable({ html: '#schedTable', theme: 'grid', styles: { fontSize: 7 } });
        doc.save("Schedule.pdf");
    }

    window.onload = initTable;
</script>
</body>
</html>
