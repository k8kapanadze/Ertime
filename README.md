<!DOCTYPE html>
<html lang="ka">

<head>

<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>ErtimeCMC</title>

<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>

<style>

:root{
--blue:#001f3f;
--red:#8b0000;
--bg:#f4f7f6;
}

body{
font-family:Segoe UI,Tahoma;
margin:15px;
background:var(--bg);
}

.container{
max-width:1400px;
margin:auto;
background:white;
padding:20px;
border-radius:12px;
box-shadow:0 4px 20px rgba(0,0,0,0.15);
}

.top-grid{
display:grid;
grid-template-columns:1fr 1fr;
gap:20px;
margin-bottom:20px;
}

.box{
padding:20px;
background:#fafafa;
border-radius:10px;
border:1px solid #ddd;
}

.controls-bar{
display:flex;
justify-content:space-between;
align-items:center;
gap:10px;
flex-wrap:wrap;
margin-bottom:15px;
background:#eee;
padding:12px;
border-radius:8px;
}

#tableBox{
overflow-x:auto;
}

table{
width:100%;
min-width:900px;
border-collapse:collapse;
background:white;
}

th,td{
border:1px solid #ccc;
padding:10px;
text-align:center;
}

th{
background:var(--blue);
color:white;
position:sticky;
top:0;
}

.weekend{
background:#fff0f0;
}

.real-col{
background:#f8fbff;
cursor:pointer;
font-weight:bold;
color:var(--blue);
}

.has-note{
background:#fff9db;
}

input,select{
padding:8px;
border:1px solid #ccc;
border-radius:6px;
}

.btn{
padding:8px 16px;
border:none;
border-radius:6px;
cursor:pointer;
background:var(--blue);
color:white;
font-weight:bold;
}

.btn-green{
background:#28a745;
font-size:11px;
padding:4px 8px;
margin-top:5px;
}

#modal{
display:none;
position:fixed;
top:50%;
left:50%;
transform:translate(-50%,-50%);
background:white;
padding:25px;
width:400px;
border-radius:12px;
box-shadow:0 0 40px rgba(0,0,0,0.6);
z-index:1000;
}

.modal-overlay{
display:none;
position:fixed;
top:0;
left:0;
width:100%;
height:100%;
background:rgba(0,0,0,0.7);
z-index:999;
}

textarea{
width:100%;
height:100px;
padding:10px;
margin-top:10px;
border-radius:6px;
border:1px solid #ccc;
resize:none;
}

@media(max-width:768px){

.top-grid{
grid-template-columns:1fr;
}

.controls-bar{
flex-direction:column;
align-items:stretch;
}

input,select,.btn{
width:100%;
}

th,td{
padding:7px;
font-size:13px;
}

}

@media print{

.no-print,.top-grid,.btn-green{
display:none;
}

}

</style>

</head>

<body>

<div class="container">

<div class="top-grid no-print">

<div class="box">

<h3>რეალური საათი</h3>

<input id="rName" placeholder="ექთანი" list="nList">
<datalist id="nList"></datalist>

<select id="rDay"></select>

<select id="rHrs">
<option value="24">24</option>
<option value="16">16</option>
<option value="8">8</option>
<option value="0">0</option>
</select>

<button class="btn" onclick="saveReal()">შენახვა</button>

</div>

<div class="box">

<h3>ძებნა</h3>

<input id="sName" placeholder="სახელი">

<button class="btn" onclick="search()">ძებნა</button>

<div id="sOut" style="margin-top:10px;font-weight:bold"></div>

</div>

</div>

<div class="controls-bar no-print">

<div>

<input id="nurseInp" placeholder="სახელი">
<button class="btn" onclick="addNurse()">დამატება</button>

</div>

<div>

<select id="mSel" onchange="load()"></select>

<input id="ySel" type="number" value="2026" style="width:90px" onchange="load()">

<button class="btn" onclick="window.print()">ბეჭდვა</button>

</div>

</div>

<div id="tableBox"></div>

</div>

<div class="modal-overlay" id="overlay" onclick="closeModal()"></div>

<div id="modal">

<h3 id="modalTitle"></h3>

<div id="expDiv">

<label>სამუშაო გამოცდილება</label>
<textarea id="noteExp"></textarea>

</div>

<label>დღიური</label>
<textarea id="noteDaily"></textarea>

<button class="btn" style="width:100%;margin-top:10px" onclick="saveNote()">შენახვა</button>

</div>

<script>

const firebaseConfig={
apiKey:"YOURKEY",
authDomain:"YOURDOMAIN",
databaseURL:"YOURDB",
projectId:"YOURID"
};

firebase.initializeApp(firebaseConfig);

const db=firebase.database();

let nurses=[];
let scheduleData={};
let activeCell=null;

const months=["იანვარი","თებერვალი","მარტი","აპრილი","მაისი","ივნისი","ივლისი","აგვისტო","სექტემბერი","ოქტომბერი","ნოემბერი","დეკემბერი"];

db.ref().on("value",snap=>{
const val=snap.val()||{};
nurses=val.nurses||[];
scheduleData=val.scheduleData||{};
load();
});

window.onload=()=>{

const mSel=document.getElementById("mSel");

months.forEach((m,i)=>{
mSel.innerHTML+=`<option value="${i}">${m}</option>`;
});

mSel.value=new Date().getMonth();

document.getElementById("ySel").value=new Date().getFullYear();

};

function addNurse(){

const inp=document.getElementById("nurseInp");

const name=inp.value.trim();

if(!name)return;

if(nurses.includes(name)){
alert("არსებობს");
return;
}

nurses.push(name);

db.ref("nurses").set(nurses);

inp.value="";

}

function load(){

const m=parseInt(document.getElementById("mSel").value);
const y=parseInt(document.getElementById("ySel").value);

const daysInMonth=new Date(y,m+1,0).getDate();

let h=`<table><thead><tr><th rowspan="2">დღე</th>`;

nurses.forEach(n=>{

h+=`<th colspan="2">

${n}

<span class="no-print" onclick="delNurse('${n}')" style="cursor:pointer">❌</span>

<br>

<button class="btn btn-green no-print" onclick="autoFillByFirst('${n}')">წლის შევსება</button>

</th>`;

});

h+=`</tr><tr>`;

nurses.forEach(()=>{

h+=`<th>გეგმა</th><th>რეალური</th>`;

});

h+=`</tr></thead><tbody>`;

for(let d=1;d<=daysInMonth;d++){

const dayName=["კვ","ორ","სამ","ოთ","ხუთ","პარ","შაბ"][new Date(y,m,d).getDay()];

const isWknd=[0,6].includes(new Date(y,m,d).getDay());

h+=`<tr class="${isWknd?"weekend":""}">`;

h+=`<td><b>${d}</b><br><span style="font-size:10px">${dayName}</span></td>`;

nurses.forEach(n=>{

const pVal=(scheduleData[`${y}-${m}-${n}-p`]&&scheduleData[`${y}-${m}-${n}-p`][d])||0;

const rVal=(scheduleData[`${y}-${m}-${n}-r`]&&scheduleData[`${y}-${m}-${n}-r`][d])||0;

h+=`<td>

<select onchange="upd('${n}',${d},this.value,'p')">

<option value="0">-</option>

<option value="8">8</option>

<option value="16">16</option>

<option value="24">24</option>

</select>

</td>

<td class="real-col" onclick="openJournal('${n}',${d},${pVal},${rVal})">

${rVal||"-"}

</td>`;

});

h+=`</tr>`;

}

document.getElementById("tableBox").innerHTML=h+"</tbody></table>";

updateDLists(daysInMonth);

}

function upd(n,d,v,t){

const m=document.getElementById("mSel").value;
const y=document.getElementById("ySel").value;

db.ref(`scheduleData/${y}-${m}-${n}-${t}/${d}`).set(parseInt(v));

}

function saveReal(){

const n=document.getElementById("rName").value;

const d=document.getElementById("rDay").value;

const h=document.getElementById("rHrs").value;

const m=document.getElementById("mSel").value;
const y=document.getElementById("ySel").value;

db.ref(`scheduleData/${y}-${m}-${n}-r/${d}`).set(parseInt(h));

}

function autoFillByFirst(nurse){

const m=parseInt(document.getElementById("mSel").value);
const y=parseInt(document.getElementById("ySel").value);

const pKey=`${y}-${m}-${nurse}-p`;

let days=Object.keys(scheduleData[pKey]||{}).filter(d=>scheduleData[pKey][d]>0).map(Number).sort((a,b)=>a-b);

if(days.length===0){

alert("ჯერ მონიშნე პირველი სამუშაო დღე გეგმაში");

return;

}

const firstDay=days[0];

const hrs=scheduleData[pKey][firstDay];

if(!confirm(`${firstDay} რიცხვიდან შეივსოს ყოველ 4 დღეში?`))return;

let curr=new Date(y,m,firstDay);

let updates={};

while(curr.getFullYear()==y){

updates[`scheduleData/${curr.getFullYear()}-${curr.getMonth()}-${nurse}-p/${curr.getDate()}`]=hrs;

curr.setDate(curr.getDate()+4);

}

db.ref().update(updates);

}

function delNurse(n){

if(confirm("წავშალოთ?")){

nurses=nurses.filter(x=>x!==n);

db.ref("nurses").set(nurses);

}

}

function openJournal(nurse,day){

activeCell={nurse,day};

document.getElementById("modalTitle").innerText=nurse+" "+day;

document.getElementById("modal").style.display="block";
document.getElementById("overlay").style.display="block";

}

function closeModal(){

document.getElementById("modal").style.display="none";
document.getElementById("overlay").style.display="none";

}

function saveNote(){

const {nurse,day}=activeCell;

const m=document.getElementById("mSel").value;
const y=document.getElementById("ySel").value;

const exp=document.getElementById("noteExp").value;
const daily=document.getElementById("noteDaily").value;

db.ref(`scheduleData/${y}-${m}-${nurse}-notes/${day}`).set({exp,daily});

closeModal();

}

function updateDLists(d){

let opt="";

for(let i=1;i<=d;i++){

opt+=`<option value="${i}">${i}</option>`;

}

document.getElementById("rDay").innerHTML=opt;

let list="";

nurses.forEach(n=>{

list+=`<option value="${n}">`;

});

document.getElementById("nList").innerHTML=list;

}

function search(){

const sn=document.getElementById("sName").value;

const m=document.getElementById("mSel").value;
const y=document.getElementById("ySel").value;

const days=(scheduleData[`${y}-${m}-${sn}-p`])?

Object.keys(scheduleData[`${y}-${m}-${sn}-p`]).filter(d=>scheduleData[`${y}-${m}-${sn}-p`][d]>0)

:[];

document.getElementById("sOut").innerHTML=days.length?days.join(", "):"ვერ მოიძებნა";

}

</script>

</body>
</html>
