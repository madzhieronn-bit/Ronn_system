<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>MADEA$¥</title>

<!-- GOOGLE FONTS -->
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;600&family=JetBrains+Mono&display=swap" rel="stylesheet">

<style>
/* ===== GLOBAL ===== */
body{
    margin:0;
    min-height:100vh;
    background:radial-gradient(circle at top,#0f1a3a,#05070f 70%);
    font-family:'Orbitron',sans-serif;
    color:#e8faff;
    display:flex;
    justify-content:center;
    padding:20px;
}
.container{width:100%;max-width:560px}

/* ===== BOXES ===== */
.box{
    background:rgba(10,15,45,0.75);
    border:1px solid rgba(0,255,255,0.35);
    border-radius:16px;
    padding:16px;
    margin-bottom:22px;
    box-shadow:0 0 25px rgba(0,255,255,0.15);
    backdrop-filter:blur(12px);
}

/* ===== LOGO ===== */
.logo{
    font-size:30px;
    letter-spacing:4px;
    color:#9fffff;
    text-shadow:0 0 14px #00ffff;
    margin-bottom:10px;
}

/* ===== INPUTS ===== */
input,select,button{
    width:90%;
    padding:10px;
    margin:6px;
    border-radius:8px;
    border:1px solid rgba(0,255,255,0.45);
    background:#080c25;
    color:#ffffff;
    font-family:'Orbitron',sans-serif;
}
button{cursor:pointer;font-weight:600}

.start{background:#00d4aa;color:#002}
.stop{background:#ff3d5e;color:#fff}
.disconnect{background:#777;color:#fff}

/* ===== DASHBOARD ===== */
.account{
    font-size:14px;
    margin-top:8px;
}
#balance{color:#7fe7ff}
#profit{color:#00ff99}
#connStatus{margin-top:6px}

/* ===== STATUS ===== */
.status{
    margin-top:8px;
    font-size:14px;
    color:#bdf6ff;
}
.signal{
    color:#ffd94d!important;
    animation:pulse 1s infinite;
}
@keyframes pulse{
    0%{text-shadow:0 0 5px #ffd94d}
    50%{text-shadow:0 0 22px #ffd94d}
    100%{text-shadow:0 0 5px #ffd94d}
}

/* ===== FLASH ===== */
.flash{animation:flash 0.8s ease}
@keyframes flash{
    0%{box-shadow:0 0 0 rgba(0,255,180,0)}
    50%{box-shadow:0 0 45px rgba(0,255,180,0.85)}
    100%{box-shadow:0 0 0 rgba(0,255,180,0)}
}

/* ===== LOG ===== */
.log{
    margin-top:10px;
    height:120px;
    overflow-y:auto;
    background:#02040d;
    border:1px solid rgba(0,255,180,0.35);
    border-radius:8px;
    padding:10px;
    font-family:'JetBrains Mono',monospace;
    font-size:13px;
    color:#00ffae;
    text-align:left;
}
h3{margin:6px;color:#d9fbff}
</style>
</head>

<body>
<div class="container">

<!-- DASHBOARD -->
<div class="box">
    <div class="logo">MADEA$¥</div>
    <input id="token" placeholder="Deriv API Token">
    <button onclick="connect()">Connect</button>
    <button class="disconnect" onclick="disconnect()">Disconnect</button>
    <div id="connStatus">Disconnected</div>
    <div class="account">
        <span id="balance">Balance: --</span> |
        <span id="profit">Profit: 0.00</span>
    </div>
</div>

<!-- SCREEN 1 -->
<div class="box" id="screen1">
    <h3>SCREEN 1 (DIFFERS)</h3>
    <select id="market1"></select><br>
    Price: <span id="price1">--</span><br>
    <input id="stake1" type="number" value="1"><br>
    Ticks left: <span id="ticks1">0</span><br>
    <button class="start" onclick="start(1)">START</button>
    <button class="stop" onclick="stop(1)">STOP</button>
    <div class="status" id="status1">Waiting for signal.</div>
    <div class="log" id="log1"></div>
</div>

<!-- SCREEN 2 -->
<div class="box" id="screen2">
    <h3>SCREEN 2 (UNDER)</h3>
    <select id="market2"></select><br>
    Price: <span id="price2">--</span><br>
    <input id="stake2" type="number" value="1"><br>
    Ticks left: <span id="ticks2">0</span><br>
    <button class="start" onclick="start(2)">START</button>
    <button class="stop" onclick="stop(2)">STOP</button>
    <div class="status" id="status2">Waiting for signal.</div>
    <div class="log" id="log2"></div>
</div>

</div>

<script>
let ws=null,connected=false;
let running={1:false,2:false};
let inTrade={1:false,2:false};
let timers={1:null,2:null};
let lastBalance=0,totalProfit=0;

const symbols=["R_10","R_25","R_50","R_75","R_100"];

function log(id,msg){
    const el=document.getElementById("log"+id);
    el.innerHTML+=`[${new Date().toLocaleTimeString()}] ${msg}<br>`;
    el.scrollTop=el.scrollHeight;
}

function connect(){
    const token=tokenInput();
    if(!token)return alert("Enter API token");
    ws=new WebSocket("wss://ws.binaryws.com/websockets/v3?app_id=1089");
    ws.onopen=()=>ws.send(JSON.stringify({authorize:token}));
    ws.onmessage=e=>{
        const d=JSON.parse(e.data);
        if(d.authorize){
            connected=true;
            connStatus.innerText="Connected ✅";
            symbols.forEach(s=>{
                market1.add(new Option(s,s));
                market2.add(new Option(s,s));
            });
            ws.send(JSON.stringify({balance:1}));
            log(1,"Connected");
            log(2,"Connected");
        }
        if(d.balance){
            if(lastBalance){
                totalProfit+=(d.balance.balance-lastBalance);
                profit.innerText="Profit: "+totalProfit.toFixed(2);
            }
            lastBalance=d.balance.balance;
            balance.innerText="Balance: "+d.balance.balance;
        }
        if(d.tick)handleTick(d.tick.symbol,d.tick.quote);
    };
}

function disconnect(){
    if(ws)ws.close();
    connected=false;
    connStatus.innerText="Disconnected";
}

function start(id){
    if(!connected)return alert("Connect first");
    running[id]=true;
    status(id,"Waiting for signal.");
    log(id,"Strategy started");
    ws.send(JSON.stringify({ticks:market(id).value,subscribe:1}));
}

function stop(id){
    running[id]=false;
    inTrade[id]=false;
    clearInterval(timers[id]);
    ticks(id,0);
    status(id,"Waiting for signal.");
    log(id,"Strategy stopped");
}

function handleTick(symbol,price){
    [1,2].forEach(id=>{
        if(symbol===market(id).value){
            document.getElementById("price"+id).innerText=price.toFixed(2);
            const digit=price.toFixed(2).slice(-1);
            if(running[id]&&!inTrade[id]&&digit==="9"){
                inTrade[id]=true;
                status(id,"Signal Detected.");
                setTimeout(()=>executeTrade(id),5000);
            }
        }
    });
}

function executeTrade(id){
    log(id,"Executing trade");
    const box=document.getElementById("screen"+id);
    box.classList.add("flash");
    let t=10;
    ticks(id,t);
    timers[id]=setInterval(()=>{
        t--;ticks(id,t);
        if(t<=0){
            clearInterval(timers[id]);
            inTrade[id]=false;
            status(id,"Waiting for signal.");
            ws.send(JSON.stringify({balance:1}));
        }
    },1000);

    ws.send(JSON.stringify({
        buy:1,
        price:stake(id).value,
        parameters:{
            amount:stake(id).value,
            basis:"stake",
            contract_type:id===1?"DIGITDIFF":"DIGITUNDER",
            barrier:9,
            duration:10,
            duration_unit:"t",
            symbol:market(id).value,
            currency:"USD"
        }
    }));
}

function status(id,msg){
    const el=document.getElementById("status"+id);
    el.classList.remove("signal");
    el.innerText=msg;
    if(msg.includes("Signal"))el.classList.add("signal");
}
function ticks(id,v){document.getElementById("ticks"+id).innerText=v}
function market(id){return document.getElementById("market"+id)}
function stake(id){return document.getElementById("stake"+id)}
function tokenInput(){return document.getElementById("token").value.trim()}
</script>

</body>
</html>
