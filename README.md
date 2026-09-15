DERIV DIGIT ANALYZER
Live ticks • statistics • backtesting • guarded demo execution

OFFLINE
Symbol 
Volatility 100 (1s)
 History 
500
 Connect Stop
Last digit—
Ticks0
Model score—
Validated edge—
Digit distribution
Signal
WAIT
Best MATCH candidate— Best DIFFER candidate— Sample quality—
Rolling windows
Window	Top digit	Frequency	Score	Status
Live tick stream
Backtest
The backtester walks through historical ticks and scores predictions without placing trades.

Minimum history 
200
 Threshold 
0.80
 Run on current ticks
No backtest yet.
Execution guard
 Demo-only mode Max stake 
1
 Max consecutive losses 
3
 Connect authenticated demo
Execution locked. Analysis and backtesting are available without credentials.const PUBLIC_WS = "wss://api.derivws.com/trading/v1/options/ws/public";

let ws = null;
let ticks = [];
let connected = false;
let pipSize = 2;

const $ = id => document.getElementById(id);

function digitFromQuote(quote) {
  const decimals = Math.max(0, Number(pipSize || 2));
  const s = Number(quote).toFixed(decimals);
  const parts = s.split(".");
  const tail = parts[1] || "";
  return Number(tail.at(-1) || s.at(-1));
}

function counts(arr) {
  const c = Array(10).fill(0);
  for (const d of arr) if (Number.isInteger(d) && d >= 0 && d <= 9) c[d]++;
  return c;
}

function normalized(c) {
  const n = c.reduce((a,b)=>a+b,0);
  return c.map(x => n ? x/n : 0);
}

/*
  This score is a ranking heuristic, not a guaranteed probability.
  It combines smoothed frequency, recent frequency, and transition behavior.
*/
function analyze(arr) {
  if (!arr.length) return null;
  const long = arr.slice(-Math.min(1000, arr.length));
  const short = arr.slice(-Math.min(100, arr.length));
  const lc = normalized(counts(long));
  const sc = normalized(counts(short));

  const trans = Array.from({length:10},()=>Array(10).fill(0));
  for(let i=1;i<long.length;i++) trans[long[i-1]][long[i]]++;
  const last = arr.at(-1);
  const row = trans[last];
  const rowTotal = row.reduce((a,b)=>a+b,0);
  const tc = row.map(x => rowTotal ? x/rowTotal : 0);

  // Shrink estimates toward uniform to avoid overconfidence.
  const uniform = 0.1;
  const score = Array.from({length:10},(_,d) =>
    0.45*(0.85*lc[d]+0.15*uniform) +
    0.35*(0.85*sc[d]+0.15*uniform) +
    0.20*(0.85*tc[d]+0.15*uniform)
  );

  const top = [...Array(10).keys()].sort((a,b)=>score[b]-score[a]);
  const best = top[0];
  const confidence = score[best];
  return {lc,sc,tc,score,best,confidence,counts:counts(long)};
}

function render() {
  const a = analyze(ticks);
  $("tickCount").textContent = ticks.length.toLocaleString();
  $("lastDigit").textContent = ticks.at(-1)?.digit ?? "—";
  if (!a) return;

  const max = Math.max(...a.counts,1);
  $("digits").innerHTML = a.counts.map((n,d)=>`
    <div class="digit"><b>${d}</b><span>${(n/a.counts.reduce((x,y)=>x+y,0)*100).toFixed(1)}%</span>
    <div class="bar"><i style="width:${(n/max)*100}%"></i></div></div>`).join("");

  $("modelScore").textContent = `${(a.confidence*100).toFixed(1)}%`;
  const threshold = 0.80;
  const enough = ticks.length >= 200;
  const signal = enough && a.confidence >= threshold ? "ANALYZE / VERIFY" : "WAIT";
  $("signal").textContent = signal;
  $("signal").className = "signal " + (signal === "WAIT" ? "wait" : "trade");

  $("bestMatch").textContent = `${a.best} (${(a.confidence*100).toFixed(1)})`;
  const differ = [...Array(10).keys()].sort((x,y)=>a.score[x]-a.score[y])[0];
  $("bestDiffer").textContent = `${differ} (${(a.score[differ]*100).toFixed(1)})`;
  $("sampleQuality").textContent = enough ? "usable" : `need ${200-ticks.length}`;

  const windows = [50,100,500,1000].filter(n=>ticks.length>=n);
  $("windows").innerHTML = windows.map(n=>{
    const sub=ticks.slice(-n).map(x=>x.digit), aa=analyze(sub);
    return `<tr><td>${n}</td><td>${aa.best}</td><td>${(aa.counts[aa.best]/n*100).toFixed(1)}%</td><td>${(aa.confidence*100).toFixed(1)}%</td><td>${aa.confidence>=threshold?"CHECK":"WAIT"}</td></tr>`;
  }).join("");
}

function addTick(quote,time) {
  const digit = digitFromQuote(quote);
  ticks.push({quote:Number(quote),time:Number(time),digit});
  if(ticks.length>10000) ticks=ticks.slice(-10000);
  const line = `[${new Date(Number(time)*1000).toLocaleTimeString()}] ${quote} → ${digit}`;
  $("stream").prepend(document.createTextNode(line + "\n"));
  while($("stream").childNodes.length>80) $("stream").lastChild.remove();
  render();
}

function connectPublic() {
  if(ws) ws.close();
  ticks=[];
  $("status").textContent="CONNECTING";
  ws=new WebSocket(PUBLIC_WS);
  ws.onopen=()=>{
    connected=true;
    $("status").textContent="LIVE";
    const symbol=$("symbol").value;
    const count=Number($("historyCount").value);
    ws.send(JSON.stringify({ticks_history:symbol,end:"latest",count,style:"ticks",subscribe:0,req_id:1}));
    ws.send(JSON.stringify({ticks:symbol,subscribe:1,req_id:2}));
  };
  ws.onmessage=e=>{
    const d=JSON.parse(e.data);
    if(d.error){console.error(d.error);$("status").textContent="ERROR";return}
    if(d.msg_type==="history"){
      pipSize=Number(d.pip_size ?? 2);
      const prices=d.history?.prices||[];
      const times=d.history?.times||[];
      ticks=prices.map((p,i)=>({quote:Number(p),time:Number(times[i]||Date.now()/1000),digit:digitFromQuote(p)}));
      render();
    }
    if(d.msg_type==="tick") addTick(d.tick.quote,d.tick.epoch);
  };
  ws.onclose=()=>{connected=false;$("status").textContent="OFFLINE"};
  ws.onerror=()=>{$("status").textContent="ERROR"};
}

function runBacktest() {
  const min=Number($("btMin").value||200);
  const threshold=Number($("btThreshold").value||0.8);
  if(ticks.length<min+10){$("backtestResult").textContent=`Need at least ${min+10} ticks.`;return}

  let predictions=0, correct=0, skipped=0;
  const outcomes=[];
  for(let i=min;i<ticks.length-1;i++){
    const history=ticks.slice(0,i).map(x=>x.digit);
    const a=analyze(history);
    if(!a){continue}
    if(a.confidence>=threshold){
      predictions++;
      const actual=ticks[i].digit;
      const ok=actual===a.best;
      if(ok) correct++;
      outcomes.push(ok?1:0);
    } else skipped++;
  }
  const accuracy=predictions?correct/predictions:0;
  $("backtestResult").textContent =
`Predictions: ${predictions}
Correct: ${correct}
Accuracy: ${(accuracy*100).toFixed(2)}%
Skipped: ${skipped}
NOTE: This is historical simulation, not a guarantee of future performance.`;
}

async function authDemo() {
  $("executionStatus").textContent="Requesting authenticated demo connection...";
  try{
    const r=await fetch("/api/auth-url",{method:"POST"});
    const d=await r.json();
    if(!r.ok) throw new Error(d.error||"Authentication failed");
    $("executionStatus").textContent="Authenticated demo URL obtained. This build does not auto-place orders.";
    console.log("Authenticated demo WebSocket URL received.");
    // Deliberately do not expose or store the URL in the UI.
  }catch(e){
    $("executionStatus").textContent=e.message;
  }
}

$("connect").onclick=connectPublic;
$("stop").onclick=()=>{if(ws)ws.close();};
$("runBacktest").onclick=runBacktest;
$("authDemo").onclick=authDemo;
$("symbol").onchange=()=>{if(connected)connectPublic();};
:root{font-family:Inter,system-ui,Arial,sans-serif;background:#090b10;color:#eef1f6}
*{box-sizing:border-box}body{margin:0}.top{display:flex;justify-content:space-between;align-items:center;padding:24px;max-width:1200px;margin:auto}
h1{font-size:22px;margin:0 0 5px}.top p,.muted{color:#8e98a8}.pill{padding:8px 12px;border-radius:999px;background:#222838;font-size:12px}
main{max-width:1200px;margin:auto;padding:0 16px 50px}.card{background:#11151d;border:1px solid #252b38;border-radius:14px;padding:18px;margin:14px 0;box-shadow:0 10px 30px rgba(0,0,0,.15)}
.controls{display:flex;gap:12px;align-items:end;flex-wrap:wrap}.controls label,.guard label,.backtest-controls label{display:flex;flex-direction:column;gap:6px;color:#aeb7c5;font-size:12px}
select,input{background:#0b0e14;color:#eee;border:1px solid #303746;border-radius:8px;padding:10px}
button{background:#f2f4f7;color:#0b0e14;border:0;border-radius:8px;padding:10px 16px;font-weight:700;cursor:pointer}
button.secondary{background:#252b38;color:#eee}.grid{display:grid;grid-template-columns:repeat(4,1fr);gap:14px}.grid.two{grid-template-columns:1.3fr 1fr}
.metric span{color:#8e98a8;font-size:12px}.metric strong{display:block;font-size:28px;margin-top:8px}
.digits{display:grid;grid-template-columns:repeat(10,1fr);gap:8px}.digit{background:#0b0e14;border:1px solid #29303d;border-radius:10px;padding:10px;text-align:center}.digit b{display:block;font-size:20px}.bar{height:5px;background:#303746;border-radius:5px;margin-top:7px;overflow:hidden}.bar i{display:block;height:100%;background:#dfe4eb}
.signal{font-size:40px;font-weight:900;text-align:center;padding:28px;border-radius:12px;margin-bottom:15px}.signal.wait{background:#25200f;color:#e9c65b}.signal.trade{background:#102419;color:#70d992}
.kv{display:grid;grid-template-columns:1fr auto;gap:10px;color:#9ca6b6}.kv b{color:#fff}
table{width:100%;border-collapse:collapse}th,td{padding:10px;border-bottom:1px solid #252b38;text-align:left;font-size:13px}
.stream{height:150px;overflow:auto;font-family:monospace;line-height:1.7;color:#b9c1ce}
.backtest-controls,.guard{display:flex;gap:12px;align-items:end;flex-wrap:wrap}pre{white-space:pre-wrap;background:#0a0d12;border-radius:10px;padding:14px;color:#cbd2dd}
@media(max-width:800px){.grid,.grid.two{grid-template-columns:1fr 1fr}.digits{grid-template-columns:repeat(5,1fr)}}@media(max-width:500px){.grid,.grid.two{grid-template-columns:1fr}.top{align-items:flex-start;gap:10px}.digits{grid-template-columns:repeat(5,1fr)}}
import express from "express";
import path from "path";
import { fileURLToPath } from "url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const app = express();
const PORT = Number(process.env.PORT || 8787);

app.use(express.json({ limit: "1mb" }));
app.use(express.static(path.join(__dirname, "public")));

app.get("/api/config", (_req, res) => {
  res.json({
    configured: Boolean(process.env.DERIV_APP_ID && process.env.DERIV_PAT && process.env.DERIV_ACCOUNT_ID),
    accountType: process.env.DERIV_ACCOUNT_TYPE || "demo",
    liveExecutionEnabled: false
  });
});

/*
  Returns an authenticated WebSocket URL.
  The PAT never leaves this server.
  Real-money execution is deliberately disabled in this starter.
*/
app.post("/api/auth-url", async (_req, res) => {
  const appId = process.env.DERIV_APP_ID;
  const pat = process.env.DERIV_PAT;
  const accountId = process.env.DERIV_ACCOUNT_ID;
  const accountType = process.env.DERIV_ACCOUNT_TYPE || "demo";

  if (!appId || !pat || !accountId) {
    return res.status(400).json({ error: "Authenticated Deriv credentials are not configured." });
  }

  if (accountType !== "demo") {
    return res.status(403).json({
      error: "This build intentionally blocks real-account execution. Set DERIV_ACCOUNT_TYPE=demo."
    });
  }

  const url = `https://api.derivws.com/trading/v1/options/accounts/${encodeURIComponent(accountId)}/otp`;

  try {
    const r = await fetch(url, {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${pat}`,
        "Deriv-App-ID": appId,
        "Content-Type": "application/json"
      }
    });

    const data = await r.json();
    if (!r.ok || !data?.data?.url) {
      return res.status(r.status || 502).json({
        error: "Deriv authentication failed.",
        details: data
      });
    }

    res.json({ wsUrl: data.data.url, accountType: "demo" });
  } catch (e) {
    res.status(502).json({ error: "Could not contact Deriv.", details: String(e) });
  }
});

app.get("*", (_req, res) => {
  res.sendFile(path.join(__dirname, "public", "index.html"));
});

app.listen(PORT, () => {
  console.log(`Deriv Digit Analyzer running at http://localhost:${PORT}`);
});
