<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>言葉のライフゲーム</title>
<style>
  :root{
    --bg:#fbfaf6;
    --ink:#2a2a28;
    --ink-faint:rgba(42,42,40,0.38);
  }
  html,body{
    margin:0; padding:0; width:100%; height:100%;
    background:var(--bg); overflow:hidden;
    font-family:-apple-system, BlinkMacSystemFont, "Noto Sans JP", sans-serif;
  }
  canvas{ display:block; touch-action:none; }

  #overlay{
    position:fixed; inset:0; z-index:20;
    background:var(--bg);
    display:flex; align-items:center; justify-content:center;
    cursor:pointer;
    transition:opacity 1.4s ease;
  }
  #overlay.hidden{ opacity:0; pointer-events:none; }
  .poem{
    display:flex; flex-direction:row-reverse; gap:1.6em;
  }
  .poem span{
    writing-mode:vertical-rl;
    font-family:"Hiragino Mincho ProN","Yu Mincho","Noto Serif JP",serif;
    font-size:clamp(15px,2.1vw,19px);
    line-height:2.3; letter-spacing:0.18em;
    color:var(--ink); opacity:0.82;
  }
  .hint{
    position:absolute; bottom:9vh; left:0; right:0;
    text-align:center;
    font-size:11px; letter-spacing:0.32em;
    color:var(--ink-faint);
  }
  #touchHint{
    position:fixed; left:0; right:0; bottom:7vh; z-index:4;
    text-align:center; font-size:11px; letter-spacing:0.28em;
    color:var(--ink-faint);
    opacity:0; display:none;
    transition:opacity 1s ease;
    pointer-events:none;
  }

  #hud{
    position:fixed; left:18px; bottom:16px; z-index:5;
    font-size:11px; letter-spacing:0.18em; color:var(--ink-faint);
    display:none; user-select:none;
  }
  #muteBtn{
    position:fixed; right:18px; bottom:16px; z-index:5;
    width:34px; height:34px; border-radius:50%;
    border:1px solid rgba(42,42,40,0.22); background:transparent;
    color:var(--ink-faint); font-size:14px;
    display:none; align-items:center; justify-content:center;
    cursor:pointer;
  }
  #muteBtn:hover{ border-color:rgba(42,42,40,0.4); color:var(--ink); }
</style>
</head>
<body>

<canvas id="c"></canvas>

<div id="overlay">
  <div class="poem">
    <span>白い余白に、</span>
    <span>言葉が浮かぶ。</span>
    <span>偶然と必然が、</span>
    <span>束の間の詩を紡ぐ。</span>
  </div>
  <div class="hint">触れて、はじめる</div>
</div>

<div id="hud">世代 <span id="gen">0</span> ・ <span id="pop">0</span>語</div>
<button id="muteBtn" aria-label="音を切り替える">♪</button>
<div id="touchHint">言葉に触れると、増殖する</div>

<script>
(function(){
  "use strict";

  /* ---------- 語彙と構造 ---------- */
  const WORDS = [
    "雨","光","波","夜","海","影","炎","霧","闇","樹","苔","塵",
    "存在","記憶","沈黙","痕跡","螺旋","反復","境界","呼吸","残響","余白","無限","消滅","胎動","忘却",
    "構造","鏡像","器官","差異","閾","無意識","現存在","他者","言語","記述","祈り"
  ];
  
  // 自発的な変容に使う修飾語
  const PREFIXES = ["透明な","残酷な","不在の","静かな","構造としての","崩壊する","記述された"];
  const SUFFIXES = ["をし続ける私","の彼方","を反復する","と他者","の限界","の残響","をまなざす"];
  const CONNECTORS = ["の","と","へ","より","さえ","なす","たる","なる","は","、"];

  const rand = (a,b)=>a+Math.random()*(b-a);
  const pick = (arr)=>arr[(Math.random()*arr.length)|0];
  const clamp = (v,a,b)=>Math.max(a,Math.min(b,v));
  const reduceMotion = window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches;

  /* ---------- キャンバス ---------- */
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  let W = innerWidth, H = innerHeight;
  function resize(){
    const dpr = window.devicePixelRatio || 1;
    W = innerWidth; H = innerHeight;
    canvas.width = W*dpr; canvas.height = H*dpr;
    canvas.style.width = W+'px'; canvas.style.height = H+'px';
    ctx.setTransform(dpr,0,0,dpr,0,0);
  }
  window.addEventListener('resize', resize);
  resize();

  /* ---------- 語=個体 ---------- */
  let entities = [];
  let sparks = []; 
  let fusedCount = 0; 

  function makeEntity(text, x, y, fusions, vx, vy){
    const r = 20 + text.length*3.5 + fusions*4;
    return {
      text, x, y, r, fusions,
      vx: (vx!==undefined) ? vx : rand(-0.06,0.06),
      vy: (vy!==undefined) ? vy : -(rand(0.10,0.24) + fusions*0.015) * (reduceMotion?0.5:1),
      hue: rand(0,360),
      phase: rand(0, Math.PI*2),
      born: performance.now(),
      life: rand(11000,17000) + fusions*3200,
      alpha: 0,
      consumed: false, 
      popping: false, popStart: 0,
      lastInteraction: 0
    };
  }

  // 上・右・下・左の四辺から漂い込む
  function spawnPrimordial(){
    const margin = 60;
    const side = (Math.random() * 4) | 0; // 0:上 1:右 2:下 3:左
    let x, y, vx, vy;
    const speed = rand(0.12, 0.28) * (reduceMotion ? 0.5 : 1);
    const drift = rand(-0.12, 0.12);

    if (side === 0) {          // 上から
      x = rand(0, W);
      y = -margin - rand(0, 40);
      vx = drift;
      vy = speed;
    } else if (side === 1) {   // 右から
      x = W + margin + rand(0, 40);
      y = rand(0, H);
      vx = -speed;
      vy = drift;
    } else if (side === 2) {   // 下から
      x = rand(0, W);
      y = H + margin + rand(0, 40);
      vx = drift;
      vy = -speed;
    } else {                   // 左から
      x = -margin - rand(0, 40);
      y = rand(0, H);
      vx = speed;
      vy = drift;
    }

    entities.push(
      makeEntity(pick(WORDS), x, y, 0, vx, vy)
    );
  }

  function combine(a, b){
    const seed = (a.length + b.length + fusedCount) % 7;
    let text;
    switch(seed){
      case 0: text = a + b; break;
      case 1: text = a + "の" + b; break;
      case 2: text = a + "と" + b; break;
      case 3: text = b + a; break;
      case 4: text = a + "へ" + b; break;
      case 5: text = a + pick(CONNECTORS) + b; break;
      default: text = a + "、" + b;
    }
    if (Array.from(text).length > 16){
      text = a + b; 
    }
    return text;
  }

  /* ---------- 音 ---------- */
  let audioEnabled = false;
  let voices = [];
  function loadVoices(){ voices = speechSynthesis.getVoices ? speechSynthesis.getVoices() : []; }
  if ('speechSynthesis' in window){
    loadVoices();
    speechSynthesis.onvoiceschanged = loadVoices;
  }
  function sing(text){
    if (!audioEnabled || !('speechSynthesis' in window)) return;
    if (speechSynthesis.speaking) return; 
    try{
      const u = new SpeechSynthesisUtterance(text);
      const jaVoices = voices.filter(v => /ja/i.test(v.lang));
      const femaleOrAndrogynous = jaVoices.filter(v => !/Ichiro|Otoya|Keita|Osamu|Rokuro/i.test(v.name));
      const v = femaleOrAndrogynous.length ? pick(femaleOrAndrogynous) : (jaVoices.length ? pick(jaVoices) : null);
      
      if (v) { u.voice = v; u.lang = v.lang; } else { u.lang = 'ja-JP'; }
      
      u.pitch = rand(1.3, 1.7); 
      u.rate = rand(0.72, 1.02);
      u.volume = rand(0.35, 0.6);
      speechSynthesis.speak(u);
    }catch(e){ }
  }

  /* ---------- 描画 ---------- */
  function drawWord(e){
    const maxFontSize = clamp(e.r*0.4, 12, 24);
    const fontSize = Array.from(e.text).length > 8 ? maxFontSize * 0.8 : maxFontSize;
    
    ctx.save();
    ctx.globalAlpha = e.alpha;
    ctx.fillStyle = '#2a2a28'; 
    ctx.font = `${fontSize}px "Hiragino Mincho ProN","Yu Mincho","Noto Serif JP",serif`;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    
    ctx.shadowColor = 'rgba(251,250,246,0.8)';
    ctx.shadowBlur = 4;

    const chars = Array.from(e.text);
    const lineH = fontSize*1.12;
    const totalH = chars.length*lineH;
    let y0 = e.y - totalH/2 + lineH/2;
    chars.forEach((ch,i)=>{
      if (ch === '、' || ch === '。'){
        ctx.fillText(ch, e.x + fontSize*0.30, y0+i*lineH - fontSize*0.28);
      } else {
        ctx.fillText(ch, e.x, y0+i*lineH);
      }
    });
    ctx.restore();
  }

  /* ---------- 生成・干渉規則 ---------- */
  const MAX_POP = 60;
  let lastRuleTick = 0;
  let lastFusionTick = 0;
  let lastSpawnTick = 0;
  let lastMutationTick = 0;
  const genEl = document.getElementById('gen');
  const popEl = document.getElementById('pop');

  function applyLifeRules(now){
    const R = 130;
    for (let i=0;i<entities.length;i++){
      const e = entities[i];
      if (e.consumed || e.popping) continue;
      let neighbors = 0;
      for (let j=0;j<entities.length;j++){
        if (i===j) continue;
        const o = entities[j];
        if (o.consumed || o.popping) continue;
        const dx=e.x-o.x, dy=e.y-o.y;
        if (dx*dx+dy*dy < R*R) neighbors++;
      }
      const age = now - e.born;
      if (neighbors===0 && age>4000){
        e.life = Math.min(e.life, age+1400); 
      } else if (neighbors>=6){
        e.life = Math.min(e.life, age+1100); 
      } else if (neighbors>=1 && neighbors<=4){
        e.life += 260; 
      }
    }
  }

  function applyMutations(now){
    for (let i=0;i<entities.length;i++){
      const e = entities[i];
      if (e.consumed || e.popping || Array.from(e.text).length > 12) continue;
      
      const age = now - e.born;
      if (age > 2000 && Math.random() < 0.05) {
        let r = Math.random();
        if (r < 0.4) {
          e.text = pick(PREFIXES) + e.text;
        } else if (r < 0.8) {
          e.text = e.text + pick(SUFFIXES);
        } else {
          e.text = e.text + "、あるいは" + pick(WORDS);
        }
        
        e.r = 20 + e.text.length*3.5 + e.fusions*4;
        e.life += 3000;
        sing(e.text);
        
        for(let j=0; j<3; j++) {
           spawnSparks({text: e.text.charAt(j%e.text.length), x:e.x, y:e.y, hue:e.hue}, true);
        }
      }
    }
  }

  function applyInteractions(now){
    for (let i=0;i<entities.length;i++){
      const a = entities[i];
      if (a.consumed || a.popping) continue;
      if (now - a.lastInteraction < 2000) continue;

      for (let j=i+1;j<entities.length;j++){
        const b = entities[j];
        if (b.consumed || b.popping) continue;
        if (now - b.lastInteraction < 2000) continue;

        const dx=a.x-b.x, dy=a.y-b.y;
        const dist = Math.sqrt(dx*dx+dy*dy);
        
        if (dist < (a.r+b.r)*0.6){
          const randAction = Math.random();
          
          if (randAction < 0.3) {
            a.lastInteraction = now; 
            b.lastInteraction = now;
            
          } else if (randAction < 0.65) {
            if (a.text.length < 14 && b.text.length < 14) {
              const aChar = Array.from(a.text)[0];
              const bChar = Array.from(b.text)[0];
              a.text = a.text + "の" + bChar;
              b.text = aChar + "と" + b.text;
              
              a.lastInteraction = now; 
              b.lastInteraction = now;
              a.r += 5; b.r += 5;
              
              sing(a.text);
              spawnSparks({text: a.text.charAt(0), x:a.x, y:a.y, hue:a.hue}, true);
            }
            
          } else {
            if (a.fusions < 6 && b.fusions < 6) {
              const text = combine(a.text, b.text);
              if (Array.from(text).length <= 16 && entities.length < MAX_POP+4){
                a.consumed = true; b.consumed = true;
                const cvx = (a.vx+b.vx)/2;
                const cvy = (a.vy+b.vy)/2;
                const child = makeEntity(text, (a.x+b.x)/2, (a.y+b.y)/2, Math.max(a.fusions,b.fusions)+1, cvx, cvy);
                entities.push(child);
                fusedCount++;
                genEl.textContent = fusedCount;
                sing(text);
              }
            }
          }
          break;
        }
      }
    }
  }

  function dissolveWord(e){
    const now = performance.now();
    const chars = Array.from(e.text);
    chars.forEach(ch=>{
      const ang = rand(0, Math.PI*2);
      const speed = rand(0.04, 0.14);
      sparks.push({
        ch, x:e.x, y:e.y,
        vx: Math.cos(ang)*speed,
        vy: Math.sin(ang)*speed - 0.015,
        born: now,
        life: rand(2800, 4600),
        hue: e.hue
      });
    });
  }

  /* ---------- メインループ ---------- */
  let running = false;
  let lastFrame = 0;
  function frame(now){
    if (!running) return;
    const dt = Math.min(now-lastFrame, 48);
    lastFrame = now;

    if (now-lastSpawnTick > (reduceMotion?1400:850)){
      lastSpawnTick = now;
      if (entities.length < MAX_POP && Math.random()<0.7) spawnPrimordial();
    }
    if (now-lastRuleTick > 550){ lastRuleTick = now; applyLifeRules(now); }
    if (now-lastFusionTick > 300){ lastFusionTick = now; applyInteractions(now); }
    if (now-lastMutationTick > 1000){ lastMutationTick = now; applyMutations(now); }

    ctx.fillStyle = 'rgba(251,250,246,0.16)';
    ctx.fillRect(0,0,W,H);

    for (let i=entities.length-1;i>=0;i--){
      const e = entities[i];
      const age = now-e.born;

      if (e.consumed){
        entities.splice(i,1);
        continue;
      }
      if (!e.popping && age >=
