/**
 * GRAWDAYTRADER — 24/7 Telegram Flip Alert Bot
 * Works even when you're offline / chart closed
 * 
 * Setup:
 * 1. Create bot: Open Telegram → search @BotFather → /newbot → name it → copy TOKEN
 * 2. Get your Chat ID: Search @userinfobot on Telegram → start → it gives your ID (e.g. 123456789)
 * 3. Deploy this file to Render.com / Railway.app / VPS (free tier)
 * 4. Set env vars: TELEGRAM_TOKEN and TELEGRAM_CHAT_ID
 * 
 * This bot polls Binance every 10 seconds for your Top 100 coins,
 * calculates live 20-candle range on GLOBAL TF (default 15m),
 * and sends Telegram when GREEN or RED flip happens.
 * 
 * Node 18+ required. Run: npm install node-fetch@2 && node telegram-flip-bot.js
 */

const TELEGRAM_TOKEN = process.env.TELEGRAM_TOKEN || 'PUT_YOUR_BOT_TOKEN_HERE';
const TELEGRAM_CHAT_ID = process.env.TELEGRAM_CHAT_ID || 'PUT_YOUR_CHAT_ID_HERE';
const GLOBAL_TF = process.env.GLOBAL_TF || '15m'; // change to 1h, 5m, etc
const TOP_N = parseInt(process.env.TOP_N || '100');
const POLL_INTERVAL_MS = parseInt(process.env.POLL_INTERVAL_MS || '10000'); // 10s

const FAPI = 'https://fapi.binance.com';
const SKIP = /DOWN|UP|BEAR|BULL|_[0-9]/;

let lastZones = {}; // symbol -> zone
let flipHistory = {}; // to avoid spam: symbol -> last alert timestamp

async function get(url){
  try{
    const res = await fetch(url);
    if(!res.ok) throw new Error('HTTP '+res.status);
    return await res.json();
  }catch(e){
    console.error('Fetch fail', url, e.message);
    return null;
  }
}

function computeLiveZone(klines){
  if(!klines || klines.length < 21) return null;
  const candles = klines.map(k=>({high: parseFloat(k[2]), low: parseFloat(k[3]), close: parseFloat(k[4])}));
  const period = 20;
  const slice = candles.slice(candles.length-period-1, candles.length-1);
  const rollHigh = Math.max(...slice.map(c=>c.high));
  const rollLow = Math.min(...slice.map(c=>c.low));
  const rollRange = rollHigh - rollLow;
  if(rollRange===0) return null;
  const last = candles[candles.length-1];
  const rPos = (last.close - rollLow)/rollRange;
  const zone = rPos>0.65 ? 'GREEN' : rPos<0.35 ? 'RED' : 'MID';
  return {zone, rPos, price: last.close};
}

async function sendTelegram(text){
  if(TELEGRAM_TOKEN.includes('PUT_')) {
    console.log('[DRY RUN - No token] Would send:', text);
    return;
  }
  const url = `https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage`;
  try{
    const res = await fetch(url, {
      method: 'POST',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify({
        chat_id: TELEGRAM_CHAT_ID,
        text: text,
        parse_mode: 'Markdown',
        disable_web_page_preview: true
      })
    });
    const data = await res.json();
    if(!data.ok) console.error('Telegram error', data);
    else console.log('✅ Telegram sent:', text.slice(0,80));
  }catch(e){
    console.error('Telegram send fail', e.message);
  }
}

async function checkFlips(){
  console.log(`\n[${new Date().toLocaleTimeString()}] Checking ${TOP_N} coins @ ${GLOBAL_TF}...`);
  const tickers = await get(FAPI+'/fapi/v1/ticker/24hr');
  if(!tickers) return;
  const pool = tickers.filter(t=>t.symbol.endsWith('USDT') && !SKIP.test(t.symbol))
                      .sort((a,b)=>parseFloat(b.quoteVolume)-parseFloat(a.quoteVolume))
                      .slice(0, TOP_N);
  
  // Batch fetch klines in chunks of 10 to avoid rate limit
  const chunks = [];
  for(let i=0;i<pool.length;i+=10) chunks.push(pool.slice(i,i+10));
  
  for(const chunk of chunks){
    await Promise.all(chunk.map(async (t)=>{
      const sym = t.symbol;
      const klines = await get(FAPI+`/fapi/v1/klines?symbol=${sym}&interval=${GLOBAL_TF}&limit=21`);
      const live = computeLiveZone(klines);
      if(!live) return;
      
      const prev = lastZones[sym];
      if(prev && prev!==live.zone && live.zone!=='MID'){
        // FLIP DETECTED
        const now = Date.now();
        const lastAlert = flipHistory[sym] || 0;
        if(now - lastAlert < 5*60*1000){ // 5 min cooldown per coin to avoid spam
          console.log(`⏭️ Skip ${sym} — alerted recently`);
          return;
        }
        flipHistory[sym] = now;
        
        const base = sym.replace('USDT','');
        const emoji = live.zone==='GREEN' ? '🟢' : '🔴';
        const side = live.zone==='GREEN' ? 'LONG ZONE' : 'SHORT ZONE';
        const rPosPct = (live.rPos*100).toFixed(0);
        const chg = parseFloat(t.priceChangePercent).toFixed(2);
        
        const msg = `${emoji} *${base} FLIP ${live.zone}* @ ${GLOBAL_TF}\n`+
                    `Price: $${live.price}\n`+
                    `Live Range: ${rPosPct}% → ${side}\n`+
                    `24h: ${chg}% | Vol $${(parseFloat(t.quoteVolume)/1e6).toFixed(1)}M\n`+
                    `Time: ${new Date().toLocaleString()}\n`+
                    `View chart: https://fapi.binance.com (search ${sym})`;
        
        console.log(`🚨 FLIP ${sym} ${prev} -> ${live.zone} ${rPosPct}%`);
        await sendTelegram(msg);
      }
      lastZones[sym] = live.zone;
    }));
    // small delay between chunks to respect Binance rate limit
    await new Promise(r=>setTimeout(r, 800));
  }
  console.log(`Done. Tracking ${Object.keys(lastZones).length} zones.`);
}

async function main(){
  if(TELEGRAM_TOKEN.includes('PUT_')){
    console.log('⚠️ No Telegram token set — running in DRY RUN mode (logs only). Set env vars to enable alerts.');
  } else {
    await sendTelegram(`🚀 *GRAWDAYTRADER Bot Started*\nGlobal TF: ${GLOBAL_TF}\nTop N: ${TOP_N}\nPoll: ${POLL_INTERVAL_MS/1000}s\nYou will get alerts even when offline.`);
  }
  
  // Initial scan
  await checkFlips();
  
  // Loop
  setInterval(checkFlips, POLL_INTERVAL_MS);
  
  // Keep alive ping every hour
  setInterval(async ()=>{
    console.log('[Keepalive] Bot alive at', new Date().toISOString());
  }, 60*60*1000);
}

main().catch(console.error);
