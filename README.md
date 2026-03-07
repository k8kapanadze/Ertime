<!DOCTYPE html>
<html lang="ka">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>მორიგეობის სისტემა</title>

<style>

body{
font-family:Arial;
background:#f4f6fb;
margin:0;
padding:20px;
}

h1{
text-align:center;
}

table{
width:100%;
border-collapse:collapse;
overflow-x:auto;
}

th,td{
border:1px solid #ddd;
text-align:center;
padding:6px;
font-size:13px;
}

th{
background:#0d6efd;
color:white;
}

.plan{
background:#eef3ff;
}

.real{
background:#fff;
cursor:pointer;
position:relative;
}

.note{
position:absolute;
top:2px;
right:2px;
font-size:12px;
}

select{
width:60px;
}

#panel{
display:flex;
gap:10px;
flex-wrap:wrap;
margin-bottom:20px;
}

button{
padding:8px 14px;
border:none;
background:#0d6efd;
color:white;
cursor:pointer;
border-radius:6px;
}

button:hover{
opacity:0.9;
}

input{
padding:6px;
}

#modal{
position:fixed;
top:0;
left:0;
width:100%;
height:100%;
background:rgba(0,0,0,0.4);
display:none;
align-items:center;
justify-content:center;
}

#modalBox{
background:white;
padding:20px;
border-radius:10px;
width:90%;
max-width:400px;
}

textarea{
width:100%;
height:80px;
}

</style>

</head>

<body>

<h1>მორიგეობის მართვა</h1>

<div id="panel">

<input id="search" placeholder="ძებნა ექთანის მიხედვით">

<select id="nurseSelect"></select>

<input type="number" id="dayInput" placeholder="რიცხვი">

<select id="hourInput">
<option>8</option>
<option>16</option>
<option>24</option>
</select>

<button onclick="saveReal()">რეალური საათი</button>

<button onclick="autoFill()">წლის შევსება</button>

</div>

<table id="schedule"></table>


<div id="modal">

<div id="modalBox">

<h3 id="modalTitle"></h3>

<div id="workField">
<label>სამუშაო გამოცდილება</label>
<textarea id="workText"></textarea>
</div>

<label>პირადი დღიური</label>
<textarea id="noteText"></textarea>

<br><br>

<button onclick="saveJournal()">შენახვა</button>
<button onclick="closeModal()">დახურვა</button>

</div>

</div>

<script type="module">

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore,doc,setDoc,getDoc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";


const firebaseConfig = {

apiKey:"YOUR_KEY",
authDomain:"YOUR_DOMAIN",
projectId:"YOUR_PROJECT"

};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);


const nurses=["ანა","მარი","ნინო","თეა"]

const days=365

let planData={}
let realData={}
let journalData={}

let currentCell=null


const table=document.getElementById("schedule")

function buildTable(){

let html="<tr><th>ექთანი</th>"

for(let d=1;d<=days;d++){
html+=`<th>${d}</th>`
}

html+="</tr>"

nurses.forEach(n=>{

html+=`<tr><td>${n}</td>`

for(let d=1;d<=days;d++){

let key=n+"_"+d

let plan=planData[key]||""
let real=realData[key]||""

let note=journalData[key]?"📝":""

html+=`
<td class="plan">
<select onchange="setPlan('${key}',this.value)">
<option></option>
<option ${plan==8?"selected":""}>8</option>
<option ${plan==16?"selected":""}>16</option>
<option ${plan==24?"selected":""}>24</option>
</select>
</td>

<td class="real" onclick="openJournal('${key}','${n}',${d})">
${real}
<span class="note">${note}</span>
</td>
`

}

html+="</tr>"

})

table.innerHTML=html

}


window.setPlan=function(key,val){

planData[key]=val

}


window.saveReal=function(){

let nurse=document.getElementById("nurseSelect").value
let day=document.getElementById("dayInput").value
let hour=document.getElementById("hourInput").value

let key=nurse+"_"+day

realData[key]=hour

buildTable()

}


window.openJournal=function(key,nurse,day){

currentCell=key

document.getElementById("modal").style.display="flex"

document.getElementById("modalTitle").innerText=nurse+" - "+day

let plan=planData[key]

if(plan){
document.getElementById("workField").style.display="block"
}else{
document.getElementById("workField").style.display="none"
}

let j=journalData[key]||{}

document.getElementById("workText").value=j.work||""
document.getElementById("noteText").value=j.note||""

}


window.closeModal=function(){

document.getElementById("modal").style.display="none"

}


window.saveJournal=function(){

journalData[currentCell]={

work:document.getElementById("workText").value,
note:document.getElementById("noteText").value

}

closeModal()

buildTable()

}


window.autoFill=function(){

let first=null

for(let k in planData){

if(planData[k]==24){
first=k
break
}

}

if(!first) return

let parts=first.split("_")

let nurse=parts[0]
let start=parseInt(parts[1])

for(let d=start;d<=days;d+=4){

planData[nurse+"_"+d]=24

}

buildTable()

}


function buildNurseSelect(){

let s=document.getElementById("nurseSelect")

nurses.forEach(n=>{

let o=document.createElement("option")

o.textContent=n

s.appendChild(o)

})

}

buildNurseSelect()
buildTable()

</script>

</body>
</html>
