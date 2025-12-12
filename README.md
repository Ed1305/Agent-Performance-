<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Agent Performance Dashboard</title>
<script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
<style>
  body {font-family: Arial, sans-serif; background:#f4f6f9; color:#111; padding:20px;}
  h2 {text-align:center; margin-bottom:15px;}
  .controls {display:flex; flex-wrap:wrap; gap:10px; margin-bottom:15px; align-items:center;}
  .controls input, .controls button, .controls select {padding:5px 8px; font-size:13px;}
  table {width:100%; border-collapse:collapse; font-size:12px; background:#fff; margin-bottom:20px;}
  th, td {border:1px solid #ccc; padding:4px 6px; text-align:center; position: relative;}
  th {background:#e6e9ef;}
  tfoot td {font-weight:bold; background:#e6e9ef;}
  .good {background-color:#c6f6d5;}
  .bad {background-color:#fed7d7;}
  .lightgrey {background-color:#e0e0e0;}
  td[data-tooltip]:hover::after {
    content: attr(data-tooltip);
    position:absolute;
    background:#333; color:#fff; padding:2px 6px; border-radius:3px; font-size:11px;
    top:100%; left:50%; transform:translateX(-50%); white-space:nowrap; z-index:10;
  }
</style>
</head>
<body>

<h2>Agent Performance Dashboard</h2>

<div class="controls">
  <input type="file" id="fileInput">
  <label>Total Working Days: <input type="number" id="totalDays" value=""></label>
  <label>Days Worked: <input type="number" id="daysWorked" value=""></label>
  <label>Days Remaining: <input type="number" id="daysRemaining" value="" readonly></label>
  <button onclick="downloadTable()">Download Table as Image</button>
  <label>SECTION:
    <select id="sectionSelect">
      <option value="name">AGENT'S NAME</option>
      <option value="calls">CALLS</option>
      <option value="sales">SALES</option>
      <option value="closing">% CLOSING</option>
      <option value="pending">PENDINGS</option>
      <option value="pendingPct">% PENDING</option>
      <option value="net">NET SALES</option>
      <option value="avg">AVERAGE</option>
      <option value="projected">PROJECTED SALES</option>
      <option value="target">TARGET</option>
      <option value="shortfall">SHORTFALL</option>
      <option value="req">REQ/DAY</option>
      <option value="comm">PROJECTED COMM (ZAR)</option>
    </select>
  </label>
  <label>Sort:
    <select id="sortOrder">
      <option value="desc">Largest to Smallest</option>
      <option value="asc">Smallest to Largest</option>
    </select>
  </label>
  <button onclick="applySorting()">Confirm</button>
</div>

<table id="agentTable">
  <thead>
    <tr>
      <th>AGENT'S NAME</th>
      <th>CALLS</th>
      <th>SALES</th>
      <th>% CLOSING</th>
      <th>PENDINGS</th>
      <th>% PENDING</th>
      <th>NET SALES</th>
      <th>AVERAGE</th>
      <th>PROJECTED SALES</th>
      <th>TARGET</th>
      <th>SHORTFALL</th>
      <th>REQ/DAY</th>
      <th>PROJECTED COMM (ZAR)</th>
    </tr>
  </thead>
  <tbody></tbody>
  <tfoot>
    <tr id="totalsRow"></tr>
    <tr id="averagesRow"></tr>
  </tfoot>
</table>

<script>
let data = [];

document.getElementById("fileInput").addEventListener("change", async function(e){
  const file = e.target.files[0];
  const text = await file.text();
  parseCSV(text);
});

document.getElementById("totalDays").addEventListener("input", recalcDays);
document.getElementById("daysWorked").addEventListener("input", recalcDays);

function recalcDays(){
  const total = +document.getElementById("totalDays").value;
  const worked = +document.getElementById("daysWorked").value;
  document.getElementById("daysRemaining").value = total - worked;
  renderTable();
}

function parseCSV(text){
  const rows = text.split("\n").map(r=>r.split(","));
  const header = rows.shift().map(h=>h.trim().toLowerCase());
  data = rows.map(r=>{
    let obj={ full_name: r[header.indexOf("full_name")] };
    obj.dialed = +r[header.indexOf("dialed")]||0;
    obj.sale = +r[header.indexOf("sale")]||0;
    obj.pen = +r[header.indexOf("pen")]||0;
    return obj;
  });
  renderTable();
}

// Calculated row
function calcRow(d){
  const total = +document.getElementById("totalDays").value;
  const worked = +document.getElementById("daysWorked").value;
  const remaining = total - worked;

  const calls = +d.dialed||0, sales = +d.sale||0, pending = +d.pen||0;
  const net = sales - pending;
  const closing = calls ? ((sales/calls)*100).toFixed(1) : 0;
  const pendingPct = net ? ((pending/net)*100).toFixed(1) : 0;
  const avg = worked ? (net/worked).toFixed(1) : 0;
  const projected = Math.round(avg*total);
  const target = 12*total;
  const shortfall = target - net;
  const reqDay = remaining>0 ? Math.round(shortfall/remaining) : 0;

  let comm=0;
  if(avg<11.5) comm=0;
  else if(avg<=16.5) comm=sales*5;
  else if(avg<=20) comm=sales*10;
  else if(avg<=25) comm=sales*15;
  else comm=sales*20;

  return {
    name: d.full_name.replace(/"/g,""),
    calls,
    sales,
    closing,
    pending,
    pendingPct,
    net,
    avg,
    projected,
    target,
    shortfall,
    req:reqDay,
    comm,
    tooltipPending: `${pending}/${net} = ${pendingPct}%`,
    tooltipReq: `${shortfall}/${remaining} = ${reqDay}`
  };
}

// Render table
function renderTable(precomputed=null){
  const tbody = document.querySelector("#agentTable tbody");
  tbody.innerHTML="";
  if(!data.length) return;

  let computed = precomputed || data.map(calcRow);

  const avgClosing = computed.reduce((s,r)=>s+parseFloat(r.closing),0)/computed.length;
  const avgAverage = computed.reduce((s,r)=>s+parseFloat(r.avg),0)/computed.length;
  const avgReq = computed.reduce((s,r)=>s+r.req,0)/computed.length;

  computed.forEach(r=>{
    let tr = document.createElement("tr");
    tr.innerHTML=`
      <td>${r.name}</td>
      <td>${r.calls}</td>
      <td>${r.sales}</td>
      <td class="${r.closing>=avgClosing?'good':'bad'}">${r.closing}%</td>
      <td>${r.pending}</td>
      <td class="lightgrey" data-tooltip="${r.tooltipPending}">${r.pendingPct}%</td>
      <td>${r.net}</td>
      <td class="${r.avg>=avgAverage?'good':'bad'}">${r.avg}</td>
      <td>${r.projected}</td>
      <td>${r.target}</td>
      <td>${r.shortfall}</td>
      <td class="${r.req<=avgReq?'good':'bad'}" data-tooltip="${r.tooltipReq}">${r.req}</td>
      <td>R ${r.comm.toFixed(2)}</td>
    `;
    tbody.appendChild(tr);
  });

  renderTotals(computed);
  renderAverages(computed);
}

// Totals & averages
function renderTotals(rows){
  const t = document.getElementById("totalsRow");
  if(!rows.length){t.innerHTML="";return;}
  const sum = key => rows.reduce((a,b)=>a+(+b[key]||0),0);
  t.innerHTML=`
    <td>TOTAL</td>
    <td>${sum("calls")}</td>
    <td>${sum("sales")}</td>
    <td>${(sum("closing")/rows.length).toFixed(1)}%</td>
    <td>${sum("pending")}</td>
    <td>${(sum("pendingPct")/rows.length).toFixed(1)}%</td>
    <td>${sum("net")}</td>
    <td>${(sum("avg")/rows.length).toFixed(1)}</td>
    <td>${sum("projected")}</td>
    <td>${sum("target")}</td>
    <td>${sum("shortfall")}</td>
    <td>${Math.round(sum("req")/rows.length)}</td>
    <td>R ${sum("comm").toFixed(2)}</td>
  `;
}

function renderAverages(rows){
  const t = document.getElementById("averagesRow");
  if(!rows.length){t.innerHTML="";return;}
  const avg = key => rows.reduce((a,b)=>a+(+b[key]||0),0)/rows.length;
  t.innerHTML=`
    <td>AVERAGE</td>
    <td>${avg("calls").toFixed(1)}</td>
    <td>${avg("sales").toFixed(1)}</td>
    <td>${avg("closing").toFixed(1)}%</td>
    <td>${avg("pending").toFixed(1)}%</td>
    <td>${avg("pendingPct").toFixed(1)}%</td>
    <td>${avg("net").toFixed(1)}</td>
    <td>${avg("avg").toFixed(1)}</td>
    <td>${avg("projected").toFixed(0)}</td>
    <td>${avg("target").toFixed(0)}</td>
    <td>${avg("shortfall").toFixed(0)}</td>
    <td>${Math.round(avg("req"))}</td>
    <td>R ${avg("comm").toFixed(2)}</td>
  `;
}

// Download table as image
function downloadTable(){
  const table = document.getElementById("agentTable");
  html2canvas(table, {scale:2}).then(canvas=>{
    const link = document.createElement('a');
    link.href = canvas.toDataURL("image/png");
    link.download = "agent_table.png";
    link.click();
  });
}

// SECTION sorting
function applySorting(){
  if(!data.length) return;
  const col = document.getElementById("sectionSelect").value;
  const order = document.getElementById("sortOrder").value;

  let computed = data.map(calcRow);

  computed.sort((a,b)=>{
    let va = a[col], vb = b[col];
    va = isNaN(+va) ? va : +va;
    vb = isNaN(+vb) ? vb : +vb;
    if(!isNaN(va) && !isNaN(vb)) return order==="asc" ? va-vb : vb-va;
    return order==="asc" ? va.localeCompare(vb) : vb.localeCompare(va);
  });

  renderTable(computed);
}
</script>

</body>
</html>
