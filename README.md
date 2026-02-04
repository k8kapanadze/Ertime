<!DOCTYPE html>
<html lang="ka">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ექთნების მართვის სისტემა</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js"></script>
    <style>
        :root {
            --primary-blue: #001f3f;
            --real-row-bg: #d1d8e0;
            --total-red: #8b0000;
        }

        body { font-family: 'Segoe UI', sans-serif; margin: 20px; background: #f4f7f6; }
        .container { max-width: 1300px; margin: auto; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        
        /* Header Controls */
        .top-bar { display: flex; justify-content: space-between; align-items: flex-end; margin-bottom: 20px; gap: 15px; flex-wrap: wrap; }
        .main-controls { display: flex; gap: 10px; align-items: center; }
        
        .btn { padding: 10px 18px; border: none; border-radius: 4px; cursor: pointer; color: white; font-weight: bold; transition: 0.3s; }
        .btn-autofill { background: var(--primary-blue); }
        .btn-pdf { background: #dc3545; }
        .btn-add { background: #28a745; }

        /* Table */
        .table-container { overflow-x: auto; margin-top: 10px; }
        table { width: 100%; border-collapse: collapse; font-size: 13px; }
        th, td { border: 1px solid #ccc; text-align: center; padding: 6px; min-width: 35px; }
        th { background: #f8f9fa; position: sticky; top: 0; }
        
        .nurse-cell { 
            font-weight: bold; width: 180px; text-align: left; cursor: pointer; 
            user-select: none; position: relative;
        }
        .real-row { background-color: var(--real-row-bg); }
        .total-real { color: var(--total-red); font-weight: bold; }

        /* Context Menu for Long Press */
        #contextMenu {
            display: none; position: absolute; background: white; border: 1px solid #ccc;
            box-shadow: 0 2px 10px rgba(0,0,0,0.2); z-index: 1000; padding: 5px 0; border-radius: 4px;
        }
        #contextMenu button {
            display: block; width: 100%; padding: 8px 15px; border: none; background: none;
            text-align: left; cursor: pointer; font-size: 14px;
        }
        #contextMenu button:hover { background: #eee; }

        /* Forms Area */
        .footer-forms { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 30px; border-top: 2px solid #eee; padding-top: 20px; }
        .form-box { background: #fdfdfd; padding: 15px; border-radius: 6px; border: 1px solid #ddd; }
        input, select { padding: 8px; border: 1px solid #ccc; border-radius: 4px; margin-bottom: 5px; }

        .search-output { margin-top: 10px; padding: 10px; background: #e3f2fd; border-left: 4px solid #2196f3; display: none; }
    </style>
</head>
<body>

<div class="container">
    <div class="top-bar">
        <div class="main-controls">
            <button class="btn btn-autofill" onclick="autoFillAll()">ავტომატური შევსება (მე-4 დღე)</button>
            <input type="text" id="newNurseName" placeholder="ექთნის სახელი">
            <button class="btn btn-add" onclick="addNurse()">დამატება</button>
        </div>
        <div>
            <select id="monthSelect" onchange="renderTable()"></select>
            <input type="number" id="yearSelect" value="2024" style="width: 80px;" onchange="renderTable()">
            <button class="btn btn-pdf" onclick="exportPDF()">PDF ექსპორტი</button>
        </div>
    </div>

    <div class="table-container" id="tableWrapper"></div>

    <div class="footer-forms">
        <div class="form-box">
            <h3>რეალური საათის დაფიქსირება</h3>
            <input type="text" id="realNurseName" placeholder="ექთნის სახელი" list="nurseList">
            <datalist id="nurseList"></datalist>
            <select id="realDaySelect"></select>
            <select id="realHoursSelect">
                <option value="8">8 სთ</option>
                <option value="16">16 სთ</option>
                <option value="24">24 სთ</option>
                <option value="0">0 სთ</option>
            </select>
            <button class="btn btn-add" onclick="saveRealHours()">შენახვა</button>
        </div>

        <div class="form-box">
            <h3>ძებნა</h3>
            <div style="display: flex; gap: 5px; flex-direction: column;">
                <input type="text" id="searchNurse" placeholder="ჩაწერეთ სახელი (მორიგეობის დღეები)">
                <input type="number" id="searchDay" placeholder="ჩაწერეთ რიცხვი (ვინ მორიგეობს)">
                <button class="btn btn-autofill" onclick="performSearch()">ძებნა</button>
            </div>
            <div id="searchOutput" class="search-output"></div>
        </div>
    </div>
</div>

<div id="contextMenu">
    <button onclick="editNurseName()">რედაქტირება</button>
    <button onclick="deleteNurse()" style="color: red;">წაშლა</button>
</div>

<script>
    let nurses = JSON.parse(localStorage.getItem('nurses')) || [];
    let scheduleData = JSON.parse(localStorage.getItem('scheduleData')) || {};
    let activeNurse = null;
    let longPressTimer;

    const months = ["იანვარი", "თებერვალი", "მარტი", "აპრილი", "მაისი", "ივნისი", "ივლისი", "აგვისტო", "სექტემბერი", "ოქტომბერი", "ნოემბერი", "დეკემბერი"];

    window.onload = () => {
        const mSelect = document.getElementById('monthSelect');
        months.forEach((m, i) => mSelect.innerHTML += `<option value="${i}">${m}</option>`);
        mSelect.value = new Date().getMonth();
        renderTable();
        updateNurseList();
    };

    function renderTable() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = new Date(year, parseInt(month) + 1, 0).getDate();
        
        let html = `<table><thead><tr><th>ექთანი</th>`;
        for (let i = 1; i <= days; i++) html += `<th>${i}</th>`;
        html += `<th>ჯამი</th></tr></thead><tbody>`;

        nurses.forEach(nurse => {
            const keyBase = `${year}-${month}-${nurse}`;
            let pTotal = 0, rTotal = 0;

            // Planned Row
            html += `<tr><td class="nurse-cell" onmousedown="startPress(event, '${nurse}')" onmouseup="endPress()" onmouseleave="endPress()">${nurse}</td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-p'] && scheduleData[keyBase + '-p'][i]) || 0;
                pTotal += parseInt(val);
                html += `<td><select onchange="updateCell('${nurse}', ${i}, this.value, 'p')">
                    <option value="0" ${val==0?'selected':''}>-</option>
                    <option value="8" ${val==8?'selected':''}>8</option>
                    <option value="16" ${val==16?'selected':''}>16</option>
                    <option value="24" ${val==24?'selected':''}>24</option>
                </select></td>`;
            }
            html += `<td>${pTotal}</td></tr>`;

            // Real Row
            html += `<tr class="real-row"><td><small>რეალური</small></td>`;
            for (let i = 1; i <= days; i++) {
                const val = (scheduleData[keyBase + '-r'] && scheduleData[keyBase + '-r'][i]) || 0;
                rTotal += parseInt(val);
                html += `<td>${val || ''}</td>`;
            }
            html += `<td class="total-real">${rTotal}</td></tr>`;
        });

        html += `</tbody></table>`;
        document.getElementById('tableWrapper').innerHTML = html;
        updateDayDropdowns(days);
    }

    // Long Press Logic
    function startPress(e, nurse) {
        activeNurse = nurse;
        longPressTimer = setTimeout(() => {
            const menu = document.getElementById('contextMenu');
            menu.style.display = 'block';
            menu.style.left = e.pageX + 'px';
            menu.style.top = e.pageY + 'px';
        }, 600);
    }

    function endPress() { clearTimeout(longPressTimer); }

    document.addEventListener('click', () => document.getElementById('contextMenu').style.display = 'none');

    function deleteNurse() {
        if(confirm(`წავშალოთ ${activeNurse}?`)) {
            nurses = nurses.filter(n => n !== activeNurse);
            save();
            renderTable();
        }
    }

    function editNurseName() {
        const newName = prompt("შეიყვანეთ ახალი სახელი:", activeNurse);
        if(newName && newName !== activeNurse) {
            const idx = nurses.indexOf(activeNurse);
            nurses[idx] = newName;
            save();
            renderTable();
        }
    }

    function addNurse() {
        const name = document.getElementById('newNurseName').value.trim();
        if(name && !nurses.includes(name)) {
            nurses.push(name);
            document.getElementById('newNurseName').value = '';
            save();
            renderTable();
            updateNurseList();
        }
    }

    function autoFillAll() {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const days = new Date(year, parseInt(month) + 1, 0).getDate();

        nurses.forEach(nurse => {
            const key = `${year}-${month}-${nurse}-p`;
            if(!scheduleData[key]) return;
            let firstDay = Object.keys(scheduleData[key]).find(d => scheduleData[key][d] > 0);
            if(firstDay) {
                let hrs = scheduleData[key][firstDay];
                for(let i = parseInt(firstDay); i <= days; i += 4) scheduleData[key][i] = hrs;
            }
        });
        save();
        renderTable();
    }

    function saveRealHours() {
        const nurse = document.getElementById('realNurseName').value;
        const day = document.getElementById('realDaySelect').value;
        const hrs = document.getElementById('realHoursSelect').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;

        if(!nurses.includes(nurse)) return alert("ექთანი არ არსებობს!");
        const key = `${year}-${month}-${nurse}-r`;
        if(!scheduleData[key]) scheduleData[key] = {};
        scheduleData[key][day] = parseInt(hrs);
        save();
        renderTable();
    }

    function performSearch() {
        const sNurse = document.getElementById('searchNurse').value.trim();
        const sDay = document.getElementById('searchDay').value;
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const output = document.getElementById('searchOutput');
        output.style.display = 'block';

        if(sNurse && !sDay) {
            const key = `${year}-${month}-${sNurse}-p`;
            const days = scheduleData[key] ? Object.keys(scheduleData[key]).filter(d => scheduleData[key][d] > 0) : [];
            output.innerHTML = days.length ? `<strong>${sNurse}</strong> მორიგეობს: ${days.join(', ')} რიცხვში.` : "მორიგეობები ვერ მოიძებნა.";
        } else if(sDay) {
            let found = [];
            nurses.forEach(n => {
                const key = `${year}-${month}-${n}-p`;
                if(scheduleData[key] && scheduleData[key][sDay] > 0) found.push(n);
            });
            output.innerHTML = found.length ? `${sDay} რიცხვში მორიგეობენ: <strong>${found.join(', ')}</strong>` : "ამ დღეს მორიგე არავინაა.";
        }
    }

    function updateCell(nurse, day, val, type) {
        const month = document.getElementById('monthSelect').value;
        const year = document.getElementById('yearSelect').value;
        const key = `${year}-${month}-${nurse}-${type}`;
        if(!scheduleData[key]) scheduleData[key] = {};
        scheduleData[key][day] = parseInt(val);
        save();
        renderTable();
    }

    function save() {
        localStorage.setItem('nurses', JSON.stringify(nurses));
        localStorage.setItem('scheduleData', JSON.stringify(scheduleData));
    }

    function updateDayDropdowns(days) {
        let options = "";
        for(let i=1; i<=days; i++) options += `<option value="${i}">${i}</option>`;
        document.getElementById('realDaySelect').innerHTML = options;
    }

    function updateNurseList() {
        const dl = document.getElementById('nurseList');
        dl.innerHTML = "";
        nurses.forEach(n => dl.innerHTML += `<option value="${n}">`);
    }

    function exportPDF() {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF('l', 'mm', 'a4');
        const monthText = document.getElementById('monthSelect').options[document.getElementById('monthSelect').selectedIndex].text;
        doc.text(`განრიგი: ${monthText} ${document.getElementById('yearSelect').value}`, 14, 10);
        doc.autoTable({ 
            html: 'table', 
            startY: 15, 
            theme: 'grid', 
            styles: { fontSize: 7 },
            headStyles: { fillColor: [0, 31, 63] }
        });
        doc.save('ganrigi.pdf');
    }
</script>
</body>
</html>
