<!DOCTYPE html>
<html lang="uk">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Binance Моніторинг Ордерів + Telegram</title>
  <style>
    body { font-family: system-ui, sans-serif; background:#111; color:#0f0; margin:0; padding:15px; }
    h1 { color:#0f8; text-align:center; margin:1em 0 0.6em; }
    input, button { display:block; width:100%; margin:10px 0; padding:12px; font-size:16px; box-sizing:border-box; border-radius:6px; }
    button { color:white; border:none; font-weight:bold; cursor:pointer; }
    button.green  { background:#0a0; }
    button.red    { background:#a00; }
    button.gray   { background:#444; }
    button:disabled { background:#333; cursor:not-allowed; opacity:0.6; }
    #log { background:#000; color:#0f0; padding:10px; height:45vh; overflow-y:auto; white-space:pre-wrap; font-family:monospace; font-size:13px; border:1px solid #333; border-radius:6px; margin-top:15px; }
    .hidden { display:none !important; }
    label { display:block; margin:12px 0 4px; font-size:14px; color:#ccc; }
    .btn-row { display:flex; gap:10px; margin:15px 0; }
    .btn-row button { flex:1; padding:14px; font-size:16px; }
  </style>
</head>
<body>

<div id="setup">
  <h1>Моніторинг відкритих ордерів (Futures)</h1>

  <label>API Key (Binance):</label>
  <input type="text" id="apiKey" placeholder="Встав API Key" autocomplete="off"/>

  <label>Secret Key (Binance):</label>
  <input type="password" id="secret" placeholder="Встав Secret Key" autocomplete="off"/>

  <label>Telegram Bot Token (опціонально):</label>
  <input type="text" id="botToken" placeholder="123456789:ABCdef..." autocomplete="off"/>

  <label>Telegram Chat ID (опціонально):</label>
  <input type="text" id="chatId" placeholder="123456789" autocomplete="off"/>

  <button id="startBtn" class="green">Запустити моніторинг</button>

  <p style="color:#888; font-size:0.85em; text-align:center; margin-top:20px;">
    Ключі зберігаються тільки в браузері.<br>
    <strong>Рекомендую read-only ключі!</strong><br>
    Для Telegram: створи бота в @BotFather, отримай Chat ID через getUpdates.
  </p>
</div>

<div id="monitor" class="hidden">
  <h1>Моніторинг активний</h1>

  <div class="btn-row">
    <button id="stopBtn" class="red">Зупинити</button>
    <button id="backBtn" class="gray">Повернутися</button>
  </div>

  <button id="clearBtn" class="gray">Очистити лог</button>

  <div id="log">Очікую оновлення... (кожні 10 секунд)</div>
</div>

<script>
// Правильна HMAC-SHA256 через Web Crypto API
async function hmacSHA256(key, data) {
  try {
    const encoder = new TextEncoder();
    const keyData = encoder.encode(key);
    const messageData = encoder.encode(data);

    const cryptoKey = await crypto.subtle.importKey(
      "raw",
      keyData,
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["sign"]
    );

    const signature = await crypto.subtle.sign("HMAC", cryptoKey, messageData);
    const hashArray = Array.from(new Uint8Array(signature));
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
  } catch (e) {
    console.error("HMAC помилка:", e);
    return 'error-signature';
  }
}

let intervalId = null;
let previousOrderIds = new Set(); // Для виявлення нових ордерів
const logEl = document.getElementById('log');
const setupDiv = document.getElementById('setup');
const monitorDiv = document.getElementById('monitor');

function log(msg) {
  const time = new Date().toLocaleTimeString('uk-UA', {hour12:false});
  logEl.textContent += `[${time}] ${msg}\n`;
  logEl.scrollTop = logEl.scrollHeight;
}

async function sendTelegramMessage(botToken, chatId, text) {
  if (!botToken || !chatId) return;
  try {
    const response = await fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ chat_id: chatId, text: text, parse_mode: 'HTML' })
    });
    if (response.ok) {
      log("Повідомлення в Telegram відправлено!");
    } else {
      log("Помилка Telegram: " + response.status);
    }
  } catch (err) {
    log("Помилка відправки в Telegram: " + err.message);
  }
}

document.getElementById('startBtn').onclick = async () => {
  const apiKey = document.getElementById('apiKey').value.trim();
  const secret = document.getElementById('secret').value.trim();
  const botToken = document.getElementById('botToken').value.trim();
  const chatId = document.getElementById('chatId').value.trim();

  if (!apiKey || !secret) {
    alert("Введи ключі Binance!");
    return;
  }

  localStorage.setItem('apiKey', apiKey);
  localStorage.setItem('secret', secret);
  localStorage.setItem('botToken', botToken);
  localStorage.setItem('chatId', chatId);

  setupDiv.classList.add('hidden');
  monitorDiv.classList.remove('hidden');

  previousOrderIds.clear();
  log("Запущено. Перше оновлення...");
  await checkOrders();
  intervalId = setInterval(checkOrders, 10000);
};

document.getElementById('stopBtn').onclick = () => {
  if (intervalId) clearInterval(intervalId);
  log("Моніторинг зупинено.");
};

document.getElementById('clearBtn').onclick = () => {
  logEl.textContent = "";
};

document.getElementById('backBtn').onclick = () => {
  if (intervalId) clearInterval(intervalId);
  localStorage.clear();
  setupDiv.classList.remove('hidden');
  monitorDiv.classList.add('hidden');
  logEl.textContent = "";
  document.getElementById('apiKey').value = "";
  document.getElementById('secret').value = "";
  document.getElementById('botToken').value = "";
  document.getElementById('chatId').value = "";
};

async function checkOrders() {
  const apiKey = localStorage.getItem('apiKey');
  const secret = localStorage.getItem('secret');
  const botToken = localStorage.getItem('botToken');
  const chatId = localStorage.getItem('chatId');

  if (!apiKey || !secret) return;

  try {
    const ts = Date.now();
    const query = `timestamp=${ts}`;
    const signature = await hmacSHA256(secret, query);

    const url = `https://fapi.binance.com/fapi/v1/openOrders?\( {query}&signature= \){signature}`;

    const resp = await fetch(url, {
      method: 'GET',
      headers: { 'X-MBX-APIKEY': apiKey }
    });

    if (!resp.ok) {
      const errText = await resp.text();
      log(`ПОМИЛКА Binance: HTTP ${resp.status} — ${errText}`);
      return;
    }

    const data = await resp.json();

    if (Array.isArray(data)) {
      const currentOrderIds = new Set(data.map(o => o.orderId));
      const newOrderIds = [...currentOrderIds].filter(id => !previousOrderIds.has(id));

      previousOrderIds = currentOrderIds;

      if (data.length === 0) {
        log("Відкритих ордерів немає");
      } else if (newOrderIds.length > 0) {
        log(`Нові ордери виявлено (${newOrderIds.length})`);
        data.filter(o => newOrderIds.includes(o.orderId)).forEach(o => {
          const price = o.price !== "0.00000000" ? o.price : o.type;
          const msg = `🔔 Новий ордер!\nСимвол: ${o.symbol}\nСторона: ${o.side}\nКількість: ${o.origQty}\nЦіна/Тип: ${price}\nСтатус: ${o.status}\nЧас: ${new Date(o.time).toLocaleString('uk-UA')}`;
          log(msg);
          sendTelegramMessage(botToken, chatId, msg);
        });
      } else {
        log(`Ордери без змін (${data.length} відкритих)`);
      }
    } else {
      log("Неочікувана відповідь: " + JSON.stringify(data));
    }
  } catch (err) {
    log("Критична помилка: " + err.message);
  }
}

// Автозапуск, якщо ключі збережені
if (localStorage.getItem('apiKey') && localStorage.getItem('secret')) {
  setupDiv.classList.add('hidden');
  monitorDiv.classList.remove('hidden');
  log("Збережені ключі знайдено — запускаю...");
  checkOrders();
  intervalId = setInterval(checkOrders, 10000);
}
</script>
</body>
</html>
