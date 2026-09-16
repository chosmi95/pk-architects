# pk-architects

Single-file static site: `index.html` holds the markup, CSS, JS and every image
(as base64 data URIs). No build step, no dependencies, no test suite. Open the
file in a browser and it runs.

Push directly to `main` — the owner asked for this; do not open a branch or a PR
unless asked.

## The artifact is the source

The owner edits the page as an artifact on claude.ai and hands it over to be
pushed:

    https://claude.ai/artifact/MKZQdMQ2XbCJe8kDGyrq15

This session watches that artifact, so a republish from another conversation
wakes it. To sync:

1. `Artifact` tool, `action: "read"`, that `url` — it saves the full HTML locally
   and prints the path.
2. Copy that file over `index.html`.
3. Re-apply the invariants below. **They do not survive a round trip** — the
   artifact is regenerated in a chat that does not know about them, and both
   have already been lost twice.
4. Verify in the browser (see below), then commit and push.
5. Republish the corrected file back to the same artifact `url`, so the next
   round starts from a version that already has the fixes.

## Invariants — check these on every sync

**1. The theme is forced light.** `<html>` carries `data-theme="light"`, `:root`
declares `color-scheme:light`, and there are no `@media (prefers-color-scheme:
dark)` or `:root[data-theme="dark"]` blocks. Without this the page goes dark for
anyone whose OS is in dark mode.

    grep -c "prefers-color-scheme: dark" index.html   # must be 0
    sed -n '2p' index.html                            # must carry data-theme="light"

**2. The mobile-nav block comes before the routing block.** `router()` runs
during init and calls `closeMobileNav()`, which reads the `nav` and `toggle`
variables. If those are still assigned below the routing block, the call throws
`TypeError: Cannot read properties of undefined`, the rest of the IIFE never
runs, and *nothing renders on any route* while the mobile menu button does
nothing.

    grep -n "mobile nav toggle\|---- routing" index.html   # nav must come first

## Verifying a change

Chromium is at `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`. Playwright
itself is not installed; drive it with flags.

Append a probe script to a **copy** of the page in the scratchpad, then read the
result out of `document.title`:

    chrome --headless --no-sandbox --disable-gpu --window-size=500,900 \
      --virtual-time-budget=6000 --dump-dom "file://<copy>.html#/bio"

Check that a section gets `is-active`, the project grid and team cards are
populated, and `getComputedStyle(document.body).backgroundColor` is
`rgb(250, 250, 248)`.

Two headless traps worth knowing:

- CSS transitions do not advance under `--virtual-time-budget`, so a faded-in
  element reads `opacity: 0` forever. Set `el.style.transition = 'none'` before
  measuring.
- Chrome clamps `--window-size` to a minimum width around 485px, so a `390`
  request silently becomes 485 and screenshots crop the right side. Use 500+ and
  read `document.documentElement.clientWidth` before concluding anything about
  layout at a given width.

## Content still pending

Team photos: `TEAM` entries carry `photo: null`, which renders a placeholder
tile. Swap in a `data:image/...` string per person when the owner supplies them.
