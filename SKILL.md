---
name: webkit-webview-ai-skill
description: "Hard-won WebKit/WKWebView behaviors for Mac apps whose UI is a web page inside a WKWebView (Tauri apps, native URL-wrapper apps, any Solid/React/TS UI shipped in a Mac bundle). Trigger whenever (1) something works in Chrome or the browser mock but not in the installed app — a click that 'does nothing' (including one that works only on the SECOND try, right after switching back to the app — Tauri swallows the first mouse by default), a shortcut that goes dead after using a text field, a popover that won't close, a pane that freezes while the rest of the app works, a right-click → Copy Link (or link drag) that pastes a custom-scheme/sink URL instead of the real one; (2) you need to VERIFY WebKit-only behavior without showing a window or triggering a TCC prompt (the offscreen WKWebView probe pattern); (3) you're designing a web UI that will run in WKWebView and want to avoid the traps up front (focus, native date inputs, iframes, drag-drop, resource errors); (4) the caret jumps to another line WHILE TYPING, or you are adding the macOS spelling panel / Check Document Now / grammar checking / right-click spelling suggestions to a WKWebView editor; or (5) a setting flipped in Settings \"does nothing\" in a window or surface that was ALREADY open, kept mounted, or pre-warmed (per-window stores, and SolidJS reads that are not tracked); or (6) a popup's own ↑/↓ / Enter handling \"does nothing\" while the list behind it moves instead. Also (7) an `evaluateJavaScript` / eval-hook probe returns nothing, a stale value or 'unsupported type', (8) a one-shot celebration or reveal replays after a redraw or when you come back to the screen, (9) an inline rename / contenteditable field commits immediately or 'presses Enter for me' due to synthetic WebKit blurs on menu dismissals or focus shifts, (10) a flex sidebar auto-fitting its width from child `.label` bounds grows by +7px on every autosave redraw while typing, (11) unhiding a modal list (Command Palette / Quick Switcher) under a stationary cursor fires synthetic `mouseenter` and steals selection from row 0, or (12) a key chord sent from native code into a page lands in the wrong tab or fires the app menu, or one web view must render dark while another renders light. A Chromium browser mock lies about every item here."
---

# WebKit / WKWebView gotchas for web UIs inside Mac apps

Many Mac apps ship a web UI inside a WKWebView (Tauri, native URL wrappers).
It is tempting to verify in a Chromium browser pane or a browser mock; the installed app runs
WebKit. Every item below is a place where those two disagree, learned the hard way.
Each carries its trigger, the near-miss to rule out, and the fix.

## 1. Buttons never take focus on click — a focused input keeps the keyboard

**Trigger:** a single-key shortcut (`#`, `e`, `j`) "does nothing" in the installed app
after the user touched a search/text field, then clicked a row, a pill, a toolbar
button. Chromium moves focus to the clicked button; WebKit does NOT — the input stays
`document.activeElement`, so the key types into it (or the keymap's "typing" guard
rightly swallows it). **Discrimination:** if the character APPEARS in the field, it's
this; if nothing appears anywhere and the menu bar is also dead, it's a hang (see section 5).
When you can't tell, assume this — it's free to fix and always correct.
**Action:** every list-side control the user clicks after using a text field must
release that field's focus explicitly (`if (activeElement is the search input) blur()`);
a "clear" ✕ on the field blurs after clearing, exactly like Esc. Use
`onMouseDown={e => e.preventDefault()}` on such buttons only when you WANT focus to
stay (a completion list that would close on blur) — and then blur on click yourself.

## 2. `<input type="datetime-local">`'s calendar popover stays open while the field has focus

**Trigger:** the user picks a day in the native calendar and the popover keeps
covering the UI while they edit the hour. WebKit exposes no API to dismiss it; only
losing focus closes it. **Action:** two inputs, `type=date` + `type=time`, never one
combined field. On the date field's `change`, if no keystroke landed in it in the last
~400 ms it was a calendar click → `timeInput.focus()` (closes the popover, hour ready
to type); typed segments keep focus put. The Chromium mock closes the popover itself,
so it cannot show the bug — verify in the installed app.

## 3. Hit-testing over an icon returns the `<path>`, and its `className` is an object

**Trigger:** `document.elementFromPoint(x, y)` at a button's center "returns nothing" /
prints `(none)`. The hit is the SVG `<path>` inside the button and `className` on SVG
elements is an `SVGAnimatedString`, so `h.className || ''` is an object or empty.
**Action:** walk `parentElement` (or `closest('button')`) and print `tagName#id.class`
per level; test `typeof className === 'string'` before using it. Don't conclude the
click can't reach the button from that one read.

## 4. Verifying WebKit behavior headlessly — the offscreen WKWebView probe

**Trigger:** you need proof under real WebKit and the browser mock can't give it.
**Action:** a Swift script (`swift scripts/<thing>-probe.swift`) that copies the BUILT
frontend (`dist/`, root-relative asset paths rewritten to relative), loads it in a
`WKWebView` inside an `NSWindow` that is never ordered front, with
`NSApplication.shared.setActivationPolicy(.prohibited)` — no Dock icon, no focus, no
TCC. Drive it with `evaluateJavaScript`; poll for readiness (`window.__edx…` or a DOM
marker). Facts about that environment:
- **Native KEY events work** through `win.sendEvent(NSEvent.keyEvent(…))` — the
  frame-focus probe proves keymap routing that way.
- **Cheapest real-key path in a LIVE app:** give the app a
  `<scheme>://key?code=<keycode>[&chars=<c>][&out=<path>]` URL command that builds
  `NSEvent.keyEvent(with: .keyDown/.keyUp, … windowNumber: win.windowNumber, keyCode:)`
  and calls `win.sendEvent` on the frontmost window — WebKit's own routing included, no
  probe script, no activation (53 = Escape, chars default `\u{1B}`). It proved the find
  panel's Escape path in one call. Caveat from section 20: a synthetic key never raises the
  autocorrect bubble, so it cannot reproduce that specific swallow — read the screenshot.
- **Native MOUSE clicks DO land — but only when handed to the web view itself.**
  `NSWindow.sendEvent(NSEvent.mouseEvent(…))` routes nothing in a never-shown window, so the first probe silently fell back to dispatching the DOM
  sequence (`pointerdown → … → click` on `elementFromPoint`) and "verified" a ✕ that
  stayed dead for the user — a DOM dispatch cannot fail the way a real click can. Call the view's NSResponder methods directly instead:
  `web.mouseDown(with: down); web.mouseUp(with: up)` with `NSEvent.mouseEvent`s whose
  location is in the window's bottom-left frame (`y = H - pageY`) — WebKit then runs
  its own hit-testing and the full mousedown → mouseup → click sequence in the web
  process. **Every clicking probe carries a CONTROL click** on something whose state is
  visible (a list row's selected index) and prints whether it landed; a pass whose
  control says "native clicks DO NOT land" is not a pass — never let the probe fall
  back to DOM dispatch quietly.
- **A JavaScript evaluation that never returns is the hang detector**: schedule a
  timer after the action; if the callback hasn't fired in ~5 s, print `HANG` and exit 3.
- Four useful probe shapes: `scripts/frame-focus-probe.swift` (iframe focus + native keys +
  ⌘C to pasteboard), `scripts/search-clear-probe.swift` (search ✕, hit-test report,
  control click + native click via the view, hang check),
  `scripts/newest-pin-probe.swift` (SCROLL geometry sampled every 100 ms for 3 s while
  iframes grow — the shape for any scroll-position feature, see section 8), and
  `scripts/copy-link-probe.swift` (a NATIVE CONTEXT MENU: `rightMouseDown:` handed to
  the view, the menu observed through `NSMenuDidBeginTracking`, an item's action sent
  programmatically, the menu cancelled from a delayed block, the pasteboard polled —
  and the user's clipboard saved/restored around the run, see section 18). Copy their
  skeleton; keep the "no window shown" invariant.
- **The hidden Chromium browser pane runs NO rendering steps, so it cannot show scroll
  behavior at all:** a programmatic `scrollTop` write fires no `scroll`
  event, ResizeObserver never fires, and a transient layout (an iframe collapsed to
  0 px for a measure) never clamps the scroller — a scroll-position feature passed
  every pane check and failed in the installed app. Trigger: the change touches
  scrollTop, scroll events, sticky positioning, or iframe heights → prove it with the
  probe, never the pane.
**Discrimination:** a probe running on the browser MOCK proves the frontend's own
behavior under WebKit, not the Tauri IPC or real data. A symptom that survives the
probe is in the app shell or the backend.

## 5. A Solid resource in error state THROWS on read — one failed fetch can freeze a pane

**Trigger:** part of the UI stops updating for the rest of the session while the rest
keeps working (a reading pane stuck on the previous item while list clicks change the
highlight). **Discrimination:** not a selection/cursor bug if the LIST still reacts —
look for a thrown read in that pane's render. **Action:** resource fetchers catch and
report (an error toast via the app's `setLastError`) and return a value, never
rethrow; wrap the pane's content in `<ErrorBoundary fallback={(err, reset) => …}>` that
shows the error TEXT and a Try Again; make formatters (Intl-based dates, etc.) fall
back instead of throwing. Silence is the failure — the boundary turns the next
occurrence into a one-line report from the user.

## 6. Older WebKit-vs-Chromium traps (listed so you look)

- HTML5 drag-and-drop is dead under Tauri's native drag layer; use pointer events.
- Parent-attached listeners never fire inside a scriptless sandboxed iframe; links
  must work via DEFAULT actions (a custom protocol), and the frame eats the keyboard —
  a focus guard reclaims it.
- Tauri's injected drag-region script `stopImmediatePropagation()`s document
  `mousedown` on `data-tauri-drag-region` elements — popover click-away must listen for
  capture-phase `pointerdown` on `window`, not document `mousedown`.
- `pointer-events: none` layers, `z-index` stacking, and `:hover` all behave the same
  as Chromium — when a click "does nothing", check section 1 (focus) and section 5 (a dead render)
  before stacking.

## 7. The click that REACTIVATES the app is swallowed by AppKit — Tauri's `acceptFirstMouse` is off

**Trigger:** "clicking X does nothing" where the screenshot shows the app FRONTMOST and
X is a control you click ONCE right after coming back from another app (a search ✕, a
toolbar verb, a menu chip); often reported repeatedly because a previous "fix" targeted
the page. Check in one grep: `acceptFirstMouse` in `tauri.conf.json` windows and
`.accept_first_mouse(` on every `WebviewWindowBuilder` — Tauri's default is `false`
(`tauri-utils` `WindowConfig`, verified on 2.11), so the first click on an
inactive window only activates it and the webview never sees it.
**Discrimination:** controls clicked repeatedly (list rows) never show it, because the
first press is absorbed and the second lands; a probe with real WebKit clicks (section 4)
passes, because the probe's window is the key window. The tell: the app being
frontmost in the report IS the swallowed click's effect, and a second click works. When
the probe passes and the user still fails, suspect this before the page. **Action:**
`"acceptFirstMouse": true` on every window in tauri.conf.json and
`.accept_first_mouse(true)` on every builder; state the tradeoff (a reactivating click
can fire a destructive control — make sure those verbs are undoable). Swift wrappers:
override `acceptsFirstMouse(for:)` → `true` on the WKWebView subclass if the same
symptom appears.

## 8. An iframe collapsed to 0 px for a measure CLAMPS the parent scroller — and the clamp's scroll event looks like the user

**Trigger:** a "keep this element in view while content grows" feature (a pin to the
newest message, a chat-style stick-to-bottom) that holds in the browser mock and lets go
in the installed app, leaving the view one frame's height too high. **Cause:** the
frame's own measure sets `iframe.style.height = "0px"` and reads the child document's
`scrollHeight`; WebKit's `Document::updateLayout` lays out the OWNER document first, so
the parent scroller's content shrinks for that layout, its offset is clamped, and a
`scroll` event nobody wrote is queued — a "not our write → the user scrolled" rule
then releases. **Discrimination:** the mock never shows it (section 4's last bullet); in the
app the drift equals the collapsed frame's height and lands right after a re-measure.
**Action:** hold the frame's wrapper at `min-height: <current>` across the collapsed
read so the parent only ever sees the final height (this also stops a reader below
the frame jumping on late re-measures); read user intent from INPUT (wheel, a press in
the pane, scroll keys on `window`, touch, the app's own navigation verbs), never from
geometry; if a scroll-event fallback stays, treat "scrollHeight changed AND scrollTop
is on the bottom clamp" as content, everything else as the user. Verify with the
sampling probe (section 4), keyed on account+item id so a refetch never re-pins.

## 9. Plain-`http://` images are MIXED CONTENT in the app's secure origin — the broken glyph is WebKit, not the setting

**Trigger:** "Load remote images is on but I still see broken images" where the broken
ones are `http://` (`t1.gstatic.com` logos in a Google alert) while `https://` siblings
in the same mail render; `curl -I` answers 200 for both. **Cause:** the webview's
custom scheme (`tauri://`, any registered-as-secure scheme) is a secure context, so
WebKit blocks `http:` subresources as mixed content regardless of `img-src` allowing
`http:` — the scheme is permitted, the load is refused. **Action:** upgrade
`http://` → `https://` on the src you activate (the hosts that matter serve https) and
add `upgrade-insecure-requests` to the CSP as the backstop (`srcdoc` frames inherit
it). Don't "fix" it by widening `img-src`. Verify the shipped CSP with `strings` on the
installed binary — Tauri injects it into the embedded HTML, not `dist/`.

## 10. TipTap 3 emits `update` for PROGRAMMATIC writes, and StarterKit's TrailingNode edits the document on focus

**Trigger:** a composer marks itself edited (prompts "save as draft", autosaves a
draft) for a message the user never typed into; or "dirty" flips true right after
open. **Cause:** in TipTap 3 `setContent(content, { emitUpdate = true })` — the
opposite of v2's default — so every store-side content write arrives through
`onUpdate` like typing; and StarterKit 3 ships TrailingNode, which appends an empty
paragraph to a loaded body the moment it gets focus, through the same event.
**Discrimination:** the debug log shows an update whose HTML differs from the stored
body only past its first N chars, with no keystroke; `npm ls @tiptap/core` says 3.x
(the worktree's `node_modules` may be empty — look in the main checkout's).
**Action:** every programmatic `setContent` passes `{ emitUpdate: false }` and mirrors
the body by hand; `onUpdate({ editor, transaction })` forwards
`transaction.docChanged` — the DISPATCHED transaction's own flag, which is false for
appended (extension) changes — and only that marks a user edit; gate any "did they
edit?" prompt on that flag, not on a `dirty` that the autosave clears. Probing the
store from the page via a dynamic `import('/src/…')` HANGS the pane — instrument with
a temporary `console.debug` instead.

## 11. Tauri's opener plugin shells out to `/usr/bin/open` — a browser router sees the CLI as the sender, never your app

**Trigger:** "the app isn't honoring the browser I picked in OpenIn / Velja / Choosy",
"my per-app browser rule doesn't fire for this app", or links from a Tauri app landing
in the router's DEFAULT browser while the same rule works for Spark/Slack. Check in one
grep: `plugin-opener` / `tauri_plugin_opener::open_url` / `app.opener().open_url` in
the tree — and confirm the router IS the https handler (`plutil -p
~/Library/Preferences/com.apple.LaunchServices/com.apple.launchservices.secure.plist |
grep -B3 'LSHandlerURLScheme" => "https"'`). The mechanism is in the crate source, not
a guess: `tauri-plugin-opener` → the Rust `open` crate → `Command::new("/usr/bin/open")`
(`open-5.x/src/macos.rs`), so the Apple Event the router receives names the short-lived
`open` process as its sender and a rule keyed on the app's bundle path can never match.
**Discrimination:** Swift wrappers calling `NSWorkspace.shared.open` and Electron's
`shell.openExternal` (NSWorkspace underneath) are attributed correctly — only the
`/usr/bin/open` path loses the sender. And the router's HISTORY table is only evidence
if it was recording: OpenIn's "Store history" is often Never, and rows may be a year
old from another Mac (`ZHOST`, `min/max(ZTIMESTAMP)` — check before citing a count).
**Action:** open web links from the app's OWN process via `objc2-app-kit`
`NSWorkspace::sharedWorkspace().openURL(&NSURL)` (sender = the app; a router rule
matches by bundle PATH, so only the `/Applications` install matches, never a worktree
bundle), and give the user a Settings "Open Links With" select built from
`URLsForApplicationsToOpenURL(https://example.com/)` (every https handler; `NSBundle`
id + `NSFileManager.displayNameAtPath` name) that opens via
`openURLs_withApplicationAtURL_configuration_completionHandler` when set — `mailto:`
always to the system handler. Keep the opener plugin only for `open_path` (files) and
`reveal_item_in_dir`. Mirror the choice into app state at boot + on change so a
protocol handler (which must never take the DB mutex) can read it. Related: Tauri apps
are not AppleScriptable (no `NSAppleScriptEnabled`/sdef); the cheap scripting surface
is a URL scheme via `tauri-plugin-deep-link`, one-way.

## 12. `position: fixed` under a TRANSFORMED ancestor is positioned against the ancestor — an overlay born inside an animated card is never seen

**Trigger:** a fixed-position popover/bubble/toolbar "doesn't appear" although its state
says it's rendered (`document.querySelector` finds it, its inline `left/top` look right,
the control that raises it is lit), and it lives inside a container that animates in —
Settings cards, panels, dialogs with `animation: … both`. Check in one call:
`getComputedStyle(el.closest('[class*=card],[class*=panel]')).transform !== "none"`
(for example, a Settings card that keeps `transform: translateY(0)` after
its entry animation because of `animation-fill-mode: both` — an identity transform still
creates the containing block). The link bubble computed viewport coordinates, was
placed relative to the card, and sat clipped outside its scroll area.
**Discrimination:** a popover that appears but at the WRONG spot is the same class; one
that appears in the right spot but under something else is z-index, not this. And a
probe's DOM numbers are NOT proof of visibility — `document.elementFromPoint(x+10,
y+10)` at the overlay's rect must return the overlay, and a screenshot must show it.
**Action:** render every fixed overlay through a `Portal` at the document root
(Solid `solid-js/web` `Portal`; React `createPortal`) so window coordinates mean the
window; keep the overlay's z-index above the host panel's (e.g. Settings at 600, the bubble at 700). Do not "fix" it by removing the card animation's fill
mode — other cards depend on it.

## 13. A link inside a `contenteditable` NAVIGATES the WebView — and Tauri hands that navigation to the SYSTEM browser

**Trigger:** "links in the editor open in the wrong browser" / "…ignore the browser I
chose in Settings" for links clicked inside a TipTap/ProseMirror/contenteditable
surface, while links elsewhere in the app honor the setting. Mechanism: WebKit follows
an `<a href>` click inside editable content (TipTap's `link.openOnClick: false` only
stops TipTap's own handler), the WebView attempts a top-level navigation to an external
URL, and Tauri's navigation handler opens it via `/usr/bin/open` → the system default
browser — the same sender-less path as #11, bypassing any in-app "Open Links With"
choice and any router rule keyed on the app.
**Discrimination:** a link that opens NOTHING is CSP or the #11 opener plugin; a link
that opens the wrong browser from an editor is this.
**Action:** the editor host swallows link clicks — `onClick`: `closest("a[href]")` →
`preventDefault()`; ⌘-click opens through the app's own path (`openExternal` →
`links::open_url`, #11) — and gives the user a deliberate way to open/copy/edit/remove
the link (a bubble under the link the cursor sits in).

## 14. An entry animation with `fill-mode: both` HOLDS its end frame — and beats every later `opacity` rule

**Trigger:** a "dim the non-matches" / "fade the disabled ones" rule (`.filtering .tile:not(.hit)
{ opacity: .16 }`) that never takes effect on elements that also carry a load-in animation
(`animation: enter .45s ease-out both` with `from { opacity: 0 } to { opacity: 1 }`);
`getComputedStyle(el).opacity` reads `1` (or `0` mid-animation) while the class is present.
Mechanism: animated values sit above author styles in the cascade, and `both` / `forwards`
keeps applying the last keyframe forever — the `to { opacity: 1 }` wins over any later rule
that isn't `!important`. Same in Chromium; it is the spec, not a WebKit bug — listed here
because it is easy to inherit from a mockup that never actually dimmed anything.
**Discrimination:** a dim that works after a reload but not right away is the animation still
running (read again after its duration); one that never works is the held fill. A probe that
reads styles right after a class flip also sees TRANSITION start values — see #15.
**Action:** `animation-fill-mode: backwards` (the `from` frame applies before the delay, nothing
is held after) and set the resting look with ordinary rules; animate only properties the
resting rules never touch (a breathing `box-shadow` on lit tiles is fine). Verify:
`getComputedStyle(nonHit).opacity === "0.16"` a second after the class lands.

## 15. `getComputedStyle` right after a class change returns the TRANSITION's start value

**Trigger:** a headless probe (#4, or an `eval` command into the app's page) adds a class /
injects a `.__hover` style to simulate hover, reads `backgroundColor` / `transform` /
`borderColor` in the same tick, and gets the RESTING values — the new rule looks ignored.
Mechanism: the element has `transition: background .22s, transform .22s …`; computed style
during a transition is the interpolated value, and at t=0 that is the old value.
**Discrimination:** if a property WITHOUT a transition (say `outline`) changes while the
transitioned ones don't, it is this; if none change, the selector or the custom property
(`rgba(var(--hue-rgb), .2)` with `--hue-rgb` unset → the whole declaration invalid) is wrong.
**Action:** the injected probe style carries `transition: none !important` for the element and
its children, or the probe waits past the longest transition before reading. For a screenshot
of the hover state, add the class, wait `max(transition, 250 ms)`, then snapshot; remove the
class and the style afterwards so the live page is untouched.

## 16. `Cross-Origin-Resource-Policy: same-origin` kills a cross-origin `<img>` — fetch remote images from the APP PROCESS, never the webview

**Trigger:** "remote images are enabled but not showing" where the image is `https://`, `curl -I <url>` answers 200 `image/png`, and the CSP `img-src` already allows `https:` — so section 9 (mixed content) is ruled out. Run
`curl -sI <url> | grep -i cross-origin-resource-policy`: `same-origin` (some sites send it on every asset) means WebKit refuses the no-cors `<img>` load from any other origin, silently, `complete=true` with `naturalWidth=0`. A Referer-keyed hotlink rule (403 only with a foreign Referer) is the sibling case with the same fix.
**Discrimination:** section 9 is `http://` only and shows as a broken glyph the same way — check the scheme first. A CORS error in the console is NOT this (CORP blocks silently, no console line in WebKit). A `curl` 200 proves the SERVER serves it, never that the webview may load it — prove the block with the offscreen probe (section 4): `loadHTMLString` a page with the suspect `<img>` and a known-good control (`https://www.google.com/s2/favicons?domain=apple.com&sz=64`) side by side, read both `naturalWidth`s after 6 s; suspect 0 + control 64 = CORP/hotlink, both 0 = network/machine.
**Action:** don't touch the sanitizer or widen the CSP — register a custom scheme (`app-img://localhost/?u=<encodeURIComponent(url)>`, Tauri `register_asynchronous_uri_scheme_protocol`) whose handler fetches the URL from the app process on a worker thread (reqwest: http/https only, ~20 s timeout, size cap, a mainstream browser UA — some CDNs challenge library UAs — no cookies, no Referer), answers 200 with the upstream `content-type` + `cache-control: max-age=3600` + `access-control-allow-origin: *`, and an UNCACHED 404 on any failure so a reopen retries; add the scheme to `img-src`; point every activated `<img>` at it in the app only (the browser mock has no handler — keep the direct URL there). Gmail web, Spark and Mail.app all proxy for exactly this reason, so any email client's "load remote images" must. Keep an `#[ignore]` network unit test that fetches the exact asset that broke, and run it with `-- --ignored` before shipping.

## 17. No EVENT crosses a scriptless sandboxed iframe — but `:hover` STATE does. Read it, don't listen

**Trigger:** a feature needs to know what the pointer is on INSIDE a `sandbox="allow-same-origin"` frame with no `allow-scripts` (a hovered link's URL, a hovered image, "is the pointer over the quote?"), and the obvious move is `frameDoc.addEventListener("mouseover", …)` from the parent. That listener fires in Chromium (the browser mock goes green) and NEVER in WKWebView (section 6) — the parent also gets no `mousemove` while the pointer is over the frame, so there is no event path at all.
**Discrimination:** rendering state is not an event. The engine maintains `:hover` on the frame's elements with or without scripts, and the hover chain climbs into the OWNER `<iframe>` element in the parent document — so `frame.matches(":hover")` and `frame.contentDocument.querySelectorAll("a[href]:hover")` are plain same-origin DOM reads that work in both engines. A `title` attribute is the other event-free option (a native tooltip via default UI) but arrives after ~1 s in system styling, and a `title` on the IFRAME itself surfaces as a stray tooltip over the whole body (such as "Email content") — use `aria-label` for the frame's accessible name.
**Action:** a module-level poll (≈80 ms, started when a frame mounts, stopped when none is left) checks `frame.matches(":hover")` per frame and queries the hovered anchor only inside a hovered one; publish the target AND its box (`anchor.getBoundingClientRect()` + the frame's rect — a body scaled with a CSS transform already reports transformed rects; the frame never scrolls internally) as a signal, and draw the preview in the PARENT (portaled, `position: fixed`, `pointer-events: none` so hovering the preview still hovers the link). Place it ABOVE the target, left-aligned: the arrow cursor covers down-right of its hotspot, so a pill under the link sat under the pointer. Verify with a real pointer hover in a real browser (only a real pointer drives `:hover`) and in the installed app — the mock's event path proves nothing here.

## 18. The native context menu's "Copy Link" copies the REWRITTEN href — fix it at the MENU, in the app process

**Trigger:** a link inside a webview carries a rewritten href — a custom-scheme sink (`app-open://open/?u=<encoded>` so a click can leave a scriptless sandboxed frame, section 6), a proxy URL, a tracking wrapper — and the user reports that right-click → Copy Link (or a drag of the link) pasted THAT string instead of the real URL. Concrete check: `grep -rn 'setAttribute("href"' <ui>` — any rewrite that changes what an anchor's href SAYS is what WebKit's menu will copy, verbatim.
**Discrimination:** the page cannot fix this. No event crosses the sandbox (section 6/section 17), the context menu is native, and — measured — the web process writes the pasteboard AFTER the menu has closed: right after the item's action fires, `NSPasteboard.general.changeCount` is still unchanged. So a one-shot read in `didCloseMenu`/`NSMenuDidEndTracking` reads the OLD clipboard and "proves" nothing happened; a `copy` listener never fires; un-rewriting the anchors breaks the click path (the sink IS how the click escapes). Also don't hand-roll a WKWebView subclass or swizzle `willOpenMenu:` — the global `NSMenuDidBeginTrackingNotification` / `NSMenuDidEndTrackingNotification` fire for WebKit's context menu with no view involvement, and WebKit stamps every context-menu item with an `identifier` (`WKMenuItemIdentifierCopyLink`, `…OpenLink`, `…OpenLinkInNewWindow`, `…DownloadLinkedFile`, `…ShareMenu`) — the menu bar's items never match, so the check is exact. Failure asymmetry: gate the rewrite on the STRING PREFIX of your own sink scheme, never on "the clipboard changed" — then a foreign copy in the poll window is untouched by construction.
**Action:** in the app process, observe both notifications (`addObserverForName:object:queue:usingBlock:`, main thread, once at setup). Begin: the menu (`notification.object`) has an item whose `identifier` is `WKMenuItemIdentifierCopyLink` → remember `changeCount`. End: a block-based `NSTimer` polls every 16 ms for ≤ 1.5 s; on the first change, read `public.utf8-plain-text`, and only if it starts with your sink prefix decode the real URL (same decoder + allowlist as the click path: http/https/mailto), `clearContents`, write it as `public.utf8-plain-text` AND `public.url`, invalidate the timer. Rust/objc2: `block2::RcBlock::new(|note: *mut AnyObject| …)` passed as `&*block`, forget it (the center copies it). Prove it with an offscreen probe (section 4): hand a `rightMouseDown:` NSEvent straight to the WKWebView over a sink link in a `sandbox="allow-same-origin"` srcdoc frame, `NSApp.sendAction(item.action, to: item.target, from: item)` inside the begin observer, `menu.cancelTracking()` from a 50 ms `asyncAfter` (the tracking loop runs common modes, so the main queue drains), then poll; the probe must save the general pasteboard's items before and restore them on every exit — it overwrites the user's clipboard otherwise. "Open Link in New Window" in that menu is a WebKit new-window request Tauri drops without a `new_window_req_handler` — a known no-op, note it rather than chase it.

## 19. A HIDDEN app window runs no layout — eval/snapshot show stale DOM, not the bug

**Trigger:** the app's state dump says the window is not visible (`windowVisible:
false` — app hidden with ⌘H, minimized, or a background relaunch the user never surfaced) and
your headless eval returns only `cm-gap` / zero-height rects, `scrollTop` writes don't
"take", or `takeSnapshot` gives a blank page with just the title.
**Discrimination:** the DOM is REAL but frozen at the last layout the window had while
visible — it is neither evidence for nor against the rendering bug you are chasing; and
activating the app to "make it render" steals focus (never). A visible-but-behind
window (`orderBack`) DOES lay out — that's the normal background-relaunch state and it
verifies fine.
**Action:** don't debug rendering there. Build a BROWSER HARNESS of the web layer: esbuild
the real UI modules with the native bridge stubbed (a small harness entry file plus a build script), serve it locally, drive it in a browser, spy on
bridge messages by replacing `window.webkit.messageHandlers.<name>.postMessage`. Reproduce
and fix there, rebuild the app, then confirm with ONE eval once the window is visible
(a `--relaunch` puts it on screen behind other windows).

## 20. Escape "does nothing" in a text field — macOS autocorrect's bubble ate it

**Trigger:** the user reports Escape not closing a find/filter/search field's panel, and
their screenshot shows the little suggestion pill (`The ×`) under the field; or an
in-process key event (`window.sendEvent(NSEvent.keyEvent(… keyCode: 53 …))`) DOES reach
the page's keydown and closes the panel while a real press doesn't.
**Discrimination:** it looks like a keymap/scope bug (CodeMirror's `search-panel`
binding, a global listener swallowing the key) — the code reads fine because it is
fine. WebKit shows macOS's autocorrect bubble under any `<input>` that has not opted
out, and the first Escape DISMISSES THE BUBBLE inside WebKit; the page never sees a
keydown. Synthetic events never raise the bubble, which is why every headless check
passes. A Chromium harness never shows it either.
**Action:** applied at the element factory: every `input`/`textarea`
the app creates gets `autocorrect="off" autocapitalize="off" autocomplete="off"
spellcheck="false"` (in the one helper that creates elements), and a helper stamps the same on
fields a library builds (CodeMirror's find panel → `disableAutocorrect(panel)` the
first time the panel appears). Keep a shell-side fallback for Escape anyway
(`WKWebView.cancelOperation(_:)` → push an `escape` event) — it costs one override.

## 21. `requestAnimationFrame` NEVER fires in a window that is not on screen — draw synchronously

**Trigger:** a Settings / editor surface renders its content inside `requestAnimationFrame(...)` (or waits a frame "so
layout exists") and a headless check of a background-opened window finds the container EMPTY — no rows, no samples,
`offsetWidth` 0 — while the same code works when the user opens the window. `grep -n "requestAnimationFrame" <ui js>`
on any first-paint path is the check.
**Discrimination:** this is not entry 19 (a hidden window's stale layout): here the DOM is simply never BUILT, because the
callback never runs. A node appended to a connected parent already has layout — reading `clientWidth` right after
`append` forces it; the frame wait buys nothing. Timers (`setTimeout`) still fire in the background; rAF does not.
**Action:** build and measure synchronously after the append (`pane.append(card); draw();`). Keep rAF only for things
that are purely visual and can wait for visibility (scroll resets, entrance classes). When polling a condition across
frames (a settle loop), accept that it only converges once the window is visible — and make the first synchronous pass
correct on its own.

## 22. 3D / extruded TEXT: the shadow goes on a COPY behind the face, per-letter 3D carries its own perspective

**Trigger:** you are building a logo-style title — extrusion via a `text-shadow` stack, a gradient face
(`background-clip: text`), a tilt, or letters bent around a curve.
**Discrimination / the three traps, each measured:**
1. **`text-shadow` paints THROUGH `background-clip: text`.** With a transparent fill the shadow shows inside the glyphs and
   muddies the gradient. Put the shadow on a second copy of the text BEHIND the face: `.ch::before { content:
   attr(data-text) / ""; position: absolute; inset: 0; z-index: -1; text-shadow: var(--stack); }` and keep the face
   shadow-free. The face must be a CHILD span (its background paints above a negative z-index sibling); a `::before` of
   the same element that carries the clipped background paints OVER that background. Build the stack in **em** so it
   survives any later font-size fit.
2. **Letters inside a `transform-style: preserve-3d` parent rendered turned but with NO perspective** (orthographic: edge
   letters narrower, never smaller) — and `isolation`, `opacity < 1`, `filter`, `overflow` on the parent flatten it
   silently. Sturdy form: no 3D parent at all — EVERY letter carries the whole projection itself,
   `perspective(7em) translate3d(…) … rotateY(…) rotateX(tilt)`, about ONE shared `transform-origin` (the title's bottom
   centre, expressed per letter as `calc`/em offset). Mathematically the same picture, no 3D context to lose. Depth-sort
   with `z-index` from the computed z.
3. **Those negative `z-index` values fall BEHIND any ancestor with a background** (a Settings card): the far letters
   vanish and the preview shows one letter. Give the title wrapper `isolation: isolate` (safe now — nothing depends on
   a 3D context).
**Action:** verify a curve from a `snapshot` at a LARGE size, in BOTH directions — two renders that look identical mean
the transform is not reaching the screen. Push depth along the SCREEN's z (translate before the tilt) so the baseline
stays level however far a letter recedes.

## 23. A glyph the UI font lacks draws an EMPTY control — use an icon, not a character

**Trigger:** a button whose label is a symbol character (`✕ ✓ ◀ ▶ ⋮ ↗`) set in a display or mono font, or a screenshot
showing a blank bordered box beside labelled buttons. `grep -nE '"(✕|✖|✓|◀|▶|⋮)"' <ui js>` lists the candidates.
**Discrimination:** the same character renders fine in another font stack a few lines away (system UI falls back to a
symbol font; a bundled mono such as JetBrains Mono has no fallback glyph for it in WKWebView) — so "it works elsewhere"
proves nothing. Hover/title text still works, which is why it survives review.
**Action:** draw it with the app's SVG icon (`svg("i-x", 14)`), give the button `aria-label`, and size the svg in CSS
(a 0×0 icon is the same bug from the other side).

## 24. A window that stays LOADED across close / open shows yesterday's UI — reload it when its source changed

**Trigger:** a Settings / editor / inspector WKWebView that is created once and only hidden on close (the usual Mac
pattern — fast to reopen), in an app whose web layer is served LIVE from the checkout; and a report shaped like "that
option isn't in Settings" right after you added it. Check: `grep -n "loaded" <SettingsWindowController>` — a
`if !loaded { load }` guard with nothing that ever invalidates it is the defect.
**Discrimination:** your own headless check passes, because you reloaded the window by hand while testing; the user's window
was loaded hours earlier and nothing told it to refresh. The content windows are a different case — they have an
explicit reload command and shortcut; secondary windows usually do not.
**Action:** record `loadedAt` when the page finishes loading; expose "newest mtime of the web sources" (html + js + css
directories) from the instance; on every show, if the window is HIDDEN and the sources are newer than `loadedAt`, drop
the loaded flag and load again (never while the window is visible — never reload under the user). Log one line when it
happens. And before telling the user a Settings feature exists, confirm it in a freshly loaded window, not the one you
have been reloading.

## 25. `evaluateJavaScript` cannot return a Promise — "JavaScript execution returned a result of an unsupported type"

**Trigger:** a headless check evaluates an `async` function or anything that returns a Promise through `WKWebView.evaluateJavaScript` (the app's `eval` / `settings js` command) and gets `unsupported type` back — while the side effect DID happen. **Discrimination:** the error reads like the script failed; it only means the completion value could not be serialized. Re-running "to make it work" repeats the side effect (a page reorder can run twice that way). **Action:** make the evaluated expression synchronous — start the async work, stash its outcome on `window.__r`, end the script with a plain value (`1`), then read `window.__r` in a second call after a short wait; or use `callAsyncJavaScript` in the shell when the app's command surface can be changed. Read the result from the app's state / the file on disk before assuming failure.

## 26. `@property`-driven conic gradients animate in WKWebView — the one-element travelling border

Verified on macOS 26: `@property --angle { syntax: "<angle>"; inherits: false; initial-value: 0deg; }` plus `background: conic-gradient(from var(--angle), …) border-box`, a transparent border, and the double mask (`-webkit-mask: linear-gradient(#000 0 0) padding-box, linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude`) gives a bright arc that travels an element's perimeter from ONE pseudo-element and one keyframe (`to { --angle: 360deg }`). **Trigger:** a "moving line around the window / card" effect. **Discrimination:** animating `background-position` on a linear gradient only slides along one axis and jumps at the corners; rotating an oversized child needs `overflow: hidden`, which clips glows. **Action:** the recipe above on a `position: fixed; inset: 0; pointer-events: none` layer; drive its duration from a CSS variable so the speed can be a setting.

## 27. Every webview window has its OWN JavaScript store — a setting saved in one never reaches the others

**Trigger:** the app opens more than one webview window (Tauri `WebviewWindowBuilder`, a
second `WKWebView`, pre-warmed hidden "spare" windows) and the user reports that a
Settings change "did nothing" in a compose / reply / editor window — check with
`grep -rnE 'readPref|localStorage|hydrate' <stores>`: a store that reads its prefs ONCE at
boot and has no listener for other windows' writes has this bug. Pre-warmed windows are the
worst case: they booted at launch, so EVERY later change is invisible to them.
**Discrimination:** the same symptom inside ONE window is not this — that is a read that
is not reactive (section 28); test by flipping the setting and looking at the SAME window
first. When both could apply, fix both: they stack (fixing only
one of them can ship a "fix" that changes nothing).
**Action:** (1) the backend's pref-write command broadcasts `prefs:changed <key>` to every
window after the write; (2) each store registers ONE listener (inside its hydrate
function, guarded by a module flag) that re-runs hydrate for its own key; (3) the ECHO
GUARD is the point — the store remembers the JSON text it last wrote OR read, skips a
re-hydrate for identical text, and its autosave effect never rewrites an unchanged
snapshot. Without the guard every window's re-hydrate writes again and pings every other
window forever.

**Second shape, same root:** each window kept a copy of ONE settings blob
(settings + combos + per-folder last note) and wrote the WHOLE blob back on any change — so a
window that hadn't heard a sibling's edit silently reverted it on its next save. Fix: the native
`setSettings` stores the blob and pushes `settingsChanged` to every page EXCEPT the sender; each
page replaces its copy (`parseSettings`) and re-applies. The sender exclusion is the echo guard.

## 28. SolidJS: a `<For>` callback runs ONCE per item and tracks NOTHING — an `if (signal())` inside it freezes the item in the state it was built in

**Trigger:** a toggle "does nothing" to a list / toolbar / strip that is already on screen
(or kept mounted, or pre-warmed) but works after a full reload — grep the component for a
helper called from `<For each=…>{(m) => helper(m)}</For>` or `.map(…)` that branches with a
plain `if` / ternary on a signal: `grep -nE 'if \(!?(ui|settings\(\)|store)\.' <component>`.
**Discrimination:** the SAME read inside JSX — `<Show when={sig()}>`, an attribute,
`classList={{…}}` — IS tracked (the compiler wraps it in an effect); only statements in the
callback body are not. A helper called inside `createMemo` / `createEffect` is tracked too.
**Action:** keep the item's node constant and move the condition INTO the JSX:
`<span>{renderItem(m)}<Show when={ui.showKeys() && keyOf(m.id)}><kbd>…</kbd></Show></span>`.
Then verify in the FAILING ORDER: build the surface first, flip the setting after, and
flip it back — a probe that flips the setting before the surface is ever built passes on
the broken code (a fix can ship twice that way before the real cause is found).

## 29. Third-party HTML on a THEMED paper: text that declares its color but no background disappears — measure contrast per element, never force white or invert

**Trigger:** the app renders someone else's HTML (an email, a feed item, a pasted page) inside a
frame whose background follows the app's theme, and a report says "this one is hard to read" /
"it fades into the background". Check one failing element: `getComputedStyle(el).color` is a
declared dark (or light) color and NO ancestor inside the document paints a `background-color`
or `background-image` — the author assumed white paper.
**Discrimination:** text over a background the document paints ITSELF (a white card, a colored
button, a hero image) is the author's design — leave it alone, whatever its contrast; only text
whose effective background is the app's paper is yours to fix. Forcing white paper for the whole
document throws away the themed look for every mail; a global `filter: invert()` or a blanket
`* { color: … !important }` wrecks logos, buttons and cards. When a background IMAGE is behind the
text its color is unknowable — treat it as painted and skip.
**Action:** from the PARENT (a scriptless `sandbox="allow-same-origin"` frame still gives computed
styles): walk the text nodes; per parent element, climb ancestors for a painted background; if none,
compute the WCAG contrast of its color against the paper; below 4.5 set an attribute
(`data-…-contrast="text|muted|link"`) and let ONE injected rule re-point marked elements to the
theme's text / muted / accent with `!important`. Keep the author's hierarchy: a color that was
low-emphasis on the paper it was designed for (contrast under 7 against white for dark text, against
black for light text) maps to muted; links map to the accent; fall back to the text color when the
theme's muted / accent would itself fail. Clear the marks BEFORE each pass (so the colors read are
the author's), write marks AFTER all reads, re-run on every theme / setting change, and retry when
the frame was not rendered (a `display: none` frame has no computed styles — no parsable `color`
anywhere means "not measured", not "all fine"). Resolve theme variables through a probe element's
computed `color` so hex / rgb / custom notations all arrive as `rgb(…)`. Run it with a white paper
too — light text whose dark background was stripped is the same class. Not fixable by nature: dark
artwork on a transparent PNG.

## 30. "Settings opens ready to search" in a web UI: focus on every show, and hand a stray keystroke to the field WITHOUT swallowing it

**Trigger:** you are making Settings open ready to search (the search field is focused on open; typing
anywhere reaches it) in an in-window web overlay or a web-rendered Settings window — open it, click
a section, press a letter: nothing in the field = not done.
**Discrimination:** `autofocus` runs once per element creation — enough only when the panel is
re-created per open; a panel kept mounted needs the focus call on every show. Do NOT re-dispatch or
synthesize the character: moving focus inside the `keydown` handler and NOT calling
`preventDefault` makes the engine deliver the character to the newly focused input on its own.
**Action:** (1) on open: `el.focus({ preventScroll: true })` + caret to the end, from a microtask
after the markup exists; (2) a `window` `keydown` listener alive only while Settings is open: bail
when `e.defaultPrevented`, any of ⌘ / ⌃ / ⌥, `e.isComposing`, `e.key.length !== 1`, Space, a
shortcut recorder is listening, another overlay (picker, palette) stands over Settings, or
`document.activeElement` is an input / textarea / select / contenteditable — else focus the field
and return; (3) Esc in the field: a non-empty query is cleared and the key stopped (focus stays),
an EMPTY field lets Esc through so it closes Settings — with autofocus making Esc blur the
field costs an extra keypress on every close; (4) the ✕ `preventDefault`s its `mousedown` and
refocuses; (5) reset the query when Settings closes or it reopens on a stale result list. Verify in
both orders: open → type, and click a section → type; read `document.activeElement.id` and the
event's `defaultPrevented` (must be false).

## 31. A ⌘-chord the page does NOT `preventDefault` falls through to the menu bar — a "dead" app shortcut is usually the native menu item firing instead

**Trigger:** a ⌘-chord the web UI registers "does nothing", or does something the page never
asked for — ⌘A selects all the page's TEXT, ⌘W closes the window, ⌘Z undoes typing — while the
same chord works in some other state of the UI (e.g. in a mail app, ⌘A selects the list's rows only until
an email is clicked; then it selects page text).
**Discrimination:** WKWebView's `performKeyEquivalent:` hands the key to the web content FIRST;
only when the page's `keydown` did not call `preventDefault` does WebKit re-send it to
`[NSApp mainMenu]`, where a predefined item (Edit → Select All, Undo, Close) or any item with that
accelerator fires. So the chord DID reach JavaScript — the app's own dispatcher declined it. The
usual reason is a CONTEXT gate: the action is registered for one pane ("list") and a click in
another pane (opening an email → focus moves to the reading pane) silently removes it. A wrong
accelerator or a missing ACL grant looks different: those fail in EVERY state.
**Action:** (1) prove which side handled it — `window.addEventListener("keydown", e => console.log(e.key, e.metaKey, e.defaultPrevented), true)`,
press the chord in the failing state: the event arrives with `defaultPrevented === false`; (2) find
the gate in the dispatcher (e.g. an `activeContexts()` helper; the action's `contexts`),
and register the action for every pane where its verb makes sense — a list act like select-all
belongs to the list AND the reading pane, like every mail verb; (3) keep `skipWhenTyping` (or the
equivalent) so the chord still stands down inside inputs, where the native menu action IS what the
user wants; (4) verify in the FAILING ORDER in the browser mock — click into the other pane first,
then dispatch the chord on `window` and read the app's own state, not the DOM selection.

## 32. `spellcheck="true"` draws no red squiggles — WebKit's continuous spell checking is OFF by default

**Trigger:** a WKWebView editor (contenteditable, CodeMirror, textarea) carries
`spellcheck="true"`, the app's "Spell check" setting is on, and misspelled words show no
squiggle. **Discrimination:** the attribute only gates checking PER ELEMENT; the process-wide
switch lives in the UI process's `TextChecker`, which reads `WebContinuousSpellCheckingEnabled`
from the APP's own defaults and treats a missing key as off. A programmatic insert
(`view.dispatch`, `execCommand`) is never checked either — so an eval-driven test shows nothing
even when the fix is in; only real typing (or a real NSEvent key through the window) is.
**Action:** (1) in `applicationWillFinishLaunching` — before the first web view, which a Finder
open creates before didFinishLaunching — set `WebContinuousSpellCheckingEnabled = true`, and
`WebAutomaticSpellingCorrectionEnabled`, `WebAutomaticTextReplacementEnabled`,
`WebAutomaticQuoteSubstitutionEnabled`, `WebAutomaticDashSubstitutionEnabled` = false (so nothing
autocorrects); (2) **the setting must switch WebKit itself, not only the attribute** —
turning `spellcheck` off on the element leaves every squiggle already drawn. WKWebView answers only `toggleContinuousSpellChecking:` (no
`isContinuousSpellCheckingEnabled` / setter — probed); read the live state by validating a menu
item with that action — `let item = NSMenuItem(title: "", action: sel, keyEquivalent: "");
_ = (webView as NSUserInterfaceValidations).validateUserInterfaceItem(item); item.state == .on`
— and `perform(sel)` when it differs from the setting. Off unmarks every word in every web view;
the toggle persists the defaults key itself. (3) verify with real key events into a SCRATCH note,
snapshot with the setting on, flip it off, snapshot again — `takeSnapshot` renders the squiggles.

**Existing text needs a re-check pass.** WebKit marks
words only while typing or when the selection LEAVES them, so a note that opens, text that was there
when checking turned on, and lines CodeMirror redraws (scroll, syntax reveal) show no squiggles.
Rebuilding the DOM does not help, and a bounce in ONE task checks nothing (WebKit keeps the
pre-batch selection). What works (measured): per visible line, `sel.collapse(lineNode, off)` in one
task, restore the saved anchor/focus in the NEXT task (`setTimeout 0` each) — WebKit checks the whole
line it left, and CodeMirror never adopts the parked selection (head read back unchanged mid-walk).
Guard it: a capture `keydown` / `mousedown` listener restores the selection first, abort when the doc
changes, skip when there is no selection to restore. **Unfocused tests LIE here (the symptom: flicker, and the
cursor position jumps around):** CodeMirror ignores selectionchange while it lacks focus,
so every headless run showed the caret steady — focused, it adopted each parked line (caret 7 → 2 → 543
→ 331, syntax revealed line by line). Fix: `EditorState.transactionFilter` returning `[]` for
`!tr.docChanged && tr.isUserEvent("select")` while a `parking` flag is set; any key/click ends the walk and
clears the flag BEFORE CodeMirror handles it; reset the flag at the start of every run so a cancelled walk
can't leave clicks blocked; never schedule the walk on caret moves. Verify in a browser harness after a
REAL click — assert `view.hasFocus === true` first, then sample the head every frame during the walk.
And the native toggle reaches only the ONE web
view it is sent to — toggle every OTHER view twice so its content process gets the state.

**A walk must never START while the user types (the symptom: the cursor
jumps while typing).** *Trigger:* any code that moves the DOM selection on a timer in a WKWebView editor.
*Discrimination:* a `keydown` guard looks sufficient and passes every Chromium test — but in
WKWebView a key press is TWO messages from the UI process, `keydown` and then the text insertion,
so a timer can fire BETWEEN them: the guard has already run, the walk parks the caret, and the
letter lands on the parked line. *Action:* (1) gate the timer on input silence — permanent capture
listeners stamp `lastInputAt` on `keydown` / `beforeinput` / `compositionstart` / `mousedown`, and a
due walk re-schedules itself until 1.2 s have passed (and while `view.composing`); (2) add
`beforeinput` and `compositionstart` to the events that END a running walk — `beforeinput` fires
right before insertion, so the selection is home in time. Verify: harness, real click, then real
typing while a recheck is requested every 60 ms → 0 walks and every letter contiguous; plus a walk
interrupted mid-park by a dispatched `beforeinput` (wrap `Selection.prototype.collapse` to catch the
parked moment) → selection restored immediately.

## 33. The standard macOS spelling tools in a WKWebView editor — panel, Check Document Now, grammar

**Trigger:** the user asks for "the default Mac spell checker" / ⌘: / ⌘; / grammar checking in a
web-UI editor. **Discrimination:** nothing needs building — WKWebView responds to
`showGuessPanel:` (the system Spelling and Grammar panel), `checkSpelling:` (select the next
misspelled word), `changeSpelling:` (the panel's Change) and `toggleGrammarChecking:` (probed with
`responds(to:)`). The trap is the TEST: in a background window a CodeMirror editor shows nothing
selected after `checkSpelling:` — WebKit DID select the word, but CM only adopts a DOM selection
while `view.hasFocus`, and otherwise writes its own selection back (`onSelectionChange → flush →
docView.updateSelection`, caught by wrapping `updateSelection`). Not a bug in the focused app.
**Action:** (1) an Edit ▸ "Spelling and Grammar" submenu: Show Spelling and Grammar ⌘: , Check
Document Now ⌘; , Check Spelling While Typing, Check Grammar With Spelling — the first two as
rebindable registry actions (⌘: stored as ⇧⌘; , LABELLED ⌘: everywhere) whose handler focuses the
editor and asks native to `makeFirstResponder(webView)` then `NSApp.sendAction(sel, to: webView)`;
(2) grammar is its own persisted setting (default off, like TextEdit), applied with the same
validate-then-toggle probe as #32 (`toggleGrammarChecking:`, seed `WebGrammarCheckingEnabled` in
willFinishLaunching), and DISABLED (Settings row grayed with a "Turn on Spell check" note, menu
item `validateMenuItem → false`, bound key a no-op) while spell check is off; (3) verify with the
offscreen probe over the browser harness with `document.hasFocus = () => true` in the page:
`checkSpelling:` → `String(getSelection())` is the word and CM's selection matches; then
`sendAction("changeSpelling:", to: v, from: NSTextField(string: "fix"))` → the doc text changed (so
autosave follows). Add an `app://spelling[?action=panel]` URL command for the in-app path.

## 34. The editor's right-click menu: WebKit's own is TextEdit-grade — trim it, and own its Spelling submenu

**Trigger:** "add a right-click menu with spelling suggestions" in a WKWebView editor.
**Discrimination:** don't build one — WebKit's default menu on a misspelled word already has the
guesses, Ignore / Learn Spelling, Look Up, Translate, Cut / Copy / Paste, Writing Tools,
Substitutions, Transformations, Speech, Share, and picking a guess replaces the word through
WebKit (a CodeMirror doc sees a normal edit). Unless the page `preventDefault`s `contextmenu`
there, it is already live. Two real defects in it: its **Spelling and Grammar submenu toggles the
checkers directly**, bypassing the app's persisted setting, and **Font / Paragraph Direction /
Selection Direction** inject rich formatting into plain text (Reload also appears on
non-editable areas). **Action:** override `willOpenMenu(_:with:)` on the WKWebView subclass:
remove `WKMenuItemIdentifierReload` and the submenus titled Font / Paragraph Direction / Selection
Direction (they carry no identifier), replace `WKMenuItemIdentifierSpellingMenu`'s submenu with
the app's own builder (the SAME function the menu bar uses, targets on the app delegate so
validation draws the check marks), then collapse doubled / trailing separators. **Verify without
popping a menu on screen:** offscreen probe WKWebView subclass over the harness, synthetic
`rightMouseDown`/`Up` via `window.sendEvent` at `coordsAtPos` of the word, dump `menu.items` in
`willOpenMenu`, then `menu.cancelTracking()`; `menu.performActionForItem(at: 0)` then reading the doc
proves a guess edits the note.

## 35. A popup's ↑/↓ move the LIST behind it — the app keymap listens on window CAPTURE

**Trigger:** a menu or picker adds its own arrow-key walk (`onKeyDown` on the menu, with
`stopPropagation`) and the arrows do nothing in it, or the list cursor underneath moves
instead. Check: `grep -n 'addEventListener("keydown"' src` — a keymap registered with
`{ capture: true }` on `window` runs BEFORE any element's handler, so the popup's
`stopPropagation` comes too late. **Discrimination:** a text field inside the popup is
protected by the keymap's "typing" rule; a focused BUTTON row is not, so a plain arrow
reaches the global bindings. The probe tell: the event comes back `defaultPrevented` while
focus did not move (the keymap consumed it). **Action:** the popup publishes an "open" signal
(`ui.setScopeMenuOpen(open())` in an effect, cleared on cleanup) and the keymap's context
resolver returns its MODAL context while it is set, the same list the other pickers and dialogs
are on. Then the popup's own handler gets the arrows. Every new keyboard-driven popup joins that
list. Verify in the mock with a dispatched `keydown` on `document.activeElement` and read the new
`activeElement`.

## 36. A native event pushed while the page boots arrives BEFORE the page's `ready` reply — hold it, or it is silently dropped

*Trigger:* a bridge where native queues `push(event:)` calls until the page says `ready`, and
the page builds the object that handles the event (a Settings panel, an editor) inside the
`ready` reply's `.then`. The report: "the FIRST time it opens it ignores what I asked for" —
Settings opening on its first section instead of the one a ⌃-letter chord or a "Customize…"
row asked for; every later open works. *Discrimination:* a repeat open works because the
page is already booted, so testing on an open window proves nothing — test on a FRESH
window (kill + relaunch, then the command). A handler written `on("x", d => panel?.show(d))`
reads as "safe" and is exactly the bug: `markPageReady()` flushes the queue before the reply
is sent, so `panel` is still null when the event lands. *Action:* the handler stores the
latest payload while its target is null (`pendingShown = data`), and the boot code applies it
right after creating the target (`panel.shown(pending?.section, pending?.filter); pending =
null`). Verify on a fresh window with the app's URL command (`settings?section=shortcuts`)
and read the active section back.

## 37. A WKWebView inside a plain container view shows a BLANK window while its page is fine

- **Trigger:** you move a WKWebView off `window.contentView` into a container `NSView` (to add a native strip, sidebar or toolbar beside it), and the window paints only its background color. `evaluateJavaScript` shows the DOM rendered, `document.visibilityState` is `visible`, `takeSnapshot` returns the full page, and the frame is correct.
- **Discrimination:** every headless check passes: snapshots render out of process, so they can't see it. Only the on-screen window is blank. That is what makes it slip through. A container whose sibling draws with `draw(_:)` (a title strip filling its rect) is the usual culprit.
- **Action:** `container.wantsLayer = true`, and give sibling chrome a layer color (`wantsLayer = true` in `init`, `layer?.backgroundColor = …`) instead of `draw(_:)`. Guard it: dump `window.contentView?.layer != nil` in the app's state hook and fail the build's UI check when it is false.

## 38. `node --check file.js` does not prove an ES module loads in WebKit

- **Trigger:** you syntax-check the page's `type="module"` scripts from the shell before a build.
- **Discrimination:** plain `node --check` passed a file with an extra `}` while WebKit reported "SyntaxError: Parser error" and the whole page (every module importing it) stayed blank; the guard only said "guard missing".
- **Action:** check modules as modules: `node --input-type=module --check < file.js` for every file under the web folder; when a page is blank, `import('./js/<m>.js').catch(String)` each module from an eval to find the one that fails.
- **Make it a test, not a habit:** a second `let editTimer` added to a function that already has one can blank EVERY window of the app (a build guard then reports only "guard missing" on every view). Run the module check from the unit tests (`WebUIGuardTests.testEveryWebScriptParses`: `Process` running node on each `js/*.js`, looking in `/opt/homebrew/bin`, `/usr/local/bin` and `~/.local/bin`, skipped if none), so `swift test` fails before a build can ship it. Before adding a timer or helper name to a big view function, `grep -nw '<name>'` the file.

## 39. `null` printed on the page: EVERY DOM insert method writes the word, not just `append`

- **Trigger:** a view builds children with `cond ? el : null` and passes them to `append`, `prepend`, `replaceChildren`, `before`, `after` or `replaceWith`; the page shows a stray "null" (or "nullnull").
- **Discrimination:** patching only `append` is not enough; the next stray word comes from a call like `replaceChildren(…, flagsCard(…) /* null when clean */)`. Patching the one method in the screenshot is the failure mode.
- **Action:** one patch at the top of the shared UI module, for `Element.prototype` AND `DocumentFragment.prototype`, over all six methods: `proto[m] = function (...kids) { return native.apply(this, kids.flat(Infinity).filter((k) => k != null && k !== false)); }`. Guard it twice: a source test that lists the six names, and the layout guard failing on any text node equal to "null"/"undefined" in every view — which only works when every screen has a sample in the guard.

## 40. What a person typed must live in page STATE, not in the field — a redraw rebuilds the field from the saved record

- **Trigger:** a screen with editable fields also has actions that redraw it (approve a finding, clean an image, refresh a list).
- **Discrimination:** the edited repo name went back to the original after "Approve": the redraw rebuilt the field from the saved project. Saving on every keystroke is not the fix (it can commit half-typed values).
- **Action:** a per-screen form object in the view's state (`st.approveForm = { owner, repo, vis }`, keyed by the record so another record starts fresh), fields built FROM it with `oninput` writing back; reset clears it. A source test pins the pattern.
- **State is not storage:** fields can be typed, "Save Progress" can show ✅, and after a restart they are gone: they lived only in `st`, and the save stored WHERE the user was, not WHAT they typed. So the form object must also reach disk while the user types; see item 42. Half-typed values are fine to store as long as the record is a draft (nothing is published until the final action).

## 41. Nothing blinks: a redraw keeps what is on screen, a late answer updates only its own card

- **Trigger:** a view's `draw()` rebuilds its window (`clear(root).append(…)`, `root.innerHTML = …`), a card loads something slow (a `claude -p` suggestion, a scan, a GitHub call) and then calls the page's `draw()`, or an action replaces the page with `working("…")` until it is done. Symptom: the page flashes, pictures reload, a glow or spinner jumps, a scrolled list snaps to the top, a README goes back to "Loading…".
- **Discrimination:** Typical case: on an approval page, a card's late suggestion arrived and called `draw()`; the page started over with its "Checking everything once more…" spinner and a second scan, so the whole page blinked. Fixing that one card was not enough: the same blink was in every window, because every redraw rebuilt the DOM from scratch. A skeleton the first time data arrives is fine; a spinner in place of content already seen is the defect. A DOM-morphing library is the wrong fix when listeners are closures added by `h()`: morphing keeps the OLD nodes with the OLD closures.
- **Action:** in the shared UI module:
  1. `steady(root, ...tree)` replaces `clear(root).append(...)` in EVERY top-level draw (and any sub-pane that redraws, such as the Settings pane and nav). Before the swap it records: `img[src]` / `video[src]` by tag + src, `root.getAnimations({ subtree: true })` currentTime keyed by `animationName | pseudoElement | tag.className`, every element with `scrollTop || scrollLeft` by child-index path, and the focused input/textarea with its selection. After the swap it moves the OLD media nodes into the new tree (copying the new attributes, so nothing reloads, not even `loading="lazy"`), sets each matching new animation's `currentTime` (glows carry on, and a finished one-shot fade-in stays finished instead of replaying), and restores scroll and focus when the node at that path has the same tag + className.
  2. `busyOver(holder, textOrWorkingEl)`: the page stays, `.busy-over > :not(.busy-pill) { opacity: .45; pointer-events: none }`, with a sticky working pill prepended. Keep it in state (`st.busy`, marked `.over`) and re-apply it after every `draw()` while the action runs, so a redraw mid-action keeps the pill; views that used to `return st.busy` skip it when `.over`.
  3. A card that loads late swaps ITSELF: `root.querySelector(".about-card")?.replaceWith(aboutCard(...))`, never `draw()`. While loading it shows skeleton rows the size of the real ones (a shimmer), then the content with a one-shot `.fade-in`.
  4. Every async answer a view shows (a scan, the README text, the next release) is kept in state keyed by its record (`st.approveLast = { name, result }`) and painted at once on the next draw, with the re-check running behind it; repaint only if the JSON differs. Clear the cache when the user leaves that screen, so a stale "Clean" is never shown after work elsewhere; an action's own answer updates the cache at once.
  5. Guard test: no `clear(root).append(` in any view file, no `replaceChildren(busy)` / `append(st.busy)`, and `steady` / `busyOver` present.

## 42. Save as you type, and finish every pending save before a window closes or the app quits

- **Trigger:** a page has fields or editors a person types into (a wizard page, Settings, a small "new item" window), you are about to write a debounced save (`setTimeout(save, 350)`), an **Apply** button, a save that runs only on `change` / blur, or a Save toast. Symptom: "I saved and it was gone after a restart", or the last few characters typed before closing Settings are lost.
- **Discrimination:** three separate holes, each easy to miss: (1) typed values lived only in page state and the "✅ Progress Saved" toast covered only the step the user was on; (2) Settings fields saved on blur or on an Apply button, and Browse… filled a field without saving it; (3) a debounced save still waiting is lost when the window closes, because `windowWillClose` removes the script message handler, and when the app quits. A toast is only honest when it follows the write it names. Explicit editors with Save / Cancel keep those buttons: store their UNSAVED text as a draft (restored with the editor open), and let Save / Cancel decide.
- **Action:**
  1. One helper in the shared UI module: `autosaver(fn, ms)` returns `save()` (debounced) with `save.now()`, and registers each pending run in a set; `onFlush(fn)` registers one-off work (text typed into an "add a chip" box, not added yet); `flushPending()` runs them all. The page exposes `window.__app.flush = () => flushPending()`. Every debounced save uses `autosaver`, never a bare `setTimeout`.
  2. Native, before the page can no longer answer: `windowShouldClose` returns false once, awaits `webView.callAsyncJavaScript("if (window.__app && window.__app.flush) await window.__app.flush();", arguments: [:], in: nil, in: .page)` with a 1.5 s cap, then `performClose` again (a flag lets the second call through). `applicationShouldTerminate` flushes every window through a `DispatchGroup` and returns `.terminateLater`, replying `true` when the group finishes. A page's own "close window" bridge call must use `performClose`, not `close()`, so it goes through the same path.
  3. Settings: every text field is a `liveField` (saves on input through `autosaver`, and at once on `change`); remove Apply buttons; a Browse… that fills a field also saves it.
  4. Wizard-style pages: one `edited()` every field and open editor calls. It records the page state for the native close handler, and when the app's auto-save setting is on it writes the draft record and the saved run (`SavedProgress.edits`). With the setting off, nothing is written until ⌘S, and ⌘S always writes everything first, then shows ✅. Reopening restores it (`restoreEdits`).
  5. A "new item" window (e.g. a new rule) under auto-save adds the item as you type it and updates it in place; Cancel / Esc takes it back out; removing its text removes it. Blank rows are dropped by the store's load and save (`Rules.withoutBlanks`), so a never-filled "＋ Add" row doesn't pile up.
  6. Guard tests: a source test listing every editable field and editor with its `edited()` / `liveField` call and failing on a new one without it; a test that the saved record round-trips the edits; a test that windows flush before closing and quitting.

## 43. `evaluateJavaScript` probes share ONE global scope, and a Promise result is an error

- **Trigger:** you verify a WKWebView page from outside with repeated `evaluateJavaScript` calls (an app's `eval` URL hook, a test harness), and a later probe returns a stale value, nothing, or an "unsupported type" error.
- **Discrimination:** WebKit runs each call as a classic script in the page's global lexical scope. A top-level `const f = …` in probe 2 throws `SyntaxError: Can't create duplicate variable: 'f'` when probe 1 declared it too, and the whole probe does nothing. This can make a field look stuck for several inputs in a row while the app is fine, or let a probe step silently NEVER run, so its "pass" is the previous step's state; so a step that plants state must assert the state is there before it acts. A probe whose last expression is a Promise (`import(...).then(...)`) fails with `WKErrorDomain Code=5 "JavaScript execution returned a result of an unsupported type"`. Before you call either one a bug, check the raw error text your harness wrote.
- **Action:** wrap every probe in an IIFE (`(() => { const f = …; return …; })()`) or use `var`. For async answers, start the work and stash the result (`….then(r => { window.__r = JSON.stringify(r) }); 1`), then read `window.__r` in a second call. Or use `callAsyncJavaScript` in native code, which awaits a Promise. Every probe returns the preconditions it relied on next to its result.

## 44. A celebration that plays ONCE through redraws: pin Web Animations to one start time

- **Trigger:** a one-shot animated moment (a finish screen, a success finale, a staged reveal) inside a view that redraws by rebuilding its DOM (`steady()`, item 41), or that the user can leave and come back to.
- **Discrimination:** CSS animations restart whenever an element is rebuilt or removed and re-inserted. `steady()`'s carry-over only matches running animations by key, so coming BACK to the screen replays the whole thing. Also, `element.className` on an SVG element is an `SVGAnimatedString` object, not a string: any key built from it (`${tag}.${el.className}`) collapses every SVG element into one bucket. Use `getAttribute("class")`.
- **Action:** store one start time in view state the first time the moment shows (`st.finale = { key, t0: document.timeline.currentTime }`). Build every entry animation with `el.animate(keyframes, { delay, duration, fill: "both" })` and set `anim.startTime = t0`, so a rebuilt node lands on the same frame and a return after the end shows the final state. Infinite ambient loops (a breathing glow) are pinned the same way with `fill: "none"`. A whole-window layer (a fixed canvas for particles, a wash) is appended to `document.body` OUTSIDE the redrawn root, keyed so it plays once, and removes itself when done. Persistent pieces (a tower built step by step) are kept as nodes in state and moved by the redraw, never rebuilt; a CSS drop-in flips from `.falling` to `.settled` on `animationend` so re-insertion can't replay it. Theme colors reach a canvas through `getComputedStyle(document.documentElement).getPropertyValue("--token")`.

## 45. A view that throws while drawing freezes the LAST screen: a finished run looks stuck forever

- **Trigger:** a progress screen (a spinner, stages, a running clock) stays up after the work behind it is done, and its Stop / Cancel button "does nothing". Or you write a view function that shows a remembered value at once and calls helpers to finish it.
- **Discrimination:** it looks like a hung backend, but read the signals first. The native side is idle (`sample <pid> 2` shows no app frames), the output is on disk, and the page still answers an eval, so JS is not hung either. The page's clock froze at the moment the work ended, so the reply DID arrive and the code after the `await` ran (it cleared the ticker). What failed is the redraw after it. The usual cause is a `const helper = () => …` called above its definition inside the same view function. That is a temporal-dead-zone ReferenceError that fires only on the branch that calls it early. For example, a branch that shows a value seen before after a "nothing yet" check. So every quick test passed, and the error was an unhandled rejection nobody saw. Stop looked dead because it only stopped the worker, which had already finished.
- **Action:**
  - (1) Reproduce, don't reason: collect errors in the page with an eval that installs `error` / `unhandledrejection` listeners, then replay the flow. The stack names the line.
  - (2) Fix at the call site: call helpers only below their definitions.
  - (3) Fix the class: the view's `draw()` wraps building the step in try/catch and renders a `.page-error` banner (the message plus Reload), never leaving the old screen up. The layout guard fails on any `.page-error`.
  - (4) Stop ends the run on the page FIRST (clear the ticker, drop the run state, redraw), then asks native to kill the worker.
  - (5) Add a guard demo that ENDS the run from the state that broke it (every path), and assert the progress card is gone.
  - (6) Test end to end with a stand-in worker. Put a fake CLI on `PATH` for a second app instance started with `open -n --env PATH=… --env <APP>_HOME=<scratch store>`.

## 46. A module the entry module imports must not READ the entry's exports while it loads: every window goes blank

- **Trigger:** every window of the app is blank at once, or the layout guard reports "guard missing" on every view, right after you added a top-level line like `const DEMO = query.demo === "x"` to a module that `main.js` imports (and that imports `{ query }` from `main.js`).
- **Discrimination:** it parses fine (`node --input-type=module --check` passes, `testEveryWebScriptParses` passes), so it is not item 38. ES module cycles evaluate the imported module FIRST, so `main.js`'s `export const query = …` is still in its temporal dead zone: the top-level read throws a ReferenceError and the whole graph fails to load. The same import used INSIDE a function is fine (it runs after `main.js` finished).
- **Action:** at a module's top level, read only what it owns: `new URLSearchParams(location.search)` instead of the entry's `query`, or move the read into the function that needs it. Note the rule in the project's CLAUDE.md. The layout guard catches it (every view fails), so always run the full guard after touching module tops.

## 47. A field's caret is drawn ONCE, then never again: a positioned field above a scroll container with no stacking context

- **Trigger:** "the cursor blinks once then disappears", "I can't click into X", "Tab doesn't reach X", while focus checks say X IS focused (`document.activeElement`, the AX focused element) and typing still lands in it. Above all on a screen whose content below the field is long (in an email composer, a forward carries the whole original email, so the body always scrolls; a new message's short body never shows it).
- **Discrimination:** not a focus bug, so no focus fix helps (explicit Tab handling, an inert preview frame and an activation repair all fail). A headless or OFFSCREEN WKWebView is never the active window and draws NO caret at all, so every probe "passes"; a DOM / focus log records focus, never paint. The cause is WebKit's repaint: a field inside a POSITIONED box (`position: relative`, here the anchor of an autocomplete dropdown) above a sibling scroll container (`overflow: auto`) that has NO stacking context of its own gets its caret painted once and never repainted. A sibling field that is not inside a positioned box (the Subject) blinks normally in the same window.
- **Action:** give the SCROLLER its own stacking context: `isolation: isolate` (no layout change; `will-change: transform` or `position: relative; z-index: 0` measured the same). A stacking context on the FIELD's box (`z-index`, `will-change` there) did NOT fix it. Don't add isolation to every scroller blindly: a `position: fixed` popup rendered inside an isolated scroller can no longer stack above later siblings. Guard it with a CSS-scanning test on that rule. **Measure in the installed, ACTIVE app:** drive it with any UI automation tool and diff 4 screenshots 0.2 s apart (a 3-4 px column at the field's edge is the caret). To find which CSS property matters, ship ONE temporary build whose suspects are switches read from a prefs row (`<html data-dbg>` + CSS), then flip them between runs instead of rebuilding per guess.

## 48. Contenteditable / inline-edit fields must NEVER commit on `blur` — WebKit fires synthetic blurs on menu dismissal, window focus shifts, and first-responder handoffs

- **Trigger:** an inline rename or contenteditable field (sidebar file/folder rename, note title rename, table cell edit, tag editor) commits immediately / "presses Enter for me" / vanishes right after opening, especially when invoked from a native `NSMenu` (right-click context menu, menu bar item) or during window activation changes.
- **Discrimination:** it looks like a phantom `Enter` keypress or an immediate submit. In reality, the field committed because of a `blur` listener (`field.addEventListener("blur", () => finish(true))`). In WKWebView, AppKit native menu tracking dismissals, window focus shifts, first-responder handoffs, and system spellcheck cycles asynchronously fire synthetic `blur` events on editable elements. Even adding a grace period (`Date.now() - startTime < 250ms`) fails because native event draining and menu animations can easily exceed that window.
- **Action:**
  1. **Zero `blur` reliance:** NEVER commit or finish an inline edit or rename in a `blur` listener. Do not attach `blur` listeners to inline editing inputs or contenteditable elements.
  2. **Commit ONLY on explicit user actions:**
     - `keydown` with `Enter`: commit (`finish(true)`).
     - `keydown` with `Escape`: cancel (`finish(false)`).
     - Window-level outside click: register capture-phase `pointerdown` and `mousedown` listeners on `window` after a 100ms arming delay (`setTimeout(..., 100)`). If `field.contains(e.target)` is false, commit (`finish(true)`).
  3. **Idempotent cleanup:** Ensure `finish()` removes the window listeners immediately so subsequent clicks are unaffected.
  4. **Guard test:** Add an automated source scan (e.g. `web/test/rename.test.ts`) that inspects rename/inline-edit code and fails if any `addEventListener("blur"` or `.onblur` is attached to an editable field.

## 49. Clipboard: a page writes only inside a gesture, and its custom HTML is re-serialized

- **Trigger:** the app copies something WITHOUT a click or key press (a code copied the moment its mail arrives, a value
  copied when a sync lands), or copies rich text (links) out of a sandboxed / rewritten view.
- **Discrimination:** `navigator.clipboard.writeText` and `execCommand("copy")` need a user gesture in WebKit; from an
  event handler with no gesture (a native event, a timer, the app in the background) they fail or do nothing, and the
  headless test pane rejects them too, so a "works on click" check proves nothing about the automatic path. Inside a
  gesture, a one-shot `copy` listener calling `clipboardData.setData("text/html", …)` works, but WebKit re-serializes
  that HTML (computed inline styles, a `<meta charset>` head) before it reaches the pasteboard: links survive, your exact
  markup does not. A view that rewrote its links (for example, an email viewer that points every link at a private `app-open://` sink) copies the
  rewritten href unless the copy rebuilds the HTML with the real targets.
- **Action:** automatic copies go through native code: a command that writes `NSPasteboard.general` on the main thread. A copy the app owns (⌘C over a sandboxed frame) writes text/plain AND
  text/html with every anchor pointed at its real target, and the receiving editor decodes any private href that still
  arrives (`transformPastedHTML`). Prove it with a real-WebKit probe that reads `NSPasteboard.general.string(forType: .html)`.

## 50. Probing a real page: overlays win `elementFromPoint`, framework nodes must not be removed, and `takeSnapshot` sees a hidden window

- **Trigger:** a WKWebView probe checks that a control is on top, clears old state between steps, or needs a picture of
  an interactive state that a simple load-wait-shoot screenshot tool cannot reach.
- **Discrimination:** (1) a launch animation or splash covering the window for its first seconds makes
  `elementFromPoint` return the overlay (e.g. a boot-grid `DIV`), which reads as "the button is covered"; (2) deleting toast
  or list nodes the framework rendered (`el.remove()` on a Solid `<Show>` child) leaves the framework writing into the
  detached node, so the NEXT state looks missing; (3) `WKWebView.takeSnapshot(with:)` works on a titled window that was
  never ordered on screen, so no window has to appear for a picture.
- **Action:** wait for the overlay to leave (`!document.querySelector('.app-boot')`) before any on-top check; never remove
  framework nodes, assert on content unique to each step instead; for a picture, call `takeSnapshot` with a
  `WKSnapshotConfiguration.rect` around the control, write the PNG to the scratchpad behind an env var
  (`APP_PROBE_SNAPSHOT=<png>`), and Read it. **Two traps in that picture:** a never-shown window runs no
  CSS animations, so anything that enters with an animation (`opacity: 0` → 1, a slide) stays at frame 0 and the snapshot
  shows what is BEHIND it (the Settings overlay was invisible; the DOM checks had passed). Inject
  `*,*::before,*::after{animation:none!important;transition:none!important}` right before the snapshot. And a rect
  taller than the web view is cut at its bottom: make the view tall enough (2000px) and `scrollIntoView({block:'start'})`
  the element first.

## 51. An auto-growing textarea is cut by its own border under `box-sizing: border-box`

- **Trigger:** a textarea that grows with its text (`height = scrollHeight`) so a long value wraps instead of scrolling.
- **Discrimination:** `scrollHeight` excludes the borders; with the global `border-box`, the box is 2px shorter than its
  text and the last line loses its bottom pixels (measured: `scrollHeight` 67, `clientHeight` 65).
- **Action:** `el.style.height = "0px"; el.style.height = el.scrollHeight + el.offsetHeight - el.clientHeight + "px"`,
  and verify with `scrollHeight <= clientHeight + 1` after a long value.

## 52. Auto-fitting a flex sidebar from child `.label` bounds creates a `+7px` growth feedback loop on every autosave redraw unless `.label` is `flex: none` during measurement

- **Trigger:** when the sidebar is open and the user types in the editor, the sidebar grows wider by a few pixels on every pause/autosave and the editor screen jumps horizontally.
- **Discrimination:** autosave triggers the FSEvents watcher (`fsChanged`), which re-renders the sidebar tree and runs `fitSidebarWidth()`. Inside `.tree-item` (`display: flex; padding-right: 10px`), `.label` has `flex: 0 1 auto; min-width: 0; overflow: hidden`. Even when `#tree.measuring` sets `width: max-content` on `.tree-item`, any row already at the current `sidebarWidth` clamps `.label.getBoundingClientRect().right - treeLeft` to `sidebarWidth - 17px`. Adding `+ 24` (`Math.ceil(need + 24)`) computes `sidebarWidth + 7`, growing the sidebar by `+7px` on every `fsChanged` tick until it hits the 520px cap.
- **Action:**
  1. In CSS, force `#tree.measuring .tree-item .label { flex: none; min-width: max-content; overflow: visible }` so `.label.getBoundingClientRect().right` always measures the intrinsic text width, never the current container width.
  2. In the `fsChanged` handler, compare the incoming file tree JSON (`JSON.stringify(data.tree)`) against the last rendered tree and skip `sidebar.render()` / `fitSidebarWidth()` when only a file's contents changed on disk.

## 53. Unhiding a modal list (`CommandPalette` / `QuickSwitcher`) under a stationary cursor fires synthetic `mouseenter` in WebKit and steals selection from row 0

- **Trigger:** opening a modal list (`⌘P` Command Palette or `⌘O` Quick Switcher) highlights whichever row happens to sit under the stationary mouse pointer instead of the first row (`selected = 0`), or pressing `Enter` runs the hovered row instead of the top match.
- **Discrimination:** when an overlay flips from `hidden` to visible underneath the pointer, WebKit immediately dispatches `mouseenter` / `mouseover` on the element now under the cursor even though the mouse never moved. If rows wire `row.addEventListener("mouseenter", () => { this.selected = idx; })`, that synthetic entry overwrites `this.selected = 0` on open.
- **Action:** never use `mouseenter` to change modal list selection. Record `lastMouseX = -1, lastMouseY = -1` on `open()`, and listen for `mousemove` on each row, updating `this.selected` only when `e.clientX !== this.lastMouseX || e.clientY !== this.lastMouseY`.

## 54. Moving the DOM selection into a contenteditable steals focus from the field the user is typing in

**Trigger:** a background job sets the document selection on a timer or after a scroll (a spell-check walk that parks the caret on each line, a restore-selection step, a "reveal" helper), and keys typed in a find box, filter, rename field or table cell land in the document instead ("⌘F then ⌫ deletes content in the note").
**Discrimination:** headless tests pass because nothing else has focus. In WebKit, `getSelection().collapse(node)` inside a contenteditable moves FOCUS there; restoring the saved anchor (the input's container node) does not give focus back to the `<input>`. With a search match selected, the next ⌫ deletes the match.
**Action:** gate the job on `document.activeElement`: run only when it is `null`, `<body>` / `<html>`, or the editor's own content element (not a field inside a `contenteditable="false"` widget); otherwise reschedule. Check it both when scheduling and when the job starts.

## 55. A stub for the native bridge must parse the message, and never forward what it stubs

**Trigger:** a test replaces `window.webkit.messageHandlers.<name>.postMessage` to catch a side-effecting call (clipboard, Finder, a native menu) before it reaches native.
**Discrimination:** the page posts `JSON.stringify({ id, type, ...params })`, a STRING. A stub that tests `m.type` sees `undefined`, matches nothing, forwards everything, and the real side effect happens: a Copy-button test can overwrite the real clipboard that way.
**Action:** `const m = JSON.parse(raw)` first; for the stubbed type, record it and RETURN (never call the original). To fake a native reply to a `call()`, answer through the receive hook: `window.__recv({ id: m.id, ok: true, data: { … } })` (e.g. the item a native context menu would return).

## 56. Spelling squiggles live on the TEXT NODE, and `spellcheck="false"` is read per line

**Trigger:** you want WebKit's continuous spell check to skip part of a contenteditable (inline code, a token) and the
squiggles stay. **Discrimination:** `spellcheck="false"` on a block element (a whole line) works; on an inline span inside
a checked line it does not, because WebKit checks the paragraph the selection left and marks every misspelling in it
(measured with a selection-parking recheck walk). Swapping the attribute later never clears marks already drawn.
**Action:** marks are attached to the text node, so give that text a NEW element: a framework that re-parents the same
text node into a different element (CodeMirror after a mark's `tagName` changes) clears them; a text node replaced
behind the framework's back also clears them but breaks the framework's view (never do that). Verify with
a page-pixel snapshot, not the DOM: markers are not in the DOM.

## 57. Escape key dismissal for WKWebView modals / lightboxes requires both a capture-phase `window` listener and an AppKit `keyCode == 53` bridge, plus `.blur()` on the trigger when closing

- **Trigger:** a modal or lightbox (`#art-lightbox`) opened by clicking an element inside a `WKWebView` fails to close on `Escape` when the web view is not the window's `firstResponder` (or when a focused child swallows bubble-phase key events), or closing the modal leaves the trigger element looking slightly washed out / altered (`:focus-visible`).
- **Discrimination:** a bubble-phase `document.addEventListener("keydown")` only fires when `WKWebView` is `firstResponder` and no inner handler stops propagation. Furthermore, calling `trigger.focus()` on close activates `.linked:focus-visible { filter: brightness(1.06) }` in WebKit even after a mouse click, leaving a CSS filter raster buffer stuck on the trigger element.
- **Action:**
  1. **Capture-phase `window` listener in JS:** attach `window.addEventListener("keydown", (e) => { if (isOpen() && (e.key === "Escape" || e.key === "Esc" || e.keyCode === 27)) { e.preventDefault(); e.stopPropagation(); close(); } }, true)` and focus the dialog container (`tabindex="-1"`) on open.
  2. **AppKit `NSEvent` local monitor fallback:** in `NSEvent.addLocalMonitorForEvents(matching: .keyDown)`, when `e.keyCode == 53` (`Escape`) in a web view window and `win.firstResponder !== wc.web.webView`, restore `win.makeFirstResponder(wc.web.webView)` and invoke the JS `escape()` handler directly (`wc.web.call("escape")`).
  3. **Blur the trigger on close:** call `trigger.blur()` instead of `trigger.focus()` when closing a click-triggered lightbox so `:focus-visible` filters never stick to the artwork.

## 58. A HIDDEN window holds every CSS animation at 0, and synthetic pointer events can't be captured

- **Trigger:** a headless check reads an animated value (`getComputedStyle(el).boxShadow`, a custom property driven by
  `@keyframes`) twice and gets the same 0% value, or a drag test with `dispatchEvent(new PointerEvent(...))` does nothing.
- **Discrimination:** the window was behind other apps: `document.visibilityState === "hidden"` and the animation
  reports `playState: "running"` with `currentTime: 0`, so it is frozen, not broken (WebKit throttles occluded pages;
  the same window animated normally once in front). For drags, `setPointerCapture(e.pointerId)` throws
  `InvalidPointerId` for a synthetic event, which aborts a handler that called it first.
- **Action:**
  1. Read `document.visibilityState` before trusting an animated value. To check keyframes in a hidden window, seek:
     `const a = el.getAnimations()[0]; a.currentTime = <ms>; getComputedStyle(el)...`, then `a.play()` to restore.
     Snapshot the peak by pausing at that time (`a.pause()`) before `takeSnapshot`.
  2. Wrap capture in `try { el.setPointerCapture(e.pointerId); } catch (_) {}` so the real drag still captures and a
     dispatched test drag still runs; dispatch pointerdown / pointermove / pointerup on the element itself.
  3. Evals sent through `curl -d "{...}"` break on JS regex escapes like `\(` (invalid JSON); send them from a
     Python script with `json.dumps` instead.

## 44. An off-screen probe page has its timers held to ~1 s: test waits are a message-channel yield

*Trigger:* a headless click test (an off-screen WKWebView probe) suddenly
times out or prints nothing, after the test grew by a few steps. *Discrimination:* the page is not slow: an off-screen
window's page gets its `setTimeout`s throttled, so after a couple of seconds every `await sleep(30)` takes about a
second (measured: 13 sections at 0.2 s, 2 s, 10 s, 22 s, 31 s). `WKPreferences.inactiveSchedulingPolicy = .none` did
NOT help. *Action:* when the page renders synchronously, shadow the probe's `sleep` in the test with
`const sleep = () => new Promise((r) => { const c = new MessageChannel(); c.port1.onmessage = () => r(); c.port2.postMessage(0); });`
(31 s → 1.2 s), and let the probe take its time limit from an env var (`WEBPROBE_TIMEOUT`).

## 45. A tooltip helper that sets `position: relative` moves absolutely placed controls

*Trigger:* a `[data-tip] { position: relative }` (or any attribute-selector helper) on a page where some tooltip owner
is ALSO absolutely positioned (the ✕ inside a search field). *Discrimination:* equal specificity, so the later rule wins
and the ✕ joins the flex row: it sat on the field's border, and a hit test still passed. *Action:*
`:where([data-tip]) { position: relative; }` (zero specificity), and a guard that the ✕'s box sits INSIDE the field's.

## 59. An effect that edits a frame document and then calls `measure()` re-runs on every measure

*Trigger:* a Solid `createEffect` marks or rewrites nodes inside an email / preview frame and then re-fits the frame
(`measure()`, any height or scale write); the marks keep being rebuilt, or state set on them (a "current" attribute)
disappears a moment later. *Discrimination:* the effect only meant to depend on its input (the search terms), but
`measure` READS signals (height, scale), and an effect tracks every signal read inside it, so each measure re-runs the
effect, which re-marks, which measures again. *Action:* call the measure untracked, `untrack(measure)`, so the effect
depends only on what it should.

## 60. Anything you hang on nodes INSIDE an iframe dies when the frame reloads

*Trigger:* a "current item" / selection / flag stored as an attribute on elements inside a frame's document (the
current search match) is lost after the conversation finishes loading. *Discrimination:* the frame re-loads when its
full body arrives (hydration), every node in it is new, and the marks are re-made; the attribute was the only record.
*Action:* keep the state in the PARENT as a position or id (`hitIndex`), and re-apply the attribute after every
re-mark (`recountHits`). Never read the state back from the DOM attribute.

## 61. Two async "do X once the page settles" jobs race; the late one undoes the user

*Trigger:* an effect starts a polling job on open ("scroll to the first match once the frames load") and the user's
first keypress is undone a second later. *Discrimination:* opening a search result fires the effect twice (the query
changed AND the selection changed); both jobs poll, and the slower one scrolls back after the user stepped. Seen only
in a log of every scroll with timestamps. *Action:* a serial number per job (only the newest may act) plus a counter
the user's own action bumps (a job started before the user acted gives up).

## 62. Headless probe hygiene for a browser-mock build under real WebKit

*Trigger:* writing or extending a Swift probe that drives the mock app in an off-screen WKWebView. *Discrimination:*
each item below has cost a failed run; most failures were the probe, not the app, so read these before
blaming the code. *Action:*
- Steps as `web.callAsyncJavaScript(body, in: .page)` so a step can `await`; wrap each body in
  `try { … } catch (e) { return 'threw: ' + e.message }` so an exception names itself instead of "A JavaScript
  exception occurred".
- `cfg.websiteDataStore = .nonPersistent()`: the mock saves its prefs in localStorage, and a template made by the last
  run changed the next run's results.
- Stub `navigator.clipboard.writeText` before any copy: a mock that "writes the clipboard" writes the Mac's REAL
  pasteboard (a probe can overwrite your clipboard this way; `CFPasteboardSetExpirationDate` in the output is the tell).
- In a Swift `"""` string a JS regex escape needs two backslashes (`/\\s+/`, `/\\?/`); one is a Swift compile error
  that also shows as "cannot convert String to (String, String)" on the step array.
- A bare key in a Gmail-style keymap waits the sequence beat (~350 ms) before it runs: sleep past it.
- Record which binding a key ran (e.g. `window.__lastAction` in the dispatcher) before suspecting the handler.
- Close what earlier steps opened (Settings ate a probe's `n` as typing in its search box), and check the fixture's
  facts (a template WITH a fill-in rightly took the cursor the step expected on another line).
- Write the composer's state through its editor (`.ProseMirror`'s `.editor`), not synthetic `beforeinput` events,
  which WebKit doesn't route for synthetic input.

## 63. Key handlers key off STATE, never `e.target`; element handlers skip keys the app already handled

- **Trigger:** a key works "only sometimes" (Esc leaves a filter on some presses and not others), or one key press does
  two things (Space played / paused AND toggled the seat that had focus).
- **Discrimination:** two separate causes, both measured. (1) The Escape branch tested `e.target.id === "pagefilter-input"`:
  after a click focused something else, or when the native shell's key monitor forwarded Esc through a page hook that
  calls the handler with `new KeyboardEvent("keydown", { key: "Escape" })` WITHOUT dispatching it (`e.target` is null),
  the branch never ran. (2) A document-level capture handler ran the app shortcut and `preventDefault()`ed it, and the
  focused element's own `keydown` (`Enter || " "` → activate) still fired, because preventDefault does not stop
  propagation.
- **Action:**
  1. Decide from app state: "is the filter engaged" = has text, OR its field has focus, OR it was opened by its
     shortcut. Clear it, blur it, close it from any focus and from the synthetic forward path.
  2. Element handlers return early on `e.defaultPrevented`, and a list item answers Enter only, never Space when Space
     is a global shortcut.
  3. Test both from a running instance with stubbed writes: focus an item, dispatch the key on IT, and call the native
     forward hook directly, then read the state.

## 64. What makes the WebKit graphics process ("<App> Graphics and Media") busy, and how to measure it

- **Trigger:** Activity Monitor shows "<App> Graphics and Media" (`com.apple.WebKit.GPU`) high while the app sits idle,
  the Mac runs hot or freezes, or you add any looping effect to a WKWebView page.
- **Discrimination:** measured on a large (5K) window with the page idle:
  - Anything that REPAINTS every frame costs about the whole window, whatever the element's size: six small cards
    breathing via `box-shadow` 54%, a conic border turned through `@property --angle` 26%.
  - Moving HUGE layers is just as costly even with `transform` only: four 60vw blurred aurora blobs drifting by
    `transform` cost 58%; removing their `filter: blur` changed nothing (59%); ONE blob alone cost 60%. Small
    opacity / transform loops (a 220px art halo, a pulsing dot, a comet sliding along a 2.5px strip) cost about 0.
  - A tiny canvas (1/8 to 1/32 of the window) redrawn at 10-12 fps and stretched by CSS costs 3-4%; at 30 fps 7%.
  - `ctx.filter = "blur(…)"` on a 2D canvas is IGNORED by this WebKit (the property sticks, nothing blurs): shrink an
    image to a small swatch and stretch it instead.
  - A covered window reports `document.visibilityState === "hidden"` and WebKit stops drawing, but
    `inactiveSchedulingPolicy = .none` keeps every JS timer and canvas loop running: a hidden page still burns CPU.
  - Canvas 2D draws are rasterized in the GPU process, so a `display: none` canvas whose loop keeps drawing still costs.
- **Action:**
  1. Find the app's own helpers by RESPONSIBLE pid (what Activity Monitor uses), never by "newest WebKit process":
     `ctypes.CDLL("/usr/lib/libSystem.B.dylib").responsibility_get_pid_responsible_for_pid(pid)` == the app's pid;
     sample with `top -l 4 -s 3 -pid … -stats pid,cpu` and drop the first sample.
  2. Find the culprit by switching ONE effect off at a time through an injected `<style>` (`display: none`, then
     `animation: none`), 10 s per state, then put it back. Alternate A/B samples: media playback, content changes or the user's own
     clicks swing the baseline by 5-10 points between runs.
  3. Fix by moving motion onto small transform / opacity layers or a tiny low-fps canvas, pause everything while unseen (the shell reports
     `NSWindow.didChangeOcclusionStateNotification` to the page), and add a build check that fails on new per-frame repaint effects.

## 65. A bar pinned with `position: sticky; bottom` inside a PADDED scroller

*Trigger:* you dock a bar (actions, a footer) to the bottom of a scrolling pane with `position: sticky; bottom: …`, and
the scroller has bottom padding (`.prefs-body { padding: 4px 34px 34px }`). *Discrimination:* negative offsets
and margins (`bottom: -34px; margin-bottom: -34px`) to cancel the padding left the bar 8–9 px above the pane's edge in
real WebKit, and the gap looks like a layout bug. *Action:* while the bar is there the scroller drops its bottom padding
(`.prefs-body:has(.dock) { padding-bottom: 0 }`, WebKit supports `:has`), the bar sticks at `bottom: 0`,
bleeds sideways with `margin: 0 -<side padding>`, and gets a `z-index` over cards that animate (`transform` makes them
stacking contexts). Prove it with the pane made short (`max-height: 420px` on the scroller) so the form really scrolls:
the bar's bottom equals the scroller's bottom at `scrollTop = 0` AND at the end. A pane that happens to fit proves
nothing.

## 66. An off-screen probe's ENTRANCE animation never ends: freeze animations before measuring geometry

*Trigger:* a probe compares two boxes (`getBoundingClientRect`) inside a panel that opens with a transform animation
(`translateY(14px) scale(0.99)` → none). *Discrimination:* the same family as item 58: an off-screen page holds the
animation at its first frame, so the panel stays scaled; `clientHeight` (885) disagreed with the rect's height (876)
and every comparison was off by about 1%. It read as a real 9 px gap. *Action:* before measuring, inject
`*,*::before,*::after{animation:none!important;transition:none!important}` (the same style the snapshot helpers use),
wait a beat, then measure. When `clientHeight` and the rect height differ, a transform is in play: freeze first.

**The probe's own frame must be bigger than what it measures.** *Trigger:*
an `elementFromPoint` hit test near the content's far edge returns `null` or the wrong element in a probe, while the
same control works in the app. *Discrimination:* the app's window sizes itself to the content; the probe's
`WKWebView(frame:)` does not, and a point outside that frame hits nothing (a 996 px card in a 900 px probe read as
"right edge missing"). The rise-in animation left the same card 6 px low, so top-edge hits missed too. *Action:* make
the probe frame wider and taller than the largest content it loads, and set `style.animation = 'none'` on the
animated element before any hit test.

## 67. A native TEXT drag inside the page reaches Tauri's drag-drop layer as a file drag with NO paths

*Trigger:* a "drop files here" zone driven by `getCurrentWebview().onDragDropEvent`, on a page with selectable text or
chips. *Discrimination:* pressing on selected text (an address chip) and moving started WebKit's own text drag; Tauri
reported it as `enter` / `over` with `paths: []`, and a zone lit on any `enter` said "Drop to Attach" for a chip
being moved between fields. *Action:* (1) the zone wakes only when `enter` carried `paths.length > 0` (remember it for
the `over` events that follow); (2) in-page drags are pointer events end to end (`preventDefault` on the element's
`mousedown`, window `mousemove` / `mouseup`, a ghost with `pointer-events: none`, `elementFromPoint` for the target),
plus `-webkit-user-drag: none` and `onDragStart = preventDefault` on the element.

## 68. Two capture listeners on `window`: the one ADDED FIRST runs first, and only `stopImmediatePropagation` stops the other

*Trigger:* a small surface that must own Esc / Enter / ↑↓ while it is open (a type-ahead list, an emoji picker)
in an app whose keymap already listens on `window` in the capture phase. *Discrimination:* `stopPropagation()` in a
window capture listener does NOT stop another listener on the same window target; the keymap still ran and Esc
closed the composer under the list. Adding the surface's listener AFTER the keymap's makes it run second, too late.
*Action:* install the surface's listener BEFORE the keymap installs its own, and call `preventDefault()` +
`stopImmediatePropagation()` for the keys it handles. A listener that can only be added later (a drag that starts
mid-session) cannot win this way: route its Esc through the app's own escape hook instead.

## 69. ⌘Z inside a `contenteditable` island does nothing, and two quick selection writes scroll the page

- **Trigger:** an app keeps its own document history (CodeMirror, ProseMirror, an app undo stack) and also has
  `contenteditable` islands inside the page (table cells, inline fields in a widget); or the page sets the DOM
  selection from script while focus is inside an editable.
- **Discrimination:** (1) WebKit gives each editing host its own undo stack. Content set by script (`textContent`,
  `innerHTML`) records nothing, so ⌘Z inside the island is WebKit's no-op, and the app's history never hears the key
  unless the island forwards it. Edit ▸ Undo through the responder chain hits the same no-op. (2) Setting the selection
  inside a FOCUSED editable makes WebKit scroll it into view synchronously. Write a stale position and then the right
  one in two steps, and the page has already jumped to the first (measured: over 1,000px); Chrome often hides
  this.
- **Action:** (1) the island's keydown takes ⌘Z / ⇧⌘Z (and ⌘Y), commits its own pending text, then runs the
  document's undo / redo; never rely on the native undo inside a script-filled editable. (2) Compute the final
  selection first and apply it in ONE update (CodeMirror: `view.dispatch([tr, fixTr])`).

## 70. A synthetic key sent through `window.sendEvent` can land in the WRONG web view or in the app's MENU; `webView.appearance` is the per-view `prefers-color-scheme` switch

- **Trigger:** native code sends a key chord into a web page to drive the site (a mode toggle, a shortcut) with
  `NSEvent.keyEvent` + `window.sendEvent`, or you need one web view dark and another light.
- **Discrimination:** two silent failures, both seen in the logs of a tabbed web-wrapper app. (1) A HIDDEN `WKWebView` (a background
  tab) refuses first responder, so the event goes to whatever view already has focus: the visible tab got two toggles
  and the background tab none. (2) When the page does not consume the key (still loading, a modal's input focused),
  WebKit hands the unhandled event back up the responder chain to the main menu's key equivalents: an app menu item on
  the same chord fired 14 ms later. A guard in the local key monitor does not cover the menu path.
- **Action:** prefer a state the view owns: `webView.appearance = NSAppearance(named: .darkAqua)` makes THAT view's
  `prefers-color-scheme` dark (live, hidden views included), so a site on its "follow system" setting follows per view.
  When a key really must reach the page, dispatch a `KeyboardEvent` inside it with `evaluateJavaScript` (it cannot
  leave the page), and send a native event only to the visible web view of the key window. Any app menu item that
  shares the chord checks the same "my own keystroke, last 1 s" flag as the key monitor.

