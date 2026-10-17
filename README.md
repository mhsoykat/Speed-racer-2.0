<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no">
<title>Speed Racer 2.0</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#080f1c;color:white;font-family:Arial;overflow:hidden}
#game{display:block;width:100vw;height:100dvh;touch-action:none}
#hud{position:absolute;top:10px;left:5%;width:90%;display:flex;justify-content:space-between;font-size:13px;font-weight:bold;text-shadow:1px 2px 3px black}
#controls{position:absolute;bottom:22px;width:100%;display:flex;justify-content:space-around}
button{background:#1688df;color:white;border:0;border-radius:12px;padding:13px 18px;font-size:17px;font-weight:bold;touch-action:none}
#controls button{min-width:65px;background:#173b60cc}
#controls .boost{background:#f28b18}
.panel{position:absolute;inset:0;background:#081322f5;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:13px;padding:20px;overflow:auto}
.hidden{display:none}
h1{color:#27c7ff;text-align:center}
p{color:#b9c9da;text-align:center}
#garageList{display:grid;grid-template-columns:1fr 1fr;gap:10px;width:100%;max-width:420px}
.car{padding:15px;background:#19314b;border:1px solid #48799e;border-radius:12px;text-align:center}
.car button{font-size:12px;margin-top:8px}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="hud">
<span id="level">LEVEL 1</span>
<span id="score">SCORE 0</span>
<span id="coins">🪙 0</span>
</div>
<div id="controls">
<button id="left">◀</button>
<button id="nitro" class="boost">NITRO</button>
<button id="pause">Ⅱ</button>
<button id="right">▶</button>
</div>
<div id="menu" class="panel">
<h1>🏎️ SPEED RACER 2.0</h1>
<p>Night Highway Racing</p>
<p id="best">Best Score: 0</p>
<p id="wallet">Coins: 0</p>
<button onclick="startGame()">▶ START RACE</button>
<button onclick="openGarage()">🚘 GARAGE</button>
<p>Collect coins • Avoid traffic • Use Nitro</p>
</div>
<div id="garage" class="panel hidden">
<h1>CAR GARAGE</h1>
<p id="garageCoins"></p>
<div id="garageList"></div>
<button onclick="showMenu()">BACK</button>
</div>
<div id="paused" class="panel hidden">
<h1>PAUSED</h1>
<button onclick="resumeGame()">RESUME</button>
<button onclick="startGame()">RESTART</button>
<button onclick="showMenu()">MAIN MENU</button>
</div>
<div id="over" class="panel hidden">
<h1>GAME OVER</h1>
<p id="result"></p>
<button onclick="startGame()">RESTART</button>
<button onclick="showMenu()">MAIN MENU</button>
</div>
<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const $=id=>document.getElementById(id);
let W,H,roadX,roadW;
let running=false,paused=false;
let playerX,playerY,playerW=43,playerH=77;
let traffic=[],coins=[],particles=[];
let score=0,level=1,elapsed=0;
let direction=0,nitro=false,nitroFuel=100;
let last=0,spawnTimer=0,coinTimer=0;
let audio;
const cars=[
{name:"Starter",color:"#17baff",price:0},
{name:"Red Fury",color:"#ff334f",price:50},
{name:"Golden GT",color:"#ffca27",price:120},
{name:"Emerald X",color:"#16d99c",price:250},
{name:"Purple Storm",color:"#a879ff",price:450},
{name:"White Phantom",color:"#e9f3ff",price:750}
];
let save;
try{save=JSON.parse(localStorage.getItem("speedRacer2"))||{}}
catch(e){save={}}
save.wallet=save.wallet||0;
save.best=save.best||0;
save.selected=save.selected||0;
save.unlocked=save.unlocked||[0];
function persist(){
try{localStorage.setItem("speedRacer2",JSON.stringify(save))}catch(e){}
}
function resize(){
W=canvas.width=innerWidth;
H=canvas.height=innerHeight;
roadW=Math.min(W*.84,410);
roadX=(W-roadW)/2;
playerY=H-155;
if(!running)playerX=W/2;
}
addEventListener("resize",resize);
function sound(freq=300,duration=.08){
try{
audio=audio||new (window.AudioContext||window.webkitAudioContext)();
const osc=audio.createOscillator();
const gain=audio.createGain();
osc.type="sawtooth";
osc.frequency.value=freq;
gain.gain.setValueAtTime(.045,audio.currentTime);
gain.gain.exponentialRampToValueAtTime(.001,audio.currentTime+duration);
osc.connect(gain).connect(audio.destination);
osc.start();osc.stop(audio.currentTime+duration);
}catch(e){}
}
function panel(id){
["menu","garage","paused","over"].forEach(x=>$(x).classList.add("hidden"));
if(id)$(id).classList.remove("hidden");
}
function showMenu(){
running=false;paused=false;
$("best").textContent="Best Score: "+save.best;
$("wallet").textContent="Coins: "+save.wallet;
panel("menu");
}
function openGarage(){
running=false;
$("garageCoins").textContent="Your Coins: "+save.wallet;
$("garageList").innerHTML="";
cars.forEach((car,i)=>{
const owned=save.unlocked.includes(i);
const selected=save.selected===i;
const div=document.createElement("div");
div.className="car";
div.innerHTML='<div style="font-size:32px;color:'+car.color+'">🏎</div><b>'+car.name+'</b><br><button>'+(selected?"SELECTED":owned?"SELECT":"🪙 "+car.price)+'</button>';
div.querySelector("button").onclick=()=>{
if(owned){save.selected=i;sound(500)}
else if(save.wallet>=car.price){
save.wallet-=car.price;
save.unlocked.push(i);
save.selected=i;
sound(700,.2);
}else{alert("Not enough coins!")}
persist();openGarage();
};
$("garageList").appendChild(div);
});
panel("garage");
}
function startGame(){
resize();
score=0;level=1;elapsed=0;
traffic=[];coins=[];particles=[];
playerX=W/2;
direction=0;nitro=false;nitroFuel=100;
spawnTimer=0;coinTimer=0;
running=true;paused=false;
panel(null);sound(280,.2);
}
function resumeGame(){paused=false;panel(null)}
function pauseGame(){
if(!running)return;
paused=!paused;
panel(paused?"paused":null);
}
function endGame(){
running=false;
save.best=Math.max(save.best,Math.floor(score));
persist();
$("result").textContent="Score: "+Math.floor(score)+" | Level: "+level;
panel("over");sound(110,.4);
}
function drawCar(x,y,w,h,color,enemy=false){
ctx.save();
ctx.translate(x,y);
ctx.fillStyle="#0008";
ctx.fillRect(-w/2+5,-h/2+7,w,h);
ctx.fillStyle="#080b12";
ctx.fillRect(-w/2-5,-h*.32,9,h*.24);
ctx.fillRect(w/2-4,-h*.32,9,h*.24);
ctx.fillRect(-w/2-5,h*.15,9,h*.25);
ctx.fillRect(w/2-4,h*.15,9,h*.25);
ctx.fillStyle=color;
ctx.beginPath();
ctx.roundRect(-w/2,-h/2,w,h,9);
ctx.fill();
ctx.fillStyle="#122a43";
ctx.fillRect(-w*.34,-h*.22,w*.68,h*.25);
ctx.fillRect(-w*.34,h*.16,w*.68,h*.19);
ctx.fillStyle=enemy?"#ff3030":"#f8ffff";
ctx.fillRect(-w*.38,enemy?h*.40:-h*.46,w*.22,5);
ctx.fillRect(w*.16,enemy?h*.40:-h*.46,w*.22,5);
ctx.restore();
}
function collide(a,b){
return Math.abs(a.x-b.x)<(a.w+b.w)*.40 &&
Math.abs(a.y-b.y)<(a.h+b.h)*.43;
}
function spawnTraffic(){
const lanes=3;
const lane=Math.floor(Math.random()*lanes);
const x=roadX+roadW*(lane+.5)/lanes;
const colors=["#fa4554","#f3d45b","#e6e6e6","#6a9dff","#22c5a2"];
traffic.push({x,y:-100,w:43,h:75,color:colors[Math.floor(Math.random()*colors.length)]});
}
function spawnCoin(){
const lane=Math.floor(Math.random()*3);
coins.push({x:roadX+roadW*(lane+.5)/3,y:-25,r:11});
}
function update(dt){
if(!running||paused)return;
dt=Math.min(dt,.05);
elapsed+=dt;
level=Math.min(20,Math.floor(elapsed/18)+1);
const boosting=nitro&&nitroFuel>0;
const speed=(210+level*17)*(boosting?1.75:1);
nitroFuel=Math.max(0,Math.min(100,nitroFuel+(boosting?-38:13)*dt));
score+=dt*(10+level*2)*(boosting?1.7:1);
playerX+=direction*dt*300;
playerX=Math.max(roadX+playerW/2+8,Math.min(roadX+roadW-playerW/2-8,playerX));
spawnTimer+=dt;coinTimer+=dt;
if(spawnTimer>Math.max(.42,1.15-level*.025)){
spawnTraffic();spawnTimer=0;
}
if(coinTimer>1.5){spawnCoin();coinTimer=0}
traffic.forEach(t=>t.y+=speed*dt);
coins.forEach(c=>c.y+=speed*dt);
traffic=traffic.filter(t=>t.y<H+120);
coins=coins.filter(c=>c.y<H+30);
const player={x:playerX,y:playerY,w:playerW,h:playerH};
for(const t of traffic){
if(collide(player,t)){endGame();return}
}
coins=coins.filter(c=>{
if(Math.abs(c.x-playerX)<30&&Math.abs(c.y-playerY)<46){
save.wallet+=5;
persist();sound(850,.07);
return false;
}
return true;
});
$("score").textContent="SCORE "+Math.floor(score);
$("level").textContent="LEVEL "+level+"/20";
$("coins").textContent="🪙 "+save.wallet;
}
function draw(){
ctx.fillStyle="#10201a";ctx.fillRect(0,0,W,H);
ctx.fillStyle="#232b36";ctx.fillRect(roadX,0,roadW,H);
ctx.fillStyle="#d3d8da";
ctx.fillRect(roadX+5,0,4,H);
ctx.fillRect(roadX+roadW-9,0,4,H);
ctx.strokeStyle="#d6d9dc99";
ctx.lineWidth=3;ctx.setLineDash([32,35]);
const offset=(elapsed*220)%67;
ctx.lineDashOffset=-offset;
for(let i=1;i<3;i++){
const x=roadX+roadW*i/3;
ctx.beginPath();ctx.moveTo(x,0);ctx.lineTo(x,H);ctx.stroke();
}
ctx.setLineDash([]);
coins.forEach(c=>{
ctx.beginPath();ctx.arc(c.x,c.y,c.r,0,Math.PI*2);
ctx.fillStyle="#ffcf31";ctx.fill();
ctx.strokeStyle="#fff0a1";ctx.lineWidth=3;ctx.stroke();
ctx.fillStyle="#a16c00";ctx.font="bold 13px Arial";
ctx.textAlign="center";ctx.fillText("$",c.x,c.y+5);
});
traffic.forEach(t=>drawCar(t.x,t.y,t.w,t.h,t.color,true));
drawCar(playerX,playerY,playerW,playerH,cars[save.selected].color);
ctx.fillStyle="#182c3f";
ctx.fillRect(W*.25,H-82,W*.5,8);
ctx.fillStyle="#00c8ff";
ctx.fillRect(W*.25,H-82,W*.5*nitroFuel/100,8);
ctx.fillStyle="white";
ctx.font="12px Arial";
ctx.textAlign="center";
ctx.fillText("NITRO",W/2,H-91);
}
function loop(now){
const dt=(now-last)/1000||0;last=now;
update(dt);draw();
requestAnimationFrame(loop);
}
function hold(id,down,up){
const el=$(id);
el.addEventListener("pointerdown",e=>{
e.preventDefault();down();
try{el.setPointerCapture(e.pointerId)}catch(x){}
});
["pointerup","pointercancel","lostpointercapture"].forEach(type=>
el.addEventListener(type,up));
}
hold("left",()=>direction=-1,()=>direction=0);
hold("right",()=>direction=1,()=>direction=0);
hold("nitro",()=>{nitro=true;sound(160,.12)},()=>nitro=false);
$("pause").onclick=pauseGame;
canvas.addEventListener("pointermove",e=>{
if(running&&!paused&&e.buttons){
playerX=e.clientX;
}
});
canvas.addEventListener("pointerdown",e=>{
if(running&&!paused)playerX=e.clientX;
});
addEventListener("keydown",e=>{
if(e.key==="ArrowLeft")direction=-1;
if(e.key==="ArrowRight")direction=1;
if(e.key===" ")nitro=true;
if(e.key==="Escape")pauseGame();
});
addEventListener("keyup",e=>{
if(["ArrowLeft","ArrowRight"].includes(e.key))direction=0;
if(e.key===" ")nitro=false;
});
resize();showMenu();requestAnimationFrame(loop);
</script>
</body>
</html>
