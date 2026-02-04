<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>გრაფიკი ER - Final System</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root { --primary: #2c3e50; --plan: #3498db; --fact: #c0392b; --bg: #f4f7f6; }
        body { font-family: 'Segoe UI', sans-serif; margin: 0; background: var(--bg); font-size: 12px; }
        
        .header { background: var(--primary); color: white; padding: 10px 20px; display: flex; justify-content: space-between; align-items: center; }
        
        /* კონტროლის პანელი */
        .main-container { display: flex; flex-direction: column; height: 100vh; }
        .controls { background: white; padding: 15px; display: flex; flex-wrap: wrap; gap: 20px; border-bottom: 2px solid #ddd; }
        
        .btn-column { display: flex; flex-direction: column; gap: 8px; }
        button { cursor: pointer; padding: 10px 20px; border: none; border-radius: 4px; color: white; font-weight: bold; min-width: 200px; }
        .btn-save { background: #8e44ad; }
        .btn-pdf { background: #e67e22; }
        .btn-auto { background: #2980b9; }
        .btn-record { background: #27ae60; margin-top: 10px; }

        /* რეალური საათების დაფიქსირების ველი */
        .record-panel { background: #f9f9f9; padding: 15px; border: 1px solid #ddd; border-radius: 8px; }
        .record-panel select, .record-panel input { padding: 8px; margin-right: 5px; border-radius: 4px; border: 1px solid #ccc; }

        /* ცხრილი */
        .table-area { overflow: auto; flex-grow: 1; padding: 10px; }
        table { border-collapse: collapse; width: 100%; background: white; }
        th, td { border: 1px solid #ddd; padding: 4px; text-align: center; }
        th { background: #ecf0f1; position: sticky; top: 0; z-index: 10; }
        .name-col { position: sticky; left: 0; background: white; z-index: 11; font-weight: bold; min-width: 180px; text-align: left; }
        
        .row-plan { background: #f0f7ff; color: #2980b9; }
        .row-fact { background: #2c3e50; color: white; font-weight: bold; }
        .row-fact input { background: transparent; color: white; border: none; text-align: center; width: 100%; font-weight: bold; }
        
        .total-plan { color: var(--plan); font-weight: bold; }
        .total-fact { color: white; background: #c0392b; font-weight: bold; }
        
        .search-area { display: flex; gap: 10px; align-items: center; background: #eee; padding: 10px; border-radius: 5px; }
    </style>
</head>
<body>

<div class="main-container">
    <div class="header">
        <h2>✨ ChronoCare ER</h2>
        <div class="search-area">
            🔍 ძებნა: 
            <input type="text" id="searchName" placeholder="ექთნის სახელი..." onkeyup="filterTable()">
            <input type="number" id="searchDay" placeholder="რიცხვი (1-31)" oninput="filterTable()">
        </div>
    </div>

    <div class="controls">
        <div class="btn-column">
            <button class="btn-save" onclick="saveData()">💾 შენახვა</button>
            <button class="btn-pdf" onclick="downloadPDF()">📄 PDF ექსპორტი</button>
            <button class="btn-auto" onclick="autoFill4()">⚡ ყოველ მე-4 დღეს შევსება</button>
        </div>

        <div class="record-panel">
            <strong>📍 რეალური საათის დაფიქსირება:</strong><br><br>
            <select id="recNurse"></select>
            <input type="month" id="monthPicker" onchange="initTable()">
            <input type="number" id="recDay" placeholder="რიცხვი" min="1" max="31">
            <select id="recHours">
                <option value="8">8 სთ</option>
                <option value="16">16 სთ</option>
                <option value="24">24 სთ</option>
                <option value="0">0 (გაცვლა)</option>
            </select>
            <button class="btn-record" onclick="recordActual()">დაფიქსირება</button>
        </div>
    </div>

    <div class="table-area">
        <table id="schedTable">
            <thead>
                <tr id="headerRow">
                    <th class="name-col">ექთნების სია (51)</th>
                    <th>გეგმა (ჯამი)</th>
                    <th>რეალური (ჯამი)</th>
                </tr>
            </thead>
            <tbody id="tableBody"></tbody>
        </table>
    </div>
</div>

<script>
    let nurses = JSON.parse(localStorage.getItem('nurseList')) || Array.from({length: 51}, (_, i) => `ექთანი ${i+1}`);
    let scheduleData = JSON.parse(localStorage.getItem('scheduleData')) || {};

    // საწყისი ფუნქციები
    function initTable() {
        const picker = document.getElementById('monthPicker');
        if (!picker.value) {
            const now = new Date();
            picker.value = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
        }
        
        const [year, month] = picker.value.split('-').map(Number);
        const days = new Date(year, month, 0).getDate();
        const body = document.getElementById('tableBody');
        const header = document.getElementById('headerRow');

        // თარიღების სვეტების რენდერი
        while (header.cells.length > 3) header.deleteCell(1);
        for (let i = 1; i <= days; i++) {
            let th = document.createElement('th');
            th.innerText = i;
            header.insertBefore(th, header.cells[header.cells.length - 2]);
        }

        // ექთნების სელექტის შევსება (ჩამოსაშლელი მენიუ)
        const recNurse = document.getElementById('recNurse');
        if(recNurse.options.length === 0) {
            nurses.forEach((n, i) => recNurse.add(new Option(n, i)));
        }

        body.innerHTML = '';
        const mKey = picker.value;
        if (!scheduleData[mKey]) scheduleData[mKey] = {};

        nurses.forEach((name, nIdx) => {
            if (!scheduleData[mKey][nIdx]) {
                scheduleData[mKey][nIdx] = { plan: Array(days).fill(0), fact: Array(days).fill(0) };
            }

            let trPlan = document.createElement('tr'); trPlan.className = 'row-plan';
            let trFact = document.createElement('tr'); trFact.className = 'row-fact';
            trPlan.innerHTML = `<td rowspan="2" class="name-col" contenteditable="true" onblur="updateNurseName(${nIdx}, this.innerText)">${name}</td>`;

            let sumP = 0, sumF = 0;
            for (let d = 0; d < days; d++) {
                let p = scheduleData[mKey][nIdx].plan[d] || 0;
                let f = scheduleData[mKey][nIdx].fact[d] || 0;
                sumP += p; sumF += f;

                trPlan.innerHTML += `<td>${p > 0 ? p : '-'}</td>`;
                trFact.innerHTML += `<td>${f > 0 ? f : ''}</td>`;
            }

            trPlan.innerHTML += `<td class="total-plan">${sumP}</td><td rowspan="2" class="total-fact">${sumF}</td>`;
            body.appendChild(trPlan);
            body.appendChild(trFact);
        });
    }

    // რეალური საათის ჩაწერა პანელიდან
    function recordActual() {
        const mKey = document.getElementById('monthPicker').value;
        const nIdx = document.getElementById('recNurse').value;
        const day = document.getElementById('recDay').value;
        const hrs = document.getElementById('recHours').value;

        if(!day || day < 1 || day > 31) { alert("მიუთითეთ სწორი რიცხვი!"); return; }

        scheduleData[mKey][nIdx].fact[day-1] = Number(hrs);
        initTable();
    }

    // ავტო შევსება ყოველ მე-4 დღეს
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

    function updateNurseName(idx, newName) {
        nurses[idx] = newName;
        localStorage.setItem('nurseList', JSON.stringify(nurses));
    }

    function saveData() {
        localStorage.setItem('scheduleData', JSON.stringify(scheduleData));
        alert("მონაცემები შენახულია!");
    }

    function downloadPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        doc.setFontSize(16);
        doc.text(`ChronoCare ER: ${document.getElementById('monthPicker').value}`, 15, 15);
        doc.autoTable({ html: '#schedTable', theme: 'grid', startY: 20, styles: { fontSize: 7, cellPadding: 1 } });
        doc.save(`Schedule_${document.getElementById('monthPicker').value}.pdf`);
    }

    function filterTable() {
        let nameIn = document.getElementById('searchName').value.toLowerCase();
        let rows = document.querySelectorAll('#tableBody tr');
        for (let i = 0; i < rows.length; i += 2) {
            let n = rows[i].cells[0].innerText.toLowerCase();
            rows[i].style.display = rows[i+1].style.display = n.includes(nameIn) ? '' : 'none';
        }
    }

    window.onload = initTable;
</script>
</body>
</html>
