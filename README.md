<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ექთნების მორიგეობის ცხრილი</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root {
            --primary-blue: #003366;
            --dark-bg: #e9ecef;
            --real-row-bg: #d1d8e0;
            --total-red: #8b0000;
        }

        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1200px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        
        /* Header & Controls */
        .controls { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; flex-wrap: wrap; gap: 10px; }
        .btn { padding: 8px 15px; border: none; border-radius: 4px; cursor: pointer; color: white; transition: 0.3s; }
        .btn-blue { background: var(--primary-blue); }
        .btn-green { background: #28a745; }
        .btn-pdf { background: #dc3545; }
        .btn:hover { opacity: 0.8; }

        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; }

        /* Table Styling */
        .table-container { overflow-x: auto; margin-top: 20px; }
        table { width: 100%; border-collapse: collapse; font-size: 12px; }
        th, td { border: 1px solid #ccc; text-align: center; padding: 5px; min-width: 30px; }
        th { background: #f8f9fa; }
        
        .nurse-cell { font-weight: bold; background: #fff; width: 150px; text-align: left; }
        .real-row { background-color: var(--real-row-bg); }
        .total-cell { font-weight: bold; }
        .total-real { color: var(--total-red); }

        /* Form Styling */
        .forms-section { margin-top: 30px; border-top: 2px solid #eee; padding-top: 20px; display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
        .form-group { background: #fafafa; padding: 15px; border-radius: 5px; border: 1px solid #eee; }
        
        .search-results { margin-top: 10px; padding: 10px; background: #fff3cd; border-radius: 4px; display: none; }
    </style>
</head>
<body>

<div class="container">
    <h2>ექთნების მორიგეობის ცხრილი</h2>

    <div class="controls">
        <div>
            <input type="text" id="nurseName" placeholder="ექთნის სახელი">
            <button class="btn btn-blue" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="monthSelect" onchange="renderTable()">
                <option value="0">იანვარი</option>
                <option value="1">თებერვალი</option>
                <option value="2">მარტი</option>
                <option value="3">აპრილი</option>
                <option value="4">მაისი</option>
                <option value="5">ივნისი</option>
                <option value="6">ივლისი</option>
                <option value="7">აგვისტო</option>
                <option value="8">სექტემბერი</option>
                <option value="9">ოქტომბერი</option>
                <option value="10">ნოემბერი</option>
                <option value="11">დეკემბერი</option>
            </select>
            <input type="number" id="yearSelect" value="2024" style="width: 70px;" onchange="renderTable()">
        </div>
        <div>
            <button class="btn btn-blue" style="background: #001f3f;" onclick="autoFillAll()">ყველას შევსება მე-4 დღეზე</button>
            <button class="btn btn-pdf" onclick="exportPDF()">PDF ექსპორტი</button>
        </div>
    </div>

    <div class="table-container" id="mainTableContainer">
        </div>

    <div class="forms-section">
        <div class="form-group">
            <h4>რეალური საათის დაფიქსირება</h4>
            <input type="text" id="realNurseName" placeholder="ექთნის სახელი" list="nurseList">
            <datalist id="nurseList"></datalist>
            <select id="realDay"></select>
            <select id="realHours">
                <option value="0">0 სთ</option>
                <option value="8">8 სთ</option>
                <option value="16">16 სთ</option>
                <option value="24">24 სთ</option>
            </select>
            <button class="btn btn-green" onclick="saveRealHours()">შენახვა</button>
        </div>

        <div class="form-group">
            <h4>ძებნა</h4>
            <input type="text" id="searchNurse" placeholder="ექთნის სახელი">
            <input type="number" id="searchDay" placeholder="რიცხვი" min="1" max="31">
            <button class="btn btn-blue" onclick="searchData()">ძებნა</button>
            <div id="searchResults" class="search-results"></div>
        </div>
    </div>
</div>

<script>
    let nurses = JSON.parse(localStorage.getItem('nurses')) || [];
    let scheduleData = JSON.parse(localStorage.getItem('scheduleData')) || {};

    // Initialize
    window.onload = () => {
        const now = new Date();
        document.getElementById('monthSelect').value = now.getMonth();
        document.getElementById('yearSelect').value = now.getFullYear();
        updateDayDropdown();
        renderTable();
        updateNurseDatalist();
    };

    function addNurse() {
        const name = document.getElementById('nurseName').value.trim();
        if (name && !nurses.includes(name)) {
            nurses.push(name);
            saveData();
            renderTable();
            updateNurseDatalist();
            document.getElementById('nurseName').value = '';
        }
    }

    function removeNurse(name) {
        if(confirm(`დარწმუნებული ხართ, რომ გსურთ ${name}-ს წაშლა?`)) {
            nurses = nurses.filter(n => n !== name);
            saveData();
            renderTable();
            updateNurseDatalist();
        }
    }

    function getDaysInMonth(month, year) {
        return new Date(year, parseInt(month) + 1, 0).getDate();
    }

    function renderTable() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = getDaysInMonth(month, year);
        updateDayDropdown();

        let html = `<table><thead><tr><th>ექთანი / დღე</th>`;
        for (let i = 1; i <= days; i++) html += `<th>${i}</th>`;
        html += `<th>ჯამი</th><th>მოქმედება</th></tr></thead><tbody>`;

        nurses.forEach(nurse => {
            const keyBase = `${year}-${month}-${nurse}`;
            let plannedTotal = 0;
            let realTotal = 0;

            // Planned Row
            html += `<tr><td class="nurse-cell">${nurse}<br><small>(დაგეგმილი)</small></td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-p'] && scheduleData[keyBase + '-p'][i]) || 0;
                plannedTotal += parseInt(val);
                html += `<td>
                    <select onchange="updateCell('${nurse}', ${i}, this.value, 'p')">
                        <option value="0" ${val==0?'selected':''}>-</option>
                        <option value="8" ${val==8?'selected':''}>8</option>
                        <option value="16" ${val==16?'selected':''}>16</option>
                        <option value="24" ${val==24?'selected':''}>24</option>
                    </select>
                </td>`;
            }
            html += `<td class="total-cell">${plannedTotal}</td>
                     <td rowspan="2"><button onclick="removeNurse('${nurse}')" style="color:red">წაშლა</button></td></tr>`;

            // Real Row
            html += `<tr class="real-row"><td class="nurse-cell"><small>(რეალური)</small></td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-r'] && scheduleData[keyBase + '-r'][i]) || 0;
                realTotal += parseInt(val);
                html += `<td>${val || ''}</td>`;
            }
            html += `<td class="total-cell total-real">${realTotal}</td></tr>`;
        });

        html += `</tbody></table>`;
        document.getElementById('mainTableContainer').innerHTML = html;
    }

    function updateCell(nurse, day, val, type) {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const key = `${year}-${month}-${nurse}-${type}`;
        
        if (!scheduleData[key]) scheduleData[key] = {};
        scheduleData[key][day] = parseInt(val);
        saveData();
        renderTable();
    }

    function autoFillAll() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = getDaysInMonth(month, year);

        nurses.forEach(nurse => {
            const key = `${year}-${month}-${nurse}-p`;
            if (!scheduleData[key]) return;
            
            // პოულობს პირველ ჩაწერილ მორიგეობას
            let firstDay = 0;
            for (let i = 1; i <= days; i++) {
                if (scheduleData[key][i] > 0) {
                    firstDay = i;
                    break;
                }
            }

            if (firstDay > 0) {
                const hours = scheduleData[key][firstDay];
                for (let i = firstDay; i <= days; i += 4) {
                    scheduleData[key][i] = hours;
                }
            }
        });
        saveData();
        renderTable();
    }

    function saveRealHours() {
        const nurse = document.getElementById('realNurseName').value;
        const day = document.getElementById('realDay').value;
        const hours = document.getElementById('realHours').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;

        if (!nurses.includes(nurse)) { alert("ექთანი ვერ მოიძებნა"); return; }

        const key = `${year}-${month}-${nurse}-r`;
        if (!scheduleData[key]) scheduleData[key] = {};
        scheduleData[key][day] = parseInt(hours);
        
        saveData();
        renderTable();
    }

    function searchData() {
        const nurse = document.getElementById('searchNurse').value.trim();
        const day = document.getElementById('searchDay').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const resultsDiv = document.getElementById('searchResults');
        resultsDiv.style.display = 'block';
        
        let msg = "";

        if (nurse && !day) {
            const key = `${year}-${month}-${nurse}-p`;
            let days = [];
            if (scheduleData[key]) {
                for (let d in scheduleData[key]) if(scheduleData[key][d] > 0) days.push(d);
            }
            msg = days.length ? `${nurse}-ს მორიგეობები: ${days.join(', ')} რიცხვში.` : "მორიგეობები ვერ მოიძებნა.";
        } else if (day && !nurse) {
            let found = [];
            nurses.forEach(n => {
                const key = `${year}-${month}-${n}-p`;
                if (scheduleData[key] && scheduleData[key][day] > 0) found.push(n);
            });
            msg = found.length ? `${day} რიცხვში მორიგეობენ: ${found.join(', ')}` : "ამ დღეს არავინ მორიგეობს.";
        } else {
            msg = "გთხოვთ შეავსოთ სახელი ან რიცხვი.";
        }
        resultsDiv.innerText = msg;
    }

    function saveData() {
        localStorage.setItem('nurses', JSON.stringify(nurses));
        localStorage.setItem('scheduleData', JSON.stringify(scheduleData));
    }

    function updateDayDropdown() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = getDaysInMonth(month, year);
        let options = "";
        for (let i = 1; i <= days; i++) options += `<option value="${i}">${i}</option>`;
        document.getElementById('realDay').innerHTML = options;
    }

    function updateNurseDatalist() {
        let options = "";
        nurses.forEach(n => options += `<option value="${n}">`);
        document.getElementById('nurseList').innerHTML = options;
    }

    async function exportPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        const month = document.getElementById('monthSelect').options[document.getElementById('monthSelect').selectedIndex].text;
        const year = document.getElementById('yearSelect').value;

        doc.text(`მორიგეობის განრიგი: ${month} ${year}`, 14, 15);
        
        const table = document.querySelector("table");
        doc.autoTable({ 
            html: table,
            startY: 20,
            theme: 'grid',
            styles: { fontSize: 7, cellPadding: 1 },
            headStyles: { fillColor: [0, 51, 102] }
        });

        doc.save(`ganrigi_${month}_${year}.pdf`);
    }
</script>

</body>
</html>
