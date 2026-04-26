# Розробка додатку для візуалізації вимірювань радару

## Мета роботи
Розробити додаток, який зчитує дані з емульованої вимірювальної частини радару, наданої у вигляді Docker image, та відображає задетектовані цілі на графіку в полярних координатах.

### 1. Підготовка середовища

Було встановлено Docker Desktop для запуску контейнера з емулятором радара.

Після цього було завантажено Docker-образ сервісу радара та запущено локальний сервер на порту `4000`.

### 2. Створення структури веб-додатку

Було створено HTML-файл, який містить:

- область для побудови графіка;
- панель керування параметрами радара;
- блок статусу з’єднання;
- журнал подій (логи).

### 3. Реалізація підключення до WebSocket сервера

Було реалізовано підключення до сервера за адресою:

```text
ws://localhost:4000
```

Для відображення цілей використано бібліотеку Plotly.

Для кращого аналізу реалізовано різні кольори та розміри точок залежно від потужності сигналу:

слабкий сигнал - жовтий колір;
середній сигнал - помаранчевий;
сильний сигнал - червоний.

Код:

```
<!DOCTYPE html>
<html lang="uk">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Radar Control Panel</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/plotly.js/2.26.0/plotly.min.js"></script>

<style>
*{
    margin:0;
    padding:0;
    box-sizing:border-box;
}

body{
    font-family:Arial, sans-serif;
    background:#0b1220;
    color:#e2e8f0;
    min-height:100vh;
}

header{
    padding:18px 25px;
    background:#111827;
    border-bottom:1px solid #1f2937;
    font-size:28px;
    font-weight:700;
}

.wrapper{
    display:grid;
    grid-template-columns:1fr 320px;
    gap:18px;
    padding:18px;
}

.chart-box,
.panel{
    background:#111827;
    border-radius:14px;
    padding:18px;
    box-shadow:0 10px 30px rgba(0,0,0,.25);
}

.chart-box{
    min-height:760px;
}

.chart-title{
    font-size:20px;
    margin-bottom:12px;
    color:#60a5fa;
    font-weight:700;
}

#chart{
    width:100%;
    height:700px;
}

.block{
    margin-bottom:22px;
}

.block h2{
    font-size:18px;
    margin-bottom:14px;
    color:#60a5fa;
}

label{
    display:block;
    margin-top:10px;
    margin-bottom:6px;
    font-size:14px;
}

input{
    width:100%;
    padding:10px;
    border:none;
    border-radius:8px;
    background:#1f2937;
    color:white;
}

button{
    width:100%;
    padding:12px;
    margin-top:14px;
    border:none;
    border-radius:8px;
    background:#2563eb;
    color:white;
    font-weight:bold;
    cursor:pointer;
    transition:.2s;
}

button:hover{
    background:#1d4ed8;
    transform:translateY(-2px);
    box-shadow:0 0 15px rgba(37,99,235,.45);
}

.status{
    display:flex;
    align-items:center;
    gap:10px;
    margin-bottom:14px;
}

.dot{
    width:12px;
    height:12px;
    border-radius:50%;
    background:#ef4444;
}

.dot.online{
    background:#22c55e;
    box-shadow:0 0 10px #22c55e;
}

.info-row{
    display:flex;
    justify-content:space-between;
    padding:8px 0;
    border-bottom:1px solid #1f2937;
}

.log{
    background:#0b1220;
    border-radius:8px;
    padding:10px;
    height:190px;
    overflow:auto;
    font-size:13px;
    font-family:monospace;
}

.log div{
    margin-bottom:6px;
}

@media(max-width:1100px){
    .wrapper{
        grid-template-columns:1fr;
    }
}
</style>
</head>

<body>

<header>Radar Control Panel</header>

<div class="wrapper">

<div class="chart-box">
    <div class="chart-title">Detected Targets in Real Time</div>
    <div id="chart"></div>
</div>

<div class="panel">

    <div class="block">

        <div class="status">
            <div class="dot" id="stateDot"></div>
            <span id="stateText">Disconnected</span>
        </div>

        <div class="info-row">
            <span>Packets</span>
            <span id="packets">0</span>
        </div>

        <div class="info-row">
            <span>Targets</span>
            <span id="targets">0</span>
        </div>

        <div class="info-row">
            <span>Last Angle</span>
            <span id="angle">-</span>
        </div>

    </div>

    <div class="block">

        <h2>Radar Settings</h2>

        <label>Measurements Per Rotation</label>
        <input id="mpr" type="number" value="360">

        <label>Rotation Speed (RPM)</label>
        <input id="rpm" type="number" value="60">

        <label>Target Speed (km/h)</label>
        <input id="speed" type="number" value="100">

        <button onclick="saveConfig()">Apply Settings</button>

    </div>

    <div class="block">
        <h2>Logs</h2>
        <div class="log" id="logBox"></div>
    </div>

</div>
</div>

<script>
const LIGHT_SPEED = 299792458;
const MARK_LIFE = 5000;
const MAX_DISTANCE = 200;

let ws;
let scans = [];
let packetCounter = 0;

/* ---------- LOG ---------- */
function addLog(text,color="#e2e8f0"){
    const box = document.getElementById("logBox");

    const row = document.createElement("div");
    row.textContent = "[" + new Date().toLocaleTimeString() + "] " + text;
    row.style.color = color;

    box.prepend(row);

    while(box.children.length > 40){
        box.removeChild(box.lastChild);
    }
}

/* ---------- CHART ---------- */
function buildChart(){

    Plotly.newPlot("chart",[{
        type:"scatterpolar",
        mode:"markers",
        r:[],
        theta:[],
        marker:{
            size:[],
            color:[],
            opacity:[],
            colorscale:[
                [0,"#fde047"],
                [0.5,"#fb923c"],
                [1,"#ef4444"]
            ],
            cmin:0,
            cmax:1
        }
    }],
    {
        showlegend:false,
        paper_bgcolor:"#111827",
        plot_bgcolor:"#111827",
        font:{color:"#e2e8f0"},

        polar:{
            bgcolor:"#111827",

            radialaxis:{
                visible:true,
                range:[0,200],
                ticksuffix:" km",
                gridcolor:"#334155"
            },

            angularaxis:{
                rotation:90,
                direction:"clockwise",
                gridcolor:"#334155"
            }
        },

        margin:{t:20,l:40,r:40,b:30}

    },
    {responsive:true});
}

/* ---------- WEBSOCKET ---------- */
function openConnection(){

    ws = new WebSocket("ws://localhost:4000");

    ws.onopen = () => {
        document.getElementById("stateDot").classList.add("online");
        document.getElementById("stateText").textContent = "Connected";
        addLog("WebSocket connected","#22c55e");
    };

    ws.onclose = () => {
        document.getElementById("stateDot").classList.remove("online");
        document.getElementById("stateText").textContent = "Disconnected";
        addLog("Connection closed. Reconnecting...","#ef4444");

        setTimeout(openConnection,3000);
    };

    ws.onerror = () => {
        addLog("WebSocket error","#ef4444");
    };

    ws.onmessage = (event) => {

        try{
            const msg = JSON.parse(event.data);
            handlePacket(msg);
        }
        catch(error){
            addLog("Invalid packet","#ef4444");
        }

    };
}

/* ---------- HANDLE DATA ---------- */
function handlePacket(data){

    packetCounter++;
    document.getElementById("packets").textContent = packetCounter;

    document.getElementById("angle").textContent =
        data.scanAngle != null ? data.scanAngle + "°" : "-";

    const now = Date.now();

    scans = scans.filter(item =>
        now - item.timeStamp < MARK_LIFE
    );

    if(data.echoResponses && data.echoResponses.length > 0){

        data.echoResponses.forEach(item => {

            const distanceMeters =
                (LIGHT_SPEED * item.time) / 2;

            const distanceKm =
                distanceMeters / 1000;

            if(distanceKm <= MAX_DISTANCE){

                scans.push({
                    angle:data.scanAngle,
                    radius:distanceKm,
                    power:item.power,
                    timeStamp:now
                });

            }

        });
    }

    document.getElementById("targets").textContent =
        scans.length;
}

/* ---------- REDRAW ---------- */
function redraw(){

    const now = Date.now();

    const r = scans.map(x => x.radius);
    const theta = scans.map(x => x.angle);

    const color = scans.map(x =>
        Math.max(0,Math.min(1,x.power))
    );

    const size = scans.map(x => 6 + x.power * 4);

    const opacity = scans.map(x =>
        Math.max(0.2,
        1 - ((now - x.timeStamp) / MARK_LIFE))
    );

    const maxPower =
        Math.max(...color,0.1);

    Plotly.restyle("chart",{
        r:[r],
        theta:[theta],
        "marker.color":[color],
        "marker.size":[size],
        "marker.opacity":[opacity],
        "marker.cmin":[0],
        "marker.cmax":[maxPower]
    },[0]);
}

/* ---------- SAVE CONFIG ---------- */
async function saveConfig(){

    const payload = {
        measurementsPerRotation:Number(document.getElementById("mpr").value),
        rotationSpeed:Number(document.getElementById("rpm").value),
        targetSpeed:Number(document.getElementById("speed").value)
    };

    try{

        const response =
        await fetch("http://localhost:4000/config",{
            method:"PUT",
            headers:{
                "Content-Type":"application/json"
            },
            body:JSON.stringify(payload)
        });

        if(response.ok){
            addLog("Settings updated","#22c55e");
        }else{
            addLog("Failed to update settings","#ef4444");
        }

    }catch(error){
        addLog("API error","#ef4444");
    }
}

/* ---------- START ---------- */
buildChart();
openConnection();

/* redraw every 200ms */
setInterval(redraw,200);
</script>

</body>
</html>
```
