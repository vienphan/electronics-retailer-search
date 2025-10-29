
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Keyword Combination Tracker — Retail Brands</title>
  <meta name="description" content="Track keyword combinations for retail brands with result counts and top related Vietnamese keywords." />
  <link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    :root{
      --bg:#ffffff; --fg:#111; --muted:#666; --brand:#e11d48; --card:#fafafa; --ok:#16a34a; --warn:#b45309;
    }
    *{box-sizing:border-box}
    body{margin:0;font-family:system-ui,-apple-system,Segoe UI,Roboto,Inter,Arial,sans-serif;background:var(--bg);color:var(--fg);line-height:1.6}
    header{padding:24px 16px;border-bottom:1px solid #eee;background:linear-gradient(180deg,#fff,#fafafa)}
    .wrap{max-width:1200px;margin:0 auto;padding:0 16px}
    h1{margin:4px 0 6px;font-size:clamp(20px,3.5vw,28px)}
    p.sub{margin:0;color:var(--muted)}
    .card{background:var(--card);border:1px solid #eee;border-radius:16px;padding:16px;margin-top:16px}
    .row{display:grid;grid-template-columns:1fr auto;gap:12px;align-items:center}
    input[type=text]{padding:10px 14px;border-radius:999px;border:1px solid #ccc;width:240px;font-size:15px}
    button{appearance:none;border:none;background:var(--brand);color:#fff;border-radius:999px;padding:10px 16px;font-weight:600;cursor:pointer}
    button:disabled{opacity:.6;cursor:not-allowed}
    .stamp{font-size:14px;color:var(--muted)}
    table{width:100%;border-collapse:collapse;margin-top:8px;background:#fff;border-radius:12px;overflow:hidden}
    th,td{padding:10px 8px;border-bottom:1px solid #eee;text-align:left;font-size:14px}
    th{background:#f7f7f7}
    td.num{text-align:right;font-variant-numeric:tabular-nums}
    .status-ok{color:var(--ok);font-weight:600}
    .status-warn{color:var(--warn);font-weight:600}
    .flex{display:flex;gap:8px;flex-wrap:wrap;align-items:center}
    .badge{background:#fff;border:1px solid #eee;border-radius:999px;padding:6px 10px;font-size:13px}
    .small{font-size:12px;color:var(--muted)}
    canvas{width:100%;max-height:420px}
    footer{padding:24px 16px;color:#666;text-align:center;border-top:1px solid #eee;margin-top:28px}
    footer a{color:#007BFF;text-decoration:none;font-weight:600}
    footer a:hover{text-decoration:underline}
    @media(max-width:640px){
      th,td{font-size:12px;padding:8px 6px}
      input[type=text]{width:100%}
    }
  </style>
</head>
<body>
  <header>
    <div class="wrap">
      <h1>Keyword Combination Tracker</h1>
      <p class="sub">Shows search counts and top 3 Vietnamese co-occurring words for each retailer brand + keyword combination.</p>
      <div class="card">
        <div class="row">
          <div class="flex">
            <input type="text" id="extraKeyword" placeholder="Enter extra keyword (e.g., Tủ lạnh)" />
            <button id="scanBtn">Scan Now</button>
          </div>
          <span class="stamp" id="ts">Not yet scanned</span>
        </div>
      </div>
    </div>
  </header>

  <main class="wrap">
    <div class="card">
      <strong>Fixed Brand Keywords</strong>
      <div id="kwList" class="flex" style="margin-top:8px;"></div>
    </div>

    <div class="card">
      <strong>Scan Results</strong>
      <table id="resultTable">
        <thead>
          <tr>
            <th>Combined Keyword</th>
            <th>Source</th>
            <th class="num">Result Count</th>
            <th>Top 1</th>
            <th>Top 2</th>
            <th>Top 3</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody id="tbody">
          <tr><td colspan="7" class="small">No data yet. Enter a keyword and click “Scan Now”.</td></tr>
        </tbody>
      </table>
    </div>

    <div class="card">
      <strong>Visualization</strong>
      <canvas id="chart"></canvas>
    </div>
  </main>

  <footer>
    <p>Made with passion for Retail by 
      <a href="https://vienphan.github.io/karim-profile/" target="_blank" rel="noopener">Karim Noui</a>.
    </p>
    <span class="small">© <span id="year"></span></span>
  </footer>

  <script>
    const BASE_KEYWORDS = [
      "Thế Giới Di Động",
      "Nguyễn Kim",
      "Chợ Lớn",
      "Điện Máy Xanh",
      "Pico",
      "FPT",
      "CellphoneS",
      "Phong Vũ"
    ];

    // bilingual stopwords (English + Vietnamese)
    const STOPWORDS = new Set([
      "the","and","of","for","in","on","to","is","a","an","with","by","at","as","from","that","this","it","are","be","or","was","were","has","have","had",
      "và","là","có","của","cho","trên","dưới","ở","tại","một","các","những","này","kia","đó","thì","được","bị","đang","với","rằng","nên","khi","nếu","đã",
      "rất","rằng","vì","nơi","sẽ","đến","hơn","cũng","làm","thế","gì","cần","thành","nhiều","ít","sản","phẩm","trong","ra"
    ]);

    function buildBingUrl(keyword){
      const q = encodeURIComponent('"' + keyword + '"');
      return `https://www.bing.com/search?q=${q}&setlang=en-US&mkt=en-US`;
    }
    const proxied = url => `https://r.jina.ai/http://${url.replace(/^https?:\/\//,'')}`;

    function parseBingCount(text){
      const lower = text.toLowerCase().replace(/\s+/g,' ');
      const m = lower.match(/(?:about\s+)?([\d,._]+)\s+results/i);
      if(!m)return null;
      const n = Number(m[1].replace(/[,_]/g,''));
      return Number.isFinite(n)?n:null;
    }

    function extractTopWords(text, mainKeyword){
      const words = text.toLowerCase()
        .replace(/[^a-zA-ZÀ-ỹ0-9\s]/g,' ')
        .split(/\s+/);
      const mainParts = mainKeyword.toLowerCase().split(/\s+/);
      const freq = {};
      for(const w of words){
        if(!w || STOPWORDS.has(w) || mainParts.includes(w)) continue;
        freq[w] = (freq[w]||0)+1;
      }
      const sorted = Object.entries(freq)
        .sort((a,b)=>b[1]-a[1])
        .slice(0,3)
        .map(x=>x[0]);
      return [sorted[0]||'',sorted[1]||'',sorted[2]||''];
    }

    async function scanOne(keyword){
      const url = buildBingUrl(keyword);
      const prox = proxied(url);
      let status='OK';
      try{
        const res=await fetch(prox);
        if(!res.ok)throw new Error('HTTP '+res.status);
        const text=await res.text();
        const count=parseBingCount(text);
        const tops=extractTopWords(text,keyword);
        if(count==null){
          status='Fallback/Blocked';
          return{keyword,source:'Bing',count:0,top1:tops[0],top2:tops[1],top3:tops[2],status,url};
        }
        return{keyword,source:'Bing',count,top1:tops[0],top2:tops[1],top3:tops[2],status,url};
      }catch(e){
        status='Error';
        return{keyword,source:'Bing',count:0,top1:'',top2:'',top3:'',status,url};
      }
    }

    function renderKwList(){
      const box=document.getElementById('kwList');
      box.innerHTML='';
      BASE_KEYWORDS.forEach(k=>{
        const span=document.createElement('span');
        span.className='badge';
        span.textContent=k;
        box.appendChild(span);
      });
    }

    function renderTable(rows){
      const tb=document.getElementById('tbody');
      tb.innerHTML='';
      if(!rows.length){
        tb.innerHTML='<tr><td colspan="7" class="small">No data available.</td></tr>';
        return;
      }
      for(const r of rows){
        const tr=document.createElement('tr');
        tr.innerHTML=`
          <td>${r.keyword}</td>
          <td><a href="${r.url}" target="_blank" rel="noopener">Bing</a></td>
          <td class="num">${r.count.toLocaleString('en-US')}</td>
          <td>${r.top1}</td>
          <td>${r.top2}</td>
          <td>${r.top3}</td>
          <td>${r.status==='OK'
                ?'<span class="status-ok">OK</span>'
                :'<span class="status-warn">'+r.status+'</span>'}</td>`;
        tb.appendChild(tr);
      }
    }

    let chart;
    function renderChart(rows){
      const ctx=document.getElementById('chart').getContext('2d');
      const labels=rows.map(r=>r.keyword);
      const data=rows.map(r=>r.count);
      if(chart)chart.destroy();
      chart=new Chart(ctx,{
        type:'bar',
        data:{labels,datasets:[{label:'Approx. Search Results',data}]},
        options:{responsive:true,maintainAspectRatio:false,
          plugins:{legend:{display:true}},
          scales:{x:{ticks:{autoSkip:false}},y:{beginAtZero:true}}}
      });
    }

    async function scanAll(){
      const extra=document.getElementById('extraKeyword').value.trim();
      if(!extra){alert("Please enter an extra keyword first.");return;}
      const btn=document.getElementById('scanBtn');
      btn.disabled=true;btn.textContent='Scanning...';
      document.getElementById('ts').textContent='Scanning...';
      const combined=BASE_KEYWORDS.map(b=>`${b} ${extra}`);
      const results=await Promise.all(combined.map(scanOne));
      results.sort((a,b)=>b.count-a.count);
      renderTable(results);
      renderChart(results);
      document.getElementById('ts').textContent='Last scanned: '+new Date().toLocaleString();
      btn.disabled=false;btn.textContent='Scan Now';
    }

    document.getElementById('year').textContent=new Date().getFullYear();
    renderKwList();
    document.getElementById('scanBtn').addEventListener('click',scanAll);
  </script>
</body>
</html>
