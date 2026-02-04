<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <title>ექთნების მართვის პანელი</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.25/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root { --dark-blue: #1e3a8a; --dark-red: #991b1b; --bg-gray: #f3f4f6; }
        body { font-family: sans-serif; background-color: var(--bg-gray); padding: 20px; font-size: 14px; }
        .container { max-width: 1300px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        
        .header-section { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 20px; gap: 20px; }
        .nurse-management, .main-controls { background: #fff; padding: 15px; border: 1px solid #ddd; border-radius: 8px; }
        
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { border: 1px solid #ccc; text-align: center; padding: 5px; }
        th { background: #f1f5f9; }
        
        .real-row { background-color: #f1f1f1; font-weight: bold; }
        .real-total { color: var(--dark-red); }
        
        button { cursor: pointer; border: none; border-radius: 4px; padding: 8px 12px; color: white; }
        .btn-add { background: #2563eb; }
        .btn-delete { background: #dc2626; padding: 2px 6px; font-size: 10px; }
        .btn-fill { background: var(--dark-blue); font-weight: bold; width: 100%; margin-top: 10px; }
        .btn-pdf { background: #059669; }

        select { padding: 4px; border-radius: 4px; border: 1px solid #ccc; }
        .entry-form { margin-top: 20px; padding: 15px; background: #f9fafb; border: 1px solid #ddd; border-radius: 8px; }
        .input-group { display: flex; gap: 10px; align-items: center; margin-top: 10px; }
    </style>
</head>
<body>

<div class="container">
    <div class="header-section">
        <div class="nurse-management">
            <h3>ექთნების სია</h3>
            <div style="display:flex; gap:5px; margin-bottom:10px;">
                <input type="text" id="newNurseName" placeholder="სახელი გვარი">
                <button class="btn-add" onclick="addNurse()">დამატება</button>
            </div>
            <ul id="nurseUl" style="list-style:none; padding:0; max-height: 150px; overflow-y: auto;"></ul>
        </div>

        <div class="main-controls">
            <label>თვე:</label>
            <input type="month" id="monthPicker" value="2024-05" onchange="generateTable()">
            <button class="btn-pdf" onclick="exportToPDF()">PDF ექსპორტი</button>
            <button class="btn-fill" onclick="autoFillAll()">ყველას შევსება მე-4 დღეზე (მუქი ლურჯი)</button>
        </div>
    </div>

    <div id="tableWrapper" style="overflow-x: auto;"></div>

    <div class="entry-form">
        <h3>რეალური საათის დაფიქსირება</h3>
        <div class="input-group">
            <input type="text" id="realNurseName" placeholder="ჩაწერეთ სახელი">
            <select id="realDaySelect"></select>
            <select id="realHourSelect">
                <option value="0">0 სთ</option>
                <option value="8">8 სთ</option>
                <option value="16">16 სთ</option>
                <option value="24">24 სთ</option>
            </select>
            <button class="btn-add" onclick="saveRealHours()">შენახვა</button>
        </div>
    </div>
</div>

<script>
    let nurses = JSON.parse(localStorage.getItem('nurses')) || ["ნინო კ.", "მარიამ ბ."];
    let scheduleData = JSON.parse(localStorage.getItem('scheduleData')) || {};

    function saveToStorage() {
        localStorage.setItem('nurses', JSON.stringify(nurses));
        localStorage.setItem('scheduleData', JSON.stringify(scheduleData));
    }

    function addNurse() {
        const name = document.getElementById('newNurseName').value.trim();
        if (name && !nurses.includes(name)) {
            nurses.push(name);
            document.getElementById('newNurseName').value = '';
            saveToStorage();
            renderNurseList();
            generateTable();
        }
    }

    function deleteNurse(name) {
        nurses = nurses.filter(n => n !== name);
        saveToStorage();
        renderNurseList();
        generateTable();
    }

    function renderNurseList() {
        const ul = document.getElementById('nurseUl');
        ul.innerHTML = nurses.map(n => `
            <li style="display:flex; justify-content:space-between; margin-bottom:5px; border-bottom:1px solid #eee; padding-bottom:3px;">
                ${n} <button class="btn-delete" onclick="deleteNurse('${n}')">X</button>
            </li>
        `).join('');
    }

    function generateTable() {
        const month = document.getElementById('monthPicker').value;
        const daysCount = new Date(month.split('-')[0], month.split('-')[1], 0).getDate();
        const wrapper = document.getElementById('tableWrapper');

        // რეალური საათების რიცხვის სელექტის განახლება
        const daySelect = document.getElementById('realDaySelect');
        daySelect.innerHTML = Array.from({length: daysCount}, (_, i) => `<option value="${i+1}">${i+1}</option>`).join('');

        if (!scheduleData[month]) scheduleData[month] = {};
        
        let html = `<table><thead><tr><th>ექთანი</th><th>ტიპი</th>`;
        for (let i = 1; i <= daysCount; i++) html += `<th>${i}</th>`;
        html += `<th>ჯამი</th></tr></thead><tbody>`;

        nurses.forEach(nurse => {
            if (!scheduleData[month][nurse]) {
                scheduleData[month][nurse] = { planned: Array(daysCount).fill(''), real: Array(daysCount).fill('') };
            }
            const pData = scheduleData[month][nurse].planned;
            const rData = scheduleData[month][nurse].real;
            const pSum = pData.reduce((a, b) => a + (Number(b) || 0), 0);
            const rSum = rData.reduce((a, b) => a + (Number(b) || 0), 0);

            // დაგეგმილი
            html += `<tr><td rowspan="2"><b>${nurse}</b></td><td>დაგეგმ.</td>`;
            pData.forEach((val, i) => {
                html += `<td>
                    <select onchange="updateHour('${nurse}', ${i}, 'planned', this.value)">
                        <option value=""></option>
                        <option value="8" ${val == 8 ? 'selected' : ''}>8</option>
                        <option value="16" ${val == 16 ? 'selected' : ''}>16</option>
                        <option value="24" ${val == 24 ? 'selected' : ''}>24</option>
                    </select>
                </td>`;
            });
            html += `<td>${pSum}</td></tr>`;

            // რეალური
            html += `<tr class="real-row"><td>რეალ.</td>`;
            rData.forEach(val => html += `<td>${val || ''}</td>`);
            html += `<td class="real-total">${rSum}</td></tr>`;
        });

        html += `</tbody></table>`;
        wrapper.innerHTML = html;
    }

    function updateHour(nurse, day, type, val) {
        const month = document.getElementById('monthPicker').value;
        scheduleData[month][nurse][type][day] = val;
        saveToStorage();
        generateTable();
    }

    function autoFillAll() {
        const month = document.getElementById('monthPicker').value;
        nurses.forEach(nurse => {
            const planned = scheduleData[month][nurse].planned;
            let firstIdx = planned.findIndex(v => v !== '');
            if (firstIdx !== -1) {
                let val = planned[firstIdx];
                for (let i = firstIdx; i < planned.length; i += 4) {
                    planned[i] = val;
                }
            }
        });
        saveToStorage();
        generateTable();
    }

    function saveRealHours() {
        const name = document.getElementById('realNurseName').value;
        const day = document.getElementById('realDaySelect').value - 1;
        const hour = document.getElementById('realHourSelect').value;
        const month = document.getElementById('monthPicker').value;

        if (nurses.includes(name)) {
            scheduleData[month][name].real[day] = hour;
            saveToStorage();
            generateTable();
        } else {
            alert("ასეთი ექთანი ვერ მოიძებნა!");
        }
    }

    async function exportToPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        const month = document.getElementById('monthPicker').value;
        doc.text(`Schedule: ${month}`, 14, 10);
        doc.autoTable({ html: 'table', startY: 15, theme: 'grid', styles: {fontSize: 7} });
        doc.save(`Schedule_${month}.pdf`);
    }

    renderNurseList();
    generateTable();
</script>
</body>
</html>
