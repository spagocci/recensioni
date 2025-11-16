// Spagocci Bot 1.9 — versione limitata alla vetrina

require('dotenv').config();
const { Client, GatewayIntentBits } = require('discord.js');
const fs = require('fs');
const axios = require('axios');
const puppeteer = require('puppeteer');

/* =========================================================
 * (1) HEADER & CONFIG
 * =======================================================*/
console.clear();
console.log("\x1b[36m╔════════════════════════════════════════════════════════════════════╗");
console.log("║ 🤖  SPAGOCCI BOT 1.9 — VETRINA ONLY                              ║");
console.log("╚════════════════════════════════════════════════════════════════════╝\x1b[0m\n");

const LOG_FILE = 'log.txt';
const DATA_FILE = 'data.json';
const CACHE_FILE = 'cache.json';

const VETRINA_ID = '1433444235851468980';
const CHANNEL_IDS = [
  VETRINA_ID // Operazioni limitate al canale vetrina
];

const BROWSER_RESET_INTERVAL = 60 * 60 * 1000;
const CACHE_RESET_INTERVAL   = 60 * 60 * 1000;

fs.writeFileSync(LOG_FILE, `🗓️ Log avviato il ${new Date().toLocaleString()}\n\n`);

/* =========================================================
 * (2) LOGGING
 * =======================================================*/
const colors = { reset:"\x1b[0m", red:"\x1b[31m", green:"\x1b[32m", yellow:"\x1b[33m", cyan:"\x1b[36m", gray:"\x1b[90m" };
function log(type, msg) {
  const t = new Date().toLocaleTimeString();
  const prefix = { ok:'✅', warn:'⚠️', err:'❌', info:'ℹ️' }[type] || '';
  const color  = { ok:colors.green, warn:colors.yellow, err:colors.red, info:colors.cyan }[type] || colors.reset;
  console.log(`${colors.gray}[${t}]${colors.reset} ${color}${prefix} ${msg}${colors.reset}`);
  fs.appendFileSync(LOG_FILE, `[${new Date().toLocaleString()}] ${prefix} ${msg}\n`);
}
const info = (m)=>log('info', m);
const ok   = (m)=>log('ok', m);
const warn = (m)=>log('warn', m);
const err  = (m)=>log('err', m);

function loadJSON(file) {
  try { if (!fs.existsSync(file)) return []; return JSON.parse(fs.readFileSync(file)); }
  catch { return []; }
}
function saveJSON(file, data) { fs.writeFileSync(file, JSON.stringify(data, null, 2)); }

/* =========================================================
 * (3) UPLOAD SU GITHUB (coda + retry anti-409)
 * =======================================================*/
const GITHUB_REPO   = 'spagocci/recensioni';
const GITHUB_PATH   = 'data.json';
const GITHUB_BRANCH = 'main';
let __uploadQueue = Promise.resolve();

function b64OfFile(p) { return Buffer.from(fs.readFileSync(p)).toString('base64'); }

async function _getGitHubFileSha() {
  const token = process.env.GITHUB_TOKEN;
  const headers = {
    Authorization: `Bearer ${token}`,
    Accept: 'application/vnd.github+json',
    'X-GitHub-Api-Version': '2022-11-28'
  };
  const url = `https://api.github.com/repos/${GITHUB_REPO}/contents/${GITHUB_PATH}?ref=${encodeURIComponent(GITHUB_BRANCH)}`;
  try {
    const res = await axios.get(url, { headers });
    return res.data.sha;
  } catch (e) {
    if (e.response?.status === 404) return null;
    throw e;
  }
}
async function _putGitHubFile({ message, contentB64, sha }) {
  const token = process.env.GITHUB_TOKEN;
  if (!token) throw new Error('GITHUB_TOKEN mancante');
  const headers = {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
    Accept: 'application/vnd.github+json',
    'X-GitHub-Api-Version': '2022-11-28'
  };
  const url = `https://api.github.com/repos/${GITHUB_REPO}/contents/${GITHUB_PATH}`;
  return axios.put(url, { message, content: contentB64, sha, branch: GITHUB_BRANCH }, { headers });
}
async function uploadToGitHub(message = 'Aggiornamento automatico') {
  __uploadQueue = __uploadQueue.then(() => _uploadWithRetry(message)).catch(()=>{});
  return __uploadQueue;
}
async function _uploadWithRetry(message) {
  const contentB64 = b64OfFile(DATA_FILE);
  const maxRetries = 3;
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const sha = await _getGitHubFileSha();
      info(`☁️ Upload tentativo ${attempt}/${maxRetries} — sha=${sha ?? 'NEW'}`);
      await _putGitHubFile({ message, contentB64, sha });
      ok('☁️ Upload completato su GitHub.');
      return;
    } catch (e) {
      const status = e.response?.status;
      warn(`⚠️ Upload fallito (tentativo ${attempt}) — ${status || e.code || e.message}`);
      if (status === 409 || status === 422) { await new Promise(r => setTimeout(r, 1000 * (2 ** (attempt - 1)))); continue; }
      throw e;
    }
  }
  throw new Error('Errore upload GitHub dopo 3 tentativi');
}

/* =========================================================
 * (4) CACHE amzn.to
 * =======================================================*/
let linkCache = loadJSON(CACHE_FILE) || { _createdAt: Date.now(), data: {} };
function cacheReset(force=false) {
  if (force || (Date.now() - (linkCache._createdAt || 0)) > CACHE_RESET_INTERVAL) {
    linkCache = { _createdAt: Date.now(), data: {} };
    saveJSON(CACHE_FILE, linkCache);
    info('♻️ Cache amzn.to svuotata automaticamente (ogni 60 min)');
  }
}
setInterval(() => cacheReset(), 10 * 60 * 1000);

/* =========================================================
 * (5) UTILITIES LINK/ASIN
 * =======================================================*/
function extractASIN(str = '') {
  const m = str.match(/(?:^|\/)(?:dp|gp\/product)\/([A-Z0-9]{10})(?![A-Z0-9])/i);
  return m ? m[1].toUpperCase() : null;
}
function containsAmazonCom(text) { return !!text && /https?:\/\/[^ \n]*amazon\.com/i.test(text); }
async function expandAmznTo(shortUrl) {
  try {
    const res = await axios.head(shortUrl, { maxRedirects: 0, validateStatus: null });
    if (res.status >= 300 && res.status < 400 && res.headers.location) return res.headers.location;
  } catch (e) { warn(`⚠️ Errore expandAmznTo: ${e.message}`); }
  return shortUrl;
}

/* =========================================================
 * (6) RIMOZIONE ASIN persi.txt (solo vetrina)
 * =======================================================*/
async function rimuoviASINPersi(client) {
  const FILE_PERSI = 'persi.txt';
  if (!fs.existsSync(FILE_PERSI)) return info('🟢 Nessun file persi.txt trovato.');
  const asinPersi = fs.readFileSync(FILE_PERSI, 'utf8')
    .split(/\r?\n/).map(x => x.trim().toUpperCase())
    .filter(x => /^[A-Z0-9]{10}$/.test(x));
  if (asinPersi.length === 0) return info('🟢 persi.txt vuoto.');

  info(`📄 Rimozione ASIN da persi.txt (${asinPersi.length})`);
  let eliminatiDiscord = 0, eliminatiJSON = 0;

  for (const id of CHANNEL_IDS) {
    const channel = await client.channels.fetch(id).catch(() => null);
    if (!channel?.isTextBased()) continue;
    const msgs = await channel.messages.fetch({ limit: 100 });
    for (const m of msgs.values()) {
      const testo = [m.content, m.embeds?.[0]?.url, m.embeds?.[0]?.description, m.embeds?.[0]?.title].filter(Boolean).join(' ');
      const asin = extractASIN(testo);
      if (asin && asinPersi.includes(asin)) {
        await m.delete().catch(()=>{});
        eliminatiDiscord++; await new Promise(r=>setTimeout(r,200));
      }
    }
  }

  let data = loadJSON(DATA_FILE) || [];
  const prima = data.length;
  data = data.filter(p => {
    if (p.sourceChannel !== VETRINA_ID) return true;
    const asin = (p.asin || '').toUpperCase();
    return !asinPersi.includes(asin);
  });
  eliminatiJSON = prima - data.length;
  if (eliminatiJSON > 0) saveJSON(DATA_FILE, data);

  if (eliminatiDiscord > 0 || eliminatiJSON > 0)
    ok(`📦 Rimozione completata — Discord: ${eliminatiDiscord}, JSON: ${eliminatiJSON}`);
  else info('🟢 Nessun ASIN da rimuovere trovato.');
}

/* =========================================================
 * (7) ELIMINA amazon.com (solo vetrina)
 * =======================================================*/
async function eliminaAmazonCom(client) {
  let eliminatiDiscord = 0, eliminatiJSON = 0;
  let data = loadJSON(DATA_FILE);

  for (const id of CHANNEL_IDS) {
    const channel = await client.channels.fetch(id).catch(() => null);
    if (!channel?.isTextBased()) continue;
    const msgs = await channel.messages.fetch({ limit: 100 });
    for (const msg of msgs.values()) {
      const testo = [msg.content, msg.embeds?.[0]?.url, msg.embeds?.[0]?.description, msg.embeds?.[0]?.title]
        .filter(Boolean).join(' ');
      if (containsAmazonCom(testo)) { await msg.delete().catch(()=>{}); eliminatiDiscord++; }
    }
  }
  const prima = data.length;
  data = data.filter(p => {
    if (p.sourceChannel !== VETRINA_ID) return true;
    return !(p.link && p.link.includes('amazon.com'));
  });
  eliminatiJSON = prima - data.length;
  if (eliminatiJSON > 0) saveJSON(DATA_FILE, data);
  return { eliminatiDiscord, eliminatiJSON };
}

/* =========================================================
 * (8) PUPPETEER PREZZI
 * =======================================================*/
let browser = null;
let lastBrowserReset = Date.now();

async function initBrowser() {
  const now = Date.now();
  if (!browser || now - lastBrowserReset > BROWSER_RESET_INTERVAL) {
    if (browser) try { await browser.close(); } catch {}
    browser = await puppeteer.launch({ headless: true, args: ['--no-sandbox','--disable-setuid-sandbox'] });
    lastBrowserReset = now;
    ok('🚀 Puppeteer avviato o riavviato.');
  }
  return browser;
}
function parseEuro(text) {
  if (!text) return NaN;
  const cleaned = text.replace(/\u00A0/g, ' ').replace(/[^\d,.\s]/g, '').trim();
  let s = cleaned;
  if (s.includes('.') && s.includes(',')) s = s.replace(/\./g, '').replace(',', '.');
  else if (s.includes(',')) s = s.replace('.', '').replace(',', '.');
  return parseFloat(s);
}
function renderPrezzoHTML(priceValue) {
  const am  = Math.ceil(priceValue);
  const mio = Math.ceil(am / 2);
  return `<span style="color:green;font-weight:bold;font-size:1.5em;">Mio: ${mio}€</span><br><span style="color:red;font-weight:bold;">AM: ${am}€</span>`;
}
async function scrapePrezzoAmazon(asin, client) {
  const url = `https://www.amazon.it/dp/${asin}`;
  const browser = await initBrowser();
  const page = await browser.newPage();
  try {
    await page.setUserAgent('Mozilla/5.0 (Windows NT 10.0; Win64; x64)');
    let price = NaN;
    for (let i = 1; i <= 2 && isNaN(price); i++) {
      info(`🔎 Tentativo ${i}/2 per ${asin}...`);
      await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 25000 }).catch(()=>{});
      await new Promise(r => setTimeout(r, 1500));
      const body = await page.content();
      const match = body.match(/(\d{1,3}(?:[.,]\d{3})*[.,]\d{2})\s*€/);
      if (match) price = parseEuro(match[1]);
      if (isNaN(price) && i < 2) await new Promise(r => setTimeout(r, 2000));
    }

    if (isNaN(price)) {
      for (const id of CHANNEL_IDS) {
        const channel = await client.channels.fetch(id).catch(()=>null);
        if (!channel?.isTextBased()) continue;
        const msgs = await channel.messages.fetch({ limit: 100 });
        for (const m of msgs.values()) {
          const testo = [m.content, m.embeds?.[0]?.url, m.embeds?.[0]?.description, m.embeds?.[0]?.title].filter(Boolean).join(' ');
          const found = extractASIN(testo);
          if (found && found.toUpperCase() === asin.toUpperCase()) await m.delete().catch(()=>{});
        }
      }
      let data = loadJSON(DATA_FILE) || [];
      const prima = data.length;
      data = data.filter(p => {
        if (p.sourceChannel !== VETRINA_ID) return true;
        return (p.asin || '').toUpperCase() !== asin.toUpperCase();
      });
      if (data.length < prima) saveJSON(DATA_FILE, data);
      return null;
    }

    ok(`💰 Prezzo trovato per ${asin}: ${price.toFixed(2)}€`);
    return renderPrezzoHTML(price);
  } finally { await page.close(); }
}

/* =========================================================
 * (9) PRIORITÀ & PULIZIE (solo vetrina)
 * =======================================================*/
async function prioritaVetrina(client) {
  let data = loadJSON(DATA_FILE) || [];
  let eliminatiJSON = 0;

  const nuovaData = [];
  const visti = new Set();
  for (const p of data) {
    if (!p.asin || p.sourceChannel !== VETRINA_ID) {
      nuovaData.push(p);
      continue;
    }
    const asin = p.asin.toUpperCase();
    if (visti.has(asin)) { eliminatiJSON++; continue; }
    visti.add(asin);
    nuovaData.push(p);
  }
  if (eliminatiJSON > 0) saveJSON(DATA_FILE, nuovaData);
  return { eliminatiDiscord: 0, eliminatiJSON, aggiornatiJSON: 0 };
}
function pulisciJSONVetrina() {
  let data = loadJSON(DATA_FILE) || [];
  const prima = data.length;
  const nuovaData = [];
  const visti = new Set();
  for (const p of data) {
    if (!p.asin || p.sourceChannel !== VETRINA_ID) {
      nuovaData.push(p);
      continue;
    }
    const asin = p.asin.toUpperCase();
    if (visti.has(asin)) continue;
    visti.add(asin);
    nuovaData.push(p);
  }
  const eliminati = prima - nuovaData.length;
  if (eliminati > 0) saveJSON(DATA_FILE, nuovaData);
  return { eliminati };
}
function eliminaDuplicatiJSON() {
  let data = loadJSON(DATA_FILE) || [];
  const visti = new Set();
  const nuova = [];
  let eliminati = 0;
  for (const p of data) {
    if (!p.asin || p.sourceChannel !== VETRINA_ID) { nuova.push(p); continue; }
    const asin = (p.asin || '').toUpperCase();
    if (visti.has(asin)) { eliminati++; continue; }
    visti.add(asin); nuova.push(p);
  }
  if (eliminati > 0) saveJSON(DATA_FILE, nuova);
  return eliminati;
}

/* =========================================================
 * (9.5) COERENZA DISCORD→JSON (solo vetrina)
 * =======================================================*/
async function controllaRimozioniDiscord(client) {
  info('🔍 Controllo coerenza: verifica messaggi rimossi da Discord...');
  let data = loadJSON(DATA_FILE) || [];
  if (data.length === 0) return { rimossi: 0 };

  const esistenti = new Set();
  for (const channelId of CHANNEL_IDS) {
    const channel = await client.channels.fetch(channelId).catch(() => null);
    if (!channel?.isTextBased()) continue;
    const msgs = await channel.messages.fetch({ limit: 100 }).catch(() => null);
    if (!msgs) continue;
    for (const m of msgs.values()) esistenti.add(m.id);
  }

  const prima = data.length;
  const nuovaData = data.filter(p => {
    if (p.sourceChannel !== VETRINA_ID) return true;
    return esistenti.has(p.id);
  });
  const rimossi = prima - nuovaData.length;
  if (rimossi > 0) saveJSON(DATA_FILE, nuovaData);
  return { rimossi };
}

/* =========================================================
 * fetchAll + sync (solo vetrina)
 * =======================================================*/
async function fetchAllMessages(channel, max = 5000) {
  let allMessages = [];
  let lastId = null;
  while (true) {
    const options = { limit: 100 };
    if (lastId) options.before = lastId;
    const batch = await channel.messages.fetch(options).catch(()=>null);
    if (!batch || batch.size === 0) break;
    allMessages.push(...batch.values());
    lastId = batch.last().id;
    info(`📜 Letti ${allMessages.length} messaggi da ${channel.name || channel.id}`);
    if (allMessages.length >= max) break;
    await new Promise(r => setTimeout(r, 1000));
  }
  return allMessages.slice(0, max);
}
async function syncOldMessages(client) {
  info('🔄 Sincronizzazione messaggi Discord → data.json ...');
  let data = loadJSON(DATA_FILE) || [];
  const visti = new Set(data.filter(p => p.sourceChannel === VETRINA_ID).map(p => p.id));
  let duplicati = 0;
  let buffer = [];
  let totNuovi = 0;

  for (const channelId of CHANNEL_IDS) {
    const channel = await client.channels.fetch(channelId).catch(()=>null);
    if (!channel?.isTextBased()) continue;

    const messagesArray = await fetchAllMessages(channel, 5000);
    const msgs = new Map(messagesArray.map(m => [m.id, m]));
    info(`📥 Lettura ${msgs.size} messaggi da ${channel.name || channelId}`);

    for (const msg of msgs.values()) {
      const embed = msg.embeds?.[0];
      const testo = [msg.content, embed?.url, embed?.description, embed?.title].filter(Boolean).join(' ');
      if (!/amazon\.|amzn\.to/i.test(testo)) continue;
      if (/amazon\.com/i.test(testo)) continue;

      const linkMatch = testo.match(/https?:\/\/[^\s)>\]]+/i);
      if (!linkMatch) continue;
      let link = linkMatch[0];
      if (link.includes('amzn.to')) link = await expandAmznTo(link);
      link = link.replace('amazon.com', 'amazon.it');

      const asin = extractASIN(link);
      if (!asin) continue;
      if (visti.has(msg.id) || data.some(x => x.asin === asin && x.sourceChannel === VETRINA_ID)) { duplicati++; continue; }

      buffer.push({
        id: msg.id,
        timestamp: msg.createdAt?.toISOString() || new Date().toISOString(),
        text: embed?.title || msg.content || '',
        link,
        image: msg.attachments.first()?.url || embed?.thumbnail?.url || '',
        price: null,
        asin,
        sourceChannel: channelId
      });
      visti.add(msg.id);
      totNuovi++;
      if (buffer.length >= 10) { data.push(...buffer); saveJSON(DATA_FILE, data); buffer = []; }
    }
  }
  if (buffer.length > 0) { data.push(...buffer); saveJSON(DATA_FILE, data); }
  if (totNuovi > 0) ok(`✅ Sync completata: ${totNuovi} nuovi messaggi.`);
  else info(`🟢 Nessun nuovo messaggio. Duplicati ignorati: ${duplicati}.`);
  return { modifiche: totNuovi > 0, totNuovi, duplicati };
}

/* =========================================================
 * (11) CICLO
 * =======================================================*/
let cicloAttivo = false;
const client = new Client({ intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages, GatewayIntentBits.MessageContent] });

client.once('ready', async () => {
  ok(`Bot attivo come ${client.user.tag}`);
  const check = await controllaRimozioniDiscord(client);
  if (check.rimossi > 0) ok(`🔁 Rimossi dal JSON: ${check.rimossi}`);
  const elim = eliminaDuplicatiJSON();
  if (elim > 0) await uploadToGitHub('Dedup iniziale data.json');
  await cicloDinamico();
});

async function cancellaMessaggiVecchi(client){
  const limite = Date.now() - 24*60*60*1000;
  for (const id of CHANNEL_IDS) {
    const ch = await client.channels.fetch(id).catch(()=>null);
    if (!ch?.isTextBased()) continue;
    const msgs = await ch.messages.fetch({ limit: 100 });
    let rimossi = 0;
    for (const m of msgs.values()) {
      if (m.createdTimestamp < limite && m.deletable) { await m.delete().catch(()=>{}); rimossi++; }
    }
    if (rimossi>0) warn(`🧹 Rimossi ${rimossi} msg vecchi in ${id}`);
  }
}
async function cicloControlloAutomatico(){
  if (cicloAttivo) { info('⏳ Ciclo già attivo, salto.'); return; }
  cicloAttivo = true;
  const start = Date.now();
  let uploadEseguitoQuestoCiclo = false;

  const uploadCiclo = async (message) => {
    await uploadToGitHub(message);
    uploadEseguitoQuestoCiclo = true;
  };
  try {
    info('🔄 Inizio ciclo automatico...');
    cacheReset();

    await rimuoviASINPersi(client);
    await cancellaMessaggiVecchi(client);
    await eliminaAmazonCom(client);
    await controllaRimozioniDiscord(client);

    const syncRes = await syncOldMessages(client);
    const priorita = await prioritaVetrina(client);
    const pulizia = pulisciJSONVetrina();
    if (pulizia.eliminati > 0) ok(`♻️ Pulizia JSON: -${pulizia.eliminati}`);

    let data = loadJSON(DATA_FILE) || [];
    let assegnati = 0, rimossiNoPrezzo = 0;
    const targets = data.filter(p => p.sourceChannel === VETRINA_ID && !p.price);
    const batchSize = 30;
    let prezziDaUltimoUpload = 0;

    for (let i = 0; i < targets.length; i += batchSize) {
      const batch = targets.slice(i, i + batchSize);
      info(`📦 Controllo prezzi (batch ${i + 1}-${i + batch.length} di ${targets.length})...`);
      for (const p of batch) {
        const prezzoHTML = await scrapePrezzoAmazon(p.asin, client);
        if (prezzoHTML) {
          p.price = prezzoHTML;
          assegnati++; prezziDaUltimoUpload++; saveJSON(DATA_FILE, data);
        } else {
          rimossiNoPrezzo++;
          data = loadJSON(DATA_FILE) || [];
        }

        if (prezziDaUltimoUpload >= 30) {
          await uploadCiclo('Prezzi: upload ogni 30 trovati');
          prezziDaUltimoUpload = 0;
          await new Promise(r => setTimeout(r, 3000));
        }
      }
      await new Promise(r => setTimeout(r, 10_000));
    }
    if (prezziDaUltimoUpload > 0) await uploadCiclo('Prezzi: upload finale residui');

    const ciSonoCambiamenti =
      (syncRes?.totNuovi || 0) > 0 ||
      (priorita?.eliminatiDiscord || 0) > 0 ||
      (priorita?.eliminatiJSON || 0) > 0 ||
      (priorita?.aggiornatiJSON || 0) > 0 ||
      (pulizia.eliminati || 0) > 0 ||
      assegnati > 0 || rimossiNoPrezzo > 0;

    if (ciSonoCambiamenti && !uploadEseguitoQuestoCiclo) await uploadToGitHub('Upload finale ciclo');
    const dur = Math.round((Date.now() - start) / 1000);
    ok(`🧾 Riepilogo — prezzi: +${assegnati}, rimossi no-prezzo: ${rimossiNoPrezzo}, durata ${dur}s`);
  } catch (e) {
    err(`Errore ciclo: ${e.message}`);
  } finally {
    cicloAttivo = false;
    info('✅ Ciclo terminato.\n');
  }
}
async function cicloDinamico(){
  try { await cicloControlloAutomatico(); setTimeout(cicloDinamico, 60_000); }
  catch(e){ err(`Errore nel ciclo dinamico: ${e.message}`); setTimeout(cicloDinamico, 30_000); }
}

client.login(process.env.DISCORD_TOKEN);

/* =========================================================
 * (13) HELPERS PREZZO MANUALE + PULIZIA JSON CANALE
 * =======================================================*/
function isValidASIN(a) { return /^[A-Z0-9]{10}$/.test(String(a || '').toUpperCase()); }
function setManualPriceForASIN(asin, prezzo) {
  asin = String(asin || '').toUpperCase().trim();
  if (!isValidASIN(asin)) throw new Error('ASIN non valido.');
  const priceStr = String(prezzo || '').trim();
  if (!priceStr) throw new Error('Prezzo mancante.');

  let data = loadJSON(DATA_FILE) || [];
  let changed = 0;
  for (const p of data) {
    if (p.sourceChannel !== VETRINA_ID) continue;
    if ((p.asin || '').toUpperCase() === asin) { p.price = priceStr; changed++; }
  }
  if (changed > 0) saveJSON(DATA_FILE, data);
  return changed;
}
async function rimuoviDaJSONPerCanale(channelId) {
  let data = loadJSON(DATA_FILE) || [];
  const prima = data.length;
  data = data.filter(p => p.sourceChannel !== channelId);
  const rimossi = prima - data.length;
  if (rimossi > 0) saveJSON(DATA_FILE, data);
  return rimossi;
}

/* =========================================================
 * (14) HANDLER 1 — comandi principali + !d
 * =======================================================*/
client.on('messageCreate', async (msg) => {
  if (!msg.content.startsWith('!')) return;
  if (msg.author.bot) return;

  const comando = msg.content.trim();
  const lower = comando.toLowerCase();

  // ☁️ Upload manuale di data.json su GitHub
  if (lower === '!d') {
    await msg.reply('☁️ Upload di `data.json` in corso su GitHub…');
    try {
      await uploadToGitHub('Upload manuale via !d');
      await msg.reply('✅ Upload completato con successo.');
    } catch (e) {
      await msg.reply(`❌ Errore durante l’upload: ${e.response?.status || e.message}`);
    }
    return;
  }

  if (lower === '!i') {
    await msg.reply('☁️ Upload manuale in corso su GitHub...');
    try { await uploadToGitHub('Upload manuale via !i'); await msg.reply('✅ Upload completato con successo.'); }
    catch (e) { await msg.reply(`❌ Errore durante l’upload: ${e.response?.status || e.message}`); }
    return;
  }

  if (lower.startsWith('!pr')) {
    const parts = comando.split(/\s+/);
    const asin = parts[1]?.toUpperCase();
    const prezzo = comando.replace(/^!pr\s+\S+\s*/i, '');
    if (!asin || !isValidASIN(asin) || !prezzo) { await msg.reply('Uso: `!pr <ASIN> <prezzo>` es. `!pr B0C123ABCD € 12,99`'); return; }
    try {
      const n = setManualPriceForASIN(asin, prezzo);
      if (n === 0) await msg.reply(`⚠️ ASIN **${asin}** non trovato nella vetrina.`);
      else { await uploadToGitHub(`Prezzo manuale ${asin}`); await msg.reply(`✅ Prezzo manuale impostato per **${asin}** su ${n} record della vetrina.`); }
    } catch (e) { await msg.reply(`❌ Errore: ${e.message}`); }
    return;
  }

  if (lower === '!ct') {
    await msg.reply('🧹 Inizio cancellazione di tutti i messaggi in vetrina...');
    let totRimossi = 0;
    for (const id of CHANNEL_IDS) {
      try {
        const channel = await client.channels.fetch(id);
        if (!channel?.isTextBased()) continue;
        const msgs = await channel.messages.fetch({ limit: 100 });
        for (const m of msgs.values()) { if (m.deletable) await m.delete().catch(()=>{}); await new Promise(r=>setTimeout(r,200)); }
        const rimossi = await rimuoviDaJSONPerCanale(id);
        totRimossi += rimossi;
      } catch (e) { warn(`⚠️ Errore pulizia canale ${id}: ${e.message}`); }
    }
    if (totRimossi > 0) await uploadToGitHub('Pulizia vetrina via !ct');
    await msg.reply(`✅ Pulizia completata. Rimossi ${totRimossi} record dal JSON della vetrina.`);
    return;
  }

  if (lower === '!cq') {
    await msg.reply(`🧽 Pulizia dei messaggi in ${msg.channel.name} in corso...`);
    try {
      const msgs = await msg.channel.messages.fetch({ limit: 100 });
      for (const m of msgs.values()) { if (m.deletable) await m.delete().catch(()=>{}); await new Promise(r=>setTimeout(r,200)); }
      const rimossi = await rimuoviDaJSONPerCanale(msg.channel.id);
      if (rimossi > 0) await uploadToGitHub('Pulizia vetrina via !cq');
      await msg.channel.send(`✅ Pulizia completata (${rimossi} record rimossi dal JSON della vetrina).`);
    } catch (e) { err(`Errore pulizia canale ${msg.channel.name}: ${e.message}`); }
    return;
  }

  if (lower === '!td') {
    await msg.reply('🔍 Rimozione dei duplicati nella vetrina in corso...');
    let totEliminati = 0;
    const visti = new Set();
    for (const id of CHANNEL_IDS) {
      try {
        const channel = await client.channels.fetch(id);
        if (!channel?.isTextBased()) continue;
        const msgs = await channel.messages.fetch({ limit: 100 });
        const ordinati = [...msgs.values()].sort((a, b) => a.createdTimestamp - b.createdTimestamp);
        for (const m of ordinati) {
          const testo = [m.content, m.embeds?.[0]?.url, m.embeds?.[0]?.description, m.embeds?.[0]?.title].filter(Boolean).join(' ');
          const asin = extractASIN(testo);
          if (!asin) continue;
          if (visti.has(asin)) { await m.delete().catch(()=>{}); totEliminati++; await new Promise(r=>setTimeout(r,200)); }
          else { visti.add(asin); }
        }
      } catch (e) { warn(`⚠️ Errore durante la rimozione duplicati in ${id}: ${e.message}`); }
    }
    await msg.reply(totEliminati > 0 ? `✅ Rimossi ${totEliminati} messaggi duplicati dalla vetrina.` : '🟢 Nessun duplicato trovato in vetrina.');
    return;
  }
});

/* =========================================================
 * (15) HANDLER 2 — include di nuovo !d + altri (solo vetrina)
 * =======================================================*/
client.on('messageCreate', async (msg) => {
  if (!msg.content.startsWith('!')) return;
  if (msg.author.bot) return;

  const comando = msg.content.trim();
  const lower = comando.toLowerCase();

  // ☁️ Upload manuale di data.json su GitHub
  if (lower === '!d') {
    await msg.reply('☁️ Upload di `data.json` in corso su GitHub…');
    try {
      await uploadToGitHub('Upload manuale via !d');
      await msg.reply('✅ Upload completato con successo.');
    } catch (e) {
      await msg.reply(`❌ Errore durante l’upload: ${e.response?.status || e.message}`);
    }
    return;
  }

  if (lower === '!i') {
    await msg.reply('☁️ Upload manuale in corso su GitHub...');
    try { await uploadToGitHub('Upload manuale via !i'); await msg.reply('✅ Upload completato con successo.'); }
    catch (e) { await msg.reply(`❌ Errore durante l’upload: ${e.response?.status || e.message}`); }
    return;
  }

  if (lower === '!ctnov') {
    await msg.reply('🧹 Pulizia della vetrina in corso...');
    for (const id of CHANNEL_IDS) {
      if (id === VETRINA_ID) continue;
    }
    await msg.reply('✅ Pulizia completata (nessun altro canale gestito).');
    return;
  }

  if (lower === '!td') {
    await msg.reply('🔍 Rimozione dei duplicati (priorità vetrina) in corso...');
    let totEliminati = 0;
    const visti = new Map();
    try {
      const vetrina = await client.channels.fetch(VETRINA_ID);
      if (vetrina?.isTextBased()) {
        const msgs = await vetrina.messages.fetch({ limit: 100 });
        for (const m of msgs.values()) {
          const testo = [m.content, m.embeds?.[0]?.url, m.embeds?.[0]?.description, m.embeds?.[0]?.title].filter(Boolean).join(' ');
          const asin = extractASIN(testo); if (asin) visti.set(asin, VETRINA_ID);
        }
      }
    } catch (e) { warn(`⚠️ Errore lettura vetrina: ${e.message}`); }

    for (const id of CHANNEL_IDS) {
      const channel = await client.channels.fetch(id).catch(()=>null);
      if (!channel?.isTextBased()) continue;
      const msgs = await channel.messages.fetch({ limit: 100 });
      const ordinati = [...msgs.values()].sort((a,b)=>a.createdTimestamp - b.createdTimestamp);
      for (const m of ordinati) {
        const testo = [m.content, m.embeds?.[0]?.url, m.embeds?.[0]?.description, m.embeds?.[0]?.title].filter(Boolean).join(' ');
        const asin = extractASIN(testo); if (!asin) continue;
        const first = visti.get(asin);
        if (!first) visti.set(asin, id);
        else if (id === VETRINA_ID) { await m.delete().catch(()=>{}); totEliminati++; await new Promise(r=>setTimeout(r,300)); }
      }
    }
    await msg.reply(totEliminati > 0 ? `✅ Rimossi ${totEliminati} duplicati dalla vetrina.` : '🟢 Nessun duplicato trovato.');
    return;
  }

  if (lower.startsWith('!pr')) {
    const parts = comando.split(/\s+/);
    const asin = parts[1]?.toUpperCase();
    const prezzo = comando.replace(/^!pr\s+\S+\s*/i, '');
    if (!asin || !isValidASIN(asin) || !prezzo) { await msg.reply('Uso: `!pr <ASIN> <prezzo>` es. `!pr B0C123ABCD € 12,99`'); return; }
    try {
      const n = setManualPriceForASIN(asin, prezzo);
      if (n === 0) await msg.reply(`⚠️ ASIN **${asin}** non trovato nella vetrina.`);
      else { await uploadToGitHub(`Prezzo manuale ${asin}`); await msg.reply(`✅ Prezzo manuale impostato per **${asin}** su ${n} record della vetrina.`); }
    } catch (e) { await msg.reply(`❌ Errore: ${e.message}`); }
    return;
  }

  if (lower === '!ct') {
    await msg.reply('🧹 Inizio cancellazione di tutti i messaggi in vetrina...');
    for (const id of CHANNEL_IDS) {
      try {
        const channel = await client.channels.fetch(id);
        if (!channel?.isTextBased()) continue;
        const msgs = await channel.messages.fetch({ limit: 100 });
        for (const m of msgs.values()) { if (m.deletable) await m.delete().catch(()=>{}); await new Promise(r=>setTimeout(r,300)); }
      } catch (e) { warn(`⚠️ Errore pulizia canale ${id}: ${e.message}`); }
    }
    await msg.reply('✅ Pulizia completa della vetrina terminata.');
    return;
  }

  if (lower === '!cq') {
    await msg.reply(`🧽 Pulizia dei messaggi in ${msg.channel.name} in corso...`);
    try {
      const msgs = await msg.channel.messages.fetch({ limit: 100 });
      for (const m of msgs.values()) { if (m.deletable) await m.delete().catch(()=>{}); await new Promise(r=>setTimeout(r,200)); }
      await msg.channel.send('✅ Pulizia completata in questo canale.');
    } catch (e) { err(`Errore pulizia canale ${msg.channel.name}: ${e.message}`); }
    return;
  }
});
