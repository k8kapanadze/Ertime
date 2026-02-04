<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ექთნების მორიგეობის ცხრილი</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.25/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root {
            --dark-blue: #1e3a8a;
            --dark-red: #991b1b;
            --bg-gray: #f3f4f6;
            --real-bg: #e5e7eb;
        }
        body { font-family: sans-serif; background-color: var(--bg-gray); padding: 20px; }
        .container { max-width: 1200px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        
        h2 { text-align: center; color: var(--dark-blue); }
        
        .controls { display: flex; gap: 10px; margin-bottom: 20px; align-items: center; flex-wrap: wrap; }
        select, input, button { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
        
        button { cursor: pointer; color: white; border: none; }
        .btn-fill { background-color: var(--dark-blue); }
        .btn-pdf { background-color: #059669; }
        .btn-save { background-color: #4f46e5; }

        /* ცხრილის სტილები */
        .table-container { overflow-x: auto; margin-top: 20px; }
        table { width: 100%; border-collapse: collapse; font-size: 12px; }
        th, td { border: 1px solid #ddd; text-align: center; padding: 4px; min-width: 30px; }
        th { background-color: #f8fafc; }
        
        .real-row { background-color: var(--real-bg); font-weight: bold; }
        .planned-total { font-weight: bold; }
        .real-total { color: var(--dark-red); font-weight: bold; }

        /* ფორმა ცხრილის ქვეშ */
        .entry-form { margin-top: 30px; padding: 20px; border-top: 2px solid #eee; background: #fafafa; }
        .search-section { margin-top: 30px; padding: 20px; border: 1px solid #ddd; border-radius: 8px; }
        
        .grid-inputs { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 15px; }
    </style>
</head>
<body>

<div class="container">
    <h2>ექთნების მორიგეობის ცხრილი</h2>

    <div class="controls">
        <label>აირჩიეთ წელი და თვე:</label>
        <input type="month" id="monthPicker" value="2024-05" onchange="generateTable()">
        <button class="btn-pdf" onclick="exportToPDF()">PDF ექსპორტი</button>
    </div>

    <div class="table-container" id="scheduleTable">
        </div>

    <div class="entry-form">
        <h3>რეალური საათის დაფიქსირება</h3>
        <div class="grid-inputs">
            <div>
                <label>ექთანი:</label>
                <select id="nurseSelect"></select>
            </div>
            <div>
                <label>რიცხვი:</label>
                <input type="number" id="daySelect" min="1" max="31">
            </div>
            <div>
                <label>საათები:</label>
                <input type="number" id="hourInput" min="0" max="24">
            </div>
            <button class="btn-save" onclick="updateRealHours()" style="margin-top: 20px;">შენახვა</button>
        </div>
    </div>

    <div class="search-section">
        <h3>ძებნა</h3>
        <div class="grid-inputs">
            <input type="text" id="searchNurse" placeholder="ექთნის სახელი" oninput="searchData()">
            <input type="number" id="searchDay" placeholder="კონკრეტული რიცხვი" oninput="searchData()">
        </div>
        <div id="searchResults" style="margin-top:15px; color: var(--dark-blue); font-weight: bold;"></div>
    </div>
</div>

<script>
    const nurses = ["ნინო კ.", "მარიამ ბ.", "ლელა მ.", "გიორგი ა.", "ეკა თ."];
    let data = {}; // მონაცემების საცავი: { "2024-05": { "ნინო": { planned: [], real: [] } } }

    function init() {
        // ექთნების სიის შევსება სელექტში
        const select = document.getElementById('nurseSelect');
        nurses.forEach(n => {
            let opt = document.createElement('option');
            opt.value = n;
            opt.textContent = n;
            select.appendChild(opt);
        });
        generateTable();
    }

    function getDaysInMonth(monthStr) {
        const [year, month] = monthStr.split('-');
        return new Date(year, month, 0).getDate();
    }

    function generateTable() {
        const month = document.getElementById('monthPicker').value;
        const daysCount = getDaysInMonth(month);
        const container = document.getElementById('scheduleTable');
        
        if (!data[month]) {
            data[month] = {};
            nurses.forEach(n => {
                data[month][n] = { 
                    planned: Array(daysCount).fill(''), 
                    real: Array(daysCount).fill('') 
                };
            });
        }

        let html = `<table><thead><tr><th>ექთანი</th><th>ტიპი</th>`;
        for (let i = 1; i <= daysCount; i++) html += `<th>${i}</th>`;
        html += `<th>ჯამი</th></tr></thead><tbody>`;

        nurses.forEach(nurse => {
            const plannedRow = data[month][nurse].planned;
            const realRow = data[month][nurse].real;
            const pSum = plannedRow.reduce((a, b) => a + (Number(b) || 0), 0);
            const rSum = realRow.reduce((a, b) => a + (Number(b) || 0), 0);

            // დაგეგმილი რიგი
            html += `<tr>
                <td rowspan="2"><b>${nurse}</b><br><button class="btn-fill" onclick="autoFill('${nurse}')" style="font-size:9px; padding:2px">მე-4 დღე</button></td>
                <td>დაგეგმ.</td>`;
            plannedRow.forEach((val, i) => {
                html += `<td><input type="number" value="${val}" onchange="updateCell('${nurse}', ${i}, 'planned', this.value)" style="width:25px; border:none; text-align:center;"></td>`;
            });
            html += `<td class="planned-total">${pSum}</td></tr>`;

            // რეალური რიგი
            html += `<tr class="real-row">
                <td>რეალ.</td>`;
            realRow.forEach(val => {
                html += `<td>${val}</td>`;
            });
            html += `<td class="real-total">${rSum}</td></tr>`;
        });

        html += `</tbody></table>`;
        container.innerHTML = html;
    }

    function updateCell(nurse, dayIndex, type, value) {
        const month = document.getElementById('monthPicker').value;
        data[month][nurse][type][dayIndex] = value;
        generateTable();
    }

    function autoFill(nurse) {
        const month = document.getElementById('monthPicker').value;
        const planned = data[month][nurse].planned;
        // ვპოულობთ პირველ ჩაწერილ მნიშვნელობას
        let firstIndex = planned.findIndex(val => val !== '');
        if (firstIndex === -1) {
            alert("გთხოვთ ჯერ ჩაწეროთ პირველი მორიგეობის საათი სასურველ რიცხვში.");
            return;
        }
        let hours = planned[firstIndex];
        for (let i = firstIndex; i < planned.length; i += 4) {
            planned[i] = hours;
        }
        generateTable();
    }

    function updateRealHours() {
        const nurse = document.getElementById('nurseSelect').value;
        const day = parseInt(document.getElementById('daySelect').value);
        const hours = document.getElementById('hourInput').value;
        const month = document.getElementById('monthPicker').value;
        const daysInMonth = getDaysInMonth(month);

        if (day > 0 && day <= daysInMonth) {
            data[month][nurse].real[day - 1] = hours;
            generateTable();
        } else {
            alert("მიუთითეთ სწორი რიცხვი");
        }
    }

    function searchData() {
        const nameQuery = document.getElementById('searchNurse').value.toLowerCase();
        const dayQuery = document.getElementById('searchDay').value;
        const month = document.getElementById('monthPicker').value;
        const resultsDiv = document.getElementById('searchResults');
        
        let results = [];

        if (nameQuery && !dayQuery) {
            // სად მუშაობს ეს ექთანი ამ თვეში
            for (let n in data[month]) {
                if (n.toLowerCase().includes(nameQuery)) {
                    let days = data[month][n].planned.map((v, i) => v !== '' ? i + 1 : null).filter(v => v);
                    results.push(`${n}: მორიგეობა ${days.join(', ')} რიცხვებში`);
                }
            }
        } else if (dayQuery) {
            // ვინ მუშაობს ამ დღეს
            let working = [];
            for (let n in data[month]) {
                if (data[month][n].planned[dayQuery - 1] !== '') working.push(n);
            }
            results.push(`${dayQuery} რიცხვში მორიგეები: ${working.length ? working.join(', ') : 'არავინ'}`);
        }

        resultsDiv.innerHTML = results.join('<br>');
    }

    async function exportToPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        const month = document.getElementById('monthPicker').value;
        
        doc.text(`მორიგეობის ცხრილი - ${month}`, 14, 15);
        
        const rows = [];
        const headers = ['ექთანი', 'ტიპი', ...Array.from({length: getDaysInMonth(month)}, (_, i) => i + 1), 'ჯამი'];

        nurses.forEach(n => {
            const pRow = [n, 'დაგეგმ.', ...data[month][n].planned, data[month][n].planned.reduce((a,b)=>a+(Number(b)||0), 0)];
            const rRow = ['', 'რეალ.', ...data[month][n].real, data[month][n].real.reduce((a,b)=>a+(Number(b)||0), 0)];
            rows.push(pRow);
            rows.push(rRow);
        });

        doc.autoTable({
            head: [headers],
            body: rows,
            startY: 20,
            styles: { fontSize: 7, cellPadding: 1 },
            didParseCell: function(data) {
                if (data.row.index % 2 !== 0) {
                    data.cell.styles.fillColor = [229, 231, 235];
                    if (data.column.index === headers.length - 1) {
                        data.cell.styles.textColor = [153, 27, 27];
                    }
                }
            }
        });

        doc.save(`Schedule_${month}.pdf`);
    }

    window.onload = init;
</script>

</body>
</html>
