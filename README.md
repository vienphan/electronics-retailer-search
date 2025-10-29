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

// Stopwords: English + Vietnamese + technical noise
const STOPWORDS = new Set([
  "the","and","of","for","in","on","to","is","a","an","with","by","at","as","from","that","this","it","are","be","or",
  "was","were","has","have","had","https","http","bing","search","result","results","com","www","img","e1","jpg","png",
  "và","là","có","của","cho","trên","dưới","ở","tại","một","các","những","này","kia","đó","thì","được","bị","đang","với",
  "rằng","nên","khi","nếu","đã","rất","vì","nơi","sẽ","đến","hơn","cũng","làm","thế","gì","cần","thành","nhiều","ít","sản",
  "phẩm","trong","ra","tốt","như","nào","điện","máy","cửa","hàng","siêu","thị"
]);

// Clean Vietnamese text and extract only meaningful words
function cleanText(text) {
  // Remove URLs, technical patterns, symbols
  let cleaned = text
    .replace(/https?:\/\/\S+/gi, " ")
    .replace(/\b(www|bing|google|result|search|href|src|img|png|jpg|jpeg|html)\b/gi, " ")
    .replace(/[^a-zA-ZÀ-ỹ\s]/g, " ")
    .replace(/\s+/g, " ")
    .trim()
    .toLowerCase();
  return cleaned;
}

function extractTopWords(text, mainKeyword) {
  const cleaned = cleanText(text);
  const words = cleaned.split(/\s+/);
  const mainParts = mainKeyword.toLowerCase().split(/\s+/);
  const freq = {};

  for (const w of words) {
    if (!w || STOPWORDS.has(w) || mainParts.includes(w) || w.length < 3) continue;
    freq[w] = (freq[w] || 0) + 1;
  }

  // Sort by frequency
  const sorted = Object.entries(freq)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 3)
    .map(x => x[0]);

  return [sorted[0] || '', sorted[1] || '', sorted[2] || ''];
}

function buildBingUrl(keyword) {
  const q = encodeURIComponent('"' + keyword + '"');
  return `https://www.bing.com/search?q=${q}&setlang=en-US&mkt=en-US`;
}
const proxied = url => `https://r.jina.ai/http://${url.replace(/^https?:\/\//, '')}`;

function parseBingCount(text) {
  const lower = text.toLowerCase().replace(/\s+/g, ' ');
  const m = lower.match(/(?:about\s+)?([\d,._]+)\s+results/i);
  if (!m) return null;
  const n = Number(m[1].replace(/[,_]/g, ''));
  return Number.isFinite(n) ? n : null;
}

async function scanOne(keyword) {
  const url = buildBingUrl(keyword);
  const prox = proxied(url);
  let status = 'OK';
  try {
    const res = await fetch(prox);
    if (!res.ok) throw new Error('HTTP ' + res.status);
    const text = await res.text();
    const count = parseBingCount(text);
    const tops = extractTopWords(text, keyword);
    if (count == null) {
      status = 'Fallback/Blocked';
      return { keyword, source: 'Bing', count: 0, top1: tops[0], top2: tops[1], top3: tops[2], status, url };
    }
    return { keyword, source: 'Bing', count, top1: tops[0], top2: tops[1], top3: tops[2], status, url };
  } catch (e) {
    status = 'Error';
    return { keyword, source: 'Bing', count: 0, top1: '', top2: '', top3: '', status, url };
  }
}

function renderKwList() {
  const box = document.getElementById('kwList');
  box.innerHTML = '';
  BASE_KEYWORDS.forEach(k => {
    const span = document.createElement('span');
    span.className = 'badge';
    span.textContent = k;
    box.appendChild(span);
  });
}

function renderTable(rows) {
  const tb = document.getElementById('tbody');
  tb.innerHTML = '';
  if (!rows.length) {
    tb.innerHTML = '<tr><td colspan="7" class="small">No data available.</td></tr>';
    return;
  }
  for (const r of rows) {
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td>${r.keyword}</td>
      <td><a href="${r.url}" target="_blank" rel="noopener">Bing</a></td>
      <td class="num">${r.count.toLocaleString('en-US')}</td>
      <td>${r.top1}</td>
      <td>${r.top2}</td>
      <td>${r.top3}</td>
      <td>${r.status === 'OK'
        ? '<span class="status-ok">OK</span>'
        : '<span class="status-warn">' + r.status + '</span>'}</td>`;
    tb.appendChild(tr);
  }
}

let chart;
function renderChart(rows) {
  const ctx = document.getElementById('chart').getContext('2d');
  const labels = rows.map(r => r.keyword);
  const data = rows.map(r => r.count);
  if (chart) chart.destroy();
  chart = new Chart(ctx, {
    type: 'bar',
    data: { labels, datasets: [{ label: 'Approx. Search Results', data }] },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: { legend: { display: true } },
      scales: { x: { ticks: { autoSkip: false } }, y: { beginAtZero: true } }
    }
  });
}

async function scanAll() {
  const extra = document.getElementById('extraKeyword').value.trim();
  if (!extra) { alert("Please enter an extra keyword first."); return; }
  const btn = document.getElementById('scanBtn');
  btn.disabled = true; btn.textContent = 'Scanning...';
  document.getElementById('ts').textContent = 'Scanning...';
  const combined = BASE_KEYWORDS.map(b => `${b} ${extra}`);
  const results = await Promise.all(combined.map(scanOne));
  results.sort((a, b) => b.count - a.count);
  renderTable(results);
  renderChart(results);
  document.getElementById('ts').textContent = 'Last scanned: ' + new Date().toLocaleString();
  btn.disabled = false; btn.textContent = 'Scan Now';
}

document.getElementById('year').textContent = new Date().getFullYear();
renderKwList();
document.getElementById('scanBtn').addEventListener('click', scanAll);
</script>
