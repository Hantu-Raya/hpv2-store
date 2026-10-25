# hpv2-store

The save page for the **HP Colors v2/v3** Deadlock mods. The mods load this page in a hidden browser panel inside the game and use it to keep settings on your PC between game launches.

Live page: <https://hantu-raya.github.io/hpv2-store/>

## Why it exists

Deadlock's HUD can contain a small browser panel (`CitadelHTMLPanel`). Until the game update of 1 October 2026, the mod opened a local `file:///` page in that panel and controlled it with `javascript:` addresses. The update changed the panel so it only opens addresses that start with `https://`; anything else becomes a blank page. The panel has no other way to run script, so the local page stopped working and saving broke.

This page is the replacement. It is served over `https://`, so the panel still opens it.

## How it works

```text
Deadlock HUD (mod script)                        This page (in Steam's browser)
-------------------------                        ------------------------------
SetURL(page + "#" + request)   ───────────────►  reads location.hash
                                                 reads/writes localStorage
HTMLTitle event                ◄───────────────  sets document.title = reply
```

1. At startup the mod opens `https://hantu-raya.github.io/hpv2-store/#<hello>`. The page answers with its version number.
2. For every read, write, or delete, the mod changes only the part of the address after `#`. The browser does not reload the page; the page's `hashchange` handler runs the request.
3. The page answers by setting its title. The game reports title changes to the mod.
4. Settings are kept in the page's `localStorage`, which Steam keeps in its browser profile on your PC (`htmlcache`). They survive game and PC restarts.

### Requests (mod → page)

Each request is `#` followed by URL-encoded JSON:

| Field | Meaning |
|---|---|
| `i` | Unique message id. Every message has a new id, so the address always changes and the page always sees it. |
| `o` | Operation: `hello`, `r` (read), `w` (write), `d` (delete), `lv` (latest release, `latest` capability). |
| `r` | Id of the whole read or write. It stays the same across that operation's chunks. |
| `p` | Chunk index. |
| `k` | Required on reads; optional on writes and deletes. A keyless write targets `hantu.hpcolors.v2/state`; a keyless delete removes both v2 keys. A keyed delete removes only its named key. |
| `n`, `v` | Write only: total chunk count and this chunk's text. |
| `e` | Write only: checksum of the save the mod last verified. The page moves an old current save to that version's `.prev` key only when it matches. |
| `m` | Write only, optional (`cas` capability): the checksum of the whole value the mod last saw at the target key, or `none` if it saw no value. The page commits only if the key still holds that value (or already holds the value being written, for a retried commit); otherwise it refuses with `conflict` and the mod reloads. Writes without `m` (v2) are unconditional, as before. |
| `a` | `lv` only: the mod's id in `latest.json`, 1-16 lowercase letters or digits (HP Colors v3 sends `hpv3`). |

Saves are sent in chunks of up to 3,000 characters (64 chunks at most) because replies travel in the page title, which the game caps at 4,096 characters.

### Replies (page → mod)

The page sets its title to `HPV2S1:` followed by JSON with the same `i` and `o`, `ok: true` or `ok: false` with an error, and the result. A `hello` reply also carries the page's protocol version `v: 1`, its address `h`, and `c: ["vkeys", "cas", "latest"]` (versioned-key, compare-at-commit and latest-release capabilities). v2 checks `v` and `h` and ignores `c`; v3 requires `vkeys` and `cas` so an old cached page fails closed, and asks `lv` only when `latest` is listed. The mod ignores replies whose id it is not waiting for, and accepts the `hello` only from this page's exact address.

An `lv` reply carries `l`, the newest released version for `a` as `X.Y.Z` (or `null` when unknown or malformed); `d`, that release's download page per build (`{"normal": url, "qollock": url}`, only `https://gamebanana.com/` links, or `null`); and `s`: `pending` while the page's first fetch of `latest.json` is still loading, otherwise `done`. The page fetches `./latest.json?n=<time>` with `cache: "no-store"` once when it loads and again at most every 10 minutes (every minute after a failure), triggered only by `lv` requests. It answers from the copy it already holds and never writes the title on its own, so each title still answers one request. GitHub Pages can serve an older `latest.json` for a few minutes after a push.

### What it stores

Only keys matching `^hantu\.hpcolors\.v\d+/state(\.prev)?$` may be read, written or deleted. In particular:

- `hantu.hpcolors.v2/state`: the v2 current save.
- `hantu.hpcolors.v2/state.prev`: the v2 previous save, kept as a backup.
- `hantu.hpcolors.v3/state` and `hantu.hpcolors.v3/state.prev`: separate v3 save and backup.

Every save is checksummed. The page rejects a save whose checksum is wrong, and the mod refuses to overwrite a save it could not read. A v3 save also names the value it expects to replace (`m`), so a save written meanwhile by another game window or a newer mod version is never overwritten.

### Buff Timer page (`/bt/`)

`bt/index.html` (<https://hantu-raya.github.io/hpv2-store/bt/>) saves the settings of the Buff Timer mod (pak98, `buff_timer_virgin/` in Deadlock-mods-collection). It uses the same `#` request / title reply transport, with these differences:

- Replies start with `BTS1:` and the page has its own `VERSION` (1). Changing it never affects the HP Colors page above.
- It stores one key, `hantu.bufftimer.v1/settings`, a checksummed record of a few hundred characters at most. Requests are single messages: `hello`, `r` (read) and `w` (write, field `v` = the record). There is no chunking, backup key or delete.
- It never reads, writes or deletes the HP Colors keys. Both pages share this site's storage, so that is a rule of the code, not a browser boundary.

The mod-side end-to-end test is `buff_timer_virgin/scripts/validate-settings-store.js`; it runs this page's script against a simulated game panel.

### DLF broadcast page (`/dlf/`)

`dlf/index.html` (<https://hantu-raya.github.io/hpv2-store/dlf/>) saves the match details of the DLF observer HUD (pak01, `dlf/` in Deadlock-mods-collection): season and match numbers, series format, team names, series scores and hero bans. It follows the Buff Timer page's rules:

- Replies start with `DLFS1:` and the page has its own `VERSION` (1).
- It stores one key, `hantu.dlf.v1/match`, a checksummed record of at most 1024 characters. Requests are `hello`, `r` and `w`; there is no chunking, backup key or delete.
- It never reads, writes or deletes the HP Colors or Buff Timer keys.

The mod-side end-to-end test is `dlf/scripts/validate-dlf.js`; it runs this page's script against a simulated game panel.

## The fix in the mod

From `hp_colors_rewrite_v2/panorama/scripts/hp_colors_v2_storage.js`. Lines marked `// …` are trimmed.

**Before:** the mod opened a local page and sent it script. Since the 1 October update the panel turns both kinds of address into `about:blank`.

```js
var PAGE_URLS = ["file:///C:/", "file://"];

panel.SetURL(PAGE_URLS[navigations - 1]);           // open the local page
panel.SetURL("javascript:" + code + ";void(0);");   // run each request in it
// code = "window.__hpv2s&&window.__hpv2s.r(...)"
```

**After:** the mod opens this `https://` page once, then sends each request as data after `#`.

```js
var PAGE_URL = "https://hantu-raya.github.io/hpv2-store/";
var PAGE_VERSION = 1;

function send(message) {
  if (!isValid(panel)) return false;
  try {
    panel.SetURL(PAGE_URL + "#" + encodeURIComponent(JSON.stringify(message)));
    return true;
  } catch (error) {
    log("SetURL threw: " + String(error));
    return false;
  }
}

// Example read request; `i` is new for every message so the address always changes.
send({ "i": "h7", "r": "h6", "p": 0, "o": "r", "k": "hantu.hpcolors.v2/state" });
```

Replies still arrive through the page title. The mod trusts the page only when the `hello` reply comes from this exact address with the expected version:

```js
function isStoragePage(href) {
  return typeof href === "string" && href.split("#")[0] === PAGE_URL;
}

function onReady(message) {
  if (ready || unavailable || message.i !== helloId) return;
  if (!isStoragePage(message.h)) {
    log("ignored hello from untrusted page " + String(message.h).slice(0, 80));
    return;
  }
  if (message.v !== PAGE_VERSION || message.ok !== true) {
    markUnavailable("protocol_version (hosted page missing or incompatible)");
    return;
  }
  ready = true;
  // …
}

function onTitle(panelOrTitle, eventTitle) {
  var title = arguments.length > 1 ? eventTitle : panelOrTitle;
  if (typeof title !== "string" || !title) return;
  if (title.indexOf(TITLE_PREFIX) !== 0) return;   // TITLE_PREFIX = "HPV2S1:"
  // … parse the JSON after the prefix into `message`
  if (message.o === "ready") onReady(message);
  else onReply(message);
}
```

On the page side, `index.html` runs each request when the `#` part changes (simplified):

```js
function handle() {
  var request = JSON.parse(decodeURIComponent(location.hash.slice(1)));
  // … run the hello, read, write or delete against localStorage
  document.title = "HPV2S1:" + JSON.stringify(reply);
}

window.addEventListener("hashchange", handle);
handle();   // also handles the request the page was first opened with
```

## Privacy

- Your settings never leave your PC. They travel in the part of the address after `#`, which browsers do not send to the server, and they are stored locally.
- GitHub serves the page file like any other website, so it sees an ordinary page request (IP address, browser user agent) when the game loads it. It does not see your settings.
- The page loads nothing else: no scripts, trackers, or requests to other sites.

## Limits

- **Internet needed to load the page.** If the page cannot be reached, the mod cannot load your saved settings: it starts on defaults and does not save during that session. Your save stays untouched for the next launch.
- **Saves from before 1 October 2026 are not carried over.** Browsers keep each site's storage separate, so this page cannot read what the old local `file:///` page stored. Use the mod's import/export codes to move settings.
- **Updates take a while to arrive.** GitHub Pages lets browsers cache the page for up to 10 minutes, so a new version may not reach the game immediately.

## Versioning

The page reports protocol version `1` (`VERSION` in `index.html`) and the `vkeys`, `cas` and `latest` capabilities in its ready reply. Installed v2 mods require exactly version 1; v3 also requires `vkeys` and `cas` to reject a cached old page. The added optional key field, the optional `m` write field, the `lv` operation and the capabilities leave v2 messages and replies intact. Incompatible changes to the v2 format must update both the page and bridge before release.

`latest.json` lists the newest **released** version of each mod and, under `download`, its GameBanana download page per build (`{"hpv3": "3.1.0", "download": {"hpv3": {"normal": url, "qollock": url}}}`). The mod validates each link again and offers DOWNLOAD only for its own build. Raise a mod's entry, and point its links at the new files, only after that release is downloadable; a 3.0.x HP Colors v3 never asks, and any release whose version is lower than the entry warns its players once per game session.

## Files

- `index.html`: the whole page. A single inline script handles the requests; it has no dependencies.
- `latest.json`: newest released version per mod, read by the page (see Versioning).
- `bt/index.html`, `dlf/index.html`: the Buff Timer and DLF broadcast pages, each with its own key and version.

The mod side is `hp_colors_rewrite_v2/panorama/scripts/hp_colors_v2_storage.js` in [Deadlock-mods-collection](https://github.com/Hantu-Raya/Deadlock-mods-collection). Its end-to-end tests (`scripts/validate-hp-colors-rewrite-v2-storage.test.js`) run this page's script against a simulated game panel.
