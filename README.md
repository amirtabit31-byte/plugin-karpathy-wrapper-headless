> **🌐 Read this in [Italiano](README.it.md)**

---

# How I ran an 82,000-line Obsidian plugin headless, and delivered it to a company on-premise

I sold a small local company an on-premise AI infrastructure: an agent that lives inside their building, with data that never leaves the premises. The hardest part to design was the knowledge base, a wiki that maintains itself, with no screen and without the customer ever touching it.

To pull that off I had to run an Obsidian plugin headless. The normal one lives inside the desktop app, with a screen, a mouse and drop-down menus: here it had to be orchestrated by an AI agent.

The road looked simple. Rewrite the logic as a Hermes skill and move on. Then I opened the plugin bundle and found 3.6MB of compiled code, 82,009 lines. Years of development and dozens of features: dedup, aliases, schema enforcement, automatic lint, LLM analysis.

Rewriting them would have meant weeks of work, with the risk of producing something divergent that would break at the first update.

So I did something else. I built an Obsidian API stub in Node.js, convinced the plugin it was on a desktop, and ran it headless on a remote server.

The result: zero crashes across 82,000 lines, 16 bugs fixed, 68 tests passed out of 68, and a wiki that maintains itself.

![Architecture diagram of the system delivered on-premise](files/diagramma.png)

> *Diagram: the customer's building with the loaned mini PC (AI agent, Honcho memory, headless wrapper, wiki, MCP server), the customer's phone connected via MCP, and the only outgoing flow towards my homelab for monitoring. No content data ever leaves the premises.*

![Wrapper snippet: intercepting require("obsidian")](files/spezzone-codice.png)

> *The heart of the wrapper: `Module.prototype.require` is intercepted, so every `obsidian` request from the 82,000-line bundle gets our stub instead, along with the fake browser globals (`window`, `setTimeout`, `MutationObserver`) the plugin expects.*

---

## Failure 1 of 5: a flood of TypeErrors

First launch of the wrapper: `node index.js ingest file.md`. The answer:

```
TypeError: Cannot read properties of undefined (reading 'getSecret')
TypeError: plugin.addRibbonIcon is not a function
TypeError: plugin.addStatusBarItem is not a function
TypeError: this.registerView is not a function
TypeError: app.vault.getMarkdownFiles(...).filter is not a function
```

Six errors in a row. The plugin expected a full Obsidian environment, with security APIs, UI, workspace and vault, and found `undefined` everywhere.

The root problem is that the Karpathy LLM Wiki plugin (v1.26.0) calls 39 Obsidian API methods, spread across four main objects (App, Plugin, Vault, Notice). Some are obvious, like reading and writing files. Others are surprising: `secretStorage.getSecret()` for API keys, `addRibbonIcon()` for the UI, `registerView()` for the query window.

Every missing method is a TypeError, and every TypeError is a crash.

### The fix: a tracing stub

Instead of guessing which APIs were needed, I put a JavaScript Proxy in place that intercepts every access to an undefined property, logs it and returns a default value. With the tracing stub I launched the plugin and within five minutes I knew exactly which 39 calls were required.

```
Phase 1 — API discovery (5 minutes):
  App:         secretStorage.getSecret, secretStorage.setSecret, workspace, vault
  Plugin:      loadData, saveData, registerView, addCommand, addRibbonIcon, addStatusBarItem
  Vault:       read, create, modify, delete, getMarkdownFiles, getAbstractFileByPath, process, cachedRead, getRoot
  Notice:      Notice(), hide(), setMessage()
```

Without the Proxy I would have spent hours reading the plugin docs or throwing guesses at it.

---

## Obstacle 1 of 5: Obsidian does not exist on a server

With the APIs mapped out, the real work was implementing them. Some are trivial to fake, to the point that `addRibbonIcon()` can just return an empty object. Others have to actually work against the filesystem.

### Vault, the biggest block

The plugin reads and writes files constantly: entities, concepts, sources, index, logs. A stub answering `{}` is not enough, you need an implementation that can read `.md` files from disk, create new folders and files in the right place, update the index whenever something changes, and support atomic read plus write (`Vault.process()`).

In the end I wrote 18 Vault methods working directly on the filesystem through Node.js' `fs` module. No database and no cache, just real operations on real files.

### Secret Storage, the API keys

Plugin v1.26.0 introduced Obsidian's Secret Storage (API v1.25+) to keep Gemini keys safe. Headless, it does not exist, so I put a stub in place that reads and writes a `.secrets.json` file inside the plugin folder. The keys stay on the server.

### Window, the browser globals

The bundle expects `window`, `document`, `setTimeout` and `MutationObserver`. In Node.js they are not there, so I built a fake browser environment:

```javascript
globalThis.window = {
  setTimeout: setTimeout.bind(globalThis),
  clearTimeout: clearTimeout.bind(globalThis),
  setInterval: setInterval.bind(globalThis),
  document: { createElement: () => ({}), addEventListener: () => {} },
  MutationObserver: class { observe() {} disconnect() {} },
  location: { href: '' },
  navigator: { userAgent: 'Node.js' },
  matchMedia: () => ({ matches: false })
};
```

Without `setTimeout` the plugin timed out on `startupCheck()`. An `await new Promise(r => setTimeout(r, 3000))` that never resolved.

---

## Failure 2 of 5: the ReferenceError that drove me crazy

Once the stub was fixed I tried the ingest again:

```
TypeError: (0 , import_obsidian.requestUrl) is not a function
```

The plugin uses Obsidian's `requestUrl()` for every HTTP call to the Gemini API, and headless `obsidian.requestUrl` does not exist. I implemented it with Node.js' native `fetch`.

First call: works. Second call:

```
TypeError: Body is unusable: Body has already been read
```

The plugin read `response.text()`, then `response.json()`, then `response.arrayBuffer()`: three reads of the same HTTP stream. In Node.js a `fetch` body can only be consumed once.

I fixed it by reading `response.text()` once and deriving `json` and `arrayBuffer` from the string. The plugin does not need the original stream, it needs the data.

It is the kind of bug that never shows up in development, because Obsidian has its own HTTP layer, and that explodes headless right away.

---

## Design decision: why a stub and not a rewrite

I thought about rewriting the whole thing as a Python script more than once. I dropped the idea for three reasons:

| Approach | Pros | Cons |
|-----------|----------|-----------|
| **Rewrite** | Total control, no dependency | Weeks of work, divergent behaviour, breaks at every update |
| **Stub + Plugin** | Same logic, same behaviour, automatic updates | Stub complexity, 3.6MB loaded on every call |

The plugin has features that would have been insane to replicate: duplicate merge with LLM semantic analysis, alias matching for broken wikilinks, schema enforcement with configurable templates, automatic retag against a validated vocabulary, smart fix repairing dozens of issues in a single run.

Rewriting all of it would have been a second project, not a wrapper.

The numbers confirmed the stub choice: 68 tests passed out of 68, 16 bugs fixed across 6 iterations, two days of total development.

To give an idea of the complexity, the `Vault` object alone has 18 implemented methods (`read`, `create`, `modify`, `delete`, `getMarkdownFiles`, `getAbstractFileByPath`, `process`, `cachedRead`, `getRoot`, `trashFile`, `append`, `exists`, `getFileByPath`, `getFolderByPath`, `createFolder`, `readBinary`, `writeBinary`, `getAllLoadedFiles`), each one with error handling, logging and permissions. The whole stub, adding up Vault, App, Plugin, Notice and Workspace, is over 600 lines. That is not a small amount of code, but it is nothing compared to the 82,000 lines it allows to run.

---

## Failure 3 of 5: the smart fix that never finished

The plugin's lint behaves in a sneaky way. `lintWiki()` is an `async` function that resolves right after the initial scan, while the smart fix keeps running in the background between Promises and Notices. Hermes was therefore reading the partial report, the pre-fix one, and answering the user with incomplete data.

### Problem

```
1. Lint starts → initial scan (3 seconds)
2. lintWiki() resolves → Hermes reads the partial report
3. [IN BACKGROUND] Smart fix continues for 300 seconds
4. Hermes has already answered the user with wrong data
```

### The fix: fs.watch on log.md

The plugin writes the report to `log.md` twice. The first time after a few seconds, with the scan result, the second time between 10 and 300 seconds with the final post-fix report. I put an `fs.watch` in place that counts writes: the first is ignored, the second unblocks the worker, which reads the final report and returns it.

```javascript
const watcher = fs.watch(logPath, (eventType) => {
  if (eventType === 'change') {
    writeCount++;
    if (writeCount >= 2) {
      watcher.close();
      resolve(); // unblocks the worker
    }
  }
});
```

There is no timeout and no polling, and it also works on a 100-page lint or more that takes five minutes.

### Second problem: trashFile really deleted things

During the smart fix the plugin deletes duplicate pages by calling `trashFile()`. In my first stub that method ended up in `fs.unlinkSync()`, meaning permanent deletion with no trash bin.

In Obsidian `trashFile()` moves files to the system trash, so I implemented my own trash: `.trash/<path_with_timestamp>.trash`. Everything is recoverable, and the wiki is saved automatically before every lint.

### Third problem: backups piled up

Every lint left a backup in `.hermes/tmp/wiki-backup-<ts>/`. After ten lints, ten backups, and nobody cleaned them up. I added an `fs.rmSync(backupDir, { recursive: true, force: true })` once the lint completes, so nothing is left behind.

---

## In production: from my VPS to the customer's premises

Everything I have told so far happened on my VPS, in my validation lab. I was not interested in making a plugin work headless: I was interested in being certain that the same bundle would hold up where nobody sits behind the console. And that is exactly the place where I delivered it: a small local company, anonymous at their request, using it as the knowledge management layer of their on-premise AI infrastructure.

The architecture I designed and installed on site:

| Component | What it does | Where it runs |
|-----------|---------|----------|
| **AI agent** | The customer's counterpart over chat; maintains the wiki through the headless wrapper described above | Mini PC on site (hardware I own, given on free loan) |
| **User persistent memory** | Recall of interactions across sessions, managed by Honcho | Same mini PC |
| **Wiki — general knowledge base** | The customer's knowledge: the agent maintains it (lint, merge, aliases), the customer reads and enriches it | Same mini PC |
| **Obsidian mobile via MCP server** | The app on the customer's phone, connected to the MCP server serving the wiki | Customer's smartphone |

The core of the project is two consumers on a single source of truth. The agent writes to the wiki through the headless plugin, the same `main.js` byte for byte, while the customer reads and adds from the phone. Neither of them knows the other exists: they sync on the filesystem and there are no divergent versions.

Three things I was able to guarantee to the customer during the sales phase, all of them possible thanks to the stub:

- Wiki content never leaves the building. Only technical status travels to my homelab for silent monitoring, healthchecks and minimal logs.
- Remote access is revoked with one click. The connection is outbound-only over a private mesh network and no port is left open on the customer's router.
- The logic maintaining the wiki is the same between my devices and the customer's mini PC. What I validated here also runs there.

---

## Testing as a progression: from 5/10 to 10/10

Validation was done through 6 progressive tests on real files, with a wiki of 54 entities, 58 concepts and 13 sources. Here is the progression:

| Test | Version | Duration | Gap | Score |
|------|----------|:------:|:---:|:----:|
| T1 — Stub and loading | v0.1→v0.7 | 35 min | 7 | 10/10 |
| T2 — Single ingest | v1.0.0 | ~60s | 5 | 10/10 |
| T3 — Folder/multi ingest | v1.0.0 | ~300s | 1 | 10/10 |
| T4 — Lint and reindex | v1.0.0 | 4-120s | 1 | 10/10 |
| T5 — Fast CLI | v1.0.0 | <100ms | 0 | 10/10 |
| T6 — Real production (126KB) | v1.2.0 | 123+308s | 3 | 10/10 |

The score never dropped below 10/10 because I fixed every gap before moving to the next test. Bugs surfaced, I fixed them, and the following test started on the already corrected version.

### What the wrapper did in production (T6)

Ingest of a real 126KB file (788 lines):

| Operation | Result |
|-----------|-----------|
| Rate limiter | Active: 16s between requests, 15 req/min |
| Pages created | 1 source + 7 entities + concepts |
| Schema placeholders | 12 cosmetic warnings (fixed) |
| Total time | 123 seconds |

Production lint, on more than 100 pages:

| Category | Count | Detail |
|:----------|:--------:|:----------|
| 🔧 Sources normalised | 9 files | gemini, hetzner, karpathy, node-js, syncthing, layer-meccanico, rate-limiter, stub, wrapper |
| 🔗 Wikilinks fixed | 4 | claude-code, composio-mcp, gemini, hermes-agent |
| 📄 Stubs created | 2 | entities/gmail.md, concepts/automazione.md |
| 🏷️ Aliases added | 6 entities | gmail +4, google-maps +5, google-sheets +6, automazione +6, mcp +1, skill-system +5 |
| 🔀 Duplicate merge | 1 | sources/avvia-un-business merged |

Total time: 308 seconds. Exit code: 0.

For context, the same lint inside Obsidian desktop takes about 180 seconds. The 128-second difference comes from the wrapper loading the bundle from scratch on every call, 3.6MB from disk, while Obsidian keeps it in memory. A persistent daemon would shrink that gap to a few seconds.

---

## Residual risks

1. Plugin updates: if the Karpathy plugin updates and calls new APIs, the stub has to be revised with a new tracing phase. Without it, the wrapper breaks silently.

2. Node.js cache: the 3.6MB bundle is reloaded on every call. The V8 cache helps but does not remove the cost. With frequent operations, like several queries in a row, a persistent daemon might be needed.

3. Non-standard Obsidian APIs: some plugin features use undocumented Obsidian APIs. If Obsidian deprecates them, the plugin breaks and the stub breaks with it.

4. No UI: everything works, but there is no visual feedback. No progress bar and no on-screen notifications, the output is text only.

5. Remote support on the customer side: interventions go through silent monitoring on my homelab, healthchecks and technical status logs. As long as customers stay below a manageable threshold, a single human point of contact is enough. Beyond that threshold more support layers are needed, an architecture I have planned but not built yet.

---

## Lessons and next steps

One thing comes out of this story: there is no need to rewrite what already works. Sometimes the most elegant solution is building a fake environment that convinces the software it is at home.

The patterns I am keeping:

- Before implementing a stub, let the system tell you what it needs. A Proxy logging every access to a missing property is faster than any static analysis.
- To sync with an asynchronous process, watching the filesystem is cleaner than polling every N seconds.
- Every destructive operation, whether a lint, a merge or a fix, should back up first. It costs little and saves the day.
- The wrapper uses the same `main.js` as the plugin on laptop and iPhone, so lint and ingest produce identical results on any device.

The biggest lesson is that an Obsidian plugin is just JavaScript code. If you can fool it about its execution environment, you can run it anywhere, whether that is Docker, a server, a CI/CD pipeline or an edge function. The border between desktop app and remote service is much thinner than it looks.

### Next steps

- A persistent daemon keeping the plugin warm in memory, to bring loading time down from 3 seconds to around 100 milliseconds
- LLM queries through the wrapper, instead of browsing the wiki by hand
- Automatic tracing whenever the plugin updates

The wrapper repository is on GitHub. And for anyone doing my job: this is one of the layers I design and deliver to small businesses as on-premise AI infrastructure. The technical part stays the same, only the environment changes, where it has to hold up on its own with nobody behind the console.

---

## Resources

- [GitHub repository](https://github.com/amirtabit31-byte/plugin-karpathy-wrapper-headless)
- Full design manual: `progettazioni/completate/plugin-wiki + hermes/`
- `llm-wiki-karpathy` skill, the Hermes interface to the wrapper
