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
    <span>白い余白に</span>
    <span>言葉が浮かぶ</span>
    <span>偶然と必然が</span>
    <span>束の間の詩を紡ぐ</span>
  </div>
  <div class="hint">触れてはじめる</div>
</div>

<div id="hud">世代 <span id="gen">0</span> ・ <span id="pop">0</span>語</div>
<button id="muteBtn" aria-label="音を切り替える">♪</button>
<div id="touchHint">言葉に触れると増殖する</div>

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
  const CONNECTORS = ["の","と","へ","より","さえ","なす","たる","なる","は",""];

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
      lastInteraction: 0 // 連続干渉を防ぐためのクールダウンタイマー
    };
  }

  // 複数の方向から言葉が漂い込んでくる（下・上・左・右）
  function spawnPrimordial(){
    const edge = Math.floor(rand(0,4));
    const speed = rand(0.10,0.24) * (reduceMotion?0.5:1);
    let x,y,vx,vy;
    if (edge===0){       // 下から上へ
      x = rand(0,W); y = H+30;
      vx = rand(-0.06,0.06); vy = -speed;
    } else if (edge===1){ // 上から下へ
      x = rand(0,W); y = -30;
      vx = rand(-0.06,0.06); vy = speed;
    } else if (edge===2){ // 左から右へ
      x = -30; y = rand(0,H);
      vx = speed; vy = rand(-0.06,0.06);
    } else {               // 右から左へ
      x = W+30; y = rand(0,H);
      vx = -speed; vy = rand(-0.06,0.06);
    }
    entities.push(makeEntity(pick(WORDS), x, y, 0, vx, vy));
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
      default: text = a + "" + b;
    }
    if (Array.from(text).length > 16){
      text = a + b; 
    }
    return text;
  }

  /* ---------- 音（中性的・女性的な声への絞り込み） ---------- */
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
    // 文字数が長い場合は少しフォントを小さくして調整
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
      // 「」「」は横書き用グリフの墨が字面の左下に寄っているため
      // そのまま縦に積むと列全体から見て左にはみ出して見える
      // 縦組みの慣習（右肩に寄せる）に近づくよう座標を補正する
      if (ch === '' || ch === ''){
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

  // 自発的な変容（新しい機能）
  function applyMutations(now){
    for (let i=0;i<entities.length;i++){
      const e = entities[i];
      if (e.consumed || e.popping || Array.from(e.text).length > 12) continue;
      
      const age = now - e.born;
      if (age > 2000 && Math.random() < 0.05) { // 稀に自発的に変容する
        let r = Math.random();
        if (r < 0.4) {
          e.text = pick(PREFIXES) + e.text;
        } else if (r < 0.8) {
          e.text = e.text + pick(SUFFIXES);
        } else {
          e.text = e.text + "あるいは" + pick(WORDS);
        }
        
        e.r = 20 + e.text.length*3.5 + e.fusions*4; // 半径を再計算
        e.life += 3000; // 変異すると寿命が延びる
        sing(e.text); // 声に出す
        
        // 変容の証として小さな破片を散らす
        for(let j=0; j<3; j++) {
           spawnSparks({text: e.text.charAt(j%e.text.length), x:e.x, y:e.y, hue:e.hue}, true);
        }
      }
    }
  }

  // 衝突時の多様な相互作用
  function applyInteractions(now){
    for (let i=0;i<entities.length;i++){
      const a = entities[i];
      if (a.consumed || a.popping) continue;
      if (now - a.lastInteraction < 2000) continue; // クールダウン中

      for (let j=i+1;j<entities.length;j++){
        const b = entities[j];
        if (b.consumed || b.popping) continue;
        if (now - b.lastInteraction < 2000) continue;

        const dx=a.x-b.x, dy=a.y-b.y;
        const dist = Math.sqrt(dx*dx+dy*dy);
        
        if (dist < (a.r+b.r)*0.6){
          const randAction = Math.random();
          
          if (randAction < 0.3) {
            // 【パターン1：すれ違い】何もしない影響を受けずに通り過ぎる
            a.lastInteraction = now; 
            b.lastInteraction = now;
            
          } else if (randAction < 0.65) {
            // 【パターン2：相互干渉】互いの構造を取り込み合いそれぞれが変質して生き残る
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
            // 【パターン3：統合】2つが消滅し新たな1つが生まれる（従来通り）
            if (a.fusions < 6 && b.fusions < 6) {
              const text = combine(a.text, b.text);
              if (Array.from(text).length <= 16 && entities.length < MAX_POP+4){
                a.consumed = true; b.consumed = true;
                // 生まれる子は親たちの漂う方向を引き継ぐ（急に上向きへ切り替わらないように）
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
          break; // 1フレームで1回の干渉のみ
        }
      }
    }
  }

  /* ---------- 寿命が尽きた言葉の霧散 ---------- */
  // 崩れるように文字がバラバラになりゆっくり霧散して消える
  function dissolveWord(e){
    const now = performance.now();
    const chars = Array.from(e.text);
    chars.forEach(ch=>{
      const ang = rand(0, Math.PI*2);
      const speed = rand(0.04, 0.14); // 霧のようにゆっくり漂う
      sparks.push({
        ch, x:e.x, y:e.y,
        vx: Math.cos(ang)*speed,
        vy: Math.sin(ang)*speed - 0.015, // わずかに立ちのぼる
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
    if (now-lastMutationTick > 1000){ lastMutationTick = now; applyMutations(now); } // 毎秒変異チェック

    ctx.fillStyle = 'rgba(251,250,246,0.16)';
    ctx.fillRect(0,0,W,H);

    for (let i=entities.length-1;i>=0;i--){
      const e = entities[i];
      const age = now-e.born;

      if (e.consumed){
        entities.splice(i,1);
        continue;
      }
      if (!e.popping && age >= e.life){
        e.popping = true; e.popStart = now;
        dissolveWord(e); // 崩れて霧散する破片を生成
      }
      if (e.popping){
        const t = (now-e.popStart)/750; // ゆっくり時間をかけて消える
        if (t>=1){ entities.splice(i,1); continue; }
        e.alpha = (1-t)*0.75;
        drawWord(e);
        continue;
      }

      e.x += e.vx*dt*0.06 + Math.sin(now/1500 + e.phase)*0.12;
      e.y += e.vy*dt*0.06;

      const fadeIn = clamp(age/700, 0, 1);
      const fadeOut = clamp((e.life-age)/1500, 0, 1);
      e.alpha = Math.min(fadeIn, fadeOut) * 0.85;

      if (e.y < -e.r-40 || e.y > H+e.r+40 || e.x < -e.r-60 || e.x > W+e.r+60){
        entities.splice(i,1); continue;
      }

      drawWord(e);
    }

    // 分裂した言葉の破片
    for (let i=sparks.length-1;i>=0;i--){
      const s = sparks[i];
      const age = now - s.born;
      if (age >= s.life){ sparks.splice(i,1); continue; }
      s.x += s.vx*dt*0.05;
      s.y += s.vy*dt*0.05;
      s.vx *= 0.98; s.vy = s.vy*0.98 - 0.001; 
      
      ctx.save();
      ctx.globalAlpha = clamp(1 - (age / s.life), 0, 1) * 0.85;
      ctx.fillStyle = `hsla(${s.hue},30%,40%,1)`;
      ctx.font = '14px "Hiragino Mincho ProN","Yu Mincho","Noto Serif JP",serif';
      ctx.textAlign='center'; ctx.textBaseline='middle';
      if (s.ch === '' || s.ch === ''){
        ctx.fillText(s.ch, s.x + 4.2, s.y - 3.9);
      } else {
        ctx.fillText(s.ch, s.x, s.y);
      }
      ctx.restore();
    }

    popEl.textContent = entities.length;
    requestAnimationFrame(frame);
  }

  /* ---------- 接触と増殖 ---------- */
  const PROLIFERATE_CHANCE = 0.85; 

  function splitWord(text){
    const chars = Array.from(text);
    if (chars.length <= 1) return [text, text]; 
    const parts = [];
    let i = 0;
    while (i < chars.length){
      const len = Math.min(chars.length-i, (Math.random()<0.5) ? 1 : 2);
      parts.push(chars.slice(i,i+len).join(''));
      i += len;
    }
    return parts;
  }

  // smallSpark: 変異時などの小さなエフェクト用フラグ
  function spawnSparks(e, smallSpark = false){
    const now = performance.now();
    const chars = Array.from(e.text || "　");
    chars.forEach(ch=>{
      const ang = rand(0,Math.PI*2);
      const speed = smallSpark ? rand(0.1, 0.5) : rand(0.3,1.2);
      const lifeTime = smallSpark ? rand(1000, 2000) : rand(3000,7000);
      sparks.push({
        ch, x:e.x, y:e.y,
        vx:Math.cos(ang)*speed, vy:Math.sin(ang)*speed,
        born:now, 
        life:lifeTime, 
        hue:e.hue || rand(0,360)
      });
    });
  }

  function scatterWord(e){
    e.consumed = true; 
    spawnSparks(e);
    
    if (Math.random() < PROLIFERATE_CHANCE){
      const frags = splitWord(e.text).slice(0,5); 
      frags.forEach(frag=>{
        if (entities.length >= MAX_POP+15) return;
        const ang = rand(0,Math.PI*2);
        const child = makeEntity(frag, e.x, e.y, Math.max(0, e.fusions-1));
        child.vx = Math.cos(ang)*rand(0.2,0.6);
        child.vy = -rand(0.05,0.15);
        child.life = rand(15000, 22000); 
        entities.push(child);
        fusedCount++;
      });
      genEl.textContent = fusedCount;
    }
  }

  function handlePointer(ev){
    if (!running) return;
    const rect = canvas.getBoundingClientRect();
    const x = ev.clientX - rect.left;
    const y = ev.clientY - rect.top;
    for (let i=entities.length-1;i>=0;i--){
      const e = entities[i];
      if (e.consumed || e.popping) continue;
      const dx=x-e.x, dy=y-e.y;
      if (dx*dx+dy*dy <= (e.r*1.2)*(e.r*1.2)){ 
        scatterWord(e); break; 
      }
    }
  }
  canvas.addEventListener('pointerdown', handlePointer);

  /* ---------- 開始 ---------- */
  const overlay = document.getElementById('overlay');
  const hud = document.getElementById('hud');
  const muteBtn = document.getElementById('muteBtn');
  let started = false;

  function start(){
    if (started) return;
    started = true;
    audioEnabled = true;
    overlay.classList.add('hidden');
    hud.style.display = 'block';
    muteBtn.style.display = 'flex';
    running = true;
    lastFrame = performance.now();
    lastSpawnTick = lastFrame; lastRuleTick = lastFrame; lastFusionTick = lastFrame; lastMutationTick = lastFrame;
    for (let i=0;i<8;i++) spawnPrimordial(); 
    requestAnimationFrame(frame);

    const touchHint = document.getElementById('touchHint');
    touchHint.style.display = 'block';
    requestAnimationFrame(()=>{ touchHint.style.opacity = '1'; });
    setTimeout(()=>{
      touchHint.style.opacity = '0';
      setTimeout(()=>{ touchHint.style.display = 'none'; }, 1000);
    }, 5000);
  }
  overlay.addEventListener('click', start);
  overlay.addEventListener('touchstart', start, {passive:true});

  muteBtn.addEventListener('click', ()=>{
    audioEnabled = !audioEnabled;
    muteBtn.textContent = audioEnabled ? '♪' : '×';
    if (!audioEnabled && 'speechSynthesis' in window) speechSynthesis.cancel();
  });

})();
</script>
</body>
</html>
