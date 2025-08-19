<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>十人副本报名接龙（网页表单）</title>
  <style>
    :root { --bg:#0b0f14; --card:#121821; --muted:#9fb0c0; --text:#e7eef5; --accent:#5bb7ff; --danger:#ff6b6b; --ok:#6bffaa; }
    body { margin:0; background:var(--bg); color:var(--text); font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial; }
    .wrap { max-width: 1050px; margin: 32px auto; padding: 0 16px; }
    .title { font-size: 26px; font-weight: 800; display:flex; align-items:center; gap:10px; }
    .muted { color: var(--muted); }
    .row { display:flex; flex-wrap:wrap; gap:12px; align-items:flex-end; }
    .card { background: var(--card); border: 1px solid #1e2733; border-radius: 14px; padding: 18px; box-shadow: 0 10px 30px rgba(0,0,0,.25); }
    .controls { display:grid; grid-template-columns: 1.2fr 0.8fr 1fr auto; gap:12px; }
    .controls input, .controls select, .controls button { height: 44px; border-radius: 10px; border: 1px solid #223044; background:#0f151e; color: var(--text); padding: 0 12px; font-size: 15px; }
    .controls button { background: linear-gradient(180deg,#1a2940,#152235); border-color:#2b3b55; cursor:pointer; }
    .controls button:hover { filter: brightness(1.06); }
    .btns { display:flex; gap:10px; flex-wrap:wrap; }
    .btn { padding:10px 14px; border-radius:10px; border:1px solid #2b3b55; background:#0f151e; color:var(--text); cursor:pointer; font-size:14px; }
    .btn:hover { filter:brightness(1.1); }
    .btn.danger { border-color:#5a2a2a; background:#1b1010; color:#ffc8c8; }
    .btn.ok { border-color:#2a5a3c; background:#0f1b15; color:#d4ffe8; }
    .grid { display:grid; gap:16px; grid-template-columns: repeat(2, 1fr); margin-top:16px; }
    @media (max-width: 900px){ .controls{grid-template-columns:1fr 1fr 1fr 1fr} .grid{grid-template-columns:1fr} }
    .slot { display:grid; grid-template-rows:auto auto 1fr; gap:10px; }
    .slot h3 { margin:0; font-size:18px; display:flex; align-items:center; justify-content:space-between; }
    .pill { font-size:12px; padding:2px 8px; border-radius:999px; border:1px solid #2b3b55; color:var(--muted); }
    .role-head { display:flex; gap:6px; align-items:center; font-weight:700; }
    .role { border:1px dashed #2a3750; border-radius:12px; padding:10px; display:grid; gap:8px; }
    .names { display:flex; flex-wrap:wrap; gap:8px; }
    .tag { background:#0b121b; border:1px solid #223044; border-radius:999px; padding:6px 10px; display:flex; gap:8px; align-items:center; }
    .tag button { border:none; background:transparent; color:var(--danger); font-weight:900; cursor:pointer; }
    .counts { display:flex; gap:8px; align-items:center; }
    .count { font-size:12px; color:var(--muted); }
    .sep { height:1px; background:#1f2937; margin:6px 0; opacity:.6; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="title">十人副本报名接龙 <span class="muted">（ET，美东）</span></div>

    <div class="card" style="margin-top:14px;">
      <div class="controls">
        <input id="nameInput" placeholder="角色名 / ID" />
        <select id="roleSelect">
          <option value="T">T</option>
          <option value="DPS">DPS</option>
          <option value="奶">奶</option>
        </select>
        <select id="slotSelect"></select>
        <button id="addBtn">报名</button>
      </div>
      <div class="row" style="margin-top:12px;">
        <div class="btns">
          <button class="btn ok" id="exportCsv">导出 CSV</button>
          <button class="btn" id="exportJson">导出 JSON</button>
          <button class="btn" id="copyText">复制接龙文本</button>
          <button class="btn danger" id="resetAll">清空所有报名</button>
        </div>
        <div class="counts" id="summary"></div>
      </div>
    </div>

    <div class="grid" id="slotsGrid"></div>
  </div>

  <script>
    const TIME_SLOTS = [
      { key: '周五 22:00 (ET) 末班车', label: '周五 22:00 (ET) 末班车' },
      { key: '周日 21:00 (ET)', label: '周日 21:00 (ET)' },
      { key: '周日 22:30 (ET)', label: '周日 22:30 (ET)' },
      { key: '周日 23:00 (ET)', label: '周日 23:00 (ET)' },
      { key: '周一 22:00 (ET)', label: '周一 22:00 (ET)' },
    ];

    const ROLES = ['T', 'DPS', '奶'];
    const CAPACITY = { T: 1, DPS: 7, '奶': 2 };

    const LS_KEY = 'raid_signup_web_v2';
    function emptyRoster(){ const r = {}; TIME_SLOTS.forEach(s => { r[s.key] = { T: [], DPS: [], '奶': [] }; }); return r; }
    function load(){ try{ const j = JSON.parse(localStorage.getItem(LS_KEY)||'null'); if(!j) return emptyRoster();
      TIME_SLOTS.forEach(s=>{ j[s.key] ||= { T:[], DPS:[], '奶':[] }; ROLES.forEach(role=>{ j[s.key][role] ||= []; });});
      return j; }catch{ return emptyRoster(); } }
    function save(){ localStorage.setItem(LS_KEY, JSON.stringify(state)); }

    let state = load();

    const slotSelect = document.getElementById('slotSelect');
    const roleSelect = document.getElementById('roleSelect');
    const nameInput = document.getElementById('nameInput');
    const addBtn = document.getElementById('addBtn');
    const grid = document.getElementById('slotsGrid');
    const summary = document.getElementById('summary');

    TIME_SLOTS.forEach(s => {
      const opt = document.createElement('option'); opt.value = s.key; opt.textContent = s.label; slotSelect.appendChild(opt);
    });

    function render(){
      grid.innerHTML = '';
      let total = 0, taken = 0;
      TIME_SLOTS.forEach(s => {
        const { T, DPS, 奶 } = state[s.key];
        total += CAPACITY.T + CAPACITY.DPS + CAPACITY['奶'];
        taken += T.length + DPS.length + 奶.length;
        const slotDiv = document.createElement('div'); slotDiv.className = 'card slot';
        slotDiv.innerHTML = `
          <h3><span>${s.label}</span>
            <span class="pill">剩余 ${CAPACITY.T - T.length + CAPACITY.DPS - DPS.length + CAPACITY['奶'] - 奶.length}</span>
          </h3>
          <div class="role">
            <div class="role-head">T <span class="count">(${T.length}/${CAPACITY.T})</span></div>
            <div class="names">${T.map(n=>tag(n,s.key,'T')).join('')||'<span class="muted">—</span>'}</div>
            <div class="sep"></div>
            <div class="role-head">DPS <span class="count">(${DPS.length}/${CAPACITY.DPS})</span></div>
            <div class="names">${DPS.map(n=>tag(n,s.key,'DPS')).join('')||'<span class="muted">—</span>'}</div>
            <div class="sep"></div>
            <div class="role-head">奶 <span class="count">(${奶.length}/${CAPACITY['奶']})</span></div>
            <div class="names">${奶.map(n=>tag(n,s.key,'奶')).join('')||'<span class="muted">—</span>'}</div>
          </div>`;
        grid.appendChild(slotDiv);
      });
      summary.textContent = `总名额 ${total}，已报名 ${taken}，剩余 ${total - taken}`;
      save();
    }

    function tag(name, slotKey, role){ return `<span class="tag">${escapeHtml(name)} <button onclick="removeName('${slotKey}','${role}','${encodeURIComponent(name)}')">×</button></span>`; }
    function removeName(slotKey,role,name){ const arr = state[slotKey][role]; const idx = arr.indexOf(decodeURIComponent(name)); if(idx>-1){ arr.splice(idx,1); render(); } }

    addBtn.addEventListener('click', ()=>{ const name = nameInput.value.trim(); if(!name){ alert('请输入角色名/ID'); return; }
      const role = roleSelect.value; const slotKey = slotSelect.value;
      for(const r of ROLES){ if(state[slotKey][r].includes(name)){ alert(`该ID已在此时段以 ${r} 报名`); return; } }
      if(state[slotKey][role].length >= CAPACITY[role]){ alert(`${slotKey} 的 ${role} 名额已满`); return; }
      state[slotKey][role].push(name); nameInput.value=''; render();
    });

    document.getElementById('exportCsv').addEventListener('click', ()=>{ const rows = [['时间段','T(1)','DPS(7)','奶(2)']];
      TIME_SLOTS.forEach(s=>{ const r = state[s.key]; rows.push([s.label, r.T.join(' | '), r.DPS.join(' | '), r['奶'].join(' | ')]); });
      const csv = rows.map(r=>r.map(v=>`"${(v||'').replace(/"/g,'""')}"`).join(',')).join('\\n'); download('raid_signup.csv', csv, 'text/csv;charset=utf-8;');
    });

    document.getElementById('exportJson').addEventListener('click', ()=>{ download('raid_signup.json', JSON.stringify(state, null, 2), 'application/json;charset=utf-8;'); });

    document.getElementById('copyText').addEventListener('click', ()=>{ let out = [];
      TIME_SLOTS.forEach(s=>{ const r = state[s.key]; const tRem = CAPACITY.T - r.T.length; const dRem = CAPACITY.DPS - r.DPS.length; const hRem = CAPACITY['奶'] - r['奶'].length;
        out.push(`【${s.label}】\\nT(${r.T.length}/1)：${r.T.join('，')||'—'}\\nDPS(${r.DPS.length}/7)：${r.DPS.join('，')||'—'}\\n奶(${r['奶'].length}/2)：${r['奶'].join('，')||'—'}\\n剩余：T${tRem} / DPS${dRem} / 奶${hRem}`);
      }); copyText(out.join('\\n\\n')); alert('已复制接龙文本');
    });

    document.getElementById('resetAll').addEventListener('click', ()=>{ if(confirm('确认清空所有报名？')){ state = emptyRoster(); render(); } });

    function download(filename, content, mime){ const blob = new Blob([content], {type: mime}); const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = filename; a.click(); URL.revokeObjectURL(a.href); }
    function copyText(t){ navigator.clipboard?.writeText(t).catch(()=>{ const ta = document.createElement('textarea'); ta.value=t; document.body.appendChild(ta); ta.select(); document.execCommand('copy'); ta.remove(); }); }
    function escapeHtml(s){ return s.replace(/[&<>\"']/g, c=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"}[c])); }

    render();
  </script>
</body>
</html>
