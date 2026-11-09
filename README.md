#!/usr/bin/env python3
import json, os, secrets, socket, threading, time, hmac, hashlib
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs, unquote

# Einstellungen
PASSWORD = os.environ.get("GALERIE_PASSWORT", "19656")
PORT = int(os.environ.get("PORT", "8080"))
HOST = "0.0.0.0"
BASE = os.path.dirname(os.path.abspath(__file__))
DIR = os.path.join(BASE, "bilder")
META = os.path.join(BASE, "kategorien.json")
USERS_FILE = os.path.join(BASE, "users.json")
MAX_BYTES = 100 * 1024 * 1024  # Bis zu 100 MB pro Video
TYPES = {".mp4": "video/mp4", ".webm": "video/webm", ".mov": "video/quicktime"}
LOCK = threading.Lock()
os.makedirs(DIR, exist_ok=True)

PAGE = r"""<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>fotos</title>
<style>
  :root { color-scheme: dark; --bg:#000; --text:#fff; --accent:#3b82f6; }
  * { box-sizing: border-box; }
  [hidden] { display:none !important; }
  html, body { margin:0; padding:0; width:100%; height:100%; background:var(--bg); color:var(--text); font-family:system-ui, sans-serif; overflow:hidden; }

  /* Perfekter TikTok Snap-Container */
  #feed {
    width: 100vw;
    height: 100vh;
    overflow-y: scroll;
    scroll-snap-type: y mandatory;
    -webkit-overflow-scrolling: touch;
    position: relative;
  }

  .video-card {
    width: 100vw;
    height: 100vh;
    height: 100dvh;
    scroll-snap-align: start;
    scroll-snap-stop: always;
    position: relative;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .video-card video {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
    cursor: pointer;
  }

  /* Overlay UI elemente wie bei TikTok */
  header {
    position: fixed;
    top: 0; left: 0; right: 0;
    display: flex; align-items: center; justify-content: space-between;
    padding: 16px 20px;
    z-index: 10;
    background: linear-gradient(to bottom, rgba(0,0,0,0.6), transparent);
    pointer-events: none;
  }
  header * { pointer-events: auto; }
  h1 { margin: 0; font-size: 1.2rem; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
  #menu-btn { background: none; border: none; color: #fff; font-size: 1.8rem; cursor: pointer; text-shadow: 0 2px 4px rgba(0,0,0,0.5); padding: 0; }

  /* Suchleiste / Chips als Overlay oben */
  #top-bar {
    position: fixed;
    top: 60px; left: 0; right: 0;
    padding: 0 16px;
    z-index: 10;
    display: flex;
    flex-direction: column;
    gap: 8px;
    pointer-events: none;
  }
  #top-bar * { pointer-events: auto; }
  #search {
    background: rgba(0,0,0,0.5);
    border: 1px solid rgba(255,255,255,0.2);
    color: #fff;
    padding: 8px 14px;
    border-radius: 999px;
    backdrop-filter: blur(4px);
    width: 100%;
    max-width: 400px;
    margin: 0 auto;
    font-size: 0.9rem;
  }
  #chips {
    display: flex;
    gap: 6px;
    overflow-x: auto;
    padding-bottom: 4px;
    max-width: 100%;
    scrollbar-width: none;
  }
  #chips::-webkit-scrollbar { display: none; }
  .chip {
    padding: 6px 14px;
    color: #fff;
    background: rgba(0,0,0,0.5);
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 999px;
    font-size: 0.85rem;
    white-space: nowrap;
    backdrop-filter: blur(4px);
    cursor: pointer;
  }
  .chip.active { background: var(--accent); border-color: var(--accent); }

  /* Rechte Aktionsleiste (Like) */
  .right-actions {
    position: absolute;
    right: 16px;
    bottom: 80px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    z-index: 5;
  }

  .action-btn {
    background: rgba(0,0,0,0.4);
    border: none;
    color: #fff;
    border-radius: 50%;
    width: 48px; height: 48px;
    display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    cursor: pointer;
    backdrop-filter: blur(4px);
    font-size: 1.2rem;
  }
  .action-btn.liked { color: #ff4757; }
  .action-count { font-size: 0.75rem; margin-top: 2px; font-weight: 600; }

  .video-info {
    position: absolute;
    left: 16px;
    bottom: 30px;
    right: 80px;
    z-index: 5;
    text-shadow: 0 1px 3px rgba(0,0,0,0.8);
  }
  .uploader-name { font-weight: bold; font-size: 1rem; margin-bottom: 4px; }
  .video-meta-stats { font-size: 0.85rem; opacity: 0.8; }

  /* Admin & Upload Buttons */
  .del {
    position: absolute;
    top: 20px; right: 20px;
    width: 36px; height: 36px;
    background: rgba(220,38,38,0.8);
    color: #fff; border: none; border-radius: 50%;
    font-size: 1rem; cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    z-index: 10;
    backdrop-filter: blur(4px);
  }
  .mv {
    position: absolute;
    top: 65px; right: 20px;
    padding: 6px 10px;
    background: rgba(0,0,0,0.6);
    color: #fff; border: 1px solid rgba(255,255,255,0.3);
    border-radius: 8px; font-size: .8rem;
    z-index: 10; backdrop-filter: blur(4px);
  }
  .mv option { color: #000; background: #fff; }

  #upload-fab {
    position: fixed;
    bottom: 20px; left: 50%;
    transform: translateX(-50%);
    background: var(--accent);
    color: #fff; padding: 10px 20px;
    border-radius: 999px; font-weight: 600;
    box-shadow: 0 4px 12px rgba(0,0,0,0.5);
    z-index: 15; cursor: pointer; border: none;
  }

  /* Modals & Menü */
  .modal-overlay { position: fixed; inset: 0; z-index: 30; background: rgba(0,0,0,0.7); display: flex; align-items: center; justify-content: center; backdrop-filter: blur(4px); padding: 16px; }
  .modal-box { background: #1b1f26; width: 100%; max-width: 380px; padding: 24px; border-radius: 16px; box-shadow: 0 10px 25px rgba(0,0,0,0.5); display: flex; flex-direction: column; gap: 16px; border: 1px solid #343a45; color: #e8eaed; }
  .modal-box h2 { margin: 0; font-size: 1.3rem; text-align: center; }
  .modal-box p { margin: 0; color: #9aa0a6; font-size: 0.9rem; text-align: center; }

  #menu-overlay { justify-content: flex-end; background: rgba(0,0,0,0.5); padding: 0; }
  #menu-panel { width: 320px; background: #1b1f26; height: 100%; padding: 20px; display: flex; flex-direction: column; gap: 16px; box-shadow: -2px 0 10px rgba(0,0,0,0.5); overflow-y: auto; color: #e8eaed; }
  #menu-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #343a45; padding-bottom: 10px; }
  #menu-header h2 { margin: 0; font-size: 1.1rem; }
  #close-menu { background: none; border: none; font-size: 1.3rem; color: #e8eaed; cursor: pointer; }

  input, button, select { font: inherit; }
  input[type=password], input[type=search], input[type=text] {
    padding: 10px; color: #e8eaed; background: #0f1115;
    border: 1px solid #343a45; border-radius: 8px; width: 100%; }
  button { padding: 10px 12px; color: #fff; background: var(--accent); border: 0; border-radius: 8px; cursor: pointer; font-weight: 500; }
  button.secondary { background: #343a45; color: #e8eaed; }

  #playlist-container { display: flex; flex-direction: column; gap: 6px; max-height: 200px; overflow-y: auto; }
  .playlist-item { display: flex; align-items: center; gap: 8px; background: #0f1115; padding: 6px; border-radius: 8px; cursor: pointer; border: 1px solid #343a45; }
  .playlist-item video { width: 36px; height: 36px; object-fit: cover; border-radius: 4px; }
  .playlist-item span { font-size: 0.85rem; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  
  #msg-toast { position: fixed; top: 110px; left: 50%; transform: translateX(-50%); background: rgba(0,0,0,0.8); color: #fff; padding: 6px 14px; border-radius: 999px; font-size: 0.85rem; z-index: 20; backdrop-filter: blur(4px); }
</style>
</head>
<body>

<header>
  <h1>fotos</h1>
  <button id="menu-btn" type="button" aria-label="Menü">☰</button>
</header>

<div id="top-bar">
  <input id="search" type="search" placeholder="Kategorie oder Benutzer suchen …" autocomplete="off">
  <div id="chips"></div>
</div>

<div id="welcome-modal" class="modal-overlay">
  <div class="modal-box">
    <h2>Willkommen!</h2>
    <p>Bitte wählen Sie, wie Sie fortfahren möchten:</p>
    <button type="button" id="wel-login-btn">Anmelden</button>
    <button type="button" id="wel-reg-btn" class="secondary">Konto erstellen</button>
    <button type="button" id="wel-guest-btn" class="secondary" style="background:transparent; border:1px solid #343a45;">Als Gast fortfahren</button>
  </div>
</div>

<div id="login-modal" class="modal-overlay" hidden>
  <div class="modal-box">
    <h2>Anmelden</h2>
    <form id="user-login-form" style="display:flex; flex-direction:column; gap:10px;">
      <input id="l-user" type="text" placeholder="Benutzername" required autocomplete="off">
      <input id="l-pw" type="password" placeholder="Passwort" required autocomplete="off">
      <p id="l-msg" style="color:#dc2626; font-size:0.8rem; text-align:left;" hidden></p>
      <button type="submit">Einloggen</button>
      <button type="button" id="back-to-wel-1" class="secondary">Zurück</button>
    </form>
  </div>
</div>

<div id="register-modal" class="modal-overlay" hidden>
  <div class="modal-box">
    <h2>Konto erstellen</h2>
    <form id="user-reg-form" style="display:flex; flex-direction:column; gap:10px;">
      <input id="r-user" type="text" placeholder="Gewünschter Benutzername" required autocomplete="off">
      <input id="r-pw" type="password" placeholder="Passwort" required autocomplete="off">
      <p id="r-msg" style="color:#dc2626; font-size:0.8rem; text-align:left;" hidden></p>
      <button type="submit">Registrieren & Starten</button>
      <button type="button" id="back-to-wel-2" class="secondary">Zurück</button>
    </form>
  </div>
</div>

<div id="menu-overlay" hidden>
  <div id="menu-panel">
    <div id="menu-header">
      <h2>Menü</h2>
      <button id="close-menu" type="button">✕</button>
    </div>
    
    <div id="user-info-area" style="font-size:0.9rem; color:#9aa0a6; display:flex; flex-direction:column; gap:4px;">
      <span>Angemeldet als: <strong id="current-user-label" style="color:#e8eaed;">Gast</strong></span>
      <span id="user-profile-stats" hidden>❤️ Likes erhalten: <strong id="user-total-likes" style="color:#e8eaed;">0</strong></span>
    </div>

    <div id="account-actions" style="display:flex; flex-direction:column; gap:8px;">
      <button type="button" id="menu-login-switch" class="secondary">Account wechseln / Login</button>
      <button type="button" id="menu-logout-btn" class="secondary" hidden style="background:#dc2626; color:#fff;">Abmelden</button>
    </div>

    <div id="playlist-section" hidden style="display:flex; flex-direction:column; gap:6px;">
      <p style="margin:0; font-size:0.9rem; font-weight:600; color:#e8eaed;">❤️ Meine Playlist (Geliked)</p>
      <div id="playlist-container"></div>
    </div>

    <hr style="border:0; border-top:1px solid #343a45; margin:4px 0;">

    <div id="admin-section">
      <p style="margin:0 0 8px 0; font-size:0.9rem; color:#9aa0a6;">Admin-Bereich</p>
      <form id="admin-login" style="display:flex; flex-direction:column; gap:8px;">
        <input id="admin-pw-input" type="password" inputmode="numeric" placeholder="Admin-Passwort" autocomplete="off">
        <button type="submit">Admin werden</button>
      </form>
      <button id="admin-logout" type="button" hidden style="background:#dc2626; width:100%;">Admin sperren</button>
      
      <div id="adminbar" hidden style="margin-top: 12px; display:flex; flex-direction:column; gap:8px;">
        <form id="newcat" style="display:flex; gap:6px;">
          <input id="catname" type="text" placeholder="Neue Kategorie" maxlength="40" autocomplete="off">
          <button type="submit">Anlegen</button>
        </form>
        <button id="delcat" type="button" hidden style="background:#dc2626; width:100%;">Kategorie löschen</button>
      </div>
    </div>
  </div>
</div>

<div id="msg-toast" hidden></div>
<button id="upload-fab" hidden>+ Video hochladen</button>
<input id="file" type="file" accept="video/mp4,video/webm,video/quicktime" multiple hidden>

<div id="feed"></div>

<script>
  const $ = (id) => document.getElementById(id);
  let adminPw = null;
  let currentUser = null; 
  let data = { categories: [], images: [] };
  let current = null;   
  let videoObserver = null;

  const call = (url, opts = {}) => fetch(url, {
    method: "POST", ...opts,
    headers: { 
      "X-Password": adminPw || "", 
      "X-User": currentUser ? currentUser.username : "",
      ...(opts.headers || {}) 
    }
  });
  const post = (url, obj) => call(url, {
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(obj)
  });

  function showToast(text) {
    const t = $("msg-toast");
    t.textContent = text;
    t.hidden = false;
    setTimeout(() => t.hidden = true, 3000);
  }

  const query = () => $("search").value.trim().toLowerCase();
  const countIn = (cat) => data.images.filter((i) => i.cat === cat).length;

  async function load() {
    data = await (await fetch("/api/list")).json();
    if (current && !data.categories.includes(current)) current = null;
    draw();
    updatePlaylistUI();
    updateUserProfileStats();
  }

  async function move(name, cat) {
    const r = await post("/api/move", { name, cat });
    if (r.ok) load(); else showToast("Verschoben");
  }

  async function likeVideo(name, btn) {
    if (!currentUser) {
      alert("Bitte melden Sie sich an, um Videos zu liken.");
      $("welcome-modal").hidden = false;
      return;
    }
    const r = await post("/api/like", { name });
    if (r.ok) {
      const res = await r.json();
      btn.querySelector(".action-count").textContent = res.likes;
      if (res.userLikes) currentUser.likes = res.userLikes;
      
      if (currentUser.likes.includes(name)) {
        btn.classList.add("liked");
      } else {
        btn.classList.remove("liked");
      }
      updatePlaylistUI();
    } else {
      showToast("Fehler beim Liken");
    }
  }

  function updateUserProfileStats() {
    if (!currentUser) {
      $("user-profile-stats").hidden = true;
      return;
    }
    $("user-profile-stats").hidden = false;
    const myVideos = data.images.filter(i => i.uploader === currentUser.username);
    const totalLikes = myVideos.reduce((sum, vid) => sum + (vid.likes || 0), 0);
    $("user-total-likes").textContent = totalLikes;
  }

  function updatePlaylistUI() {
    const container = $("playlist-container");
    const section = $("playlist-section");
    container.innerHTML = "";

    if (!currentUser || !currentUser.likes || currentUser.likes.length === 0) {
      section.hidden = true;
      return;
    }

    section.hidden = false;
    currentUser.likes.forEach(vidName => {
      const item = document.createElement("div");
      item.className = "playlist-item";
      
      const thumb = document.createElement("video");
      thumb.src = "/bilder/" + encodeURIComponent(vidName) + "#t=0.5";
      thumb.preload = "metadata";
      
      const title = document.createElement("span");
      title.textContent = vidName;

      item.onclick = () => {
        const el = Array.from(document.querySelectorAll(".video-card")).find(c => c.dataset.name === vidName);
        if (el) {
          el.scrollIntoView({ behavior: "smooth", block: "start" });
          $("menu-overlay").hidden = true;
        }
      };

      item.append(thumb, title);
      container.append(item);
    });
  }

  function chip(label, value, count) {
    const b = document.createElement("button");
    b.type = "button";
    b.className = "chip" + (current === value ? " active" : "");
    b.textContent = label + " (" + count + ")";
    b.onclick = () => { current = value; draw(); };
    return b;
  }

  function drawChips() {
    const q = query();
    const box = $("chips");
    box.innerHTML = "";
    box.append(chip("Alle", null, data.images.length));
    const none = countIn("");
    if ((none || adminPw || currentUser) && "ohne kategorie".includes(q)) box.append(chip("Ohne Kategorie", "", none));
    data.categories
      .filter((c) => c.toLowerCase().includes(q))
      .forEach((c) => box.append(chip(c, c, countIn(c))));
  }

  function visibleVideos() {
    const q = query();
    return data.images.filter((i) => {
      if (current !== null) return i.cat === current;
      if (!q) return true;
      const matchesCat = i.cat !== "" && i.cat.toLowerCase().includes(q);
      const matchesUser = i.uploader && i.uploader.toLowerCase().includes(q);
      return matchesCat || matchesUser;
    });
  }

  function drawFeed() {
    const files = visibleVideos();
    const feed = $("feed");
    feed.innerHTML = "";
    
    if (!files.length) {
      showToast(!data.images.length ? "Noch keine Videos vorhanden." : "Keine Videos gefunden.");
    }

    if (videoObserver) videoObserver.disconnect();

    // IntersectionObserver: Lädt Videos beim Reinscrollen und entlädt sie beim Rausscrollen komplett
    videoObserver = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        const card = entry.target;
        const vid = card.querySelector("video");
        if (!vid) return;

        if (entry.isIntersecting) {
          if (!vid.src) {
            vid.src = card.dataset.src;
          }
          vid.play().catch(() => {});
        } else {
          vid.pause();
          vid.removeAttribute("src"); // Komplett entladen, gibt RAM/Puffer frei
          vid.load();
        }
      });
    }, { threshold: 0.6 });

    files.forEach((it) => {
      const card = document.createElement("div");
      card.className = "video-card";
      card.dataset.name = it.name;
      card.dataset.src = "/bilder/" + encodeURIComponent(it.name);

      const video = document.createElement("video");
      video.loop = true;
      video.playsInline = true;
      video.preload = "none";
      
      video.onclick = () => {
        if (video.paused) video.play(); else video.pause();
      };
      card.append(video);

      const info = document.createElement("div");
      info.className = "video-info";
      const uploader = document.createElement("div");
      uploader.className = "uploader-name";
      uploader.textContent = "@" + (it.uploader || "Unbekannt");
      const stats = document.createElement("div");
      stats.className = "video-meta-stats";
      stats.textContent = "👁 " + (it.views || 0) + " Aufrufe" + (it.cat ? " • " + it.cat : "");
      info.append(uploader, stats);
      card.append(info);

      const rightActions = document.createElement("div");
      rightActions.className = "right-actions";

      const likeBtn = document.createElement("button");
      const isLiked = currentUser && currentUser.likes && currentUser.likes.includes(it.name);
      likeBtn.className = "action-btn" + (isLiked ? " liked" : "");
      likeBtn.innerHTML = '❤️<span class="action-count">' + (it.likes || 0) + '</span>';
      likeBtn.onclick = (e) => { e.stopPropagation(); likeVideo(it.name, likeBtn); };
      rightActions.append(likeBtn);
      card.append(rightActions);

      const isOwner = currentUser && it.uploader === currentUser.username;
      if (adminPw || isOwner) {
        const delBtn = document.createElement("button");
        delBtn.className = "del"; delBtn.textContent = "✕";
        delBtn.title = "Video löschen";
        delBtn.onclick = async () => {
          if (!confirm("Video löschen?")) return;
          const r = await call("/api/delete?name=" + encodeURIComponent(it.name));
          if (r.ok) load(); else showToast("Fehler beim Löschen");
        };
        card.append(delBtn);
      }

      if (adminPw) {
        const sel = document.createElement("select");
        sel.className = "mv";
        sel.append(new Option("Ohne Kategorie", ""));
        data.categories.forEach((c) => sel.append(new Option(c, c)));
        sel.value = it.cat;
        sel.onchange = () => move(it.name, sel.value);
        card.append(sel);
      }

      feed.append(card);
      videoObserver.observe(card);
    });
  }

  function draw() {
    drawChips();
    drawFeed();
    $("delcat").hidden = !(adminPw && current);
  }

  function updateInterfaceState() {
    if (currentUser || adminPw) {
      $("upload-fab").hidden = false;
    } else {
      $("upload-fab").hidden = true;
    }
    if (currentUser) {
      $("current-user-label").textContent = currentUser.username;
      $("menu-logout-btn").hidden = false;
      $("menu-login-switch").hidden = true;
    } else {
      $("current-user-label").textContent = adminPw ? "Admin" : "Gast";
      $("menu-logout-btn").hidden = true;
      $("menu-login-switch").hidden = false;
    }
    $("admin-login").hidden = !!adminPw;
    $("admin-logout").hidden = !adminPw;
    $("adminbar").hidden = !adminPw;
    updatePlaylistUI();
    updateUserProfileStats();
    draw();
  }

  $("wel-guest-btn").onclick = () => { $("welcome-modal").hidden = true; };
  $("wel-login-btn").onclick = () => { $("welcome-modal").hidden = true; $("login-modal").hidden = false; };
  $("wel-reg-btn").onclick = () => { $("welcome-modal").hidden = true; $("register-modal").hidden = false; };
  $("back-to-wel-1").onclick = () => { $("login-modal").hidden = true; $("welcome-modal").hidden = false; };
  $("back-to-wel-2").onclick = () => { $("register-modal").hidden = true; $("welcome-modal").hidden = false; };

  $("menu-btn").onclick = () => { $("menu-overlay").hidden = false; };
  $("close-menu").onclick = () => { $("menu-overlay").hidden = true; };
  $("menu-overlay").onclick = (e) => { if (e.target === $("menu-overlay")) $("menu-overlay").hidden = true; };

  $("menu-login-switch").onclick = () => {
    $("menu-overlay").hidden = true;
    $("welcome-modal").hidden = false;
  };

  $("menu-logout-btn").onclick = () => {
    currentUser = null;
    updateInterfaceState();
    $("menu-overlay").hidden = true;
    load();
  };

  $("admin-login").onsubmit = async (e) => {
    e.preventDefault();
    const attempt = $("admin-pw-input").value;
    $("admin-pw-input").value = "";
    let ok = false;
    try {
      const r = await fetch("/api/login", { method: "POST", headers: { "X-Password": attempt } });
      ok = r.ok;
    } catch (err) {}
    if (ok) {
      adminPw = attempt;
      updateInterfaceState();
      $("menu-overlay").hidden = true;
    } else {
      alert("Falsches Admin-Passwort");
    }
  };

  $("admin-logout").onclick = () => { adminPw = null; updateInterfaceState(); $("menu-overlay").hidden = true; };

  $("search").oninput = () => { current = null; draw(); };

  $("newcat").onsubmit = async (e) => {
    e.preventDefault();
    const name = $("catname").value.trim();
    if (!name) return;
    const r = await post("/api/category/add", { name });
    if (r.ok) {
      const j = await r.json();
      $("catname").value = "";
      $("search").value = "";
      current = j.name;
      await load();
    } else {
      showToast("Kategorie existiert bereits");
    }
  };

  $("delcat").onclick = async () => {
    if (!current) return;
    if (!confirm("Kategorie „" + current + "“ löschen?")) return;
    const r = await post("/api/category/delete", { name: current });
    if (r.ok) { current = null; await load(); }
  };

  $("upload-fab").onclick = () => {
    if (!currentUser && !adminPw) {
      alert("Bitte anmelden.");
      return;
    }
    $("file").click();
  };

  $("file").onchange = async () => {
    showToast("Video wird hochgeladen …");
    let err = null;
    for (const f of $("file").files) {
      const r = await call("/api/upload", {
        headers: { "X-Filename": encodeURIComponent(f.name), "X-Category": encodeURIComponent(current || "") },
        body: f
      });
      if (!r.ok) { err = r; break; }
    }
    $("file").value = "";
    await load();
    if (err) showToast("Fehler beim Hochladen"); else showToast("Erfolgreich hochgeladen!");
  };

  load();
</script>
</body>
</html>
"""

def hash_pw(pw):
    return hashlib.sha256(pw.encode()).hexdigest()

def load_users():
    try:
        with open(USERS_FILE, encoding="utf-8") as f:
            u = json.load(f)
            return u if isinstance(u, dict) else {}
    except (OSError, ValueError):
        return {}

def save_users(users):
    tmp = USERS_FILE + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(users, f, ensure_ascii=False, indent=2)
    os.replace(tmp, USERS_FILE)

def clean_name(s):
    return " ".join(str(s).split())[:40]

def load_meta():
    try:
        with open(META, encoding="utf-8") as f:
            m = json.load(f)
    except (OSError, ValueError):
        m = {}
    if not isinstance(m, dict):
        m = {}
    cats = m.get("categories")
    assign = m.get("assign")
    stats = m.get("stats")
    cats = [c for c in cats if isinstance(c, str)] if isinstance(cats, list) else []
    
    clean_assign = {}
    if isinstance(assign, dict):
        for k, v in assign.items():
            if isinstance(v, str):
                clean_assign[k] = {"cat": v, "uploader": ""}
            elif isinstance(v, dict):
                clean_assign[k] = {"cat": v.get("cat", ""), "uploader": v.get("uploader", "")}

    stats = ({k: v for k, v in stats.items() if isinstance(v, dict)}
             if isinstance(stats, dict) else {})
    return {"categories": cats, "assign": clean_assign, "stats": stats}

def save_meta(m):
    tmp = META + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(m, f, ensure_ascii=False, indent=2)
    os.replace(tmp, META)

def video_files():
    files = [f for f in os.listdir(DIR) if os.path.splitext(f)[1].lower() in TYPES]
    files.sort(key=lambda f: os.path.getmtime(os.path.join(DIR, f)), reverse=True)
    return files

class Handler(BaseHTTPRequestHandler):
    def send(self, code, body=b"", ctype="application/json"):
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def json(self, code, obj):
        self.send(code, json.dumps(obj).encode())

    def read_json(self):
        length = int(self.headers.get("Content-Length") or 0)
        if length > 10_000:
            return {}
        try:
            body = json.loads(self.rfile.read(length) or b"{}")
        except ValueError:
            return {}
        return body if isinstance(body, dict) else {}

    def pw_ok(self):
        given = self.headers.get("X-Password", "").encode()
        good = hmac.compare_digest(given, PASSWORD.encode())
        if not good:
            time.sleep(1)
        return good

    def do_HEAD(self):
        self.send(200, b"", "text/html; charset=utf-8")

    def do_GET(self):
        path = urlparse(self.path).path
        if path == "/":
            return self.send(200, PAGE.encode(), "text/html; charset=utf-8")
        if path == "/api/list":
            with LOCK:
                meta = load_meta()
            cats = meta["categories"]
            images = []
            for f in video_files():
                info = meta["assign"].get(f, {"cat": "", "uploader": ""})
                cat = info.get("cat", "")
                uploader = info.get("uploader", "")
                st = meta["stats"].get(f, {"views": 0, "likes": 0})
                images.append({
                    "name": f, 
                    "cat": cat if cat in cats else "", 
                    "uploader": uploader,
                    "views": st.get("views", 0), 
                    "likes": st.get("likes", 0)
                })
            return self.json(200, {"categories": sorted(cats, key=str.lower), "images": images})
        if path.startswith("/bilder/"):
            name = os.path.basename(unquote(path[8:]))
            ext = os.path.splitext(name)[1].lower()
            full = os.path.join(DIR, name)
            if ext in TYPES and os.path.isfile(full):
                with LOCK:
                    m = load_meta()
                    st = m["stats"].get(name, {"views": 0, "likes": 0})
                    st["views"] = st.get("views", 0) + 1
                    m["stats"][name] = st
                    save_meta(m)
                
                file_size = os.path.getsize(full)
                range_header = self.headers.get("Range")
                
                if range_header:
                    try:
                        byte_range = range_header.replace("bytes=", "").split("-")
                        start = int(byte_range[0])
                        end = int(byte_range[1]) if byte_range[1] else file_size - 1
                        length = end - start + 1
                        
                        self.send_response(206)
                        self.send_header("Content-Type", TYPES[ext])
                        self.send_header("Content-Range", f"bytes {start}-{end}/{file_size}")
                        self.send_header("Content-Length", str(length))
                        self.send_header("Accept-Ranges", "bytes")
                        self.end_headers()
                        
                        with open(full, "rb") as f:
                            f.seek(start)
                            self.wfile.write(f.read(length))
                        return
                    except Exception:
                        pass

                self.send_response(200)
                self.send_header("Content-Type", TYPES[ext])
                self.send_header("Content-Length", str(file_size))
                self.send_header("Accept-Ranges", "bytes")
                self.end_headers()
                with open(full, "rb") as f:
                    self.wfile.write(f.read())
                return
        self.send(404, b"Nicht gefunden", "text/plain; charset=utf-8")

    def do_POST(self):
        u = urlparse(self.path)

        if u.path == "/api/user/register":
            body = self.read_json()
            username = clean_name(body.get("username", ""))
            password = body.get("password", "")
            if not username or not password:
                return self.json(400, {"error": "Benutzername und Passwort erforderlich"})
            with LOCK:
                users = load_users()
                if username.lower() in [u_key.lower() for u_key in users.keys()]:
                    return self.json(409, {"error": "Benutzername bereits vergeben"})
                users[username] = {"password": hash_pw(password), "likes": []}
                save_users(users)
            return self.json(200, {"username": username, "likes": []})

        if u.path == "/api/user/login":
            body = self.read_json()
            username = body.get("username", "")
            password = body.get("password", "")
            with LOCK:
                users = load_users()
                user_data = None
                real_username = None
                for u_key, u_val in users.items():
                    if u_key.lower() == username.lower():
                        real_username = u_key
                        user_data = u_val
                        break
                if not user_data or user_data["password"] != hash_pw(password):
                    return self.json(403, {"error": "Ungültiger Benutzername oder Passwort"})
            return self.json(200, {"username": real_username, "likes": user_data["likes"]})

        if u.path == "/api/like":
            username = self.headers.get("X-User", "")
            body = self.read_json()
            name = os.path.basename(str(body.get("name", "")))
            if not username:
                return self.json(403, {"error": "Nicht angemeldet"})
            if os.path.splitext(name)[1].lower() not in TYPES or not os.path.isfile(os.path.join(DIR, name)):
                return self.json(404, {"error": "Video nicht gefunden"})
            
            with LOCK:
                users = load_users()
                if username not in users:
                    return self.json(403, {"error": "Benutzer existiert nicht"})
                
                user_likes = users[username].get("likes", [])
                m = load_meta()
                st = m["stats"].get(name, {"views": 0, "likes": 0})
                
                if name in user_likes:
                    user_likes.remove(name)
                    st["likes"] = max(0, st.get("likes", 0) - 1)
                else:
                    user_likes.append(name)
                    st["likes"] = st.get("likes", 0) + 1
                
                users[username]["likes"] = user_likes
                save_users(users)
                
                m["stats"][name] = st
                save_meta(m)
                
                total_likes = st["likes"]
            return self.json(200, {"likes": total_likes, "userLikes": user_likes})

        if u.path == "/api/upload":
            username = self.headers.get("X-User", "")
            is_admin = self.pw_ok()

            if not username and not is_admin:
                return self.json(403, {"error": "Bitte anmelden, um Videos zu posten"})
            
            uploader_name = username if username else "Admin"
            
            length = int(self.headers.get("Content-Length") or 0)
            if length > MAX_BYTES:
                return self.json(413, {"error": "Datei zu groß (max. 100 MB)"})
            ext = os.path.splitext(unquote(self.headers.get("X-Filename", "")))[1].lower()
            if ext not in TYPES:
                return self.json(400, {"error": "Nur MP4, WEBM und MOV Videos erlaubt"})
            cat = clean_name(unquote(self.headers.get("X-Category", "")))
            data = self.rfile.read(length)
            name = f"{int(time.time())}-{secrets.token_hex(3)}{ext}"
            with open(os.path.join(DIR, name), "wb") as f:
                f.write(data)
            with LOCK:
                m = load_meta()
                valid_cat = cat if cat in m["categories"] else ""
                m["assign"][name] = {"cat": valid_cat, "uploader": uploader_name}
                m["stats"][name] = {"views": 0, "likes": 0}
                save_meta(m)
            return self.json(200, {"name": name})

        if u.path == "/api/delete":
            name = os.path.basename(parse_qs(u.query).get("name", [""])[0])
            username = self.headers.get("X-User", "")
            is_admin = self.pw_ok()

            with LOCK:
                m = load_meta()
                info = m["assign"].get(name, {})
                owner = info.get("uploader", "") if isinstance(info, dict) else ""

                if not is_admin and (not username or username != owner):
                    return self.json(403, {"error": "Keine Berechtigung zum Löschen dieses Videos"})

            full = os.path.join(DIR, name)
            if os.path.splitext(name)[1].lower() in TYPES and os.path.isfile(full):
                os.remove(full)
                with LOCK:
                    m = load_meta()
                    m["assign"].pop(name, None)
                    m["stats"].pop(name, None)
                    save_meta(m)
                    users = load_users()
                    for u_val in users.values():
                        if name in u_val.get("likes", []):
                            u_val["likes"].remove(name)
                    save_users(users)
                return self.json(200, {"ok": True})
            return self.json(404, {"error": "Video nicht gefunden"})

        if not self.pw_ok():
            return self.json(403, {"error": "Falsches Admin-Passwort"})

        if u.path == "/api/login":
            return self.json(200, {"ok": True})

        if u.path == "/api/category/add":
            name = clean_name(self.read_json().get("name", ""))
            if not name:
                return self.json(400, {"error": "Name fehlt"})
            with LOCK:
                m = load_meta()
                if any(c.lower() == name.lower() for c in m["categories"]):
                    return self.json(409, {"error": "Kategorie existiert bereits"})
                m["categories"].append(name)
                save_meta(m)
            return self.json(200, {"name": name})

        if u.path == "/api/category/delete":
            name = clean_name(self.read_json().get("name", ""))
            with LOCK:
                m = load_meta()
                if name not in m["categories"]:
                    return self.json(404, {"error": "Kategorie nicht gefunden"})
                m["categories"].remove(name)
                for k, v in m["assign"].items():
                    if isinstance(v, dict) and v.get("cat") == name:
                        v["cat"] = ""
                save_meta(m)
            return self.json(200, {"ok": True})

        if u.path == "/api/move":
            body = self.read_json()
            name = os.path.basename(str(body.get("name", "")))
            cat = clean_name(body.get("cat", ""))
            if os.path.splitext(name)[1].lower() not in TYPES or not os.path.isfile(os.path.join(DIR, name)):
                return self.json(404, {"error": "Video nicht gefunden"})
            with LOCK:
                m = load_meta()
                if cat and cat not in m["categories"]:
                    return self.json(404, {"error": "Kategorie nicht gefunden"})
                if name in m["assign"]:
                    if isinstance(m["assign"][name], dict):
                        m["assign"][name]["cat"] = cat
                    else:
                        m["assign"][name] = {"cat": cat, "uploader": ""}
                else:
                    m["assign"][name] = {"cat": cat, "uploader": ""}
                save_meta(m)
            return self.json(200, {"ok": True})

        self.json(404, {"error": "Nicht gefunden"})

if __name__ == "__main__":
    ip = socket.gethostbyname(socket.gethostname())
    print("fotos (TikTok Feed - Optimiert) läuft.")
    print(f"  Auf dem Laptop:        http://localhost:{PORT}")
    print(f"  Im selben WLAN/LAN:    http://{ip}:{PORT}")
    print("  Beenden mit Strg+C")
    ThreadingHTTPServer((HOST, PORT), Handler).serve_forever()

