Deriv Digit Analyzer — V1–V7
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
See: https://developers.deriv.com/docs/data/ticks/ https://developers.deriv.com/docs/data/ticks-history/ https://developers.deriv.com/docs/workflows/ https://developers.deriv.com/docs/trading/buy/
