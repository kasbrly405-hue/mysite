# mysite
mysite
cd /home && mkdir -p relay_site && cd relay_site

cat > server.js <<'EOF'
import express from "express";
import fs from "fs";
import path from "path";
import fetch from "node-fetch";

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;
const DISCORD_WEBHOOK = process.env.DISCORD_WEBHOOK || "";

const LOG_FILE = path.join(process.cwd(), "logs.json");
if (!fs.existsSync(LOG_FILE)) fs.writeFileSync(LOG_FILE, "[]", "utf8");

function appendLog(entry) {
  const logs = JSON.parse(fs.readFileSync(LOG_FILE, "utf8"));
  logs.push(entry);
  fs.writeFileSync(LOG_FILE, JSON.stringify(logs, null, 2), "utf8");
}

app.post("/api/relay", async (req, res) => {
  try {
    const payload = req.body || {};
    const entry = {
      time: new Date().toISOString(),
      source: payload.source || "unknown",
      message: payload.message || "",
      raw: payload
    };
    appendLog(entry);

    if (DISCORD_WEBHOOK) {
      try {
        await fetch(DISCORD_WEBHOOK, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ content: `[${entry.time}] ${entry.source}: ${entry.message}` })
        });
      } catch (e) { }
    }

    res.json({ ok: true });
  } catch (e) {
    res.status(500).json({ ok: false, error: e.message });
  }
});

app.get("/api/logs", (req, res) => {
  try {
    const logs = JSON.parse(fs.readFileSync(LOG_FILE, "utf8"));
    res.json({ ok: true, logs });
  } catch (e) {
    res.status(500).json({ ok: false, error: e.message });
  }
});

app.get("/", (req, res) => {
  res.sendFile(path.join(process.cwd(), "public", "index.html"));
});

app.use(express.static(path.join(process.cwd(), "public")));

app.listen(PORT, () => console.log(`Relay site running on port ${PORT}`));
EOF

cat > package.json <<'EOF'
{
  "name": "relay_site",
  "type": "module",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "node-fetch": "^2.6.7"
  }
}
EOF

mkdir -p public

cat > public/index.html <<'EOF'
<!doctype html>
<html lang="ar">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Relay Logs</title>
  <style>
    body{font-family:system-ui,Arial,Helvetica,sans-serif;padding:16px;direction:rtl}
    .log{border-bottom:1px solid #ddd;padding:8px 0}
    .time{color:#555;font-size:12px}
    pre{white-space:pre-wrap;word-break:break-word;margin:6px 0}
    #controls{margin-bottom:12px}
    button{padding:6px 10px;border-radius:6px;border:1px solid #ccc;background:#f7f7f7;cursor:pointer}
  </style>
</head>
<body>
  <h1>سجلات الـ Relay</h1>
  <div id="controls">
    <button id="refresh">تحديث</button>
    <button id="clear">مسح السجلات محليًا</button>
  </div>
  <div id="logs">جارٍ التحميل...</div>
  <script src="app.js"></script>
</body>
</html>
EOF

cat > public/app.js <<'EOF'
async function loadLogs(){
  try{
    const r = await fetch("/api/logs");
    const j = await r.json();
    const container = document.getElementById("logs");
    if(!j.ok){ container.textContent = "خطأ"; return; }
    if(j.logs.length === 0){ container.textContent = "لا توجد سجلات بعد."; return; }
    container.innerHTML = j.logs.map(l=>`<div class="log"><div class="time">${l.time} — ${escapeHtml(l.source)}</div><pre>${escapeHtml(l.message)}</pre></div>`).join("");
  }catch(e){
    document.getElementById("logs").textContent = "فشل الحصول على السجلات";
  }
}
function escapeHtml(s){ return (s+"").replace(/[&<>"']/g, c=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;","'":"&#39;"})[c]); }
document.getElementById("refresh").addEventListener("click", loadLogs);
document.getElementById("clear").addEventListener("click", async ()=>{
  try{
    await fetch("/api/clear-logs", { method: "POST" });
  }catch(e){}
  loadLogs();
});
loadLogs();
setInterval(loadLogs, 5000);
EOF

cat > .env <<'EOF'
DISCORD_WEBHOOK=
PORT=3000
EOF

cat > start.sh <<'EOF'
#!/usr/bin/env bash
export $(grep -v "^#" .env | xargs)
npm install
node server.js
EOF
chmod +x start.sh

cat > README.md <<'EOF'
Relay Site
==========

Endpoints:
- POST /api/relay  -> قبول JSON { source, message, ... }
- GET  /api/logs   -> جلب السجلات

شغّل: ./start.sh
EOF
