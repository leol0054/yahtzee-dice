<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>快艇骰子（Yahtzee）</title>
<style>
  :root{
    --bg:#0f1220; --panel:#161a2e; --accent:#4f7cff; --text:#eef1ff; --muted:#a8b0d9;
    --good:#2ecc71; --warn:#f39c12; --bad:#e74c3c;
  }
  html,body{
    margin:0;
    padding:0;
    min-height:100%;
    background:linear-gradient(180deg,#0b0e1a,#0f1220 30%,#0f1220);
    color:var(--text);
    font-family:system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans", "PingFang TC", "Microsoft JhengHei", sans-serif;
    display:block;       /* 避免垂直置中造成裁切 */
    overflow:auto;       /* 允許捲動 */
  }
  .app{
    width:min(1100px,96vw);
    margin:24px auto;    /* 水平置中 */
    display:grid; grid-template-columns: 1.4fr 1fr; gap:20px;
  }
  .panel{
    background:var(--panel); border-radius:16px; box-shadow:0 10px 30px rgba(0,0,0,0.35);
    padding:18px;
  }
  h1{font-size:22px; margin:0 0 8px}
  .sub{color:var(--muted); font-size:13px; margin-bottom:16px}
  .dice-area{
    display:grid; grid-template-columns: repeat(5, 1fr); gap:12px; margin-bottom:14px;
  }
  .die{
    aspect-ratio:1; border-radius:14px; background:#12162a; border:2px solid transparent;
    display:grid; place-items:center; cursor:pointer; position:relative;
    transition:transform .12s ease, border-color .12s ease, background .12s ease;
  }
  .die:hover{transform:translateY(-2px)}
  .die.held{border-color:var(--accent); background:#10183a}
  .pip{
    width:12px; height:12px; background:#fff; border-radius:50%; box-shadow:0 1px 0 rgba(0,0,0,.2);
  }
  .pip-wrap{
    width:70%; height:70%; display:grid; grid-template-columns:repeat(3,1fr); grid-template-rows:repeat(3,1fr); gap:8px;
  }
  .controls{display:flex; align-items:center; gap:12px; margin:10px 0 2px;}
  .btn{
    background:var(--accent); color:#fff; border:none; border-radius:10px; padding:10px 14px;
    cursor:pointer; font-weight:600; box-shadow:0 8px 20px rgba(79,124,255,.35);
    transition:filter .12s ease, transform .06s ease;
  }
  .btn:disabled{filter:grayscale(80%) brightness(.7); cursor:not-allowed; box-shadow:none}
  .btn:active{transform:translateY(1px)}
  .badge{display:inline-block; background:#0e1429; border:1px solid #28335a; border-radius:8px; padding:4px 8px; margin-left:6px}
  .hint{color:var(--muted); font-size:12px; margin-left:8px}
  .meta{margin-top:8px; color:var(--muted); font-size:14px}
  .section-title{margin-top:4px; margin-bottom:10px; font-size:16px; border-bottom:1px dashed #28335a; padding-bottom:6px}

  .score-row{
    display:grid; grid-template-columns: 1.6fr .9fr .9fr; gap:10px; align-items:center;
    background:#12162a; padding:10px; border-radius:12px; margin-bottom:8px;
  }
  .score-row.taken{opacity:.6}
  .cat{font-weight:700; white-space:nowrap; overflow:hidden; text-overflow:ellipsis}
  .val{
    background:#0e1429; border:1px solid #28335a; border-radius:10px; padding:8px; text-align:center;
    min-height:34px; display:flex; align-items:center; justify-content:center; font-weight:700;
  }
  .pickable{cursor:pointer; position:relative; outline:2px solid transparent; transition:outline-color .1s ease, background .1s ease}
  .pickable:hover{outline-color:var(--accent); background:#0d1534}
  .pickable.can{color:var(--good)}
  .pickable.zero{color:var(--bad)}

  .totals{
    display:grid; grid-template-columns:1fr 1fr; gap:10px; margin-top:8px;
  }
  .total-box{
    background:#12162a; border-radius:12px; padding:10px; display:flex; flex-direction:column; gap:6px;
  }
  .big{font-size:20px; font-weight:800}
  .footer{color:var(--muted); font-size:13px; margin-top:8px;}
  .end-screen{display:none; margin-top:10px; padding:14px; border-radius:12px; background:#12162a;}
  .end-screen.show{display:block}
  .grid-2{display:grid; grid-template-columns:1fr 1fr; gap:12px}
</style>
</head>
<body>
  <div class="app">
    <div class="panel">
      <h1>快艇骰子（Yahtzee）</h1>
      <div class="sub">每回合最多擲骰 3 次，點擊骰子可保留。選擇一個分類記分，共 13 回合。</div>

      <div class="dice-area" id="diceArea"></div>

      <div class="controls">
        <button class="btn" id="rollBtn">擲骰</button>
        <button class="btn" id="resetHoldBtn">取消全部保留</button>
        <span class="badge" id="rollsInfo">本回合剩餘擲骰：3</span>
        <span class="hint" id="roundInfo">第 1 / 13 回合</span>
      </div>

      <div class="meta" id="statusMsg">提示：點擊想保留的骰子，再擲骰。</div>

      <div class="end-screen" id="endScreen">
        <div class="grid-2">
          <div>
            <div style="font-weight:800; font-size:18px">遊戲結束 🎉</div>
            <div id="finalBreakdown" style="margin-top:6px"></div>
          </div>
          <div style="display:flex; align-items:center; gap:10px; justify-content:flex-end">
            <button class="btn" id="newGameBtn">重新開始</button>
          </div>
        </div>
      </div>
    </div>

    <div class="panel">
      <div class="section-title">計分表</div>
      <div id="scorecard"></div>

      <div class="totals">
        <div class="total-box">
          <div><strong>上半部合計：</strong><span id="upperSum">0</span> <span class="hint">（1~6）</span></div>
          <div><strong>上半部獎勵：</strong><span id="upperBonus">0</span> <span class="hint">達 63 分加 35 分</span></div>
        </div>
        <div class="total-box">
          <div><strong>下半部合計：</strong><span id="lowerSum">0</span></div>
          <div class="big"><strong>總分：</strong><span id="totalSum">0</span></div>
        </div>
      </div>

      <div class="footer">
        規則簡述：<br>
        - 三條、四條：所有骰子之總和。<br>
        - 葫蘆：固定 25 分。<br>
        - 小順（四連）：固定 30 分。<br>
        - 大順（五連）：固定 40 分。<br>
        - 快艇（五條）：固定 50 分。<br>
        - 雜牌（機會）：所有骰子之總和。<br>
        提示：只有尚未記分的分類可選；若不符合，可記 0 分。
      </div>
    </div>
  </div>

<script>
(function(){
  // --- Game State ---
  const DICE_COUNT = 5;
  const MAX_ROLLS = 3;
  const ROUNDS = 13;

  let dice = [1,1,1,1,1];
  let held = [false,false,false,false,false];
  let rollsLeft = MAX_ROLLS;
  let round = 1;

  const categories = [
    // Upper
    {key:'ones', name:'一點（1）', sect:'upper', desc:'加總所有 1', calc:(d)=>sumOf(d,1)},
    {key:'twos', name:'二點（2）', sect:'upper', desc:'加總所有 2', calc:(d)=>sumOf(d,2)},
    {key:'threes', name:'三點（3）', sect:'upper', desc:'加總所有 3', calc:(d)=>sumOf(d,3)},
    {key:'fours', name:'四點（4）', sect:'upper', desc:'加總所有 4', calc:(d)=>sumOf(d,4)},
    {key:'fives', name:'五點（5）', sect:'upper', desc:'加總所有 5', calc:(d)=>sumOf(d,5)},
    {key:'sixes', name:'六點（6）', sect:'upper', desc:'加總所有 6', calc:(d)=>sumOf(d,6)},
    // Lower
    {key:'threeKind', name:'三條', sect:'lower', desc:'有任一點數 ≥3 顆，得所有骰子總和', calc:(d)=>hasKind(d,3)?sumAll(d):0},
    {key:'fourKind', name:'四條', sect:'lower', desc:'有任一點數 ≥4 顆，得所有骰子總和', calc:(d)=>hasKind(d,4)?sumAll(d):0},
    {key:'fullHouse', name:'葫蘆', sect:'lower', desc:'3 顆 + 2 顆，固定 25 分', calc:(d)=>isFullHouse(d)?25:0},
    {key:'smallStraight', name:'小順', sect:'lower', desc:'任一四連（1-4, 2-5, 3-6），固定 30 分', calc:(d)=>hasSmallStraight(d)?30:0},
    {key:'largeStraight', name:'大順', sect:'lower', desc:'五連（1-5 或 2-6），固定 40 分', calc:(d)=>hasLargeStraight(d)?40:0},
    {key:'yahtzee', name:'快艇（五條）', sect:'lower', desc:'五顆相同，固定 50 分', calc:(d)=>hasKind(d,5)?50:0},
    {key:'chance', name:'雜牌（機會）', sect:'lower', desc:'所有骰子總和', calc:(d)=>sumAll(d)},
  ];

  const taken = Object.fromEntries(categories.map(c=>[c.key,null])); // null or number

  // --- DOM ---
  const diceArea = document.getElementById('diceArea');
  const rollBtn = document.getElementById('rollBtn');
  const resetHoldBtn = document.getElementById('resetHoldBtn');
  const rollsInfo = document.getElementById('rollsInfo');
  const roundInfo = document.getElementById('roundInfo');
  const statusMsg = document.getElementById('statusMsg');
  const scorecardRoot = document.getElementById('scorecard');
  const upperSumEl = document.getElementById('upperSum');
  const upperBonusEl = document.getElementById('upperBonus');
  const lowerSumEl = document.getElementById('lowerSum');
  const totalSumEl = document.getElementById('totalSum');
  const endScreen = document.getElementById('endScreen');
  const finalBreakdown = document.getElementById('finalBreakdown');
  const newGameBtn = document.getElementById('newGameBtn');

  // --- Helpers (logic) ---
  function sumAll(d){ return d.reduce((a,b)=>a+b,0); }
  function sumOf(d,face){ return d.filter(x=>x===face).reduce((a,b)=>a+b,0); }
  function counts(d){
    const cnt = {1:0,2:0,3:0,4:0,5:0,6:0};
    d.forEach(x=>cnt[x]++);
    return cnt;
  }
  function hasKind(d, n){
    const c = counts(d);
    return Object.values(c).some(v=>v>=n);
  }
  function isFullHouse(d){
    const c = Object.values(counts(d)).filter(v=>v>0).sort((a,b)=>b-a);
    return c.length===2 && c[0]===3 && c[1]===2;
  }
  function hasSmallStraight(d){
    const s = new Set(d);
    const seqs = [
      [1,2,3,4],
      [2,3,4,5],
      [3,4,5,6],
    ];
    return seqs.some(seq=>seq.every(x=>s.has(x)));
  }
  function hasLargeStraight(d){
    const s = new Set(d);
    const a = [1,2,3,4,5].every(x=>s.has(x));
    const b = [2,3,4,5,6].every(x=>s.has(x));
    return a || b;
  }

  // --- UI Dice ---
  function renderDice(){
    diceArea.innerHTML = '';
    dice.forEach((val, i)=>{
      const die = document.createElement('div');
      die.className = 'die' + (held[i] ? ' held' : '');
      die.title = held[i] ? '已保留（點擊取消保留）' : '點擊保留此骰';
      die.addEventListener('click', ()=>{
        if (rollsLeft===MAX_ROLLS) return; // 未擲骰前不能保留
        held[i] = !held[i];
        renderDice();
      });
      const pipWrap = renderPips(val);
      die.appendChild(pipWrap);
      diceArea.appendChild(die);
    });
  }
  function renderPips(val){
    const wrap = document.createElement('div');
    wrap.className = 'pip-wrap';
    const positions = {
      1:[4],
      2:[0,8],
      3:[0,4,8],
      4:[0,2,6,8],
      5:[0,2,4,6,8],
      6:[0,2,3,5,6,8],
    };
    for(let i=0;i<9;i++){
      const cell = document.createElement('div');
      if(positions[val].includes(i)){
        const pip = document.createElement('div');
        pip.className='pip';
        cell.style.display='grid';
        cell.style.placeItems='center';
        cell.appendChild(pip);
      }
      wrap.appendChild(cell);
    }
    return wrap;
  }

  // --- Roll logic ---
  function roll(){
    if (rollsLeft<=0) return;
    for(let i=0;i<DICE_COUNT;i++){
      if (!held[i]) dice[i] = randDie();
    }
    rollsLeft--;
    statusMsg.textContent = rollsLeft>0 ? '可繼續擲骰或選擇分類記分。' : '本回合擲骰次數用盡，請選擇一個分類記分。';
    updateMeta();
    renderDice();
    renderScorecard(); // update possible scores
    updateTotals();
    checkEnd();
  }
  function randDie(){ return 1 + Math.floor(Math.random()*6); }

  // --- Scorecard ---
  function renderScorecard(){
    scorecardRoot.innerHTML = '';
    categories.forEach(cat=>{
      const row = document.createElement('div');
      row.className = 'score-row' + (taken[cat.key]!==null ? ' taken':'');

      // 名稱（介紹改用 title 提示）
      const info = document.createElement('div');
      const name = document.createElement('div');
      name.className = 'cat';
      name.textContent = cat.name;
      name.title = cat.desc;   // 滑鼠提示顯示介紹
      info.appendChild(name);

      // 當前可能分數
      const possible = document.createElement('div');
      possible.className = 'val';
      let possibleVal = cat.calc(dice);
      possible.textContent = taken[cat.key]!==null ? '—' : String(possibleVal);

      // 已選分數或可選分數
      const val = document.createElement('div');
      val.className = 'val';
      if (taken[cat.key]!==null){
        val.textContent = String(taken[cat.key]);
      } else {
        val.classList.add('pickable');
        if (rollsLeft<MAX_ROLLS){ // 至少擲過一次才能記分
          const good = possibleVal>0;
          val.classList.add(good ? 'can' : 'zero');
          val.title = '點擊將此分類記分為 ' + possibleVal + ' 分';
          val.addEventListener('click', ()=>pickCategory(cat.key, possibleVal));
          val.textContent = possibleVal + '（選）';
        } else {
          val.title = '尚未擲骰，不能記分';
          val.textContent = '—';
        }
      }

      row.appendChild(info);
      row.appendChild(possible);
      row.appendChild(val);
      scorecardRoot.appendChild(row);
    });
  }

  function pickCategory(key, val){
    if (taken[key]!==null) return;
    if (rollsLeft===MAX_ROLLS) return; // 必須先擲骰
    taken[key] = val;
    // 進入下一回合或結束
    if (round < ROUNDS) {
      round++;
      rollsLeft = MAX_ROLLS;
      held = [false,false,false,false,false];
      statusMsg.textContent = '新回合：請擲骰開始。';
      updateMeta();
      renderDice();
      renderScorecard();
      updateTotals();
      checkEnd();
    } else {
      updateTotals();
      renderScorecard();
      endGame();
    }
  }

  // --- Totals ---
  function updateTotals(){
    const upperKeys = ['ones','twos','threes','fours','fives','sixes'];
    const lowerKeys = ['threeKind','fourKind','fullHouse','smallStraight','largeStraight','yahtzee','chance'];
    const upperSum = sumTaken(upperKeys);
    const lowerSum = sumTaken(lowerKeys);
    const upperBonus = upperSum>=63 ? 35 : 0;
    upperSumEl.textContent = upperSum;
    upperBonusEl.textContent = upperBonus;
    lowerSumEl.textContent = lowerSum;
    totalSumEl.textContent = upperSum + upperBonus + lowerSum;
  }
  function sumTaken(keys){
    return keys.reduce((acc,k)=>acc + (taken[k]!==null ? taken[k] : 0), 0);
  }

  // --- End ---
  function checkEnd(){
    const remaining = Object.values(taken).filter(v=>v===null).length;
    if (remaining===0){
      endGame();
    }
  }
  function endGame(){
    endScreen.classList.add('show');
    rollBtn.disabled = true;
    resetHoldBtn.disabled = true;
    statusMsg.textContent = '遊戲結束。';
    const upperKeys = ['ones','twos','threes','fours','fives','sixes'];
    const lowerKeys = ['threeKind','fourKind','fullHouse','smallStraight','largeStraight','yahtzee','chance'];
    const upperSum = sumTaken(upperKeys);
    const upperBonus = upperSum>=63 ? 35 : 0;
    const lowerSum = sumTaken(lowerKeys);
    const total = upperSum + upperBonus + lowerSum;

    finalBreakdown.innerHTML =
      `<div><strong>上半部：</strong>${upperSum}，<strong>獎勵：</strong>${upperBonus}</div>
       <div><strong>下半部：</strong>${lowerSum}</div>
       <div style="margin-top:6px" class="big"><strong>總分：</strong>${total}</div>`;
  }

  // --- Meta/UI ---
  function updateMeta(){
    rollsInfo.textContent = '本回合剩餘擲骰：' + rollsLeft;
    roundInfo.textContent = `第 ${round} / ${ROUNDS} 回合`;
    rollBtn.disabled = rollsLeft<=0;
  }

  function resetHolds(){
    held = [false,false,false,false,false];
    renderDice();
  }

  function newGame(){
    dice = [1,1,1,1,1];
    held = [false,false,false,false,false];
    rollsLeft = MAX_ROLLS;
    round = 1;
    categories.forEach(c=>taken[c.key]=null);
    statusMsg.textContent = '新遊戲開始：請擲骰。';
    endScreen.classList.remove('show');
    rollBtn.disabled = false;
    resetHoldBtn.disabled = false;
    updateMeta();
    renderDice();
    renderScorecard();
    updateTotals();
  }

  // --- Init ---
  rollBtn.addEventListener('click', roll);
  resetHoldBtn.addEventListener('click', resetHolds);
  newGameBtn.addEventListener('click', newGame);

  // Start
  updateMeta();
  renderDice();
  renderScorecard();
  updateTotals();
})();
</script>
</body>
</html>
