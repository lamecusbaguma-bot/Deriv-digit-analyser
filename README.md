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
Execution locked. Analysis and backtesting are available without credentials.:root{font-family:Inter,system-ui,Arial,sans-serif;background:#090b10;color:#eef1f6}
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
{
  "name": "deriv-digit-analyzer",
  "version": "1.0.0",
  "private": true,
  "description": "Deriv digit analysis, backtesting and guarded demo-execution dashboard.",
  "type": "module",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "express": "^5.1.0",
    "ws": "^8.18.3"
  }
}Deriv Digit Analyzer — V1–V7
This is a single application containing:
Live market-data connection
Multi-window digit analysis
Frequency / recent-frequency / transition scoring
WAIT / analysis signal filtering
Historical walk-forward backtesting
Risk/execution guard UI
Authenticated DEMO connection scaffold
Important
The model is a statistical ranking system, not a guaranteed predictor. Digit outcomes can behave unpredictably, and a high historical score does not guarantee future accuracy.
The supplied build intentionally does not place real-money trades. The authenticated route is restricted to a Deriv demo account and only obtains an authenticated WebSocket URL.
Requirements
Node.js 20+ recommended
Internet connection
Install
npm install
npm start
Open:
http://localhost:8787
Live analysis
No Deriv account token is needed for public tick data. The app connects to the current public WebSocket API and retrieves ticks_history plus the live ticks stream.
Optional authenticated DEMO connection
Copy .env.example to .env and fill:
DERIV_APP_ID=your_app_id
DERIV_PAT=your_pat
DERIV_ACCOUNT_ID=your_demo_account_id
DERIV_ACCOUNT_TYPE=demo
Then restart:
npm start
Keep PAT credentials on the server. Do not put them into public/app.js.
What to improve before any trading
Store prediction outcomes in a database.
Run large out-of-sample tests.
Separate model-training data from evaluation data.
Add payout-aware expected-value calculations.
Add calibration tests (Brier score / reliability curve).
Add session and daily loss limits.
Add an explicit demo execution adapter.
Only consider live execution after extensive demo validation.
API basis
The current Deriv documentation provides public ticks and ticks_history WebSocket endpoints for market data. Authenticated trading uses an authenticated WebSocket URL obtained through the OTP flow; buy is an authenticated operation.
See: https://developers.deriv.com/docs/data/ticks/ https://developers.deriv.com/docs/data/ticks-history/ https://developers.deriv.com/docs/workflows/ https://developers.deriv.com/docs/trading/buy/import express from "express";
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
