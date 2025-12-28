<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>📶 Signal Strength & Noise</title>
<style>
html,body{
  margin:0;overflow:hidden;
  background:#00030a;
  font-family:system-ui;
}
canvas{display:block}
.hud{
  position:absolute;left:16px;top:16px;
  padding:14px 18px;border-radius:14px;
  background:rgba(255,255,255,0.06);
  backdrop-filter:blur(10px);
  color:#d9f3ff;
  font-size:13px;
  min-width:220px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.bad{color:#ff9f9f}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

// Earth
const earth={x:w/2,y:h/2,r:70,mu:9000};

// Ground station
const gs={angle:Math.PI/4,x:0,y:0};

// Satellite
const sat={x:earth.x,y:earth.y-210,vx:2.3,vy:0};

// Constants
const maxDist=320;
const txPower=100; // arbitrary units

function gravity(){
  const dx=earth.x-sat.x;
  const dy=earth.y-sat.y;
  const d=Math.hypot(dx,dy);
  const f=earth.mu/(d*d);
  sat.vx+=f*dx/d;
  sat.vy+=f*dy/d;
}

function updateGS(){
  gs.angle+=0.0008;
  gs.x=earth.x+Math.cos(gs.angle)*earth.r;
  gs.y=earth.y+Math.sin(gs.angle)*earth.r;
}

// Line of sight
function hasLOS(){
  const dx=sat.x-gs.x;
  const dy=sat.y-gs.y;
  const d=Math.hypot(dx,dy);
  if(d>maxDist) return false;

  const t=((earth.x-gs.x)*dx+(earth.y-gs.y)*dy)/(d*d);
  const px=gs.x+t*dx;
  const py=gs.y+t*dy;
  return Math.hypot(px-earth.x,py-earth.y)>earth.r;
}

// Signal model
function signalModel(){
  const dx=sat.x-gs.x;
  const dy=sat.y-gs.y;
  const dist=Math.hypot(dx,dy);

  if(!hasLOS()) return null;

  const signal = txPower / (dist*dist);      // path loss
  const noise  = 0.02 + Math.random()*0.05;  // random noise
  const snr    = signal / noise;

  return {signal, noise, snr, dist};
}

function update(){
  gravity();
  sat.x+=sat.vx;
  sat.y+=sat.vy;
  updateGS();
}

function draw(){
  ctx.fillStyle="rgba(0,3,10,0.35)";
  ctx.fillRect(0,0,w,h);

  update();

  // Earth
  ctx.beginPath();
  ctx.arc(earth.x,earth.y,earth.r,0,Math.PI*2);
  ctx.fillStyle="#0b3d91";
  ctx.fill();

  // Ground station
  ctx.beginPath();
  ctx.arc(gs.x,gs.y,4,0,Math.PI*2);
  ctx.fillStyle="#00ffcc";
  ctx.fill();

  // Satellite
  ctx.beginPath();
  ctx.arc(sat.x,sat.y,4,0,Math.PI*2);
  ctx.fillStyle="#e6f7ff";
  ctx.fill();

  // Signal
  const data=signalModel();
  let hudHTML="🌍 Ground Station<br>🛰️ Satellite<br>";

  if(data){
    let cls="good";
    if(data.snr<8) cls="bad";
    else if(data.snr<15) cls="mid";

    // Link color
    ctx.strokeStyle =
      data.snr>15 ? "rgba(120,220,255,0.9)" :
      data.snr>8  ? "rgba(255,210,120,0.7)" :
                    "rgba(255,120,120,0.6)";
    ctx.lineWidth=1.6;
    ctx.beginPath();
    ctx.moveTo(gs.x,gs.y);
    ctx.lineTo(sat.x,sat.y);
    ctx.stroke();

    hudHTML+=`
      📶 Signal: ${data.signal.toFixed(3)}<br>
      📉 Noise: ${data.noise.toFixed(3)}<br>
      📊 SNR: <span class="${cls}">${data.snr.toFixed(1)}</span><br>
      📡 Link: <span class="${cls}">
        ${data.snr>5 ? "CONNECTED" : "UNUSABLE"}
      </span>
    `;
  }else{
    hudHTML+=`📡 Link: <span class="bad">NO SIGNAL</span>`;
  }

  document.getElementById("hud").innerHTML=hudHTML;

  requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
