# tampermonkey_itchio_addon
tamper monkey script to do market research on itch.io


```
// ==UserScript==
// @name         ItchNova – Market Intel
// @namespace    https://nova.itch-intel.local
// @version      1.2.0
// @description  Browse itch.io, auto-find top sellers, save public game data, dashboard + CSV export.
// @author       NØVA
// @match        https://itch.io/*
// @match        https://*.itch.io/*
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_xmlhttpRequest
// @connect      itch.io
// @connect      *.itch.io
// @run-at       document-idle
// ==/UserScript==

(function () {
  "use strict";

  const STORAGE_KEY = "itchnova_games_v1";
  let isDiscovering = false;

  /** Built-in lists to fetch from itch.io (public browse pages). */
  const DISCOVER_PRESETS = [
    {
      id: "top-sellers-100",
      label: "Top sellers (≈100)",
      path: "/games/top-sellers",
      max: 100,
    },
    {
      id: "top-sellers-48",
      label: "Top sellers (first page)",
      path: "/games/top-sellers",
      max: 48,
    },
    {
      id: "top-rated-100",
      label: "Top rated (≈100)",
      path: "/games/top-rated",
      max: 100,
    },
    {
      id: "popular-100",
      label: "Popular (≈100)",
      path: "/games",
      max: 100,
    },
    {
      id: "new-popular-100",
      label: "New & popular (≈100)",
      path: "/games/new-and-popular",
      max: 100,
    },
    {
      id: "free-top-100",
      label: "Top free-ish (popular free)",
      path: "/games/free",
      max: 100,
    },
  ];

  // ── storage ──────────────────────────────────────────────
  function loadGames() {
    try {
      const raw = GM_getValue(STORAGE_KEY, "{}");
      const obj = typeof raw === "string" ? JSON.parse(raw) : raw || {};
      return obj && typeof obj === "object" ? obj : {};
    } catch {
      return {};
    }
  }

  function saveGames(map) {
    GM_setValue(STORAGE_KEY, JSON.stringify(map));
  }

  function upsertGames(list) {
    const map = loadGames();
    let added = 0;
    let updated = 0;
    const now = new Date().toISOString();

    for (const g of list) {
      if (!g || !g.url || !g.title) continue;
      const key = normalizeUrl(g.url);
      const prev = map[key];
      const merged = {
        ...prev,
        ...g,
        url: key,
        savedAt: prev?.savedAt || now,
        updatedAt: now,
        successScore: calcSuccessScore(
          g.rating != null ? g.rating : prev?.rating,
          g.ratingCount != null ? g.ratingCount : prev?.ratingCount
        ),
      };
      if (prev) {
        for (const f of [
          "developer",
          "price",
          "priceType",
          "rating",
          "ratingCount",
          "tags",
          "platforms",
          "playInBrowser",
          "genre",
          "listRank",
          "listName",
        ]) {
          if (
            (merged[f] === "" ||
              merged[f] == null ||
              (Array.isArray(merged[f]) && !merged[f].length)) &&
            prev[f] != null
          ) {
            merged[f] = prev[f];
          }
        }
        merged.successScore = calcSuccessScore(merged.rating, merged.ratingCount);
      }
      if (!prev) added++;
      else updated++;
      map[key] = merged;
    }
    saveGames(map);
    return { added, updated, total: Object.keys(map).length };
  }

  function normalizeUrl(url) {
    try {
      const u = new URL(url, "https://itch.io");
      u.hash = "";
      u.search = "";
      return u.href.replace(/\/+$/, "");
    } catch {
      return String(url).split("?")[0].replace(/\/+$/, "");
    }
  }

  function calcSuccessScore(rating, ratingCount) {
    const r = Number(rating);
    const n = Number(ratingCount);
    if (!r || !n || n < 1 || Number.isNaN(r) || Number.isNaN(n)) return 0;
    return Math.round(r * Math.log10(n + 1) * 100) / 100;
  }

  // ── scraping helpers ─────────────────────────────────────
  function text(el) {
    return (el?.textContent || "").replace(/\s+/g, " ").trim();
  }

  function parsePrice(raw) {
    const s = (raw || "").trim();
    if (!s) return { price: "", priceType: "unknown" };
    const lower = s.toLowerCase();
    if (lower === "free" || lower.includes("free")) {
      return { price: "Free", priceType: "free" };
    }
    if (lower.includes("name your price") || lower.includes("no minimum")) {
      return { price: s, priceType: "name-your-price" };
    }
    const m = s.match(/\$\s*[\d,.]+/);
    if (m) return { price: m[0].replace(/\s+/g, ""), priceType: "paid" };
    if (/[\d]/.test(s) && /\$|€|£|¥/.test(s)) {
      return { price: s, priceType: "paid" };
    }
    return { price: s, priceType: "unknown" };
  }

  function parseRatingFromText(s) {
    if (!s) return { rating: null, ratingCount: null };
    const a = s.match(
      /(\d+(?:\.\d+)?)\s*(?:\/\s*5)?\s*[\(\[]?\s*([\d,]+)\s*(?:ratings?|reviews?)?/i
    );
    if (a) {
      return {
        rating: parseFloat(a[1]),
        ratingCount: parseInt(a[2].replace(/,/g, ""), 10),
      };
    }
    const b = s.match(/(\d+(?:\.\d+)?)\s*stars?/i);
    const c = s.match(/([\d,]+)\s*ratings?/i);
    return {
      rating: b ? parseFloat(b[1]) : null,
      ratingCount: c ? parseInt(c[1].replace(/,/g, ""), 10) : null,
    };
  }

  function scrapeCellsFromRoot(root, sourcePage, rankOffset, listName) {
    const cells = root.querySelectorAll(".game_cell");
    const games = [];
    let rank = rankOffset || 0;

    cells.forEach((cell) => {
      rank++;
      const link =
        cell.querySelector("a.game_link") ||
        cell.querySelector(".game_title a") ||
        cell.querySelector("a[href*='.itch.io/']");
      if (!link) return;

      let href = link.getAttribute("href") || "";
      try {
        href = new URL(href, sourcePage || "https://itch.io").href;
      } catch {
        return;
      }
      if (!href || href.includes("/add-to-collection") || href.includes("/jams/")) return;

      const titleEl =
        cell.querySelector(".game_title a") ||
        cell.querySelector(".title a") ||
        cell.querySelector(".game_title") ||
        link;
      const title = text(titleEl) || text(link);
      if (!title) return;

      const devEl =
        cell.querySelector(".game_author a") ||
        cell.querySelector(".game_author") ||
        cell.querySelector(".byline a");
      const developer = text(devEl);

      const priceEl =
        cell.querySelector(".price_value") ||
        cell.querySelector(".game_cell_data .price_value") ||
        cell.querySelector(".sale_tag") ||
        cell.querySelector(".price");
      let priceRaw = text(priceEl);
      if (!priceRaw) {
        if (
          cell.textContent.toLowerCase().includes("free") &&
          !/\$\s*\d/.test(cell.textContent)
        ) {
          priceRaw = "Free";
        }
      }
      const { price, priceType } = parsePrice(priceRaw);

      const genreEl = cell.querySelector(".game_genre");
      const genre = text(genreEl);

      const platforms = [];
      cell.querySelectorAll(".game_platform [class*='icon-']").forEach((ic) => {
        const cls = ic.className || "";
        if (cls.includes("windows")) platforms.push("Windows");
        else if (cls.includes("apple") || cls.includes("osx") || cls.includes("mac"))
          platforms.push("macOS");
        else if (cls.includes("tux") || cls.includes("linux")) platforms.push("Linux");
        else if (cls.includes("android")) platforms.push("Android");
        else if (cls.includes("ios")) platforms.push("iOS");
        else if (cls.includes("html5") || cls.includes("web")) platforms.push("Browser");
      });

      const playInBrowser =
        /play in browser/i.test(cell.textContent) ||
        platforms.includes("Browser") ||
        !!cell.querySelector(".icon-html5");

      const ratingEl =
        cell.querySelector(".game_rating") ||
        cell.querySelector(".rating_count") ||
        cell.querySelector("[class*='rating']");
      const { rating, ratingCount } = parseRatingFromText(text(ratingEl) || "");

      const tags = [];
      if (genre) tags.push(genre);
      const dataTags = cell.getAttribute("data-tags") || cell.dataset?.tags;
      if (dataTags) {
        dataTags.split(/[,\s]+/).forEach((t) => {
          if (t && !tags.includes(t)) tags.push(t);
        });
      }

      const gameId = cell.getAttribute("data-game_id") || cell.dataset?.game_id || "";

      games.push({
        title,
        url: normalizeUrl(href),
        developer,
        price,
        priceType,
        rating,
        ratingCount,
        tags,
        genre,
        platforms: [...new Set(platforms)],
        playInBrowser,
        rankOnPage: rank - rankOffset,
        listRank: rank,
        listName: listName || "",
        gameId: gameId ? String(gameId) : "",
        sourcePage: sourcePage || "",
      });
    });

    return games;
  }

  function scrapeListPage() {
    return scrapeCellsFromRoot(document, location.href, 0, "current-page");
  }

  function scrapeGamesFromHtml(html, sourcePage, rankOffset, listName) {
    const doc = new DOMParser().parseFromString(html, "text/html");
    return scrapeCellsFromRoot(doc, sourcePage, rankOffset, listName);
  }

  function scrapeGamePage() {
    const host = location.hostname;
    if (!host.endsWith("itch.io") || host === "itch.io" || host === "www.itch.io") {
      return null;
    }

    const titleEl = document.querySelector(
      ".game_title, h1.game_title, .header h1, h1[itemprop='name'], h1"
    );
    const title = text(titleEl);
    if (!title) return null;

    const devEl =
      document.querySelector(".game_info_panel_widget a[href*='.itch.io']") ||
      document.querySelector(".game_author a") ||
      document.querySelector("a.user_link") ||
      document.querySelector(".info_panel_wrapper table a[href*='.itch.io']");
    let developer = text(devEl);
    if (!developer) developer = host.replace(/\.itch\.io$/, "");

    let priceRaw = "";
    const priceEl = document.querySelector(
      ".buy_row .price_value, .price_value, .meta_price, button.buy_btn .price_value"
    );
    priceRaw = text(priceEl);
    if (!priceRaw) {
      const buy = text(document.querySelector(".buy_btn, .buy_row"));
      const m = buy.match(/\$\s*[\d,.]+|Free|Name your price/i);
      if (m) priceRaw = m[0];
    }
    if (
      !priceRaw &&
      /download for free|free download/i.test(document.body.innerText.slice(0, 5000))
    ) {
      priceRaw = "Free";
    }
    const { price, priceType } = parsePrice(priceRaw || "Free");

    let rating = null;
    let ratingCount = null;
    const aggregate = document.querySelector(
      '[itemprop="ratingValue"], .aggregate_rating, .game_rating'
    );
    if (aggregate) {
      const rv =
        aggregate.getAttribute("content") ||
        aggregate.getAttribute("data-rating") ||
        text(aggregate);
      const parsed = parseRatingFromText(rv + " " + text(aggregate.parentElement));
      rating = parsed.rating;
      ratingCount = parsed.ratingCount;
    }
    const countEl = document.querySelector('[itemprop="ratingCount"], .rating_count');
    if (countEl) {
      const n = parseInt(
        (countEl.getAttribute("content") || text(countEl)).replace(/[^\d]/g, ""),
        10
      );
      if (!Number.isNaN(n)) ratingCount = n;
    }
    const starRow = document.querySelector(".star_value, .game_rating_widget");
    if (starRow && rating == null) {
      const p = parseRatingFromText(
        text(starRow) + " " + (starRow.getAttribute("title") || "")
      );
      rating = p.rating || rating;
      ratingCount = p.ratingCount || ratingCount;
    }

    const tags = [];
    document
      .querySelectorAll(
        ".game_info_panel_widget a[href*='/games/tag-'], a.game_tag, .tag_list a, .game_tags a"
      )
      .forEach((a) => {
        const t = text(a);
        if (t && !tags.includes(t)) tags.push(t);
      });

    const platforms = [];
    document
      .querySelectorAll(".game_info_panel_widget, .info_panel_wrapper")
      .forEach((panel) => {
        const t = text(panel);
        if (/windows/i.test(t)) platforms.push("Windows");
        if (/mac|osx|os x/i.test(t)) platforms.push("macOS");
        if (/linux/i.test(t)) platforms.push("Linux");
        if (/android/i.test(t)) platforms.push("Android");
        if (/html5|web/i.test(t)) platforms.push("Browser");
      });
    const playInBrowser =
      !!document.querySelector(".play_btn, .button[href*='html'], iframe.game_frame") ||
      /play in browser/i.test(document.body.innerText.slice(0, 8000));

    let genre = "";
    document
      .querySelectorAll(".game_info_panel_widget tr, .info_panel_wrapper tr")
      .forEach((tr) => {
        const label = text(tr.querySelector("td:first-child, th"));
        if (/^genre$/i.test(label)) genre = text(tr.querySelector("td:last-child"));
      });

    return {
      title,
      url: normalizeUrl(location.href),
      developer,
      price,
      priceType,
      rating,
      ratingCount,
      tags,
      genre,
      platforms: [...new Set(platforms)],
      playInBrowser,
      rankOnPage: null,
      listRank: null,
      listName: "game-page",
      gameId: "",
      sourcePage: location.href,
    };
  }

  function gmGet(url) {
    return new Promise((resolve, reject) => {
      if (typeof GM_xmlhttpRequest !== "function") {
        reject(new Error("GM_xmlhttpRequest not available"));
        return;
      }
      GM_xmlhttpRequest({
        method: "GET",
        url,
        timeout: 20000,
        headers: {
          Accept: "text/html,application/xhtml+xml",
          "Cache-Control": "no-cache",
        },
        onload(res) {
          if (res.status >= 200 && res.status < 300) resolve(res.responseText || "");
          else reject(new Error("HTTP " + res.status + " for " + url));
        },
        onerror() {
          reject(new Error("Network error for " + url));
        },
        ontimeout() {
          reject(new Error("Timeout for " + url));
        },
      });
    });
  }

  function fetchDataJson(gameUrl, cb) {
    let dataUrl;
    try {
      const u = new URL(gameUrl);
      dataUrl = u.origin + u.pathname.replace(/\/+$/, "") + "/data.json";
    } catch {
      cb(null);
      return;
    }
    if (typeof GM_xmlhttpRequest !== "function") {
      cb(null);
      return;
    }
    GM_xmlhttpRequest({
      method: "GET",
      url: dataUrl,
      timeout: 8000,
      onload(res) {
        try {
          if (res.status < 200 || res.status >= 300) return cb(null);
          cb(JSON.parse(res.responseText));
        } catch {
          cb(null);
        }
      },
      onerror() {
        cb(null);
      },
      ontimeout() {
        cb(null);
      },
    });
  }

  function mergeDataJson(game, data) {
    if (!data) return game;
    const g = data.game || data;
    if (g.title) game.title = g.title;
    if (g.price != null) {
      const cents = Number(g.price);
      if (!Number.isNaN(cents)) {
        if (cents === 0 && g.min_price == null) {
          game.price = "Free";
          game.priceType = "free";
        } else {
          const dollars = (cents / 100).toFixed(2).replace(/\.00$/, "");
          game.price = "$" + dollars;
          game.priceType = "paid";
        }
      }
    }
    if (g.min_price != null && Number(g.min_price) === 0 && game.priceType !== "paid") {
      game.priceType = game.priceType === "free" ? "free" : "name-your-price";
    }
    if (g.user?.display_name) game.developer = g.user.display_name;
    if (g.p_windows) game.platforms = uniqPush(game.platforms, "Windows");
    if (g.p_osx) game.platforms = uniqPush(game.platforms, "macOS");
    if (g.p_linux) game.platforms = uniqPush(game.platforms, "Linux");
    if (g.p_android) game.platforms = uniqPush(game.platforms, "Android");
    if (g.type === "html" || g.p_web) {
      game.platforms = uniqPush(game.platforms, "Browser");
      game.playInBrowser = true;
    }
    if (Array.isArray(g.tags)) {
      g.tags.forEach((t) => {
        const name = typeof t === "string" ? t : t.name || t.tag;
        if (name && !game.tags.includes(name)) game.tags.push(name);
      });
    }
    if (g.id) game.gameId = String(g.id);
    return game;
  }

  function uniqPush(arr, v) {
    const a = Array.isArray(arr) ? arr.slice() : [];
    if (!a.includes(v)) a.push(v);
    return a;
  }

  function sleep(ms) {
    return new Promise((r) => setTimeout(r, ms));
  }

  function pageUrl(path, page) {
    const base = "https://itch.io" + path;
    if (page <= 1) return base;
    const join = base.includes("?") ? "&" : "?";
    return base + join + "page=" + page;
  }

  /**
   * Fetch multiple browse pages until we have ~max games.
   * itch usually shows ~36–48 games per page.
   */
  async function discoverList(preset, onProgress) {
    const max = preset.max || 100;
    const listName = preset.label || preset.id;
    const collected = [];
    const seen = new Set();
    let page = 1;
    let emptyPages = 0;
    const maxPages = Math.ceil(max / 24) + 2; // safety

    while (collected.length < max && page <= maxPages && emptyPages < 2) {
      const url = pageUrl(preset.path, page);
      if (onProgress) {
        onProgress(`Fetching ${listName}… page ${page} (${collected.length}/${max})`);
      }
      let html;
      try {
        html = await gmGet(url);
      } catch (err) {
        throw new Error((err && err.message) || "Failed to fetch " + url);
      }

      const batch = scrapeGamesFromHtml(html, url, collected.length, listName);
      let newOnPage = 0;
      for (const g of batch) {
        if (seen.has(g.url)) continue;
        seen.add(g.url);
        collected.push(g);
        newOnPage++;
        if (collected.length >= max) break;
      }

      if (newOnPage === 0) emptyPages++;
      else emptyPages = 0;

      page++;
      if (collected.length < max) await sleep(400); // be polite to itch.io
    }

    return collected.slice(0, max);
  }

  async function runDiscover(preset) {
    if (isDiscovering) {
      toast("Already finding games… please wait.");
      return;
    }
    isDiscovering = true;
    setDiscoverBusy(true);
    try {
      const games = await discoverList(preset, (msg) => toast(msg, true));
      if (!games.length) {
        toast("No games found. Try again on itch.io, or check your connection.");
        return;
      }
      const res = upsertGames(games);
      toast(
        `Found ${games.length} from “${preset.label}” · +${res.added} new · ${res.total} total`
      );
      refreshFabBadge();
      if (document.getElementById("itchnova-dashboard")?.classList.contains("open")) {
        renderDashboard();
      }
    } catch (err) {
      console.error("[ItchNova]", err);
      toast("Find failed: " + ((err && err.message) || "unknown error"));
    } finally {
      isDiscovering = false;
      setDiscoverBusy(false);
    }
  }

  async function runDiscoverCustomTag() {
    const raw = prompt(
      "Enter an itch.io tag (e.g. horror, puzzle, pixel-art, visual-novel):",
      "horror"
    );
    if (raw == null) return;
    const tag = raw
      .trim()
      .toLowerCase()
      .replace(/^tag-/, "")
      .replace(/\s+/g, "-")
      .replace(/[^a-z0-9-]/g, "");
    if (!tag) {
      toast("Invalid tag.");
      return;
    }
    const countRaw = prompt("How many games to fetch? (max 200)", "100");
    let max = parseInt(countRaw, 10);
    if (Number.isNaN(max) || max < 1) max = 100;
    max = Math.min(200, max);

    await runDiscover({
      id: "tag-" + tag,
      label: "Tag: " + tag,
      path: "/games/tag-" + tag,
      max,
    });
  }

  // ── actions ──────────────────────────────────────────────
  function toast(msg, sticky) {
    let el = document.getElementById("itchnova-toast");
    if (!el) {
      el = document.createElement("div");
      el.id = "itchnova-toast";
      el.style.cssText =
        "position:fixed;bottom:24px;left:50%;transform:translateX(-50%);background:#1a1a2e;color:#fff;padding:10px 18px;border-radius:10px;z-index:2147483647;font:14px/1.4 system-ui,sans-serif;box-shadow:0 8px 24px rgba(0,0,0,.35);max-width:90vw;transition:opacity .2s;";
      document.body.appendChild(el);
    }
    el.textContent = msg;
    el.style.opacity = "1";
    clearTimeout(el._t);
    if (!sticky) {
      el._t = setTimeout(() => {
        el.style.opacity = "0";
      }, 3200);
    }
  }

  function setDiscoverBusy(busy) {
    document.querySelectorAll("[data-discover], [data-discover-custom]").forEach((btn) => {
      btn.disabled = !!busy;
      btn.style.opacity = busy ? "0.55" : "1";
    });
  }

  function saveVisible() {
    let games = scrapeListPage();
    const pageGame = scrapeGamePage();
    if (pageGame) games = games.concat([pageGame]);

    const seen = new Set();
    games = games.filter((g) => {
      const k = normalizeUrl(g.url);
      if (seen.has(k)) return false;
      seen.add(k);
      return true;
    });

    if (!games.length) {
      toast("No games found on this page.");
      return;
    }

    const maybeEnrich = (done) => {
      if (pageGame && games.length <= 3) {
        let pending = games.length;
        if (!pending) return done();
        games.forEach((g, i) => {
          fetchDataJson(g.url, (data) => {
            if (data) games[i] = mergeDataJson(games[i], data);
            pending--;
            if (pending <= 0) done();
          });
        });
      } else {
        done();
      }
    };

    maybeEnrich(() => {
      const res = upsertGames(games);
      toast(
        `Saved ${games.length} game(s) · +${res.added} new · ${res.total} total in library`
      );
      refreshFabBadge();
    });
  }

  function clearLibrary() {
    if (!confirm("Delete all saved ItchNova games?")) return;
    saveGames({});
    toast("Library cleared.");
    refreshFabBadge();
    if (document.getElementById("itchnova-dashboard")) renderDashboard();
  }

  // ── UI ───────────────────────────────────────────────────
  function injectStyles() {
    if (document.getElementById("itchnova-styles")) return;
    const s = document.createElement("style");
    s.id = "itchnova-styles";
    s.textContent = `
      #itchnova-fab {
        position: fixed; top: 72px; right: 16px; z-index: 2147483646;
        width: 44px; height: 44px; border-radius: 12px; border: none; cursor: pointer;
        background: linear-gradient(135deg, #7c3aed, #a855f7); color: #fff;
        font: 700 20px/44px system-ui, sans-serif; text-align: center;
        box-shadow: 0 6px 20px rgba(124,58,237,.45);
      }
      #itchnova-fab:hover { filter: brightness(1.08); }
      #itchnova-fab .badge {
        position: absolute; top: -6px; right: -6px; min-width: 18px; height: 18px;
        padding: 0 5px; border-radius: 999px; background: #22c55e; color: #052e16;
        font: 700 11px/18px system-ui,sans-serif;
      }
      #itchnova-menu {
        position: fixed; top: 122px; right: 16px; z-index: 2147483646;
        background: #111827; color: #f9fafb; border-radius: 12px; padding: 8px;
        box-shadow: 0 12px 40px rgba(0,0,0,.4); min-width: 260px; max-height: 80vh; overflow: auto;
        font: 14px/1.4 system-ui,sans-serif; display: none;
      }
      #itchnova-menu.open { display: block; }
      #itchnova-menu button {
        display: block; width: 100%; text-align: left; background: transparent;
        border: none; color: inherit; padding: 9px 12px; border-radius: 8px; cursor: pointer;
        font: inherit;
      }
      #itchnova-menu button:hover { background: #1f2937; }
      #itchnova-menu button:disabled { cursor: wait; }
      #itchnova-menu .muted { color: #9ca3af; font-size: 12px; padding: 6px 12px 4px; }
      #itchnova-menu .sep { height: 1px; background: #1f2937; margin: 6px 8px; }
      #itchnova-dashboard {
        position: fixed; inset: 0; z-index: 2147483647; background: rgba(0,0,0,.55);
        display: none; align-items: center; justify-content: center; padding: 20px;
        font: 14px/1.45 system-ui, -apple-system, sans-serif;
      }
      #itchnova-dashboard.open { display: flex; }
      #itchnova-dashboard .panel {
        background: #0f172a; color: #e2e8f0; width: min(1100px, 100%);
        height: min(85vh, 820px); border-radius: 16px; display: flex; flex-direction: column;
        box-shadow: 0 24px 80px rgba(0,0,0,.5); overflow: hidden;
      }
      #itchnova-dashboard .head {
        display: flex; align-items: center; gap: 12px; padding: 14px 16px;
        border-bottom: 1px solid #1e293b; background: #111827; flex-wrap: wrap;
      }
      #itchnova-dashboard .head h1 {
        margin: 0; font-size: 16px; font-weight: 700; color: #c4b5fd; flex: 1; min-width: 140px;
      }
      #itchnova-dashboard .tabs { display: flex; gap: 4px; flex-wrap: wrap; }
      #itchnova-dashboard .tabs button {
        background: #1e293b; border: none; color: #cbd5e1; padding: 8px 12px;
        border-radius: 8px; cursor: pointer; font: inherit;
      }
      #itchnova-dashboard .tabs button.active { background: #7c3aed; color: #fff; }
      #itchnova-dashboard .close-btn {
        background: transparent; border: none; color: #94a3b8; font-size: 22px;
        cursor: pointer; line-height: 1; padding: 4px 8px;
      }
      #itchnova-dashboard .body { flex: 1; overflow: auto; padding: 16px; }
      #itchnova-dashboard .toolbar {
        display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px; align-items: center;
      }
      #itchnova-dashboard input, #itchnova-dashboard select {
        background: #1e293b; border: 1px solid #334155; color: #f1f5f9;
        border-radius: 8px; padding: 8px 10px; font: inherit;
      }
      #itchnova-dashboard input[type="search"] { min-width: 200px; flex: 1; }
      #itchnova-dashboard table { width: 100%; border-collapse: collapse; font-size: 13px; }
      #itchnova-dashboard th, #itchnova-dashboard td {
        text-align: left; padding: 8px 10px; border-bottom: 1px solid #1e293b; vertical-align: top;
      }
      #itchnova-dashboard th {
        color: #94a3b8; font-weight: 600; position: sticky; top: 0; background: #0f172a;
        cursor: pointer; white-space: nowrap;
      }
      #itchnova-dashboard th:hover { color: #c4b5fd; }
      #itchnova-dashboard a { color: #a78bfa; text-decoration: none; }
      #itchnova-dashboard a:hover { text-decoration: underline; }
      #itchnova-dashboard .score { color: #4ade80; font-weight: 700; }
      #itchnova-dashboard .btn {
        background: #7c3aed; color: #fff; border: none; border-radius: 8px;
        padding: 9px 14px; cursor: pointer; font: inherit; font-weight: 600;
      }
      #itchnova-dashboard .btn.secondary { background: #334155; }
      #itchnova-dashboard .btn:disabled { opacity: 0.55; cursor: wait; }
      #itchnova-dashboard .btn:hover { filter: brightness(1.08); }
      #itchnova-dashboard .hint { color: #94a3b8; font-size: 13px; margin: 0 0 12px; }
      #itchnova-dashboard .stat { color: #e2e8f0; margin-bottom: 8px; }
      #itchnova-dashboard .empty { color: #94a3b8; padding: 40px; text-align: center; }
      #itchnova-dashboard .sort-bar {
        display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 12px; align-items: center;
      }
      #itchnova-dashboard .sort-bar .sort-label {
        color: #94a3b8; font-size: 12px; margin-right: 4px; font-weight: 600;
      }
      #itchnova-dashboard .sort-bar button {
        background: #1e293b; border: 1px solid #334155; color: #cbd5e1;
        border-radius: 999px; padding: 6px 12px; cursor: pointer; font: 12px/1.3 system-ui,sans-serif;
      }
      #itchnova-dashboard .sort-bar button:hover { border-color: #7c3aed; color: #e2e8f0; }
      #itchnova-dashboard .sort-bar button.active {
        background: #7c3aed; border-color: #7c3aed; color: #fff; font-weight: 600;
      }
      #itchnova-dashboard .discover-grid {
        display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 10px;
      }
      #itchnova-dashboard .discover-card {
        background: #1e293b; border: 1px solid #334155; border-radius: 12px;
        padding: 14px; display: flex; flex-direction: column; gap: 8px;
      }
      #itchnova-dashboard .discover-card strong { color: #e2e8f0; font-size: 14px; }
      #itchnova-dashboard .discover-card span { color: #94a3b8; font-size: 12px; }
    `;
    document.documentElement.appendChild(s);
  }

  function refreshFabBadge() {
    const badge = document.querySelector("#itchnova-fab .badge");
    if (!badge) return;
    const n = Object.keys(loadGames()).length;
    badge.textContent = n > 999 ? "999+" : String(n);
    badge.style.display = n ? "block" : "none";
  }

  function mountChrome() {
    if (document.getElementById("itchnova-fab")) return;
    injectStyles();

    const fab = document.createElement("button");
    fab.id = "itchnova-fab";
    fab.type = "button";
    fab.title = "ItchNova";
    fab.innerHTML = 'N<span class="badge" style="display:none">0</span>';
    document.body.appendChild(fab);

    const menu = document.createElement("div");
    menu.id = "itchnova-menu";
    menu.innerHTML = `
      <div class="muted">ItchNova · Market Intel</div>
      <button type="button" data-act="save">Save visible games</button>
      <button type="button" data-act="dash">Open Dashboard</button>
      <div class="sep"></div>
      <div class="muted">Find games (auto-fetch)</div>
      <button type="button" data-discover="top-sellers-100">★ Top sellers (≈100)</button>
      <button type="button" data-discover="top-rated-100">Top rated (≈100)</button>
      <button type="button" data-discover="popular-100">Popular (≈100)</button>
      <button type="button" data-discover="new-popular-100">New & popular (≈100)</button>
      <button type="button" data-discover="free-top-100">Popular free (≈100)</button>
      <button type="button" data-discover-custom="1">Find by tag…</button>
      <div class="sep"></div>
      <button type="button" data-act="clear">Clear library…</button>
    `;
    document.body.appendChild(menu);

    fab.addEventListener("click", (e) => {
      e.stopPropagation();
      menu.classList.toggle("open");
    });
    document.addEventListener("click", () => menu.classList.remove("open"));
    menu.addEventListener("click", (e) => e.stopPropagation());

    menu.querySelector('[data-act="save"]').addEventListener("click", () => {
      menu.classList.remove("open");
      saveVisible();
    });
    menu.querySelector('[data-act="dash"]').addEventListener("click", () => {
      menu.classList.remove("open");
      openDashboard();
    });
    menu.querySelector('[data-act="clear"]').addEventListener("click", () => {
      menu.classList.remove("open");
      clearLibrary();
    });

    menu.querySelectorAll("[data-discover]").forEach((btn) => {
      btn.addEventListener("click", () => {
        menu.classList.remove("open");
        const id = btn.getAttribute("data-discover");
        const preset = DISCOVER_PRESETS.find((p) => p.id === id);
        if (preset) runDiscover(preset);
      });
    });
    menu.querySelector("[data-discover-custom]").addEventListener("click", () => {
      menu.classList.remove("open");
      runDiscoverCustomTag();
    });

    refreshFabBadge();
  }

  // ── dashboard ────────────────────────────────────────────
  let dashState = {
    tab: "games",
    q: "",
    priceType: "all",
    minRatings: 0,
    sortKey: "listRank",
    sortDir: "asc",
    listFilter: "all",
  };

  function openDashboard() {
    let root = document.getElementById("itchnova-dashboard");
    if (!root) {
      root = document.createElement("div");
      root.id = "itchnova-dashboard";
      root.innerHTML = `
        <div class="panel" role="dialog" aria-label="ItchNova Dashboard">
          <div class="head">
            <h1>ItchNova · Market Intel</h1>
            <div class="tabs">
              <button type="button" data-tab="find" class="">Find</button>
              <button type="button" data-tab="games" class="active">Games</button>
              <button type="button" data-tab="filters">Filters</button>
              <button type="button" data-tab="export">Export</button>
            </div>
            <button type="button" class="close-btn" title="Close">×</button>
          </div>
          <div class="body" id="itchnova-dash-body"></div>
        </div>
      `;
      document.body.appendChild(root);
      root.querySelector(".close-btn").addEventListener("click", () => root.classList.remove("open"));
      root.addEventListener("click", (e) => {
        if (e.target === root) root.classList.remove("open");
      });
      root.querySelectorAll(".tabs button").forEach((btn) => {
        btn.addEventListener("click", () => {
          dashState.tab = btn.getAttribute("data-tab");
          root.querySelectorAll(".tabs button").forEach((b) =>
            b.classList.toggle("active", b === btn)
          );
          renderDashboard();
        });
      });
    }
    root.classList.add("open");
    // highlight correct tab
    root.querySelectorAll(".tabs button").forEach((b) => {
      b.classList.toggle("active", b.getAttribute("data-tab") === dashState.tab);
    });
    renderDashboard();
  }

  function getFilteredGames() {
    let list = Object.values(loadGames());
    const q = (dashState.q || "").trim().toLowerCase();
    if (q) {
      list = list.filter((g) => {
        const blob = [
          g.title,
          g.developer,
          g.price,
          g.genre,
          g.listName,
          ...(g.tags || []),
          ...(g.platforms || []),
        ]
          .join(" ")
          .toLowerCase();
        return blob.includes(q);
      });
    }
    if (dashState.priceType && dashState.priceType !== "all") {
      list = list.filter((g) => (g.priceType || "unknown") === dashState.priceType);
    }
    if (dashState.listFilter && dashState.listFilter !== "all") {
      list = list.filter((g) => (g.listName || "") === dashState.listFilter);
    }
    const minR = Number(dashState.minRatings) || 0;
    if (minR > 0) {
      list = list.filter((g) => (Number(g.ratingCount) || 0) >= minR);
    }

    const key = dashState.sortKey;
    const dir = dashState.sortDir === "asc" ? 1 : -1;

    function priceNumber(g) {
      // Free / name-your-price sort as 0; unknown as null
      if (g.priceType === "free") return 0;
      if (g.priceType === "name-your-price") return 0;
      const s = String(g.price || "");
      const m = s.replace(/,/g, "").match(/(\d+(?:\.\d+)?)/);
      if (m) return parseFloat(m[1]);
      return null;
    }

    function sortValue(g, k) {
      if (k === "priceNum") return priceNumber(g);
      if (k === "price") return priceNumber(g);
      if (k === "updatedAt" || k === "savedAt") {
        const t = Date.parse(g[k] || g.updatedAt || g.savedAt || "");
        return Number.isNaN(t) ? null : t;
      }
      return g[k];
    }

    list.sort((a, b) => {
      let av = sortValue(a, key);
      let bv = sortValue(b, key);
      if (key === "title" || key === "developer" || key === "listName") {
        av = (av || "").toString().toLowerCase();
        bv = (bv || "").toString().toLowerCase();
        if (av < bv) return -1 * dir;
        if (av > bv) return 1 * dir;
        return 0;
      }
      // Missing numbers sink to the bottom for both directions
      const aMiss = av == null || av === "" || Number.isNaN(Number(av));
      const bMiss = bv == null || bv === "" || Number.isNaN(Number(bv));
      if (aMiss && bMiss) return 0;
      if (aMiss) return 1;
      if (bMiss) return -1;
      av = Number(av) || 0;
      bv = Number(bv) || 0;
      return (av - bv) * dir;
    });
    return list;
  }

  function renderDashboard() {
    const body = document.getElementById("itchnova-dash-body");
    if (!body) return;
    const all = Object.values(loadGames());
    const list = getFilteredGames();

    if (dashState.tab === "find") {
      body.innerHTML = `
        <p class="hint">
          One click pulls public browse pages from itch.io into your library.
          Top sellers is the best “what’s selling right now” list. Takes ~5–15 seconds for 100 games.
        </p>
        <div class="discover-grid" id="itchnova-discover-grid">
          ${DISCOVER_PRESETS.map(
            (p) => `
            <div class="discover-card">
              <strong>${escapeHtml(p.label)}</strong>
              <span>Fetches up to ${p.max} games from itch.io${escapeHtml(p.path)}</span>
              <button type="button" class="btn" data-discover="${escapeAttr(p.id)}" ${
              isDiscovering ? "disabled" : ""
            }>Fetch now</button>
            </div>`
          ).join("")}
          <div class="discover-card">
            <strong>Any tag</strong>
            <span>e.g. horror, puzzle, pixel-art — up to 200 games</span>
            <button type="button" class="btn secondary" data-discover-custom="1" ${
              isDiscovering ? "disabled" : ""
            }>Find by tag…</button>
          </div>
        </div>
        <p class="stat" style="margin-top:16px">Library: <strong>${all.length}</strong> games saved</p>
      `;
      body.querySelectorAll("[data-discover]").forEach((btn) => {
        btn.addEventListener("click", () => {
          const id = btn.getAttribute("data-discover");
          const preset = DISCOVER_PRESETS.find((p) => p.id === id);
          if (preset) runDiscover(preset);
        });
      });
      const custom = body.querySelector("[data-discover-custom]");
      if (custom) custom.addEventListener("click", () => runDiscoverCustomTag());
      return;
    }

    if (dashState.tab === "games") {
      const SORT_BUTTONS = [
        { key: "listRank", dir: "asc", label: "List rank" },
        { key: "successScore", dir: "desc", label: "Success score" },
        { key: "rating", dir: "desc", label: "Star rating" },
        { key: "ratingCount", dir: "desc", label: "# of ratings" },
        { key: "priceNum", dir: "asc", label: "Price: low → high" },
        { key: "priceNum", dir: "desc", label: "Price: high → low" },
        { key: "title", dir: "asc", label: "Title A–Z" },
        { key: "developer", dir: "asc", label: "Developer A–Z" },
        { key: "updatedAt", dir: "desc", label: "Recently saved" },
      ];

      const dirArrow =
        dashState.sortDir === "asc" ? "↑" : "↓";
      const activeSortLabel =
        SORT_BUTTONS.find(
          (b) => b.key === dashState.sortKey && b.dir === dashState.sortDir
        )?.label ||
        `${dashState.sortKey} ${dirArrow}`;

      body.innerHTML = `
        <p class="stat">${list.length} shown · ${all.length} saved total · sorted by <strong>${escapeHtml(
          activeSortLabel
        )}</strong></p>
        <div class="toolbar">
          <input type="search" id="itchnova-q" placeholder="Search title, tag, dev…" value="${escapeAttr(
            dashState.q
          )}" />
          <button type="button" class="btn secondary" id="go-find">Find more games</button>
        </div>
        <div class="sort-bar" id="itchnova-sort-bar">
          <span class="sort-label">Sort by</span>
          ${SORT_BUTTONS.map((b) => {
            const active =
              dashState.sortKey === b.key && dashState.sortDir === b.dir;
            return `<button type="button" data-sort-key="${escapeAttr(
              b.key
            )}" data-sort-dir="${escapeAttr(b.dir)}" class="${
              active ? "active" : ""
            }">${escapeHtml(b.label)}</button>`;
          }).join("")}
          <button type="button" id="sort-flip" title="Reverse current order">⇅ Flip order</button>
        </div>
        ${
          list.length
            ? `<table>
          <thead>
            <tr>
              <th data-sort="listRank">#</th>
              <th data-sort="successScore">Score</th>
              <th data-sort="title">Title</th>
              <th data-sort="developer">Developer</th>
              <th data-sort="priceNum">Price</th>
              <th data-sort="rating">Rating</th>
              <th data-sort="ratingCount"># Ratings</th>
              <th data-sort="listName">List</th>
              <th>Tags</th>
            </tr>
          </thead>
          <tbody>
            ${list
              .map(
                (g) => `<tr>
              <td>${g.listRank != null ? g.listRank : "—"}</td>
              <td class="score">${g.successScore || 0}</td>
              <td><a href="${escapeAttr(g.url)}" target="_blank" rel="noopener">${escapeHtml(
                  g.title
                )}</a></td>
              <td>${escapeHtml(g.developer || "")}</td>
              <td>${escapeHtml(g.price || "—")}</td>
              <td>${g.rating != null ? g.rating : "—"}</td>
              <td>${
                g.ratingCount != null ? Number(g.ratingCount).toLocaleString() : "—"
              }</td>
              <td>${escapeHtml(g.listName || "")}</td>
              <td>${escapeHtml((g.tags || []).slice(0, 5).join(", "))}</td>
            </tr>`
              )
              .join("")}
          </tbody>
        </table>`
            : `<div class="empty">No games yet.<br/>Open the <strong>Find</strong> tab and fetch Top sellers.</div>`
        }
      `;
      const q = body.querySelector("#itchnova-q");
      if (q) {
        q.addEventListener("input", () => {
          dashState.q = q.value;
          const pos = q.selectionStart;
          renderDashboard();
          const nq = document.getElementById("itchnova-q");
          if (nq) {
            nq.focus();
            try {
              nq.setSelectionRange(pos, pos);
            } catch {}
          }
        });
      }
      const goFind = body.querySelector("#go-find");
      if (goFind) {
        goFind.addEventListener("click", () => {
          dashState.tab = "find";
          document.querySelectorAll("#itchnova-dashboard .tabs button").forEach((b) => {
            b.classList.toggle("active", b.getAttribute("data-tab") === "find");
          });
          renderDashboard();
        });
      }
      body.querySelectorAll("#itchnova-sort-bar button[data-sort-key]").forEach((btn) => {
        btn.addEventListener("click", () => {
          dashState.sortKey = btn.getAttribute("data-sort-key");
          dashState.sortDir = btn.getAttribute("data-sort-dir") || "desc";
          renderDashboard();
        });
      });
      const flip = body.querySelector("#sort-flip");
      if (flip) {
        flip.addEventListener("click", () => {
          dashState.sortDir = dashState.sortDir === "asc" ? "desc" : "asc";
          renderDashboard();
        });
      }
      body.querySelectorAll("th[data-sort]").forEach((th) => {
        th.addEventListener("click", () => {
          const k = th.getAttribute("data-sort");
          if (dashState.sortKey === k) {
            dashState.sortDir = dashState.sortDir === "asc" ? "desc" : "asc";
          } else {
            dashState.sortKey = k;
            dashState.sortDir =
              k === "title" ||
              k === "developer" ||
              k === "listName" ||
              k === "listRank" ||
              k === "priceNum"
                ? "asc"
                : "desc";
          }
          renderDashboard();
        });
      });
      return;
    }

    if (dashState.tab === "filters") {
      const listNames = [
        ...new Set(all.map((g) => g.listName).filter(Boolean)),
      ].sort();
      body.innerHTML = `
        <p class="hint">Filters apply to the Games table and Export.</p>
        <div class="toolbar" style="flex-direction:column;align-items:stretch;max-width:420px">
          <label>Search<br/><input type="search" id="f-q" value="${escapeAttr(
            dashState.q
          )}" placeholder="e.g. horror, puzzle…" style="width:100%;margin-top:4px" /></label>
          <label>Price type<br/>
            <select id="f-price" style="width:100%;margin-top:4px">
              <option value="all">All</option>
              <option value="free">Free</option>
              <option value="paid">Paid</option>
              <option value="name-your-price">Name your price</option>
              <option value="unknown">Unknown</option>
            </select>
          </label>
          <label>From list<br/>
            <select id="f-list" style="width:100%;margin-top:4px">
              <option value="all">All lists</option>
              ${listNames
                .map((n) => `<option value="${escapeAttr(n)}">${escapeHtml(n)}</option>`)
                .join("")}
            </select>
          </label>
          <label>Minimum # of ratings<br/>
            <input type="number" id="f-min" min="0" step="1" value="${escapeAttr(
              String(dashState.minRatings || 0)
            )}" style="width:100%;margin-top:4px" />
          </label>
          <div style="display:flex;gap:8px;margin-top:8px">
            <button type="button" class="btn" id="f-apply">Apply filters</button>
            <button type="button" class="btn secondary" id="f-reset">Reset</button>
          </div>
        </div>
        <p class="stat" style="margin-top:16px">${list.length} games match · ${all.length} total</p>
      `;
      body.querySelector("#f-price").value = dashState.priceType || "all";
      body.querySelector("#f-list").value = dashState.listFilter || "all";
      body.querySelector("#f-apply").addEventListener("click", () => {
        dashState.q = body.querySelector("#f-q").value;
        dashState.priceType = body.querySelector("#f-price").value;
        dashState.listFilter = body.querySelector("#f-list").value;
        dashState.minRatings = Number(body.querySelector("#f-min").value) || 0;
        dashState.tab = "games";
        document.querySelectorAll("#itchnova-dashboard .tabs button").forEach((b) => {
          b.classList.toggle("active", b.getAttribute("data-tab") === "games");
        });
        renderDashboard();
      });
      body.querySelector("#f-reset").addEventListener("click", () => {
        dashState.q = "";
        dashState.priceType = "all";
        dashState.listFilter = "all";
        dashState.minRatings = 0;
        renderDashboard();
      });
      return;
    }

    // export
    const csv = toCSV(list);
    body.innerHTML = `
      <p class="hint">Exports the <strong>currently filtered</strong> list (${list.length} games).</p>
      <div class="toolbar">
        <button type="button" class="btn" id="ex-dl">Download CSV</button>
        <button type="button" class="btn secondary" id="ex-copy">Copy to clipboard</button>
      </div>
      <p class="stat">Includes list rank / list name when fetched via Find.</p>
      <textarea readonly style="width:100%;height:220px;background:#1e293b;color:#e2e8f0;border:1px solid #334155;border-radius:8px;padding:10px;font:12px/1.4 ui-monospace,monospace">${escapeHtml(
        csv.slice(0, 5000)
      )}${csv.length > 5000 ? "\n…(preview truncated)" : ""}</textarea>
    `;
    body.querySelector("#ex-dl").addEventListener("click", () => downloadCSV(csv));
    body.querySelector("#ex-copy").addEventListener("click", async () => {
      try {
        await navigator.clipboard.writeText(csv);
        toast("CSV copied to clipboard");
      } catch {
        const ta = document.createElement("textarea");
        ta.value = csv;
        document.body.appendChild(ta);
        ta.select();
        document.execCommand("copy");
        ta.remove();
        toast("CSV copied to clipboard");
      }
    });
  }

  function toCSV(list) {
    const cols = [
      "listRank",
      "listName",
      "title",
      "url",
      "developer",
      "price",
      "priceType",
      "rating",
      "ratingCount",
      "successScore",
      "tags",
      "platforms",
      "playInBrowser",
      "genre",
      "savedAt",
      "sourcePage",
    ];
    const lines = [cols.join(",")];
    for (const g of list) {
      const row = cols.map((c) => {
        let v = g[c];
        if (Array.isArray(v)) v = v.join("; ");
        if (v == null) v = "";
        return csvCell(String(v));
      });
      lines.push(row.join(","));
    }
    return lines.join("\n");
  }

  function csvCell(s) {
    if (/[",\n\r]/.test(s)) return `"${s.replace(/"/g, '""')}"`;
    return s;
  }

  function downloadCSV(csv) {
    const blob = new Blob([csv], { type: "text/csv;charset=utf-8" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `itchnova-export-${new Date().toISOString().slice(0, 10)}.csv`;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
    toast("CSV download started");
  }

  function escapeHtml(s) {
    return String(s)
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;");
  }

  function escapeAttr(s) {
    return escapeHtml(s).replace(/'/g, "&#39;");
  }

  function boot() {
    if (!document.body) return;
    mountChrome();
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", boot);
  } else {
    boot();
  }
})();

```
