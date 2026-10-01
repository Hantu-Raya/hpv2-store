# hpv2-store

The save page for the **HP Colors v2** Deadlock mod. The mod loads this page in a hidden browser panel inside the game and uses it to keep your settings on your PC between game launches.

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
| `o` | Operation: `hello`, `r` (read), `w` (write), `d` (delete). |
| `r` | Id of the whole read or write. It stays the same across that operation's chunks. |
| `p` | Chunk index. |
| `k` | Read only: which key to read (current save or backup). |
| `n`, `v` | Write only: total chunk count and this chunk's text. |
| `e` | Write only: checksum of the save the mod last verified. The page moves the old save to the backup key only when it matches. |

Saves are sent in chunks of up to 3,000 characters (64 chunks at most) because replies travel in the page title, which the game caps at 4,096 characters.

### Replies (page → mod)

The page sets its title to `HPV2S1:` followed by JSON with the same `i` and `o`, `ok: true` or `ok: false` with an error, and the result. A `hello` reply also carries the page's protocol version `v` and its address `h`. The mod ignores replies whose id it is not waiting for, and accepts the `hello` only from this page's exact address.

### What it stores

Only two keys:

- `hantu.hpcolors.v2/state`: the current save.
- `hantu.hpcolors.v2/state.prev`: the previous save, kept as a backup.

Every save is checksummed. The page rejects a save whose checksum is wrong, and the mod refuses to overwrite a save it could not read.

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

The page reports a protocol version (`VERSION` in `index.html`, currently `1`). The mod saves only when it matches the version it was built for. Otherwise it stops saving instead of risking your data. Any change to the request or reply format must raise the version, and the mod has to be updated with it.

## Files

- `index.html`: the whole page. A single inline script handles the requests; it has no dependencies.

The mod side is `hp_colors_rewrite_v2/panorama/scripts/hp_colors_v2_storage.js` in [Deadlock-mods-collection](https://github.com/Hantu-Raya/Deadlock-mods-collection). Its end-to-end tests (`scripts/validate-hp-colors-rewrite-v2-storage.test.js`) run this page's script against a simulated game panel.
