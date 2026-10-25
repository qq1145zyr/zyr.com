# zyr.com
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<title>上帝视角射击 · 周洋锐 & 大肥鱼</title>
<style>
  *{margin:0;padding:0;box-sizing:border-box;}
  html,body{
    width:100%;height:100%;overflow:hidden;background:#000;
    touch-action:none;-webkit-touch-callout:none;
    -webkit-user-select:none;user-select:none;
    -webkit-tap-highlight-color:transparent;
  }
  #game{display:block;width:100%;height:100%;touch-action:none;}
</style>
</head>
<body>
<canvas id="game"></canvas>
<script>
(function(){
'use strict';

/* =========================================================
   周洋锐 & 大肥鱼 联合开发
   · 合成背景音乐 "Take Dis Rush" 风格（Hardstyle / 电音）
   · 初始攻击 3.5 / 四位数商店 / 地面武器掉落
   ========================================================= */

const WATERMARK_TEXT = '周洋锐 & 大肥鱼 联合开发';

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d', { alpha:false });
let VW = 0, VH = 0, DPR = 1;

const clamp = (v,a,b)=> v<a?a:(v>b?b:v);
const rand  = (a,b)=> a + Math.random()*(b-a);
const TAU = Math.PI * 2;

function hexAlpha(hex, a){
  const n = parseInt(hex.slice(1), 16);
  return 'rgba(' + ((n>>16)&255) + ',' + ((n>>8)&255) + ',' + (n&255) + ',' + a + ')';
}

const TILE = 64;
const COLS = 25, ROWS = 19;
const WORLD_W = COLS * TILE;
const WORLD_H = ROWS * TILE;
const STICK_R = 72;

let map = [];
let currentRooms = [];
const mapCanvas = document.createElement('canvas');

/* =========================================================
   音频系统 —— "Take Dis Rush" 风格合成电音
   灵感：Hardstyle / Reverse Bass / 持续底鼓 + 主旋律
   ========================================================= */
let audioCtx = null, masterGain = null, musicGain = null, sfxGain = null;
let musicOn = true, sfxOn = true;
let musicTimer = null, nextNoteTime = 0, stepIdx = 0;
let noiseBuf = null;
let lastShootSfx = 0;

/* Hardstyle 节奏：BPM 150，16分音符 */
const BPM = 150;
const STEP_DUR = 60 / BPM / 4;
const STEPS = 64;

/* 底鼓（每拍都有，Hardstyle 特征） */
const KICK_STEPS = [];
for(let i=0;i<64;i+=4) KICK_STEPS.push(i);

/* 拍手/军鼓 */
const SNARE_STEPS = [4,12,20,28,36,44,52,60];

/* 开镲 */
const HAT_OPEN = [6,14,22,30,38,46,54,62];
/* 闭镲 */
const HAT_CLOSED = [2,10,18,26,34,42,50,58];

/* 反向贝斯 —— 每拍前半拍重低频，Hardstyle 标志性 */
const REVERSE_BASS_FREQ = 55;  // A1

/* 主旋律（"Take Dis Rush" 感觉的上行音阶） */
const LEAD_NOTES = [
  0,0,523.25,0, 0,659.25,0,0, 0,0,783.99,0, 0,659.25,0,0,
  0,0,523.25,0, 0,587.33,0,0, 0,0,659.25,0, 0,523.25,0,0,
  0,0,523.25,0, 0,659.25,0,0, 0,0,783.99,0, 0,880.00,0,0,
  0,0,783.99,0, 0,659.25,0,0, 0,0,587.33,0, 0,523.25,0,0
];

/* 和弦铺底（Pad）—— 每4拍换一次 */
const PAD_NOTES = [
  0,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0,
  440,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0,
  523.25,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0,
  659.25,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0
];

function ensureAudio(){
  if(!audioCtx){
    try{
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      masterGain = audioCtx.createGain();
      masterGain.gain.value = 0.85;
      masterGain.connect(audioCtx.destination);

      musicGain = audioCtx.createGain();
      musicGain.gain.value = musicOn ? 0.28 : 0;
      musicGain.connect(masterGain);

      sfxGain = audioCtx.createGain();
      sfxGain.gain.value = 0.45;
      sfxGain.connect(masterGain);

      const len = Math.floor(audioCtx.sampleRate * 0.5);
      noiseBuf = audioCtx.createBuffer(1, len, audioCtx.sampleRate);
      const d = noiseBuf.getChannelData(0);
      for(let i=0;i<len;i++) d[i] = Math.random()*2-1;
    }catch(e){ return; }
  }
  if(audioCtx.state === 'suspended') audioCtx.resume();
  if(musicOn && !musicTimer) startMusic();
}

function startMusic(){
  if(!audioCtx || musicTimer) return;
  nextNoteTime = audioCtx.currentTime + 0.1;
  stepIdx = 0;
  musicTimer = setInterval(musicTick, 25);
}
function stopMusic(){
  if(musicTimer){ clearInterval(musicTimer); musicTimer = null; }
}
function musicTick(){
  if(!audioCtx || !musicOn) return;
  while(nextNoteTime < audioCtx.currentTime + 0.15){
    scheduleStep(stepIdx, nextNoteTime);
    nextNoteTime += STEP_DUR;
    stepIdx = (stepIdx + 1) % STEPS;
  }
}
function scheduleStep(i, t){
  // 底鼓（Hardstyle 每拍）
  if(KICK_STEPS.indexOf(i) >= 0) playHardKick(t);

  // 反向贝斯：每拍前1/8音符，Hardstyle 标志
  if(i % 4 === 0) playReverseBass(t);

  // 军鼓
  if(SNARE_STEPS.indexOf(i) >= 0) playHardSnare(t);

  // 开镲
  if(HAT_OPEN.indexOf(i) >= 0) playOpenHat(t);
  // 闭镲
  if(HAT_CLOSED.indexOf(i) >= 0) playClosedHat(t);

  // 主旋律
  if(LEAD_NOTES[i]) playLeadLead(LEAD_NOTES[i], t, STEP_DUR*2.0);

  // Pad 铺底
  if(PAD_NOTES[i]) playPad(PAD_NOTES[i], t, STEP_DUR*4);
}

/* ---- Hardstyle 底鼓：低频下潜 + 失真冲击 ---- */
function playHardKick(t){
  const o1 = audioCtx.createOscillator();
  const g1 = audioCtx.createGain();
  o1.type = 'sine';
  o1.frequency.setValueAtTime(160, t);
  o1.frequency.exponentialRampToValueAtTime(38, t+0.14);
  g1.gain.setValueAtTime(1.0, t);
  g1.gain.exponentialRampToValueAtTime(0.001, t+0.28);
  o1.connect(g1); g1.connect(musicGain);
  o1.start(t); o1.stop(t+0.3);

  // 冲击噪声层
  const s = audioCtx.createBufferSource();
  s.buffer = noiseBuf;
  const f = audioCtx.createBiquadFilter();
  f.type = 'lowpass';
  f.frequency.setValueAtTime(3000, t);
  f.frequency.exponentialRampToValueAtTime(200, t+0.06);
  const g2 = audioCtx.createGain();
  g2.gain.setValueAtTime(0.18, t);
  g2.gain.exponentialRampToValueAtTime(0.001, t+0.06);
  s.connect(f); f.connect(g2); g2.connect(musicGain);
  s.start(t); s.stop(t+0.08);
}

/* ---- 反向贝斯：短促重低频，Hardstyle 核心特征 ---- */
function playReverseBass(t){
  const o = audioCtx.createOscillator();
  o.type = 'sawtooth';
  o.frequency.setValueAtTime(REVERSE_BASS_FREQ, t);
  o.frequency.linearRampToValueAtTime(REVERSE_BASS_FREQ*1.5, t+STEP_DUR*3);
  o.frequency.linearRampToValueAtTime(REVERSE_BASS_FREQ, t+STEP_DUR*4);

  const f = audioCtx.createBiquadFilter();
  f.type = 'lowpass';
  f.frequency.setValueAtTime(220, t);
  f.frequency.linearRampToValueAtTime(900, t+STEP_DUR*3);
  f.frequency.linearRampToValueAtTime(220, t+STEP_DUR*4);

  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.001, t);
  g.gain.linearRampToValueAtTime(0.28, t+STEP_DUR*0.5);
  g.gain.linearRampToValueAtTime(0.001, t+STEP_DUR*3.5);

  o.connect(f); f.connect(g); g.connect(musicGain);
  o.start(t); o.stop(t+STEP_DUR*4);
}

/* ---- Hardstyle 军鼓 ---- */
function playHardSnare(t){
  const s = audioCtx.createBufferSource();
  s.buffer = noiseBuf;
  const f = audioCtx.createBiquadFilter();
  f.type = 'highpass';
  f.frequency.value = 1600;
  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.32, t);
  g.gain.exponentialRampToValueAtTime(0.001, t+0.16);
  s.connect(f); f.connect(g); g.connect(musicGain);
  s.start(t); s.stop(t+0.18);

  // 音调层
  const o = audioCtx.createOscillator();
  o.type = 'triangle';
  o.frequency.setValueAtTime(180, t);
  o.frequency.exponentialRampToValueAtTime(140, t+0.12);
  const g2 = audioCtx.createGain();
  g2.gain.setValueAtTime(0.18, t);
  g2.gain.exponentialRampToValueAtTime(0.001, t+0.12);
  o.connect(g2); g2.connect(musicGain);
  o.start(t); o.stop(t+0.14);
}

function playOpenHat(t){
  const s = audioCtx.createBufferSource();
  s.buffer = noiseBuf;
  const f = audioCtx.createBiquadFilter();
  f.type = 'highpass';
  f.frequency.value = 8000;
  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.09, t);
  g.gain.exponentialRampToValueAtTime(0.001, t+0.08);
  s.connect(f); f.connect(g); g.connect(musicGain);
  s.start(t); s.stop(t+0.1);
}
function playClosedHat(t){
  const s = audioCtx.createBufferSource();
  s.buffer = noiseBuf;
  const f = audioCtx.createBiquadFilter();
  f.type = 'highpass';
  f.frequency.value = 9000;
  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.06, t);
  g.gain.exponentialRampToValueAtTime(0.001, t+0.03);
  s.connect(f); f.connect(g); g.connect(musicGain);
  s.start(t); s.stop(t+0.05);
}

/* ---- 主旋律：Hardstyle 感 —— 方波 + 滤波 + 快速衰减 ---- */
function playLeadLead(freq, t, dur){
  const o1 = audioCtx.createOscillator();
  const o2 = audioCtx.createOscillator();
  o1.type = 'square';
  o2.type = 'sawtooth';
  o1.frequency.value = freq;
  o2.frequency.value = freq * 1.005; // 轻微失谐

  const f = audioCtx.createBiquadFilter();
  f.type = 'lowpass';
  f.frequency.setValueAtTime(3800, t);
  f.frequency.exponentialRampToValueAtTime(1800, t+dur);
  f.Q.value = 4;

  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.001, t);
  g.gain.linearRampToValueAtTime(0.08, t+0.012);
  g.gain.exponentialRampToValueAtTime(0.001, t+dur);

  o1.connect(f); o2.connect(f);
  f.connect(g); g.connect(musicGain);
  o1.start(t); o2.start(t);
  o1.stop(t+dur+0.02); o2.stop(t+dur+0.02);
}

/* ---- Pad 铺底，营造氛围 ---- */
function playPad(freq, t, dur){
  const o = audioCtx.createOscillator();
  o.type = 'sine';
  o.frequency.value = freq;
  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.001, t);
  g.gain.linearRampToValueAtTime(0.045, t+dur*0.3);
  g.gain.linearRampToValueAtTime(0.001, t+dur);
  o.connect(g); g.connect(musicGain);
  o.start(t); o.stop(t+dur+0.02);
}

/* ---- 音效 ---- */
function sfxShoot(){
  if(!audioCtx || !sfxOn) return;
  const now = audioCtx.currentTime;
  if(now - lastShootSfx < 0.045) return;
  lastShootSfx = now;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'square';
  o.frequency.setValueAtTime(880, now);
  o.frequency.exponentialRampToValueAtTime(220, now+0.06);
  g.gain.setValueAtTime(0.055, now);
  g.gain.exponentialRampToValueAtTime(0.001, now+0.07);
  o.connect(g); g.connect(sfxGain);
  o.start(now); o.stop(now+0.08);
}
function sfxHit(){
  if(!audioCtx || !sfxOn) return;
  const now = audioCtx.currentTime;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'triangle';
  o.frequency.setValueAtTime(620, now);
  o.frequency.exponentialRampToValueAtTime(1240, now+0.03);
  g.gain.setValueAtTime(0.045, now);
  g.gain.exponentialRampToValueAtTime(0.001, now+0.05);
  o.connect(g); g.connect(sfxGain);
  o.start(now); o.stop(now+0.06);
}
function sfxExplode(){
  if(!audioCtx || !sfxOn) return;
  const now = audioCtx.currentTime;
  const s = audioCtx.createBufferSource();
  s.buffer = noiseBuf;
  const f = audioCtx.createBiquadFilter();
  f.type = 'lowpass';
  f.frequency.setValueAtTime(1200, now);
  f.frequency.exponentialRampToValueAtTime(120, now+0.4);
  const g = audioCtx.createGain();
  g.gain.setValueAtTime(0.45, now);
  g.gain.exponentialRampToValueAtTime(0.001, now+0.45);
  s.connect(f); f.connect(g); g.connect(sfxGain);
  s.start(now); s.stop(now+0.5);
}
function sfxHurt(){
  if(!audioCtx || !sfxOn) return;
  const now = audioCtx.currentTime;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'sawtooth';
  o.frequency.setValueAtTime(220, now);
  o.frequency.exponentialRampToValueAtTime(80, now+0.2);
  g.gain.setValueAtTime(0.16, now);
  g.gain.exponentialRampToValueAtTime(0.001, now+0.22);
  o.connect(g); g.connect(sfxGain);
  o.start(now); o.stop(now+0.24);
}
function sfxPickup(){
  if(!audioCtx || !sfxOn) return;
  const now = audioCtx.currentTime;
  const o = audioCtx.createOscillator();
  const g = audioCtx.createGain();
  o.type = 'sine';
  o.frequency.setValueAtTime(523.25, now);
  o.frequency.setValueAtTime(783.99, now+0.06);
  g.gain.setValueAtTime(0.09, now);
  g.gain.exponentialRampToValueAtTime(0.001, now+0.15);
  o.connect(g); g.connect(sfxGain);
  o.start(now); o.stop(now+0.16);
}

function toggleMusic(){
  musicOn = !musicOn;
  if(musicOn){
    if(audioCtx && !musicTimer) startMusic();
    if(musicGain) musicGain.gain.value = 0.28;
  }else{
    if(musicGain) musicGain.gain.value = 0;
    stopMusic();
  }
}

/* =========================================================
   随机地图
   ========================================================= */
function carveH(x0, x1, y){
  const minX = Math.min(x0,x1), maxX = Math.max(x0,x1);
  for(let x=minX; x<=maxX; x++) for(let dy=0; dy<2; dy++)
    map[clamp(y+dy,1,ROWS-2)][clamp(x,1,COLS-2)] = 0;
}
function carveV(y0, y1, x){
  const minY = Math.min(y0,y1), maxY = Math.max(y0,y1);
  for(let y=minY; y<=maxY; y++) for(let dx=0; dx<2; dx++)
    map[clamp(y,1,ROWS-2)][clamp(x+dx,1,COLS-2)] = 0;
}

function generateRandomMap(){
  map = [];
  for(let y=0;y<ROWS;y++) map.push(new Array(COLS).fill(1));

  const rooms = [];
  const targetRooms = 8 + Math.floor(Math.random()*4);
  let attempts = 0;
  while(rooms.length < targetRooms && attempts < 160){
    attempts++;
    const w = 3+Math.floor(Math.random()*4);
    const h = 3+Math.floor(Math.random()*3);
    const x = 1+Math.floor(Math.random()*Math.max(1, COLS-w-2));
    const y = 1+Math.floor(Math.random()*Math.max(1, ROWS-h-2));
    let overlap = false;
    for(const r of rooms){
      if(x-2 < r.x+r.w && x+w+2 > r.x && y-2 < r.y+r.h && y+h+2 > r.y){ overlap = true; break; }
    }
    if(overlap) continue;
    rooms.push({ x, y, w, h });
  }

  for(const r of rooms)
    for(let yy=r.y; yy<r.y+r.h; yy++)
      for(let xx=r.x; xx<r.x+r.w; xx++)
        map[yy][xx] = 0;

  for(let i=1; i<rooms.length; i++){
    const a = rooms[i-1], b = rooms[i];
    const ax = Math.floor(a.x+a.w/2), ay = Math.floor(a.y+a.h/2);
    const bx = Math.floor(b.x+b.w/2), by = Math.floor(b.y+b.h/2);
    if(Math.random() < 0.5){ carveH(ax,bx,ay); carveV(ay,by,bx); }
    else{ carveV(ay,by,ax); carveH(ax,bx,by); }
  }
  for(let i=0;i<2;i++){
    if(rooms.length < 2) break;
    const a = rooms[Math.floor(Math.random()*rooms.length)];
    const b = rooms[Math.floor(Math.random()*rooms.length)];
    if(a===b) continue;
    const ax = Math.floor(a.x+a.w/2), ay = Math.floor(a.y+a.h/2);
    const bx = Math.floor(b.x+b.w/2), by = Math.floor(b.y+b.h/2);
    if(Math.random() < 0.5){ carveH(ax,bx,ay); carveV(ay,by,bx); }
    else{ carveV(ay,by,ax); carveH(ax,bx,by); }
  }
  currentRooms = rooms;
}

function isWall(cx, cy){
  if(cx<0||cy<0||cx>=COLS||cy>=ROWS) return true;
  return map[cy][cx] === 1;
}
function circleHitsWall(x, y, r){
  const c0 = Math.floor((x-r)/TILE), c1 = Math.floor((x+r)/TILE);
  const r0 = Math.floor((y-r)/TILE), r1 = Math.floor((y+r)/TILE);
  for(let cy=r0; cy<=r1; cy++) for(let cx=c0; cx<=c1; cx++){
    if(!isWall(cx,cy)) continue;
    const rx = cx*TILE, ry = cy*TILE;
    const nx = clamp(x,rx,rx+TILE), ny = clamp(y,ry,ry+TILE);
    const dx = x-nx, dy = y-ny;
    if(dx*dx+dy*dy < r*r) return true;
  }
  return false;
}
function moveEntity(ent, dx, dy){
  if(!circleHitsWall(ent.x+dx, ent.y, ent.r)) ent.x += dx;
  if(!circleHitsWall(ent.x, ent.y+dy, ent.r)) ent.y += dy;
}
function hasLineOfSight(a, b){
  const d = Math.hypot(b.x-a.x, b.y-a.y);
  const steps = Math.max(2, Math.ceil(d/28));
  for(let i=1;i<steps;i++){
    const t = i/steps;
    if(isWall(Math.floor((a.x+(b.x-a.x)*t)/TILE), Math.floor((a.y+(b.y-a.y)*t)/TILE))) return false;
  }
  return true;
}

function prerenderMap(){
  mapCanvas.width = WORLD_W; mapCanvas.height = WORLD_H;
  const g = mapCanvas.getContext('2d');
  g.fillStyle = '#0e0e15'; g.fillRect(0,0,WORLD_W,WORLD_H);
  for(let y=0;y<ROWS;y++) for(let x=0;x<COLS;x++){
    if(map[y][x]===1) continue;
    g.fillStyle = ((x+y)%2===0) ? '#16161f' : '#1b1b26';
    g.fillRect(x*TILE,y*TILE,TILE,TILE);
    g.strokeStyle = 'rgba(90,140,200,0.07)';
    g.lineWidth = 1;
    g.strokeRect(x*TILE+0.5,y*TILE+0.5,TILE-1,TILE-1);
  }
  for(let y=0;y<ROWS;y++) for(let x=0;x<COLS;x++){
    if(map[y][x]!==1) continue;
    const px = x*TILE, py = y*TILE;
    g.fillStyle = '#07070c'; g.fillRect(px,py,TILE,TILE);
    g.fillStyle = '#2b3a4e'; g.fillRect(px+2,py+2,TILE-4,TILE-4);
    g.fillStyle = '#3c5070'; g.fillRect(px+7,py+7,TILE-14,TILE-14);
    g.fillStyle = '#2b3a4e'; g.fillRect(px+14,py+14,TILE-28,TILE-28);
  }
}

/* 流场 */
let flowField = [];
const DIRS = [[1,0],[-1,0],[0,1],[0,-1]];
function buildFlowField(){
  const px = Math.floor(player.x/TILE), py = Math.floor(player.y/TILE);
  flowField = [];
  for(let y=0;y<ROWS;y++) flowField.push(new Array(COLS).fill(Infinity));
  if(isWall(px,py)) return;
  flowField[py][px] = 0;
  const q = [[px,py]]; let head = 0;
  while(head < q.length){
    const [cx,cy] = q[head++];
    const d = flowField[cy][cx];
    for(const [dx,dy] of DIRS){
      const nx = cx+dx, ny = cy+dy;
      if(nx<0||ny<0||nx>=COLS||ny>=ROWS) continue;
      if(map[ny][nx]===1) continue;
      if(flowField[ny][nx]!==Infinity) continue;
      flowField[ny][nx] = d+1;
      q.push([nx,ny]);
    }
  }
}

/* 水印 */
const wmCanvas = document.createElement('canvas');
function buildWatermark(){
  const w = Math.max(1, Math.round(VW));
  const h = Math.max(1, Math.round(VH));
  wmCanvas.width = w; wmCanvas.height = h;
  const g = wmCanvas.getContext('2d');
  g.clearRect(0,0,w,h);
  g.save();
  g.globalAlpha = 0.075;
  g.fillStyle = '#9fd0ff';
  g.font = 'bold 24px "Microsoft YaHei","PingFang SC",sans-serif';
  g.textAlign = 'center'; g.textBaseline = 'middle';
  g.translate(w/2,h/2); g.rotate(-Math.PI/9);
  const R = Math.hypot(w,h);
  const stepX = 400, stepY = 130;
  let row = 0;
  for(let y=-R; y<=R; y+=stepY, row++){
    const off = (row%2===0) ? 0 : stepX/2;
    for(let x=-R; x<=R; x+=stepX) g.fillText(WATERMARK_TEXT, x+off, y);
  }
  g.restore();
}

/* =========================================================
   武器定义
   ========================================================= */
const WEAPONS = {
  pistol: { name:'手枪', fireRate:0.115, dmg:3.5, bulletSpeed:900, bulletCount:1, spread:0, color:'#c8f2ff' },
  dualPistol: { name:'双持手枪', fireRate:0.10, dmg:3.5, bulletSpeed:900, bulletCount:2, spread:0.07, color:'#c8f2ff' },
  shotgun: { name:'霰弹枪', fireRate:0.17, dmg:2.1, bulletSpeed:810, bulletCount:5, spread:0.14, duration:15, color:'#ff8a3d' },
  grenadeLauncher: { name:'榴弹发射器', fireRate:0.55, dmg:35, bulletSpeed:620, bulletCount:1, spread:0, ammo:4, explosive:true, color:'#ffcc66' },
  awp: { name:'AWP', fireRate:0.9, dmg:350, bulletSpeed:1500, bulletCount:1, spread:0, duration:7.8, pierce:99, color:'#00ffcc' },
  minigun: { name:'加特林', fireRate:0.035, dmg:87.5, bulletSpeed:1100, bulletCount:3, spread:0.09, duration:5.5, color:'#ffd93d' }
};

/* 技能 */
const SKILLS = {
  lightning: {
    name:'放电', symbol:'⚡', color:'#7fd4ff', cooldown:6,
    use(){
      const dmg = 15 * player.dmg;
      effects.push({ type:'ring', x:player.x, y:player.y, maxR:190, life:0.45, maxLife:0.45, color:'#7fd4ff' });
      for(const e of enemies){
        if(e.dead) continue;
        if(Math.hypot(e.x-player.x, e.y-player.y) < 190){
          let d = dmg;
          if(e.armor > 0){ const ab = Math.min(e.armor, d); e.armor -= ab; d -= ab; }
          e.hp -= d; e.hitFlash = 0.15;
          addFloatText(e.x, e.y-e.r-4, Math.round(d), '#7fd4ff', 17);
          if(e.hp <= 0) killEnemy(e);
        }
      }
      sfxHit();
      shake = Math.max(shake, 6);
    }
  },
  grenade: {
    name:'手榴弹', symbol:'💣', color:'#ff8a3d', cooldown:8,
    use(){
      const a = player.angle;
      bullets.push({
        x: player.x+Math.cos(a)*20, y: player.y+Math.sin(a)*20,
        vx: Math.cos(a)*520, vy: Math.sin(a)*520,
        r:8, team:'player', dmg:140*player.dmg, crit:false,
        pierce:0, hitSet:null, life:1.4, color:'#ff8a3d',
        explosive:true, fuse:0.55, isGrenade:true
      });
      shake = Math.max(shake, 3);
    }
  },
  summon: {
    name:'召唤仆从', symbol:'☠', color:'#9b7bff', cooldown:20,
    use(){
      const need = 3 - minions.length;
      for(let i=0;i<need;i++){
        const a = rand(0, TAU);
        minions.push({
          x: player.x+Math.cos(a)*40, y: player.y+Math.sin(a)*40,
          r:12, hp:60, maxHp:60, angle:0, cool:0, dead:false
        });
      }
      spawnParticles(player.x, player.y, '#9b7bff', 22, 260);
      sfxPickup();
    }
  },
  invincible: {
    name:'无敌金身', symbol:'✚', color:'#ffd93d', cooldown:18,
    use(){
      player.invulnTimer = 4.8;
      spawnParticles(player.x, player.y, '#ffd93d', 26, 300);
      sfxPickup();
      shake = Math.max(shake, 5);
    }
  }
};

/* =========================================================
   商店定义 —— 四位数价格
   ========================================================= */
const SHOP_ITEMS = {
  medkit: { name:'医药包', icon:'＋', color:'#5be06b', price:1200, desc:'回复 50 生命',
    apply(){ player.hp = Math.min(player.maxHp, player.hp+50); } },
  armor: { name:'防弹衣', icon:'⬢', color:'#4fa3ff', price:1600, desc:'获得 50 护盾',
    apply(){ player.shield = Math.min(100, player.shield+50); } },
  dualPistol: { name:'双持手枪', icon:'▮▮', color:'#c8f2ff', price:5200, desc:'永久改为双持手枪（弹道+1）',
    apply(){ player.baseWeapon = 'dualPistol'; } },
  grenadeLauncher: { name:'榴弹发射器', icon:'◉', color:'#ffcc66', price:3600, desc:'4 发群伤榴弹',
    apply(){ player.tempWeapon = { type:'grenadeLauncher', ammo:4 }; } },
  awp: { name:'AWP', icon:'➤', color:'#00ffcc', price:4000, desc:'7.8 秒 · 伤害350 · 穿透',
    apply(){ player.tempWeapon = { type:'awp', timer:7.8 }; } },
  minigun: { name:'加特林', icon:'≡', color:'#ffd93d', price:4400, desc:'5.5 秒 · 极速 · 3弹道',
    apply(){ player.tempWeapon = { type:'minigun', timer:5.5 }; } },
  lightning: { name:'放电', icon:'⚡', color:'#7fd4ff', price:2800, desc:'主动技能 · 环形电击', skill:'lightning' },
  grenadeSkill: { name:'手榴弹', icon:'💣', color:'#ff8a3d', price:3200, desc:'主动技能 · 投掷爆炸', skill:'grenade' },
  summonSkill: { name:'召唤仆从', icon:'☠', color:'#9b7bff', price:4000, desc:'主动技能 · 召唤3只', skill:'summon' },
  invincibleSkill: { name:'无敌金身', icon:'✚', color:'#ffd93d', price:3600, desc:'主动技能 · 4.8秒无敌', skill:'invincible' }
};

/* =========================================================
   地面装备（含武器掉落）
   ========================================================= */
const ITEM_TYPES = {
  medkit:  { name:'医疗包', symbol:'＋', color:'#5be06b' },
  armor:   { name:'防弹衣', symbol:'⬢', color:'#4fa3ff' },
  speed:   { name:'加速鞋', symbol:'⚡', color:'#ffd93d' },
  shotgun: { name:'霰弹枪', symbol:'▰', color:'#ff8a3d' },
  awp:     { name:'AWP',    symbol:'➤', color:'#00ffcc' },
  minigun: { name:'加特林', symbol:'≡', color:'#ffd93d' },
  grenadeLauncher: { name:'榴弹发射器', symbol:'◉', color:'#ffcc66' }
};
const ENEMY_PICKABLE = ['armor','medkit','speed'];

let items = [];

function pickItemType(){
  const r = Math.random();
  if(r < 0.26) return 'medkit';
  if(r < 0.48) return 'armor';
  if(r < 0.60) return 'speed';
  if(r < 0.73) return 'shotgun';
  if(r < 0.84) return 'awp';
  if(r < 0.93) return 'minigun';
  return 'grenadeLauncher';
}

function spawnItem(){
  for(let i=0;i<120;i++){
    const cx = 1+Math.floor(Math.random()*(COLS-2));
    const cy = 1+Math.floor(Math.random()*(ROWS-2));
    if(map[cy][cx]===1) continue;
    const x = (cx+0.5)*TILE, y = (cy+0.5)*TILE;
    if(Math.hypot(x-player.x, y-player.y) < 140) continue;
    let overlap = false;
    for(const it of items){ if(Math.hypot(it.x-x, it.y-y) < 60){ overlap = true; break; } }
    if(overlap) continue;
    const type = pickItemType();
    items.push({ type, x, y, r:13, bob:Math.random()*TAU });
    return true;
  }
  return false;
}

/* =========================================================
   玩家 & 状态
   ========================================================= */
const player = {
  x:0, y:0, r:16, angle:0,
  hp:100, maxHp:100, shield:0,
  speed:245,
  cooldown:0, fireRate:0.115, bulletSpeed:900,
  invuln:0, hurtTimer:0, invulnTimer:0,
  dmg:3.5, pierce:0, critChance:0, lifesteal:0, multiShot:0, armor:0,
  baseWeapon:'pistol',
  tempWeapon:null,
  skills:[],
  speedTimer:0
};

let bullets = [];
let enemies = [];
let minions = [];
let particles = [];
let effects = [];
let floatTexts = [];
let shake = 0;

const game = {
  wave:0, score:0, kills:0,
  over:false, time:0,
  waveDelay:1.0, restartLock:0,
  buffMode:false, buffChoices:[], lastBuffWave:0,
  shopMode:false, shopItems:[], lastShopWave:0,
  bossCount:0,
  combo:0, comboTimer:0, maxCombo:0,
  paused:false, flash:0,
  toast:null
};

const cam = { x:0, y:0 };

/* 强化池 */
const BUFF_POOL = [
  { symbol:'力', name:'火力强化', desc:'子弹伤害 +1', color:'#ff6b4a', apply(){ player.dmg += 1; } },
  { symbol:'速', name:'急速射击', desc:'射击间隔缩短 18%', color:'#ffd93d', apply(){ player.fireRate = Math.max(0.035, player.fireRate*0.82); } },
  { symbol:'心', name:'生命强化', desc:'生命上限 +25，并回满', color:'#5be06b', apply(){ player.maxHp += 25; player.hp = player.maxHp; } },
  { symbol:'靴', name:'疾风之靴', desc:'移动速度 +12%', color:'#4fa3ff', apply(){ player.speed *= 1.12; } },
  { symbol:'穿', name:'穿甲弹头', desc:'子弹多穿透 1 个敌人', color:'#9b7bff', apply(){ player.pierce += 1; } },
  { symbol:'暴', name:'致命一击', desc:'暴击率 +12%', color:'#ff4d8f', apply(){ player.critChance = Math.min(0.9, player.critChance+0.12); } },
  { symbol:'血', name:'嗜血本能', desc:'击杀回复 6 生命', color:'#e0453f', apply(){ player.lifesteal += 6; } },
  { symbol:'多', name:'多重射击', desc:'每次额外发射 1 颗子弹', color:'#00d4c8', apply(){ player.multiShot += 1; } },
  { symbol:'弹', name:'高速弹头', desc:'子弹速度 +22%', color:'#7fd4ff', apply(){ player.bulletSpeed *= 1.22; } },
  { symbol:'盾', name:'战术护甲', desc:'受到伤害减少 3 点', color:'#a0b8d0', apply(){ player.armor += 3; } }
];

function pickBuffs(n){
  const pool = BUFF_POOL.slice();
  const out = [];
  while(out.length < n && pool.length > 0){
    const i = Math.floor(Math.random()*pool.length);
    out.push(Object.assign({}, pool.splice(i,1)[0]));
  }
  return out;
}

function addFloatText(x, y, text, color, size){
  if(floatTexts.length > 50) return;
  floatTexts.push({
    x: x + rand(-8,8), y, text, color, size: size||15,
    life: 0.75, maxLife: 0.75, vy: -70
  });
}
function updateFloatTexts(dt){
  for(let i=floatTexts.length-1;i>=0;i--){
    const f = floatTexts[i];
    f.life -= dt;
    if(f.life <= 0){ floatTexts.splice(i,1); continue; }
    f.y += f.vy*dt;
    f.vy *= 0.93;
  }
}

/* =========================================================
   输入
   ========================================================= */
const sticks = {
  move: { active:false, id:null, bx:0, by:0, x:0, y:0, dx:0, dy:0 },
  aim:  { active:false, id:null, bx:0, by:0, x:0, y:0, dx:0, dy:0 }
};
function startStick(s,id,x,y){ s.active=true; s.id=id; s.bx=x; s.by=y; s.x=x; s.y=y; s.dx=0; s.dy=0; }
function updateStick(s,x,y){
  s.x=x; s.y=y;
  let dx=x-s.bx, dy=y-s.by;
  const d=Math.hypot(dx,dy);
  if(d>STICK_R){ dx=dx/d*STICK_R; dy=dy/d*STICK_R; }
  s.dx=dx/STICK_R; s.dy=dy/STICK_R;
}
function endStick(s){ s.active=false; s.id=null; s.dx=0; s.dy=0; }
function findStick(id){
  if(sticks.move.active && sticks.move.id===id) return sticks.move;
  if(sticks.aim.active  && sticks.aim.id ===id) return sticks.aim;
  return null;
}

const keys = {};
const mouse = { sx:0, sy:0, wx:0, wy:0, down:false, active:false };

const pauseBtn = { x:20, y:20, w:46, h:46 };
const musicBtn = { x:20, y:74, w:46, h:46 };

function touchPos(t){
  const r = canvas.getBoundingClientRect();
  return { x: t.clientX-r.left, y: t.clientY-r.top };
}
function hitRect(px,py,r){ return r && px>=r.x && px<=r.x+r.w && py>=r.y && py<=r.y+r.h; }

let skillBtnRects = [];
function layoutSkillButtons(){
  const size = Math.min(58, VW*0.075);
  const gap = 10;
  const n = player.skills.length;
  if(n === 0){ skillBtnRects = []; return; }
  const totalH = n*size+(n-1)*gap;
  const x = VW-size-16;
  const y = VH-totalH-120;
  skillBtnRects = player.skills.map((s,i)=>({
    x, y: y+i*(size+gap), w:size, h:size, index:i
  }));
}

function handleBuffClick(px,py){
  for(let i=0;i<game.buffChoices.length;i++){
    if(hitRect(px,py,game.buffChoices[i]._rect)){ chooseBuff(i); return; }
  }
}
function chooseBuff(i){
  const b = game.buffChoices[i];
  if(!b) return;
  b.apply();
  game.lastBuffWave = game.wave;
  game.buffMode = false;
  game.buffChoices = [];
  game.waveDelay = 1.1;
  spawnParticles(player.x, player.y, b.color, 26, 340);
  sfxPickup();
  shake = Math.max(shake, 6);
  game.toast = { text:'获得强化：'+b.name+' — '+b.desc, color:b.color, t:2.2 };
}

function handleShopClick(px,py){
  const btn = game._shopLeaveBtn;
  if(hitRect(px,py,btn)){
    game.shopMode = false;
    game.waveDelay = 1.1;
    return;
  }
  for(let i=0;i<game.shopItems.length;i++){
    if(hitRect(px,py,game.shopItems[i]._rect)){ buyShopItem(i); return; }
  }
}

function buyShopItem(idx){
  const item = game.shopItems[idx];
  if(!item || item.sold) return;
  const def = SHOP_ITEMS[item.id];
  if(game.score < item.price) return;
  game.score -= item.price;
  item.sold = true;
  if(def.skill){
    if(!player.skills.some(s=>s.id===def.skill)){
      player.skills.push({ id:def.skill, cd:0 });
    }
  }else if(def.apply){
    def.apply();
  }
  sfxPickup();
  game.toast = { text:'购买：'+def.name, color:def.color, t:1.6 };
}

function generateShopItems(){
  const mult = 1.3;
  const list = [];
  list.push({ id:'medkit', price: Math.round(1200*mult) });
  list.push({ id:'armor',  price: Math.round(1600*mult) });
  if(player.baseWeapon === 'pistol'){
    list.push({ id:'dualPistol', price: Math.round(5200*mult) });
  }
  const pool = ['grenadeLauncher','awp','minigun','lightning','grenadeSkill','summonSkill','invincibleSkill'];
  const filtered = pool.filter(id=>{
    const it = SHOP_ITEMS[id];
    if(it.skill && player.skills.some(s=>s.id===it.skill)) return false;
    return true;
  });
  for(let i=filtered.length-1;i>0;i--){
    const j = Math.floor(Math.random()*(i+1));
    [filtered[i],filtered[j]] = [filtered[j],filtered[i]];
  }
  for(let i=0;i<Math.min(2, filtered.length);i++){
    const id = filtered[i];
    list.push({ id, price: Math.round(SHOP_ITEMS[id].price*mult) });
  }
  return list;
}

function openShop(){
  game.shopMode = true;
  game.shopItems = generateShopItems();
  endStick(sticks.move); endStick(sticks.aim);
  mouse.down = false;
}

function useSkill(idx){
  const s = player.skills[idx];
  if(!s || s.cd > 0) return;
  const def = SKILLS[s.id];
  def.use();
  s.cd = def.cooldown;
}

function togglePause(){
  if(game.over) return;
  game.paused = !game.paused;
  if(game.paused){
    endStick(sticks.move); endStick(sticks.aim);
    mouse.down = false;
    shake = 0;
  }
}

function handleUIButtons(px, py){
  if(hitRect(px,py,musicBtn)){ toggleMusic(); return true; }
  if(!game.over && hitRect(px,py,pauseBtn)){ togglePause(); return true; }
  return false;
}

/* ---- 触摸 ---- */
canvas.addEventListener('touchstart', (e)=>{
  e.preventDefault();
  ensureAudio();

  for(const t of e.changedTouches){
    const p = touchPos(t);
    if(handleUIButtons(p.x, p.y)) continue;

    if(game.over && game.restartLock <= 0){ resetGame(); return; }
    if(game.paused) continue;

    if(game.shopMode){ handleShopClick(p.x, p.y); continue; }
    if(game.buffMode){ handleBuffClick(p.x, p.y); continue; }

    layoutSkillButtons();
    let hitSkill = false;
    for(const b of skillBtnRects){
      if(hitRect(p.x,p.y,b)){ useSkill(b.index); hitSkill = true; break; }
    }
    if(hitSkill) continue;

    if(p.x < VW/2){
      if(!sticks.move.active) startStick(sticks.move, t.identifier, p.x, p.y);
    }else{
      if(!sticks.aim.active) startStick(sticks.aim, t.identifier, p.x, p.y);
    }
  }
}, {passive:false});

canvas.addEventListener('touchmove', (e)=>{
  e.preventDefault();
  if(game.buffMode || game.shopMode || game.paused) return;
  for(const t of e.changedTouches){
    const s = findStick(t.identifier);
    if(!s) continue;
    const p = touchPos(t);
    updateStick(s, p.x, p.y);
  }
}, {passive:false});

function handleTouchEnd(e){
  e.preventDefault();
  for(const t of e.changedTouches){
    const s = findStick(t.identifier);
    if(s) endStick(s);
  }
}
canvas.addEventListener('touchend', handleTouchEnd, {passive:false});
canvas.addEventListener('touchcancel', handleTouchEnd, {passive:false});

window.addEventListener('keydown', (e)=>{
  ensureAudio();
  keys[e.code] = true;
  if(e.code==='Space' && game.over && game.restartLock<=0) resetGame();
  if(e.code==='KeyP') togglePause();
  if(['Space','ArrowUp','ArrowDown','ArrowLeft','ArrowRight'].includes(e.code)) e.preventDefault();
});
window.addEventListener('keyup', (e)=>{ keys[e.code] = false; });

canvas.addEventListener('mousemove', (e)=>{
  const r = canvas.getBoundingClientRect();
  mouse.sx = e.clientX-r.left; mouse.sy = e.clientY-r.top;
  mouse.active = true;
  if(sticks.aim.active) mouse.active = false;
});
canvas.addEventListener('mousedown', (e)=>{
  ensureAudio();
  const r = canvas.getBoundingClientRect();
  const px = e.clientX-r.left, py = e.clientY-r.top;
  if(handleUIButtons(px, py)) return;

  if(game.over && game.restartLock<=0){ resetGame(); return; }
  if(game.paused) return;

  if(game.shopMode){ handleShopClick(px, py); return; }
  if(game.buffMode){ handleBuffClick(px, py); return; }

  layoutSkillButtons();
  for(const b of skillBtnRects){ if(hitRect(px,py,b)){ useSkill(b.index); return; } }
  mouse.down = true;
});
window.addEventListener('mouseup', ()=>{ mouse.down = false; });
canvas.addEventListener('contextmenu', (e)=> e.preventDefault());

/* =========================================================
   游戏流程
   ========================================================= */
function isBossWave(w){ return w >= 12 && w % 6 === 0; }

function resetGame(){
  generateRandomMap();
  prerenderMap();

  if(currentRooms.length > 0){
    const r = currentRooms[0];
    player.x = (r.x+r.w/2)*TILE;
    player.y = (r.y+r.h/2)*TILE;
  }else{
    player.x = 3.5*TILE; player.y = 3.5*TILE;
  }

  player.angle = 0;
  player.maxHp = 100; player.hp = 100; player.shield = 0;
  player.speed = 245; player.fireRate = 0.115; player.bulletSpeed = 900;
  player.dmg = 3.5; player.pierce = 0; player.critChance = 0;
  player.lifesteal = 0; player.multiShot = 0; player.armor = 0;
  player.baseWeapon = 'pistol';
  player.tempWeapon = null;
  player.skills = [];
  player.speedTimer = 0;
  player.invuln = 0; player.hurtTimer = 0; player.invulnTimer = 0;
  player.cooldown = 0;

  bullets = []; enemies = []; minions = []; particles = []; effects = []; items = [];
  floatTexts = [];
  shake = 0;

  game.wave = 0; game.score = 0; game.kills = 0;
  game.over = false; game.time = 0;
  game.waveDelay = 1.0; game.restartLock = 0.6;
  game.buffMode = false; game.buffChoices = []; game.lastBuffWave = 0;
  game.shopMode = false; game.shopItems = []; game.lastShopWave = 0;
  game.bossCount = 0;
  game.combo = 0; game.comboTimer = 0; game.maxCombo = 0;
  game.paused = false; game.flash = 0;
  game.toast = null;

  endStick(sticks.move); endStick(sticks.aim);

  for(let i=0;i<5;i++) spawnItem();
  buildFlowField();

  cam.x = clamp(player.x-VW/2, 0, Math.max(0, WORLD_W-VW));
  cam.y = clamp(player.y-VH/2, 0, Math.max(0, WORLD_H-VH));
}

function randomFreeSpot(minDist){
  for(let i=0;i<300;i++){
    const cx = Math.floor(Math.random()*COLS);
    const cy = Math.floor(Math.random()*ROWS);
    if(map[cy][cx]===1) continue;
    const x = (cx+0.5)*TILE, y = (cy+0.5)*TILE;
    if(Math.hypot(x-player.x, y-player.y) < minDist) continue;
    return {x,y};
  }
  return null;
}

function getEnemyArmorChance(){ return Math.min(0.6, Math.floor(game.wave/6)*0.1); }

function spawnEnemy(){
  const spot = randomFreeSpot(420);
  if(!spot) return;
  const fast = Math.random() < 0.35;
  const waveBonus = Math.floor(game.wave/4);
  const hp = (fast?2:3) + waveBonus;
  const hasArmor = Math.random() < getEnemyArmorChance();
  enemies.push({
    x:spot.x, y:spot.y, r:15,
    hp, maxHp:hp,
    armor: hasArmor ? 3 + Math.floor(game.wave/10) : 0,
    speed: fast?155:95,
    range: fast?280:360,
    fireRate: fast?1.05:1.55,
    cool: rand(0.4,1.6),
    bulletSpeed: fast?400:340,
    angle:0, hitFlash:0, dead:false,
    seed: rand(0,100),
    color: fast?'#ff8a3d':'#e0453f',
    type: fast?'fast':'normal',
    speedTimer:0
  });
}

function spawnElite(){
  const mult = Math.pow(1.012, Math.max(0, game.bossCount-1));
  const spot = randomFreeSpot(300);
  if(!spot) return;
  enemies.push({
    x:spot.x, y:spot.y, r:18,
    hp: Math.round(165*mult), maxHp: Math.round(165*mult),
    armor: Math.round(65*mult),
    speed: 120, range: 320, fireRate: 1.4,
    cool: rand(0.5,1.5), bulletSpeed: 360,
    angle:0, hitFlash:0, dead:false,
    seed: rand(0,100),
    color:'#a03080',
    type:'elite',
    speedTimer:0
  });
}

function spawnBoss(){
  game.bossCount++;
  const mult = Math.pow(1.012, game.bossCount-1);
  const spot = randomFreeSpot(500) || { x: player.x+400, y: player.y };
  enemies.push({
    x:spot.x, y:spot.y, r:34,
    hp: Math.round(800*mult), maxHp: Math.round(800*mult),
    armor: Math.round(120*mult),
    speed: 70, range: 420, fireRate: 1.2,
    cool: 1.5, bulletSpeed: 380,
    angle:0, hitFlash:0, dead:false,
    seed: rand(0,100),
    color:'#d03030',
    type:'boss', isBoss:true,
    summonCool: 5,
    speedTimer:0
  });
  for(let i=0;i<4;i++) spawnElite();
}

function startWave(){
  if(game.wave > 0) player.hp = Math.min(player.maxHp, player.hp+15);
  game.wave++;
  if(isBossWave(game.wave)){
    spawnBoss();
    game.toast = { text:'⚠ BOSS 来袭！', color:'#ff4444', t:2.5 };
    shake = Math.max(shake, 8);
  }else{
    const count = Math.min(3+game.wave, 12);
    for(let i=0;i<count;i++) spawnEnemy();
  }
  game.waveDelay = 1.8;
  while(items.length < 5 && spawnItem()){}
}

function spawnParticles(x, y, color, n, speed){
  for(let i=0;i<n;i++){
    const a = Math.random()*TAU;
    const s = rand(40, speed||240);
    particles.push({ x, y, vx:Math.cos(a)*s, vy:Math.sin(a)*s,
      life:rand(0.18,0.45), maxLife:0.45, color, size:rand(2,5) });
  }
}

function damagePlayer(dmg){
  if(player.invulnTimer > 0) return;
  if(player.invuln > 0 || game.over) return;
  let real = Math.max(1, dmg - player.armor);
  if(player.shield > 0){
    const ab = Math.min(player.shield, real);
    player.shield -= ab; real -= ab;
  }
  player.hp -= real;
  player.invuln = 0.28;
  player.hurtTimer = 0.35;
  shake = Math.max(shake, 10);
  game.combo = 0; game.comboTimer = 0;
  addFloatText(player.x, player.y-player.r-4, '-'+Math.round(real), '#ff5555', 18);
  spawnParticles(player.x, player.y, '#ff6644', 10, 200);
  sfxHurt();
  if(player.hp <= 0){
    player.hp = 0;
    game.over = true;
    game.restartLock = 0.7;
    endStick(sticks.move); endStick(sticks.aim);
  }
}

function killEnemy(e){
  e.dead = true;
  const base = (e.type==='boss') ? 12000 : (e.type==='elite') ? 3000 : (e.type==='fast') ? 1500 : 1000;

  game.combo++;
  game.comboTimer = 2.5;
  if(game.combo > game.maxCombo) game.maxCombo = game.combo;
  const mult = 1 + Math.min(2, game.combo * 0.1);
  game.score += Math.round(base * mult);

  game.kills++;
  if(player.lifesteal > 0) player.hp = Math.min(player.maxHp, player.hp+player.lifesteal);
  spawnParticles(e.x, e.y, e.color, e.type==='boss'?50:20, e.type==='boss'?450:320);
  shake = Math.max(shake, e.type==='boss'?14:5);
  if(e.type==='boss'){
    game.flash = 0.7;
    sfxExplode();
    game.toast = { text:'BOSS 被击败！+' + Math.round(base*mult) + ' 积分', color:'#ffd93d', t:2.5 };
    for(let i=0;i<3;i++) spawnItem();
  }
}

function explode(x, y, radius, dmg){
  effects.push({ type:'ring', x, y, maxR:radius, life:0.4, maxLife:0.4, color:'#ffaa44' });
  spawnParticles(x, y, '#ffaa44', 24, 380);
  sfxExplode();
  shake = Math.max(shake, 8);
  for(const e of enemies){
    if(e.dead) continue;
    const d = Math.hypot(e.x-x, e.y-y);
    if(d < radius + e.r){
      const falloff = 1 - d/(radius+e.r);
      let dd = dmg * Math.max(0.3, falloff);
      if(e.armor > 0){
        const ab = Math.min(e.armor, dd);
        e.armor -= ab; dd -= ab;
      }
      e.hp -= dd;
      e.hitFlash = 0.15;
      addFloatText(e.x, e.y-e.r-4, Math.round(dd), '#ffaa44', 18);
      if(e.hp <= 0) killEnemy(e);
    }
  }
}

function applyPlayerItem(it){
  const info = ITEM_TYPES[it.type];
  switch(it.type){
    case 'shotgun': player.tempWeapon = { type:'shotgun', timer:15 }; break;
    case 'awp': player.tempWeapon = { type:'awp', timer:7.8 }; break;
    case 'minigun': player.tempWeapon = { type:'minigun', timer:5.5 }; break;
    case 'grenadeLauncher': player.tempWeapon = { type:'grenadeLauncher', ammo:4 }; break;
    case 'armor': player.shield = Math.min(100, player.shield+50); break;
    case 'medkit': player.hp = Math.min(player.maxHp, player.hp+40); break;
    case 'speed': player.speedTimer = 15; break;
  }
  spawnParticles(it.x, it.y, info.color, 12, 200);
  sfxPickup();
  game.toast = { text:'拾取：'+info.name, color:info.color, t:1.6 };
}
function applyEnemyItem(e, it){
  if(it.type === 'armor') e.armor += 3;
  else if(it.type === 'medkit') e.hp = Math.min(e.maxHp, e.hp+2);
  else if(it.type === 'speed') e.speedTimer = 15;
  spawnParticles(it.x, it.y, ITEM_TYPES[it.type].color, 8, 150);
}
function checkPickups(){
  for(let i=items.length-1;i>=0;i--){
    const it = items[i];
    if(Math.hypot(player.x-it.x, player.y-it.y) < player.r+it.r+6){
      applyPlayerItem(it); items.splice(i,1); continue;
    }
    for(const e of enemies){
      if(e.dead) continue;
      if(ENEMY_PICKABLE.indexOf(it.type) < 0) continue;
      if(Math.hypot(e.x-it.x, e.y-it.y) < e.r+it.r+4){
        applyEnemyItem(e, it); items.splice(i,1); break;
      }
    }
  }
}

/* =========================================================
   更新
   ========================================================= */
let flowTimer = 0;

function updateCamera(dt){
  let tx, ty;
  if(WORLD_W <= VW) tx = (WORLD_W-VW)/2;
  else tx = clamp(player.x-VW/2, 0, WORLD_W-VW);
  if(WORLD_H <= VH) ty = (WORLD_H-VH)/2;
  else ty = clamp(player.y-VH/2, 0, WORLD_H-VH);
  const k = 1 - Math.exp(-14*dt);
  cam.x += (tx-cam.x)*k;
  cam.y += (ty-cam.y)*k;
}

function getCurrentWeaponType(){
  return player.tempWeapon ? player.tempWeapon.type : player.baseWeapon;
}

function tryFire(){
  if(player.cooldown > 0) return;
  const wType = getCurrentWeaponType();
  const w = WEAPONS[wType];
  player.cooldown = w.fireRate;

  const baseDmg = w.dmg * (player.dmg / 3.5);
  const n = w.bulletCount + player.multiShot;
  const spread = w.spread || 0;

  for(let s=0;s<n;s++){
    const a = player.angle + (n>1 ? (s-(n-1)/2)*spread : 0);
    bullets.push({
      x: player.x+Math.cos(a)*22, y: player.y+Math.sin(a)*22,
      vx: Math.cos(a)*w.bulletSpeed, vy: Math.sin(a)*w.bulletSpeed,
      r: w.explosive ? 7 : 5,
      team:'player',
      dmg: baseDmg,
      crit: Math.random() < player.critChance,
      pierce: (w.pierce !== undefined ? w.pierce : player.pierce),
      hitSet:null,
      life: w.explosive ? 1.2 : 1.6,
      color: w.color,
      explosive: !!w.explosive,
      fuse: w.explosive ? 0.7 : 0
    });
  }

  if(player.tempWeapon && player.tempWeapon.ammo !== undefined){
    player.tempWeapon.ammo--;
    if(player.tempWeapon.ammo <= 0) player.tempWeapon = null;
  }

  spawnParticles(player.x+Math.cos(player.angle)*26, player.y+Math.sin(player.angle)*26,
    w.color, 4, 160);
  sfxShoot();
  shake = Math.max(shake, w.explosive ? 5 : 2.5);
}

function update(dt){
  if(game.paused) return;

  game.time += dt;
  game.restartLock = Math.max(0, game.restartLock-dt);
  shake = Math.max(0, shake-dt*40);
  game.flash = Math.max(0, game.flash - dt*1.6);

  if(game.toast){
    game.toast.t -= dt;
    if(game.toast.t <= 0) game.toast = null;
  }

  if(game.over){ updateParticles(dt); updateEffects(dt); updateFloatTexts(dt); return; }
  if(game.buffMode || game.shopMode){ updateParticles(dt); updateEffects(dt); updateFloatTexts(dt); return; }

  player.cooldown = Math.max(0, player.cooldown-dt);
  player.invuln   = Math.max(0, player.invuln-dt);
  player.hurtTimer= Math.max(0, player.hurtTimer-dt);
  player.invulnTimer = Math.max(0, player.invulnTimer-dt);
  player.speedTimer = Math.max(0, player.speedTimer-dt);

  if(player.tempWeapon && player.tempWeapon.timer !== undefined){
    player.tempWeapon.timer -= dt;
    if(player.tempWeapon.timer <= 0) player.tempWeapon = null;
  }

  for(const s of player.skills){
    if(s.cd > 0) s.cd = Math.max(0, s.cd-dt);
  }

  if(game.comboTimer > 0){
    game.comboTimer -= dt;
    if(game.comboTimer <= 0) game.combo = 0;
  }

  flowTimer -= dt;
  if(flowTimer <= 0){ buildFlowField(); flowTimer = 0.25; }

  let mx = 0, my = 0;
  if(sticks.move.active){ mx = sticks.move.dx; my = sticks.move.dy; }
  if(keys['KeyW'] || keys['ArrowUp'])    my -= 1;
  if(keys['KeyS'] || keys['ArrowDown'])  my += 1;
  if(keys['KeyA'] || keys['ArrowLeft'])  mx -= 1;
  if(keys['KeyD'] || keys['ArrowRight']) mx += 1;
  let mag = Math.hypot(mx,my);
  if(mag > 1){ mx/=mag; my/=mag; mag=1; }
  if(mag > 0.02){
    const spd = player.speed * (player.speedTimer>0 ? 1.5 : 1);
    moveEntity(player, mx*spd*dt, my*spd*dt);
  }

  let aimAngle = null;
  if(sticks.aim.active && Math.hypot(sticks.aim.dx, sticks.aim.dy) > 0.22){
    aimAngle = Math.atan2(sticks.aim.dy, sticks.aim.dx);
  }else if(mouse.active && !sticks.aim.active){
    mouse.wx = mouse.sx+cam.x; mouse.wy = mouse.sy+cam.y;
    aimAngle = Math.atan2(mouse.wy-player.y, mouse.wx-player.x);
  }else if(mag > 0.15){
    aimAngle = Math.atan2(my, mx);
  }
  if(aimAngle !== null) player.angle = aimAngle;

  let wantFire = false;
  if(sticks.aim.active && Math.hypot(sticks.aim.dx, sticks.aim.dy) > 0.28) wantFire = true;
  if(mouse.down && mouse.active && !sticks.aim.active) wantFire = true;
  if(keys['Space']) wantFire = true;
  if(wantFire) tryFire();

  updateBullets(dt);
  updateEnemies(dt);
  updateMinions(dt);
  checkPickups();
  updateParticles(dt);
  updateEffects(dt);
  updateFloatTexts(dt);

  const aliveCount = enemies.filter(e=>!e.dead).length;
  if(aliveCount === 0){
    const needBuff = (game.wave > 0 && game.wave % 5 === 0 && game.lastBuffWave !== game.wave);
    if(needBuff){
      game.buffMode = true;
      game.buffChoices = pickBuffs(3);
      endStick(sticks.move); endStick(sticks.aim);
      mouse.down = false;
    }else{
      game.waveDelay -= dt;
      if(game.waveDelay <= 0){
        const nextWave = game.wave+1;
        if(isBossWave(nextWave) && game.lastShopWave !== nextWave){
          game.lastShopWave = nextWave;
          openShop();
        }else{
          startWave();
        }
      }
    }
  }else{
    game.waveDelay = 1.8;
  }

  updateCamera(dt);
}

function updateBullets(dt){
  for(let i=bullets.length-1;i>=0;i--){
    const b = bullets[i];
    b.life -= dt;
    if(b.explosive && b.fuse > 0){
      b.fuse -= dt;
      if(b.fuse <= 0){
        explode(b.x, b.y, 110, b.dmg);
        bullets.splice(i,1);
        continue;
      }
    }
    let dead = (b.life <= 0);

    const steps = 3;
    for(let s=0;s<steps && !dead;s++){
      b.x += b.vx*dt/steps;
      b.y += b.vy*dt/steps;

      if(circleHitsWall(b.x, b.y, b.r)){
        if(b.explosive) explode(b.x, b.y, 110, b.dmg);
        else spawnParticles(b.x, b.y, '#88aacc', 5, 130);
        dead = true; break;
      }

      if(b.team === 'player'){
        for(const e of enemies){
          if(e.dead) continue;
          if(b.hitSet && b.hitSet.has(e)) continue;
          if(Math.hypot(e.x-b.x, e.y-b.y) < e.r+b.r){
            if(b.explosive){
              explode(b.x, b.y, 110, b.dmg);
              dead = true;
              break;
            }
            let dmg = b.crit ? b.dmg*2 : b.dmg;
            if(e.armor > 0){
              const ab = Math.min(e.armor, dmg);
              e.armor -= ab; dmg -= ab;
            }
            e.hp -= dmg;
            e.hitFlash = 0.12;
            if(!b.hitSet) b.hitSet = new Set();
            b.hitSet.add(e);
            addFloatText(e.x, e.y-e.r-4, Math.round(dmg), b.crit?'#ffd93d':'#ffffff', b.crit?22:16);
            spawnParticles(b.x, b.y, b.crit?'#ffd93d':'#ff8a5a', b.crit?12:8, 220);
            sfxHit();
            if(e.hp <= 0) killEnemy(e);
            if(b.pierce > 0){ b.pierce--; } else { dead = true; }
            break;
          }
        }
      }else{
        for(const m of minions){
          if(m.dead) continue;
          if(Math.hypot(m.x-b.x, m.y-b.y) < m.r+b.r){
            m.hp -= b.dmg;
            spawnParticles(b.x, b.y, '#c8a0ff', 6, 150);
            if(m.hp <= 0) m.dead = true;
            dead = true;
            break;
          }
        }
        if(!dead && Math.hypot(player.x-b.x, player.y-b.y) < player.r+b.r){
          damagePlayer(b.dmg);
          spawnParticles(b.x, b.y, '#ffcc44', 8, 200);
          dead = true;
        }
      }
    }
    if(dead) bullets.splice(i,1);
  }
}

function updateEnemies(dt){
  for(const e of enemies){
    if(e.dead) continue;
    e.hitFlash = Math.max(0, e.hitFlash-dt);
    e.speedTimer = Math.max(0, e.speedTimer-dt);

    const dx = player.x-e.x, dy = player.y-e.y;
    const d  = Math.hypot(dx,dy) || 1;
    e.angle = Math.atan2(dy, dx);

    if(e.isBoss){
      e.summonCool -= dt;
      if(e.summonCool <= 0){
        e.summonCool = 8;
        const eliteCount = enemies.filter(x=>x.type==='elite' && !x.dead).length;
        const toSpawn = Math.max(0, 4 - eliteCount);
        for(let i=0;i<toSpawn;i++) spawnElite();
        if(toSpawn > 0) spawnParticles(e.x, e.y, '#ff44aa', 20, 300);
      }
    }

    const desired = e.isBoss ? 200 : e.range*0.72;
    let vx = 0, vy = 0;

    const ex = Math.floor(e.x/TILE), ey = Math.floor(e.y/TILE);
    let flowDir = null;
    if(flowField[ey] && flowField[ey][ex] !== Infinity){
      let bestD = flowField[ey][ex];
      for(const [ddx,ddy] of DIRS){
        const nx = ex+ddx, ny = ey+ddy;
        if(nx<0||ny<0||nx>=COLS||ny>=ROWS) continue;
        const val = flowField[ny][nx];
        if(val < bestD){ bestD = val; flowDir = [ddx,ddy]; }
      }
    }

    const far = d > desired+40, near = d < desired-40;

    if(far && flowDir){
      const tx = (ex+flowDir[0]+0.5)*TILE, ty = (ey+flowDir[1]+0.5)*TILE;
      const ddx = tx-e.x, ddy = ty-e.y;
      const dl = Math.hypot(ddx,ddy)||1;
      vx += ddx/dl; vy += ddy/dl;
    }else if(near){
      let worstD = -1, awayDir = null;
      if(flowField[ey] && flowField[ey][ex] !== Infinity){
        for(const [ddx,ddy] of DIRS){
          const nx = ex+ddx, ny = ey+ddy;
          if(nx<0||ny<0||nx>=COLS||ny>=ROWS) continue;
          const val = flowField[ny][nx];
          if(val !== Infinity && val > worstD){ worstD = val; awayDir = [ddx,ddy]; }
        }
      }
      if(awayDir){
        const tx = (ex+awayDir[0]+0.5)*TILE, ty = (ey+awayDir[1]+0.5)*TILE;
        const ddx = tx-e.x, ddy = ty-e.y;
        const dl = Math.hypot(ddx,ddy)||1;
        vx += ddx/dl; vy += ddy/dl;
      }else{ vx -= dx/d; vy -= dy/d; }
    }else{
      const strafe = Math.sin(game.time*1.6+e.seed)*0.8;
      vx += (-dy/d)*strafe; vy += (dx/d)*strafe;
    }

    for(const o of enemies){
      if(o === e || o.dead) continue;
      const ox = e.x-o.x, oy = e.y-o.y;
      const od = Math.hypot(ox,oy);
      if(od > 0.01 && od < e.r+o.r+10){
        vx += (ox/od)*1.2; vy += (oy/od)*1.2;
      }
    }

    const vl = Math.hypot(vx,vy);
    if(vl > 0.001){
      const spd = e.speed * (e.speedTimer>0 ? 1.5 : 1);
      moveEntity(e, (vx/vl)*spd*dt, (vy/vl)*spd*dt);
    }

    e.cool -= dt;
    if(d < e.range*1.4 && e.cool <= 0 && hasLineOfSight(e, player)){
      e.cool = e.fireRate * rand(0.8,1.3);
      const a = Math.atan2(dy,dx) + rand(-0.13,0.13);
      if(e.isBoss){
        for(let s=0;s<3;s++){
          const sa = a + (s-1)*0.12;
          bullets.push({
            x: e.x+Math.cos(sa)*34, y: e.y+Math.sin(sa)*34,
            vx: Math.cos(sa)*e.bulletSpeed, vy: Math.sin(sa)*e.bulletSpeed,
            r:6, team:'enemy', dmg:15, life:2.4
          });
        }
      }else{
        bullets.push({
          x: e.x+Math.cos(a)*20, y: e.y+Math.sin(a)*20,
          vx: Math.cos(a)*e.bulletSpeed, vy: Math.sin(a)*e.bulletSpeed,
          r:5, team:'enemy', dmg:9, life:2.2
        });
      }
    }
  }

  if(enemies.some(e=>e.dead)) enemies = enemies.filter(e=>!e.dead);
}

function updateMinions(dt){
  for(const m of minions){
    if(m.dead) continue;
    let target = null, bestD = Infinity;
    for(const e of enemies){
      if(e.dead) continue;
      const d = Math.hypot(e.x-m.x, e.y-m.y);
      if(d < bestD){ bestD = d; target = e; }
    }

    const pd = Math.hypot(player.x-m.x, player.y-m.y);
    if(pd > 90){
      const dx = (player.x-m.x)/pd, dy = (player.y-m.y)/pd;
      moveEntity(m, dx*180*dt, dy*180*dt);
    }else if(target && bestD > 220){
      const dx = (target.x-m.x)/bestD, dy = (target.y-m.y)/bestD;
      moveEntity(m, dx*180*dt, dy*180*dt);
    }

    if(target){
      m.angle = Math.atan2(target.y-m.y, target.x-m.x);
      m.cool -= dt;
      if(m.cool <= 0){
        m.cool = Math.max(0.1, player.fireRate/0.8);
        const a = m.angle;
        bullets.push({
          x: m.x+Math.cos(a)*14, y: m.y+Math.sin(a)*14,
          vx: Math.cos(a)*700, vy: Math.sin(a)*700,
          r:4, team:'player', dmg:3, crit:false,
          pierce:0, hitSet:null, life:1.2, color:'#c8a0ff'
        });
      }
    }
  }
  if(minions.some(m=>m.dead)) minions = minions.filter(m=>!m.dead);
}

function updateParticles(dt){
  for(let i=particles.length-1;i>=0;i--){
    const p = particles[i];
    p.life -= dt;
    if(p.life <= 0){ particles.splice(i,1); continue; }
    p.x += p.vx*dt; p.y += p.vy*dt;
    p.vx *= 0.94; p.vy *= 0.94;
  }
}

function updateEffects(dt){
  for(let i=effects.length-1;i>=0;i--){
    const fx = effects[i];
    fx.life -= dt;
    if(fx.life <= 0) effects.splice(i,1);
  }
}

/* =========================================================
   绘制
   ========================================================= */
function roundRect(g,x,y,w,h,r){
  if(g.roundRect){ g.beginPath(); g.roundRect(x,y,w,h,r); return; }
  g.beginPath();
  g.moveTo(x+r,y);
  g.arcTo(x+w,y,x+w,y+h,r);
  g.arcTo(x+w,y+h,x,y+h,r);
  g.arcTo(x,y+h,x,y,r);
  g.arcTo(x,y,x+w,y,r);
  g.closePath();
}

function wrapText(g,text,cx,cy,maxW,lineH,align){
  const chars = text.split('');
  let line = ''; const lines = [];
  for(const ch of chars){
    const test = line+ch;
    if(g.measureText(test).width > maxW && line !== ''){ lines.push(line); line = ch; }
    else line = test;
  }
  if(line) lines.push(line);
  const totalH = lines.length*lineH;
  let y = cy - totalH/2 + lineH/2;
  g.textAlign = align||'center';
  for(const l of lines){ g.fillText(l, cx, y); y += lineH; }
}

function layoutBuffCards(){
  const n = game.buffChoices.length;
  const cards = [];
  const horizontal = VW >= 760;
  if(horizontal){
    const gap = 26;
    const cw = Math.min(310, (VW-110-gap*(n-1))/n);
    const ch = Math.min(380, VH*0.55);
    const total = cw*n+gap*(n-1);
    const sx = (VW-total)/2;
    const y = VH*0.52-ch/2;
    for(let i=0;i<n;i++) cards.push({ x:sx+i*(cw+gap), y, w:cw, h:ch, compact:false });
  }else{
    const gap = 14;
    const cw = Math.min(VW-60, 460);
    const ch = Math.min(132, (VH*0.62-gap*(n-1))/n);
    const total = ch*n+gap*(n-1);
    const sx = (VW-cw)/2;
    const y = VH*0.58-total/2;
    for(let i=0;i<n;i++) cards.push({ x:sx, y:y+i*(ch+gap), w:cw, h:ch, compact:true });
  }
  return cards;
}

function drawBuffSelect(){
  ctx.fillStyle = 'rgba(3,6,14,0.88)';
  ctx.fillRect(0,0,VW,VH);
  const horizontal = VW >= 760;
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  const titleY = horizontal ? VH*0.15 : VH*0.13;
  ctx.fillStyle = '#9fd0ff';
  ctx.font = 'bold ' + Math.min(46, VW*0.045) + 'px "Microsoft YaHei",sans-serif';
  ctx.fillText('⚡ 强化选择 ⚡', VW/2, titleY);
  ctx.fillStyle = 'rgba(200,225,255,0.72)';
  ctx.font = 'bold 19px "Microsoft YaHei",sans-serif';
  ctx.fillText('第 '+game.wave+' 波清场完成 · 点击卡片选择一项永久强化', VW/2, titleY+46);
  const rects = layoutBuffCards();
  for(let i=0;i<game.buffChoices.length;i++){
    const b = game.buffChoices[i];
    const r = rects[i];
    b._rect = r;
    drawBuffCard(b, r, i);
  }
}

function drawBuffCard(b, r, index){
  const pulse = 0.5+0.5*Math.sin(game.time*3+index*1.3);
  ctx.save();
  ctx.shadowBlur = 16+pulse*16;
  ctx.shadowColor = b.color;
  const grd = ctx.createLinearGradient(r.x, r.y, r.x, r.y+r.h);
  grd.addColorStop(0,'rgba(24,36,56,0.97)');
  grd.addColorStop(1,'rgba(12,18,30,0.97)');
  ctx.fillStyle = grd;
  roundRect(ctx, r.x, r.y, r.w, r.h, 18); ctx.fill();
  ctx.shadowBlur = 0;
  ctx.strokeStyle = b.color; ctx.lineWidth = 3;
  roundRect(ctx, r.x, r.y, r.w, r.h, 18); ctx.stroke();
  ctx.textBaseline = 'middle';
  if(!r.compact){
    const iconR = Math.min(48, r.w*0.2);
    const iconY = r.y+r.h*0.27;
    ctx.beginPath(); ctx.arc(r.x+r.w/2, iconY, iconR, 0, TAU);
    ctx.fillStyle = hexAlpha(b.color,0.16); ctx.fill();
    ctx.strokeStyle = hexAlpha(b.color,0.9); ctx.lineWidth = 3; ctx.stroke();
    ctx.textAlign = 'center'; ctx.fillStyle = b.color;
    ctx.font = 'bold ' + Math.round(iconR*1.05) + 'px "Microsoft YaHei",sans-serif';
    ctx.fillText(b.symbol, r.x+r.w/2, iconY+2);
    ctx.fillStyle = '#eaf4ff';
    ctx.font = 'bold 27px "Microsoft YaHei",sans-serif';
    ctx.fillText(b.name, r.x+r.w/2, r.y+r.h*0.55);
    ctx.fillStyle = 'rgba(190,215,245,0.88)';
    ctx.font = '18px "Microsoft YaHei",sans-serif';
    wrapText(ctx, b.desc, r.x+r.w/2, r.y+r.h*0.74, r.w-44, 27, 'center');
  }else{
    const iconR = Math.min(36, r.h*0.34);
    const iconX = r.x+22+iconR;
    const iconY = r.y+r.h/2;
    ctx.beginPath(); ctx.arc(iconX, iconY, iconR, 0, TAU);
    ctx.fillStyle = hexAlpha(b.color,0.16); ctx.fill();
    ctx.strokeStyle = hexAlpha(b.color,0.9); ctx.lineWidth = 3; ctx.stroke();
    ctx.textAlign = 'center'; ctx.fillStyle = b.color;
    ctx.font = 'bold ' + Math.round(iconR*0.95) + 'px "Microsoft YaHei",sans-serif';
    ctx.fillText(b.symbol, iconX, iconY+2);
    const tx = iconX+iconR+20;
    const maxW = r.x+r.w-tx-20;
    ctx.textAlign = 'left'; ctx.fillStyle = '#eaf4ff';
    ctx.font = 'bold 22px "Microsoft YaHei",sans-serif';
    ctx.fillText(b.name, tx, r.y+r.h*0.35);
    ctx.fillStyle = 'rgba(190,215,245,0.88)';
    ctx.font = '16px "Microsoft YaHei",sans-serif';
    wrapText(ctx, b.desc, tx+maxW/2, r.y+r.h*0.7, maxW, 22, 'center');
  }
  ctx.restore();
}

function drawShop(){
  ctx.fillStyle = 'rgba(3,6,14,0.94)';
  ctx.fillRect(0,0,VW,VH);

  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillStyle = '#ffd93d';
  ctx.font = 'bold 40px "Microsoft YaHei",sans-serif';
  ctx.fillText('🛒 商 店', VW/2, 56);

  ctx.fillStyle = '#9fd0ff';
  ctx.font = 'bold 26px "Microsoft YaHei",sans-serif';
  ctx.fillText('积分：' + game.score, VW/2, 104);

  const n = game.shopItems.length;
  const cols = VW >= 760 ? 3 : 2;
  const rows = Math.ceil(n/cols);
  const gapX = 18, gapY = 18;
  const cw = Math.min(280, (VW-80-gapX*(cols-1))/cols);
  const ch = Math.min(200, (VH-280-gapY*(rows-1))/rows);
  const totalW = cw*cols+gapX*(cols-1);
  const totalH = ch*rows+gapY*(rows-1);
  const sx = (VW-totalW)/2;
  const sy = 150 + Math.max(0, (VH-150-110-totalH)/2);

  for(let i=0;i<n;i++){
    const col = i%cols, row = Math.floor(i/cols);
    const r = { x: sx+col*(cw+gapX), y: sy+row*(ch+gapY), w:cw, h:ch };
    game.shopItems[i]._rect = r;
    drawShopCard(game.shopItems[i], r);
  }

  const btn = { x: VW/2-90, y: VH-70, w:180, h:52 };
  game._shopLeaveBtn = btn;
  ctx.fillStyle = 'rgba(60,90,140,0.85)';
  roundRect(ctx, btn.x, btn.y, btn.w, btn.h, 14); ctx.fill();
  ctx.strokeStyle = '#9fd0ff'; ctx.lineWidth = 2;
  roundRect(ctx, btn.x, btn.y, btn.w, btn.h, 14); ctx.stroke();
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 22px "Microsoft YaHei",sans-serif';
  ctx.fillText('离开商店', btn.x+btn.w/2, btn.y+btn.h/2);
}

function drawShopCard(item, r){
  const def = SHOP_ITEMS[item.id];
  const canAfford = game.score >= item.price;
  const sold = item.sold;

  const grd = ctx.createLinearGradient(r.x, r.y, r.x, r.y+r.h);
  grd.addColorStop(0, sold ? 'rgba(30,30,30,0.92)' : 'rgba(24,36,56,0.97)');
  grd.addColorStop(1, sold ? 'rgba(20,20,20,0.92)' : 'rgba(12,18,30,0.97)');
  ctx.fillStyle = grd;
  roundRect(ctx, r.x, r.y, r.w, r.h, 14); ctx.fill();

  ctx.strokeStyle = sold ? '#555' : (canAfford ? def.color : '#666');
  ctx.lineWidth = 2;
  roundRect(ctx, r.x, r.y, r.w, r.h, 14); ctx.stroke();

  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillStyle = sold ? '#666' : def.color;
  ctx.font = 'bold 32px "Microsoft YaHei",sans-serif';
  ctx.fillText(def.icon, r.x+r.w/2, r.y+r.h*0.22);

  ctx.fillStyle = sold ? '#666' : '#eaf4ff';
  ctx.font = 'bold 20px "Microsoft YaHei",sans-serif';
  ctx.fillText(def.name, r.x+r.w/2, r.y+r.h*0.45);

  ctx.fillStyle = sold ? '#555' : 'rgba(190,215,245,0.85)';
  ctx.font = '14px "Microsoft YaHei",sans-serif';
  wrapText(ctx, def.desc, r.x+r.w/2, r.y+r.h*0.63, r.w-24, 18, 'center');

  if(sold){
    ctx.fillStyle = '#888';
    ctx.font = 'bold 18px "Microsoft YaHei",sans-serif';
    ctx.fillText('已售出', r.x+r.w/2, r.y+r.h*0.86);
  }else{
    ctx.fillStyle = canAfford ? '#ffd93d' : '#ff6666';
    ctx.font = 'bold 24px "Microsoft YaHei",sans-serif';
    ctx.fillText('💰 ' + item.price, r.x+r.w/2, r.y+r.h*0.86);
  }
}

function drawSkillButtons(){
  layoutSkillButtons();
  for(const b of skillBtnRects){
    const s = player.skills[b.index];
    const def = SKILLS[s.id];
    const ready = s.cd <= 0;
    const cx = b.x+b.w/2, cy = b.y+b.h/2, rr = b.w/2;

    ctx.beginPath(); ctx.arc(cx, cy, rr, 0, TAU);
    ctx.fillStyle = ready ? hexAlpha(def.color, 0.25) : 'rgba(40,40,50,0.75)';
    ctx.fill();
    ctx.strokeStyle = ready ? def.color : '#555';
    ctx.lineWidth = 3; ctx.stroke();

    ctx.fillStyle = ready ? def.color : '#666';
    ctx.font = 'bold ' + Math.round(b.w*0.4) + 'px "Microsoft YaHei",sans-serif';
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(def.symbol, cx, cy);

    if(!ready){
      ctx.fillStyle = 'rgba(0,0,0,0.55)';
      ctx.beginPath();
      ctx.moveTo(cx, cy);
      ctx.arc(cx, cy, rr, -Math.PI/2, -Math.PI/2 + TAU*(s.cd/def.cooldown));
      ctx.closePath();
      ctx.fill();
      ctx.fillStyle = '#fff';
      ctx.font = 'bold 16px "Microsoft YaHei",sans-serif';
      ctx.fillText(Math.ceil(s.cd), cx, cy);
    }
  }
}

function drawUIButtons(){
  ctx.save();
  ctx.fillStyle = 'rgba(10,18,30,0.72)';
  roundRect(ctx, pauseBtn.x, pauseBtn.y, pauseBtn.w, pauseBtn.h, 10); ctx.fill();
  ctx.strokeStyle = 'rgba(120,180,255,0.5)'; ctx.lineWidth = 2;
  roundRect(ctx, pauseBtn.x, pauseBtn.y, pauseBtn.w, pauseBtn.h, 10); ctx.stroke();
  ctx.fillStyle = '#9fd0ff';
  ctx.font = 'bold 20px "Microsoft YaHei",sans-serif';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText(game.paused ? '▶' : '❚❚', pauseBtn.x+pauseBtn.w/2, pauseBtn.y+pauseBtn.h/2+1);

  ctx.fillStyle = 'rgba(10,18,30,0.72)';
  roundRect(ctx, musicBtn.x, musicBtn.y, musicBtn.w, musicBtn.h, 10); ctx.fill();
  ctx.strokeStyle = musicOn ? 'rgba(255,217,61,0.6)' : 'rgba(120,120,120,0.4)';
  ctx.lineWidth = 2;
  roundRect(ctx, musicBtn.x, musicBtn.y, musicBtn.w, musicBtn.h, 10); ctx.stroke();
  ctx.fillStyle = musicOn ? '#ffd93d' : '#666';
  ctx.font = 'bold 22px "Microsoft YaHei",sans-serif';
  ctx.fillText('♪', musicBtn.x+musicBtn.w/2, musicBtn.y+musicBtn.h/2+1);
  if(!musicOn){
    ctx.strokeStyle = '#666'; ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(musicBtn.x+11, musicBtn.y+musicBtn.h-11);
    ctx.lineTo(musicBtn.x+musicBtn.w-11, musicBtn.y+11);
    ctx.stroke();
  }
  ctx.restore();
}

function drawOffscreenIndicators(){
  const margin = 34;
  for(const e of enemies){
    if(e.dead) continue;
    if(e.type !== 'boss' && e.type !== 'elite') continue;
    const sx = e.x - cam.x, sy = e.y - cam.y;
    if(sx > 0 && sx < VW && sy > 0 && sy < VH) continue;

    const cx = VW/2, cy = VH/2;
    const ang = Math.atan2(sy-cy, sx-cx);
    const halfW = VW/2 - margin;
    const halfH = VH/2 - margin;
    const cosA = Math.cos(ang), sinA = Math.sin(ang);
    const tx = Math.abs(cosA) > 1e-6 ? halfW/Math.abs(cosA) : Infinity;
    const ty = Math.abs(sinA) > 1e-6 ? halfH/Math.abs(sinA) : Infinity;
    const t = Math.min(tx, ty);
    const px = cx + cosA*t;
    const py = cy + sinA*t;

    ctx.save();
    ctx.translate(px, py);
    ctx.rotate(ang);
    ctx.beginPath();
    ctx.moveTo(14, 0);
    ctx.lineTo(-8, -9);
    ctx.lineTo(-8, 9);
    ctx.closePath();
    ctx.fillStyle = e.type === 'boss' ? '#ff4444' : '#ff44aa';
    ctx.globalAlpha = 0.85;
    ctx.fill();
    ctx.restore();
  }
}

function render(){
  ctx.fillStyle = '#05050a';
  ctx.fillRect(0,0,VW,VH);

  const sx = shake>0 ? rand(-shake,shake) : 0;
  const sy = shake>0 ? rand(-shake,shake) : 0;

  ctx.save();
  ctx.translate(-Math.round(cam.x)+sx, -Math.round(cam.y)+sy);

  ctx.drawImage(mapCanvas, 0, 0);

  drawItems();

  for(const fx of effects){
    if(fx.type === 'ring'){
      const t = 1 - fx.life/fx.maxLife;
      const rr = fx.maxR*t;
      ctx.globalAlpha = clamp(fx.life/fx.maxLife, 0, 1);
      ctx.beginPath(); ctx.arc(fx.x, fx.y, rr, 0, TAU);
      ctx.strokeStyle = fx.color; ctx.lineWidth = 4; ctx.stroke();
      ctx.globalAlpha = 1;
    }
  }

  for(const p of particles){
    const a = clamp(p.life/p.maxLife, 0, 1);
    ctx.globalAlpha = a;
    ctx.fillStyle = p.color;
    ctx.beginPath(); ctx.arc(p.x, p.y, p.size*a+1, 0, TAU); ctx.fill();
  }
  ctx.globalAlpha = 1;

  for(const b of bullets){
    const isPlayer = (b.team === 'player');
    ctx.shadowBlur = 14;
    ctx.shadowColor = isPlayer ? (b.crit?'#ffd93d':(b.explosive?'#ffaa44':'#8ee0ff')) : '#ff8844';
    ctx.fillStyle  = isPlayer ? (b.crit?'#ffe066':(b.explosive?'#ffcc66':'#c8f2ff')) : '#ffcc66';
    ctx.beginPath(); ctx.arc(b.x, b.y, b.r, 0, TAU); ctx.fill();
    ctx.shadowBlur = 0;
  }

  for(const m of minions){
    if(m.dead) continue;
    ctx.save();
    ctx.translate(m.x, m.y);
    ctx.rotate(m.angle);
    ctx.beginPath(); ctx.arc(0, 0, m.r, 0, TAU);
    ctx.fillStyle = '#9b7bff'; ctx.fill();
    ctx.lineWidth = 2; ctx.strokeStyle = '#d8c8ff'; ctx.stroke();
    ctx.fillStyle = '#c8a0ff';
    ctx.fillRect(m.r-3, -2, 8, 4);
    ctx.restore();
    const bw = 24, bh = 3;
    const pct = clamp(m.hp/m.maxHp, 0, 1);
    ctx.fillStyle = 'rgba(0,0,0,0.55)';
    ctx.fillRect(m.x-bw/2, m.y-m.r-9, bw, bh);
    ctx.fillStyle = '#9b7bff';
    ctx.fillRect(m.x-bw/2, m.y-m.r-9, bw*pct, bh);
  }

  for(const e of enemies){
    if(e.dead) continue;
    ctx.save();
    ctx.translate(e.x, e.y);
    ctx.rotate(e.angle);

    if(e.isBoss){
      ctx.fillStyle = '#4a2222';
      ctx.fillRect(8, -5, 30, 10);
      ctx.fillStyle = '#2a1010';
      ctx.fillRect(34, -3, 10, 6);
    }else{
      ctx.fillStyle = '#4a2222';
      ctx.fillRect(4, -3.5, 17, 7);
    }

    ctx.beginPath(); ctx.arc(0, 0, e.r, 0, TAU);
    ctx.fillStyle = e.hitFlash>0 ? '#ffffff' : e.color;
    ctx.fill();
    ctx.lineWidth = 3;
    ctx.strokeStyle = e.isBoss ? '#ffcc44' : 'rgba(255,255,255,0.35)';
    ctx.stroke();

    ctx.beginPath(); ctx.arc(e.isBoss?12:7, 0, e.isBoss?7:4.5, 0, TAU);
    ctx.fillStyle = 'rgba(0,0,0,0.55)'; ctx.fill();

    if(e.armor > 0){
      ctx.beginPath(); ctx.arc(0, 0, e.r+5, 0, TAU);
      ctx.strokeStyle = 'rgba(79,163,255,' + (0.6+0.3*Math.sin(game.time*6)) + ')';
      ctx.lineWidth = 3; ctx.stroke();
    }
    ctx.restore();

    if(!e.isBoss){
      const bw = 34, bh = 5;
      const hpPct = clamp(e.hp/e.maxHp, 0, 1);
      ctx.fillStyle = 'rgba(0,0,0,0.55)';
      ctx.fillRect(e.x-bw/2, e.y-e.r-14, bw, bh);
      ctx.fillStyle = hpPct>0.5 ? '#5be06b' : (hpPct>0.25 ? '#ffc93c' : '#ff4d4d');
      ctx.fillRect(e.x-bw/2, e.y-e.r-14, bw*hpPct, bh);
      if(e.armor > 0){
        const ap = clamp(e.armor/10, 0, 1);
        ctx.fillStyle = 'rgba(0,0,0,0.55)';
        ctx.fillRect(e.x-bw/2, e.y-e.r-8, bw, 3);
        ctx.fillStyle = '#4fa3ff';
        ctx.fillRect(e.x-bw/2, e.y-e.r-8, bw*ap, 3);
      }
    }
  }

  drawPlayer();

  for(const f of floatTexts){
    const a = clamp(f.life/f.maxLife, 0, 1);
    ctx.globalAlpha = a;
    ctx.font = 'bold ' + f.size + 'px "Microsoft YaHei",sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.lineWidth = 3;
    ctx.strokeStyle = 'rgba(0,0,0,0.55)';
    ctx.strokeText(f.text, f.x, f.y);
    ctx.fillStyle = f.color;
    ctx.fillText(f.text, f.x, f.y);
  }
  ctx.globalAlpha = 1;

  if(game.combo >= 2 && !game.over){
    const pulse = 1 + 0.12*Math.sin(game.time*12);
    ctx.save();
    ctx.translate(player.x, player.y - 58);
    ctx.scale(pulse, pulse);
    ctx.font = 'bold 30px "Microsoft YaHei",sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.lineWidth = 5;
    ctx.strokeStyle = 'rgba(0,0,0,0.6)';
    ctx.strokeText(game.combo + ' COMBO', 0, 0);
    ctx.fillStyle = '#ffd93d';
    ctx.fillText(game.combo + ' COMBO', 0, 0);
    ctx.restore();
  }

  ctx.restore();

  if(!game.buffMode && !game.shopMode && !game.over){
    drawSticks();
    drawSkillButtons();
    drawHUD();
    drawUIButtons();
    drawOffscreenIndicators();
  }

  if(game.buffMode) drawBuffSelect();
  if(game.shopMode) drawShop();
  if(game.over) drawGameOver();
  if(game.toast && !game.over && !game.buffMode && !game.shopMode) drawToast();

  const hpPct = player.hp/player.maxHp;
  if(hpPct < 0.3 && !game.over){
    const intensity = (0.3 - hpPct) / 0.3;
    const pulse = 0.5 + 0.5*Math.sin(game.time*6);
    const a = intensity * 0.32 * (0.6 + 0.4*pulse);
    const grd = ctx.createRadialGradient(VW/2, VH/2, Math.min(VW,VH)*0.25, VW/2, VH/2, Math.max(VW,VH)*0.75);
    grd.addColorStop(0, 'rgba(255,0,0,0)');
    grd.addColorStop(1, 'rgba(255,0,0,' + a + ')');
    ctx.fillStyle = grd;
    ctx.fillRect(0,0,VW,VH);
  }

  if(game.flash > 0){
    ctx.fillStyle = 'rgba(255,255,255,' + (game.flash*0.55) + ')';
    ctx.fillRect(0,0,VW,VH);
  }

  if(game.paused){
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0,0,VW,VH);
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillStyle = '#9fd0ff';
    ctx.font = 'bold 52px "Microsoft YaHei",sans-serif';
    ctx.fillText('已 暂 停', VW/2, VH/2 - 24);
    ctx.font = 'bold 20px "Microsoft YaHei",sans-serif';
    ctx.fillStyle = 'rgba(200,225,255,0.72)';
    ctx.fillText('点击左上角 ▶ 继续游戏', VW/2, VH/2 + 44);
  }

  if(wmCanvas.width > 1) ctx.drawImage(wmCanvas, 0, 0, VW, VH);
}

function drawItems(){
  for(const it of items){
    const info = ITEM_TYPES[it.type];
    const bob = Math.sin(game.time*3+it.bob)*4;
    ctx.shadowBlur = 18;
    ctx.shadowColor = info.color;
    ctx.beginPath(); ctx.arc(it.x, it.y+bob, it.r, 0, TAU);
    ctx.fillStyle = hexAlpha(info.color, 0.25); ctx.fill();
    ctx.shadowBlur = 0;
    ctx.beginPath(); ctx.arc(it.x, it.y+bob, it.r, 0, TAU);
    ctx.strokeStyle = info.color; ctx.lineWidth = 2.5; ctx.stroke();
    ctx.fillStyle = info.color;
    ctx.font = 'bold 16px "Microsoft YaHei",sans-serif';
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    ctx.fillText(info.symbol, it.x, it.y+bob+1);
  }
}

function drawPlayer(){
  const blink = player.invuln>0 && Math.floor(game.time*22)%2===0;
  const inv = player.invulnTimer > 0;

  ctx.save();
  ctx.translate(player.x, player.y);
  ctx.rotate(player.angle);

  const wType = getCurrentWeaponType();
  const w = WEAPONS[wType];

  ctx.fillStyle = '#cfd8e3';
  ctx.fillRect(8, -4.5, 26, 9);
  ctx.fillStyle = w.color;
  ctx.fillRect(26, -3, 10, 6);
  if(wType === 'minigun'){
    ctx.fillStyle = '#ffd93d';
    ctx.fillRect(30, -7, 4, 14);
    ctx.fillRect(34, -7, 4, 14);
  }
  if(wType === 'awp'){
    ctx.fillStyle = '#00ffcc';
    ctx.fillRect(30, -2, 20, 4);
  }

  if(player.multiShot > 0){
    ctx.beginPath(); ctx.arc(0, 0, player.r+6, 0, TAU);
    ctx.strokeStyle = 'rgba(0,212,200,' + (0.35+0.25*Math.sin(game.time*6)) + ')';
    ctx.lineWidth = 2; ctx.stroke();
  }

  ctx.shadowBlur = 20;
  ctx.shadowColor = inv ? '#ffd93d' : '#3aa0ff';
  ctx.beginPath(); ctx.arc(0, 0, player.r, 0, TAU);
  ctx.fillStyle = blink ? '#ffffff' : (player.hurtTimer>0 ? '#ff7a7a' : (inv ? '#ffe066' : '#4fa3ff'));
  ctx.fill();
  ctx.shadowBlur = 0;
  ctx.lineWidth = 3; ctx.strokeStyle = inv ? '#fff3a0' : '#b8ddff'; ctx.stroke();

  ctx.beginPath(); ctx.arc(7.5, 0, 4.5, 0, TAU);
  ctx.fillStyle = '#0b1a2a'; ctx.fill();

  ctx.restore();

  if(player.shield > 0){
    ctx.beginPath(); ctx.arc(player.x, player.y, player.r+8, 0, TAU);
    ctx.strokeStyle = 'rgba(79,163,255,' + (0.4+0.3*Math.sin(game.time*5)) + ')';
    ctx.lineWidth = 3; ctx.stroke();
  }
  if(player.speedTimer > 0){
    ctx.beginPath(); ctx.arc(player.x, player.y, player.r+13, 0, TAU);
    ctx.strokeStyle = 'rgba(255,217,61,0.5)';
    ctx.lineWidth = 2; ctx.stroke();
  }
  if(player.invulnTimer > 0){
    ctx.beginPath(); ctx.arc(player.x, player.y, player.r+17, 0, TAU);
    ctx.strokeStyle = 'rgba(255,240,120,0.8)';
    ctx.lineWidth = 4; ctx.stroke();
  }
}

function drawSticks(){
  if(sticks.move.active) drawStickVisual(sticks.move, '#4fa3ff');
  if(sticks.aim.active)  drawStickVisual(sticks.aim, '#ff7a4f');
}
function drawStickVisual(s, color){
  ctx.save();
  ctx.globalAlpha = 0.35;
  ctx.beginPath(); ctx.arc(s.bx, s.by, STICK_R, 0, TAU);
  ctx.fillStyle = 'rgba(255,255,255,0.06)'; ctx.fill();
  ctx.lineWidth = 3; ctx.strokeStyle = color; ctx.stroke();
  ctx.globalAlpha = 0.75;
  ctx.beginPath(); ctx.arc(s.bx+s.dx*STICK_R, s.by+s.dy*STICK_R, 30, 0, TAU);
  ctx.fillStyle = color; ctx.fill();
  ctx.restore();
}

function drawHUD(){
  const bx = 80, by = 20, bw = 280, bh = 26;
  ctx.fillStyle = 'rgba(0,0,0,0.5)';
  ctx.fillRect(bx, by, bw, bh);
  const pct = clamp(player.hp/player.maxHp, 0, 1);
  ctx.fillStyle = pct>0.3 ? '#5be06b' : '#ff4d4d';
  ctx.fillRect(bx+3, by+3, (bw-6)*pct, bh-6);
  ctx.strokeStyle = 'rgba(255,255,255,0.35)';
  ctx.lineWidth = 2; ctx.strokeRect(bx, by, bw, bh);
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 15px "Microsoft YaHei",sans-serif';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText('HP  ' + Math.ceil(player.hp) + ' / ' + player.maxHp, bx+bw/2, by+bh/2+1);

  let infoY = by+bh+10;
  if(player.shield > 0){
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(bx, infoY, bw, 12);
    ctx.fillStyle = '#4fa3ff';
    ctx.fillRect(bx+2, infoY+2, (bw-4)*clamp(player.shield/100,0,1), 8);
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 10px "Microsoft YaHei",sans-serif';
    ctx.fillText('护盾 ' + Math.ceil(player.shield), bx+bw/2, infoY+6);
    infoY += 18;
  }

  const wType = getCurrentWeaponType();
  const w = WEAPONS[wType];
  ctx.textAlign = 'left';
  ctx.font = 'bold 15px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = w.color;
  let wText = w.name + ' (' + w.dmg.toFixed(1) + ')';
  if(player.tempWeapon){
    if(player.tempWeapon.timer !== undefined) wText += ' · ' + player.tempWeapon.timer.toFixed(1) + 's';
    else if(player.tempWeapon.ammo !== undefined) wText += ' · 弹药 ' + player.tempWeapon.ammo;
  }
  ctx.fillText(wText, bx, infoY+10);
  infoY += 20;

  if(player.speedTimer > 0){
    ctx.fillStyle = '#ffd93d';
    ctx.fillText('⚡ 加速 ' + player.speedTimer.toFixed(1) + 's', bx, infoY+10);
  }

  ctx.textAlign = 'right';
  ctx.font = 'bold 26px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = '#9fd0ff';
  ctx.fillText('积分  ' + game.score, VW-28, 42);

  ctx.font = 'bold 19px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = 'rgba(220,240,255,0.85)';
  ctx.fillText('第 ' + game.wave + ' 波   ·   剩余 ' + enemies.filter(e=>!e.dead).length, VW-28, 76);

  ctx.font = 'bold 15px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = 'rgba(160,200,255,0.55)';
  ctx.fillText('敌方护甲率 ' + Math.round(getEnemyArmorChance()*100) + '%', VW-28, 104);

  const nextBuff = Math.ceil((game.wave+0.0001)/5)*5;
  ctx.fillText('第 ' + nextBuff + ' 波后强化', VW-28, 128);

  const boss = enemies.find(e=>e.isBoss && !e.dead);
  if(boss){
    const w2 = Math.min(600, VW-100);
    const x2 = (VW-w2)/2, y2 = VH-150;
    ctx.fillStyle = 'rgba(0,0,0,0.65)';
    roundRect(ctx, x2, y2, w2, 24, 8); ctx.fill();
    const bp = clamp(boss.hp/boss.maxHp, 0, 1);
    ctx.fillStyle = '#d03030';
    roundRect(ctx, x2+2, y2+2, (w2-4)*bp, 20, 6); ctx.fill();
    ctx.strokeStyle = '#ff8888'; ctx.lineWidth = 2;
    roundRect(ctx, x2, y2, w2, 24, 8); ctx.stroke();
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 14px "Microsoft YaHei",sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('BOSS  ' + Math.ceil(boss.hp) + ' / ' + boss.maxHp, VW/2, y2+13);
  }

  if(game.time < 9 && !game.over){
    const alpha = clamp(9-game.time, 0, 1)*0.75;
    ctx.globalAlpha = alpha;
    ctx.textAlign = 'center';
    ctx.font = 'bold 19px "Microsoft YaHei",sans-serif';
    ctx.fillStyle = '#cfe6ff';
    ctx.fillText('左半屏拖动=移动   右半屏拖动=瞄准开火   右侧圆钮=技能', VW/2, VH-42);
    ctx.globalAlpha = 1;
  }
}

function drawToast(){
  const t = game.toast;
  const a = clamp(t.t, 0, 1);
  ctx.save();
  ctx.globalAlpha = a;
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.font = 'bold 24px "Microsoft YaHei",sans-serif';
  const w = ctx.measureText(t.text).width+56;
  ctx.fillStyle = 'rgba(8,14,24,0.85)';
  roundRect(ctx, VW/2-w/2, VH-130, w, 52, 14); ctx.fill();
  ctx.strokeStyle = hexAlpha(t.color, 0.85); ctx.lineWidth = 2;
  roundRect(ctx, VW/2-w/2, VH-130, w, 52, 14); ctx.stroke();
  ctx.fillStyle = t.color;
  ctx.fillText(t.text, VW/2, VH-103);
  ctx.restore();
}

function drawGameOver(){
  ctx.fillStyle = 'rgba(0,0,0,0.78)';
  ctx.fillRect(0,0,VW,VH);
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';

  ctx.font = 'bold 64px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = '#ff5a5a';
  ctx.fillText('阵 亡', VW/2, VH/2-130);

  ctx.font = 'bold 28px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = '#ffffff';
  ctx.fillText('积分：' + game.score + '    击杀：' + game.kills + '    波数：' + game.wave,
    VW/2, VH/2-50);

  ctx.font = 'bold 22px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = '#ffd93d';
  ctx.fillText('最高连击：' + game.maxCombo + ' COMBO', VW/2, VH/2-4);

  const pulse = 0.65+0.35*Math.sin(game.time*5);
  ctx.globalAlpha = pulse;
  ctx.font = 'bold 26px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = '#9fd0ff';
  ctx.fillText('点击屏幕 / 按空格键  重新开始', VW/2, VH/2+70);
  ctx.globalAlpha = 1;

  ctx.font = 'bold 18px "Microsoft YaHei",sans-serif';
  ctx.fillStyle = 'rgba(200,220,255,0.45)';
  ctx.fillText(WATERMARK_TEXT, VW/2, VH/2+140);
}

/* =========================================================
   主循环
   ========================================================= */
let lastTime = 0;
function loop(now){
  if(!lastTime) lastTime = now;
  let dt = (now-lastTime)/1000;
  lastTime = now;
  if(dt > 0.05) dt = 0.05;
  if(dt < 0) dt = 0;
  update(dt);
  render();
  requestAnimationFrame(loop);
}

function resize(){
  DPR = Math.min(window.devicePixelRatio||1, 2);
  VW = canvas.clientWidth  || window.innerWidth;
  VH = canvas.clientHeight || window.innerHeight;
  canvas.width  = Math.round(VW*DPR);
  canvas.height = Math.round(VH*DPR);
  ctx.setTransform(DPR,0,0,DPR,0,0);
  buildWatermark();
}

window.addEventListener('resize', resize);
window.addEventListener('orientationchange', ()=> setTimeout(resize, 200));
window.addEventListener('pointerdown', ensureAudio, { once:true });

function init(){
  resize();
  resetGame();
  requestAnimationFrame(loop);
}
init();

})();
</script>
</body>
</html>
