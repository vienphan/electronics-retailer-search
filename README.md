
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Keyword Tracker — Electronics Retail Brands</title>
  <meta name="description" content="Instantly track the number of web results for key retail brands and visualize the data on a chart." />
  <link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    :root{
      --bg:#ffffff; --fg:#111; --muted:#666; --brand:#e11d48; --card:#fafafa; --ok:#16a34a; --warn:#b45309;
    }
    *{box-sizing:border-box}
    body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Inter,Arial,sans-serif;background:var(--bg);color:var(--fg);line-height:1.6}
    header{padding:24px 16px;border-bottom:1px solid #eee;background:linear-gradient(180deg,#fff, #fff 60%, #fafafa)}
    .wrap{max-width:1100px;margin:0 auto;padding:0 16px}
    h1{margin:4px 0 6px;font-size:clamp(20px,3.5vw,28px)}
    p.sub{margin:0;color:var(--muted)}
    .card{background:var(--card);border:1px solid #eee;border-radius:16px;padding:16px;margin-top:16px}
    .row{display:grid;grid-template-columns:1fr auto;gap:12px;align-items:center}
    button{appearance:none;border:none;background:var(--brand);color:#fff;border-radius:999px;padding:10px 16px;font-weight:600;cursor:pointer}
    button:disabled{opacity:.6;cursor:not-allowed}
    .stamp{font-size:14px;color:var(--muted)}
    table{width:100%;border-collapse:collapse;margin-top:8px;background:#fff;border-radius:12px;overflow:hidden}
    th,td{padding:12px 10px;border-bottom:1px solid #eee;text-align:left}
    th{font-size:14px;color:#444;background:#f7f7f7}
    td.num{text-align:right;font-variant-numeric:tabular-nums}
    .status-ok{color:var(--ok);font-weight:600}
    .status-warn{color:var(--warn);font-weight:600}
    .flex{display:flex;gap:8px;flex-wrap:wrap;align-items:center}
    .badge{background:#fff;border:1px solid #eee;border-radius:999px;padding:6px 10px;font-size:13px}
    .small{font-size:12px;color:var(--muted)}
    canvas{width:100%;max-height:420px}
    footer{padding:24px 16px;color:#666;text-align:center;border-top:1px solid #eee;margin-top:28px}
    @media (max-width:640px){
      .row{grid-template-columns:1fr}
      header{padding:20px 12px}
      .card{padding:12px}
      th,td{padding:10px 8px}
    }
  </style>
</head>
<body>
  <header>
    <div class="wrap">
      <h1>Keyword Tracker (Instant Scan)</h1>
      <p class="sub">This tool estimates the number of search results for key electronics retailers at the time of scanning, using Bing Search.</p>
      <div class="card">
        <div class="row">
          <div class="flex">
            <span class="badge">Source: Bing Search (en-US)</span>
            <span class="badge">Proxy: r.jina.ai</span>
            <span class="badge">Chart: Chart.js</span>
          </div>
          <div class="flex">
            <button id="scanBtn">Scan Now</button>
            <span class="stamp" id="ts">Not yet scanned</span>
          </div>
        </div>
      </div>
    </div>
  </header>

  <main class="wrap">
    <div class="card">
      <strong>Monitored Keywords</strong>
      <p class="small">You can modify the list directly in the source code (<code>KEYWORDS</code> variable).</p>
      <div id="kwList" class="flex" style="margin-top:8px;"></div>
    </div>

    <div class="card">
      <strong>Scan Results</strong>
      <table id="resultTable" aria-live="polite">
        <thead>
          <tr>
            <th style="width:40%">Keyword</th>
            <th>Source</th>
            <th class="num">Result Count</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody id="tbody">
          <tr><td colspan="4" class="small">No data yet. Click “Scan Now”.</td></tr>
        </tbody>
      </table>
    </div>

    <div class="card">
      <strong>Visualization</strong>
      <p class="small">Showing live results from the latest scan.</p>
      <canvas id="chart"></canvas>
    </div>

    <div class="card">
      <details>
        <summary><strong>Technical Notes</strong></summary>
        <ul class="small">
          <li>The proxy <code>r.jina.ai</code> fetches and parses search result pages (SERPs) as plain text to avoid CORS issues. This may break if the provider changes its structure.</li>
          <li>Bing results are standardized to <code>en-US</code> to ensure a stable “About N results” pattern for parsing.</li>
          <li>If a query is blocked or fails to parse, it will show <em>Fallback/Blocked</em> status.</li>
          <li>For accurate and time-filtered data (past hour/day), you should use an official API such as <b>Bing Web Search API</b>, <b>Google Custom Search API</b>, or <b>NewsAPI</b>.</li>
        </ul>
      </details>
    </div>
  </main>

  <footer>
    Made with ❤️ | <span class="small">© <span id="year"></span></span>
  </footer>

  <script>
    // ===== CONFIGURATION =====
    const KEYWORDS = [
      "Nguyễn Kim",
      "Chợ Lớn",
      "Điện Máy Xanh",
      "Pico",
      "FPT",
      "CellphoneS",
      "Phong Vũ"
    ];

    function buildBingUrl(keyword){
      const q = encodeURIComponent('"' + keyword + '"');
      return `https://www.bing.com/search?q=${q}&setlang=en-US&mkt=en-US`;
    }

    function proxied(url){
      return `https://r.jina.ai/http://${url.replace(/^https?:\/\//,'')}`;
    }

    function parseBingCount(text){
      const s = text.replace(/\s+/g, ' ');
      const m = s.match(/(?:About\s+)?([\d,._]+)\s+results/i);
      if (!m) return null;
      const num = m[1].replace(/[,_]/g,'');
      const n = Number(num);
      return Number.isFinite(n) ? n : null;
    }

    async function scanOne(keyword){
      const url = buildBingUrl(keyword);
      const prox = proxied(url);
      let status = 'OK';
      try{
        const res = await fetch(prox, { method: 'GET' });
        if(!res.ok) throw new Error('HTTP ' + res.status);
        const text = await res.text();
        const count = parseBingCount(text);
        if (count == null){
          status = 'Fallback/Blocked';
          return { keyword, source: 'Bing', count: 0, status, url };
        }
        return { keyword, source: 'Bing', count, status, url };
      }catch(e){
        status = 'Error';
        return { keyword, source: 'Bing', count: 0, status, url };
      }
    }

    function renderKwList(){
      const box = document.getElementById('kwList');
      box.innerHTML = '';
      KEYWORDS.forEach(k=>{
        const span = document.createElement('span');
        span.className = 'badge';
        span.textContent = k;
        box.appendChild(span);
      });
    }

    function renderTable(rows){
      const tb = document.getElementById('tbody');
      tb.innerHTML = '';
      if(!rows.length){
        tb.innerHTML = '<tr><td colspan="4" class="small">No data available.</td></tr>';
        return;
      }
      for(const r of rows){
        const tr = document.createElement('tr');
        tr.innerHTML = `
          <td>${r.keyword}</td>
          <td><a href="${r.url}" target="_blank" rel="noopener">Bing</a></td>
          <td class="num">${r.count.toLocaleString('en-US')}</td>
          <td>${r.status === 'OK'
                ? '<span class="status-ok">OK</span>'
                : '<span class="status-warn">'+r.status+'</span>'}</td>
        `;
        tb.appendChild(tr);
      }
    }

    let chart;
    function renderChart(rows){
      const ctx = document.getElementById('chart').getContext('2d');
      const labels = rows.map(r=>r.keyword);
      const data = rows.map(r=>r.count);

      if(chart){ chart.destroy(); }
      chart = new Chart(ctx, {
        type: 'bar',
        data: {
          labels,
          datasets: [{
            label: 'Approx. Search Results',
            data
          }]
        },
        options: {
          responsive:true,
          maintainAspectRatio:false,
          plugins:{
            legend:{ display:true },
            tooltip:{ mode:'index', intersect:false }
          },
          scales:{
            x:{ ticks:{ autoSkip:false, maxRotation:30, minRotation:0 }},
            y:{ beginAtZero:true }
          }
        }
      });
    }

    async function scanAll(){
      const btn = document.getElementById('scanBtn');
      btn.disabled = true; btn.textContent = 'Scanning...';
      const ts = new Date();
      document.getElementById('ts').textContent = 'Scanning at ' + ts.toLocaleString();

      const promises = KEYWORDS.map(k => scanOne(k));
      const results = await Promise.all(promises);

      results.sort((a,b)=>b.count - a.count);

      renderTable(results);
      renderChart(results);
      document.getElementById('ts').textContent = 'Last scanned: ' + new Date().toLocaleString();
      btn.disabled = false; btn.textContent = 'Scan Now';
    }

    document.getElementById('year').textContent = new Date().getFullYear();
    renderKwList();
    document.getElementById('scanBtn').addEventListener('click', scanAll);
  </script>
</body>
</html>
