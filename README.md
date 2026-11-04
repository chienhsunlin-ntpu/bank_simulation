# Bank Manager — classroom simulation

A single-page game (`index.html`). Students manage Harborview Bank for N days. Each day they adjust the balance sheet, preview the result, and submit. Events arrive every few days, and depositor confidence can turn into a run. There is no server and no build step.

## 1. Put it on GitHub Pages

1. Create a new public repository, e.g. `bank-game`.
2. Upload `index.html` (and this README) to the repository root.
3. Go to **Settings → Pages → Build and deployment**, set *Source* to "Deploy from a branch", choose `main` / `(root)`, and click Save.
4. After a minute the game is live at `https://<your-username>.github.io/bank-game/`.

## 2. URL parameters (treatments)

Add these to the link you give students, e.g.
`https://<you>.github.io/bank-game/?class=MB2026A&insurance=none&news=fast`

| Parameter | Values | Default | What it changes |
|---|---|---|---|
| `class` | any text | empty | Pre-fills the class code (useful for sections/treatments) |
| `days` | 15–120 | 60 | Length of the game; event timing scales with it |
| `seed` | integer or `random` | 20261105 | Same seed = same events and same random draws for everyone |
| `insurance` | `standard`, `full`, `none` | `standard` | Share of deposits that are insured (500/300, 800/0, 0/800) |
| `cb` | `on`, `off` | `on` | Central bank lending (lender of last resort) available |
| `stigma` | `on`, `off` | `on` | Whether borrowing from the central bank hurts confidence |
| `news` | `slow`, `normal`, `fast` | `normal` | How strongly rumors and others' withdrawals spread |
| `submit` | Apps Script URL | none | Turns on the **Submit results** button (see below) |

Calibration check: 20 seeds × 3 scripted strategies (passive, sensible, aggressive). Deposit insurance, news speed and central bank lending each change survival and profit in the expected direction. Combining no insurance, fast news and no central bank produces runs even for the sensible strategy.

## 3. Optional: collect results in a Google Sheet

1. Create a Google Sheet. Open **Extensions → Apps Script**, delete the sample code, and paste:

```javascript
function doPost(e) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const d = JSON.parse(e.postData.contents);

  // one row per student game
  let res = ss.getSheetByName('results') || ss.insertSheet('results');
  if (res.getLastRow() === 0) res.appendRow(['received','student','name','class','attempt','seed','days','insurance','cb','stigma','news',
    'outcome','fail_day','profit','roe','score','fines','forced_loss','min_conf','breach_days','q1','q2','q3']);
  const s = d.summary, p = d.params;
  res.appendRow([new Date(), d.student, d.name, d.cls, d.attempt, p.seed, p.days, p.insurance, p.cb, p.stigma, p.news,
    s.outcome, s.failDay, s.profit, s.roe, s.score, s.fines, s.forcedLoss, s.minConf, s.breachDays,
    d.reflection.q1, d.reflection.q2, d.reflection.q3]);

  // one row per student-day (long format, ready for Stata)
  let daily = ss.getSheetByName('daily') || ss.insertSheet('daily');
  if (d.days.length) {
    const keys = Object.keys(d.days[0]);
    if (daily.getLastRow() === 0) daily.appendRow(['student','class','attempt'].concat(keys));
    const rows = d.days.map(r => [d.student, d.cls, d.attempt].concat(keys.map(k => r[k])));
    daily.getRange(daily.getLastRow()+1, 1, rows.length, rows[0].length).setValues(rows);
  }
  return ContentService.createTextOutput('ok');
}
```

2. Click **Deploy → New deployment**, choose type **Web app**, set *Execute as* to **Me** and *Who has access* to **Anyone**, then Deploy and authorize.
3. Copy the web-app URL (`https://script.google.com/macros/s/.../exec`) and add it to the student link:
   `...?class=MB2026A&submit=https://script.google.com/macros/s/XXXX/exec`

Notes:
- Because GitHub Pages is a different site, the browser cannot confirm receipt. Students also get a **Download my data** button (JSON) as a backup.
- The `daily` sheet has columns prefixed `a_` (the student's decisions), `r_` (what happened that day) and `s_` (end-of-day state: cash, loans, deposits, confidence, ratios). Export it as CSV and `import delimited` in Stata.
- Games are auto-saved in the student's browser, so closing the tab and coming back resumes the game.

## 4. Tuning

All economic parameters are in the `CFG` object near the top of the script (initial balance sheet, regulatory minimums, fines, fire-sale discounts, panic sensitivity). The scheduled events are in `EVENT_PLAN` (day out of 60, event type), and the text and choices for each event are in `EVENTS`.
