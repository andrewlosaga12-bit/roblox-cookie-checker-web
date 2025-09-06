from flask import Flask, request, jsonify, render_template_string, send_file, abort
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests, io, os
from datetime import datetime

app = Flask(__name__)

# Menyimpan hasil terakhir
LAST_RESULTS = {"valid": [], "invalid": [], "timestamp": None}

# Template HTML langsung di-embed di kode
HTML_TEMPLATE = """
<!doctype html>
<html lang="id">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Roblox Cookie Checker</title>
<style>
:root {--bg:#0f1115;--card:#111218;--muted:#9aa0a6;--accent:#00ff95;}
*{box-sizing:border-box;font-family:Inter,system-ui,Segoe UI,Arial;}
body{margin:0;background:linear-gradient(180deg,#07070a,#0f1115);color:#e6eef3;min-height:100vh;}
.container{max-width:900px;margin:24px auto;padding:16px;}
h1{color:var(--accent);margin:8px 0;}
.subtitle{color:var(--muted);margin:0 0 12px;}
.card{background:var(--card);padding:12px;border-radius:10px;border:1px solid rgba(0,255,149,0.06);display:flex;gap:8px;align-items:center;}
input[type=file]{background:transparent;color:inherit;border:1px dashed rgba(255,255,255,0.08);padding:10px;border-radius:6px;}
button{background:var(--accent);color:#041014;border:none;padding:10px 14px;border-radius:8px;cursor:pointer;font-weight:600;}
.results{display:flex;gap:12px;margin-top:18px;flex-wrap:wrap;}
.box{flex:1;min-width:250px;background:#0b0c0f;padding:12px;border-radius:8px;border:1px solid rgba(255,255,255,0.03);}
pre{white-space:pre-wrap;word-break:break-word;max-height:360px;overflow:auto;color:#cfeede;}
.download{display:inline-block;margin-top:8px;padding:8px 10px;background:transparent;color:var(--accent);text-decoration:none;border-radius:6px;border:1px solid rgba(0,255,149,0.12);}
footer{margin-top:18px;color:var(--muted);font-size:12px;}
</style>
</head>
<body>
  <main class="container">
    <h1>Roblox Cookie Checker</h1>
    <p class="subtitle">Upload <code>data.txt</code> lalu klik <b>Check Cookies</b>.</p>

    <div class="card">
      <input id="fileInput" type="file" accept=".txt" />
      <button id="checkBtn">Check Cookies</button>
    </div>

    <div class="results">
      <section class="box">
        <h2>✅ Valid Cookies</h2>
        <pre id="validBox">Menunggu...</pre>
        <a id="dlValid" class="download" style="display:none">Download Valid</a>
      </section>
      <section class="box">
        <h2>❌ Invalid Cookies</h2>
        <pre id="invalidBox">Menunggu...</pre>
        <a id="dlInvalid" class="download" style="display:none">Download Invalid</a>
      </section>
    </div>

    <footer>
      <small>⚠️ Jangan upload cookie milik orang lain.</small>
    </footer>
  </main>

<script>
document.getElementById("checkBtn").addEventListener("click", async () => {
  const fi = document.getElementById("fileInput");
  if (!fi.files.length) { alert("Pilih file data.txt dulu"); return; }

  const fd = new FormData();
  fd.append("file", fi.files[0]);

  const checkBtn = document.getElementById("checkBtn");
  checkBtn.disabled = true;
  checkBtn.textContent = "Memeriksa...";

  try {
    const res = await fetch("/check", { method: "POST", body: fd });
    const data = await res.json();
    if (res.status !== 200) {
      alert(data.error || "Gagal memeriksa");
      document.getElementById("validBox").textContent = "Tidak ada hasil";
      document.getElementById("invalidBox").textContent = "Tidak ada hasil";
      return;
    }
    document.getElementById("validBox").textContent = data.valid.length ? data.valid.join("\\n\\n") : "Tidak ada cookie valid";
    document.getElementById("invalidBox").textContent = data.invalid.length ? data.invalid.join("\\n\\n") : "Tidak ada cookie invalid";

    const dlV = document.getElementById("dlValid");
    const dlI = document.getElementById("dlInvalid");
    if (data.valid.length) {
      dlV.style.display = "inline-block";
      dlV.href = "/download/valid";
    } else dlV.style.display = "none";
    if (data.invalid.length) {
      dlI.style.display = "inline-block";
      dlI.href = "/download/invalid";
    } else dlI.style.display = "none";

  } catch (err) {
    alert("Error: " + err.message);
  } finally {
    checkBtn.disabled = false;
    checkBtn.textContent = "Check Cookies";
  }
});
</script>
</body>
</html>
"""

# Fungsi ambil cookies dari file
def extract_cookies_from_lines(lines):
    cookies = []
    for line in lines:
        parts = line.split("|")
        for p in parts:
            p = p.strip()
            if p.startswith("_|WARNING"):
                cookies.append(p)
            elif len(p) > 40 and all(c.isalnum() or c in "._-" for c in p):
                cookies.append(p)
    seen = set()
    out = []
    for c in cookies:
        if c not in seen:
            seen.add(c)
            out.append(c)
    return out

# Fungsi cek satu cookie
def check_one_cookie(cookie):
    headers = {
        "Cookie": f".ROBLOSECURITY={cookie}",
        "User-Agent": "Mozilla/5.0"
    }
    try:
        r = requests.get("https://www.roblox.com/mobileapi/userinfo", headers=headers, timeout=10)
        if r.status_code == 200:
            try:
                data = r.json()
                username = data.get("UserName") or ""
            except:
                username = ""
            return True, cookie, username
        else:
            return False, cookie, ""
    except:
        return False, cookie, ""

@app.route("/", methods=["GET"])
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route("/check", methods=["POST"])
def check():
    file = request.files.get("file")
    if not file:
        return jsonify({"error": "No file uploaded"}), 400

    try:
        raw = file.read().decode("utf-8", errors="ignore").splitlines()
    except:
        return jsonify({"error": "Cannot read file"}), 400

    cookies = extract_cookies_from_lines(raw)
    if not cookies:
        return jsonify({"error": "No cookies found"}), 400

    valid, invalid = [], []
    with ThreadPoolExecutor(max_workers=10) as ex:
        futures = [ex.submit(check_one_cookie, c) for c in cookies]
        for fut in as_completed(futures):
            ok, token, username = fut.result()
            if ok:
                if username:
                    valid.append(f"{token}  -->  [{username}]")
                else:
                    valid.append(token)
            else:
                invalid.append(token)

    timestamp = datetime.utcnow().isoformat() + "Z"
    LAST_RESULTS["valid"] = valid
    LAST_RESULTS["invalid"] = invalid
    LAST_RESULTS["timestamp"] = timestamp

    return jsonify({"valid": valid, "invalid": invalid, "timestamp": timestamp})

@app.route("/download/<which>", methods=["GET"])
def download(which):
    if which not in ("valid", "invalid"):
        abort(404)
    data = LAST_RESULTS.get(which, [])
    b = "\n".join(data).encode("utf-8")
    name = f"{which}_cookies_{(LAST_RESULTS.get('timestamp') or 'now')}.txt"
    return send_file(io.BytesIO(b), mimetype="text/plain", as_attachment=True, download_name=name)

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port)
