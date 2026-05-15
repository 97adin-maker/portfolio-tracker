/**
 * fetch-prices.js
 * Runs via GitHub Actions every 15 minutes.
 * Fetches XETRA stock prices from Yahoo Finance → saves to Firebase Realtime Database.
 *
 * To add/remove stocks: edit the SYMBOLS array below.
 */

const admin = require('firebase-admin');

// ── Stocks to track ──────────────────────────────────────────────────────────
// These are XETRA tickers — .DE is added automatically
const SYMBOLS = ['SAP', 'BMW', 'ALV', 'SIE', 'VWCE', 'VOW3', 'ADS', 'MBG'];

// ── Firebase setup ───────────────────────────────────────────────────────────
const serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT);
const databaseURL    = process.env.FIREBASE_DATABASE_URL;

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
  databaseURL,
});

const db = admin.database();

// ── Fetch one quote from Yahoo Finance ───────────────────────────────────────
async function fetchQuote(symbol) {
  const yahooSymbol = symbol + '.DE';
  const url = `https://query1.finance.yahoo.com/v8/finance/chart/${yahooSymbol}?interval=1d&range=1d`;

  const res = await fetch(url, {
    headers: {
      'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
      'Accept': 'application/json',
      'Referer': 'https://finance.yahoo.com/',
    },
  });

  if (!res.ok) throw new Error(`HTTP ${res.status} for ${yahooSymbol}`);

  const data   = await res.json();
  const result = data?.chart?.result?.[0];
  if (!result) throw new Error(`No data for ${yahooSymbol}`);

  const m    = result.meta;
  const prev = m.chartPreviousClose ?? m.previousClose ?? 0;
  const chg  = m.regularMarketPrice - prev;

  return {
    symbol:           m.symbol,
    name:             symbol,
    price:            m.regularMarketPrice,
    currency:         m.currency || 'EUR',
    change:           parseFloat(chg.toFixed(4)),
    changePercent:    prev ? parseFloat(((chg / prev) * 100).toFixed(4)) : 0,
    open:             m.regularMarketOpen        || null,
    high:             m.regularMarketDayHigh     || null,
    low:              m.regularMarketDayLow      || null,
    previousClose:    prev,
    volume:           m.regularMarketVolume      || null,
    fiftyTwoWeekHigh: m.fiftyTwoWeekHigh         || null,
    fiftyTwoWeekLow:  m.fiftyTwoWeekLow          || null,
    exchangeName:     m.fullExchangeName         || 'XETRA',
    fetchedAt:        new Date().toISOString(),
  };
}

// ── Main ─────────────────────────────────────────────────────────────────────
async function main() {
  console.log(`Fetching ${SYMBOLS.length} symbols…`);

  const results = {};
  for (const sym of SYMBOLS) {
    try {
      results[sym] = await fetchQuote(sym);
      console.log(`  ✓ ${sym}: €${results[sym].price}`);
    } catch (err) {
      console.warn(`  ✗ ${sym}: ${err.message}`);
      results[sym] = { error: err.message, fetchedAt: new Date().toISOString() };
    }
    // Small delay to avoid rate limiting
    await new Promise(r => setTimeout(r, 300));
  }

  // Save to Firebase Realtime Database
  await db.ref('prices').set(results);
  await db.ref('lastUpdated').set(new Date().toISOString());

  console.log('Saved to Firebase ✓');
  process.exit(0);
}

main().catch(err => {
  console.error('Fatal:', err.message);
  process.exit(1);
});
