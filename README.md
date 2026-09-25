# qutebrowser-hint-caret

[中文说明](README.zh-CN.md)

Place the caret at the start of a paragraph picked via hints — a userscript for
[qutebrowser](https://qutebrowser.org/), no patching involved. Tridactyl-like
caret placement: pick where you want to start, then select from there.

    press v  →  type the hint label  →  the caret lands at the start of that
    paragraph, with caret selection mode already active, so hjkl extends the
    selection right away

## Why

Selecting a piece of text with the mouse is a context switch; caret mode alone
starts at the top of the page and makes you walk to wherever you were looking.
This userscript closes that gap: hint the paragraph you are reading, and your
caret is there.

The interaction is not new — it comes straight from vim-like browsers:

* **Tridactyl** (Firefox) does this in its caret mode, which is what this script
  set out to reproduce in qutebrowser: pick the text block you want to start
  from, and the caret is placed there,
* **VimFx** (Firefox, now unmaintained) implements the same idea — "Every element
  with non-empty `TextNodes` as direct children get a hint, and activating a
  hint places the caret at the beginning of that element" (as noted in
  [qutebrowser#1453](https://github.com/qutebrowser/qutebrowser/issues/1453)),
* [qutebrowser#1453](https://github.com/qutebrowser/qutebrowser/issues/1453)
  ("Make caret mode more efficient by adding easymotion mode", open since 2016)
  asks for the same kind of mouseless caret navigation,
* [qutebrowser#5035](https://github.com/qutebrowser/qutebrowser/issues/5035)
  ("Hints + paragraphs = mouseless copy action") asks for hints to work on
  paragraphs.

This script is a small, working slice of that: hints select a paragraph, and
the caret starts at its beginning.

## Install

```sh
mkdir -p ~/.local/share/qutebrowser/userscripts
cp caret-anchor ~/.local/share/qutebrowser/userscripts/
chmod +x ~/.local/share/qutebrowser/userscripts/caret-anchor
```

Make sure `~/.local/share/qutebrowser/userscripts` is on your
`userscripts` path if you changed it from the default.

## Configure

Add to `config.py`:

```python
c.hints.selectors['para'] = ['p', 'li', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                             'blockquote', 'dd', 'dt', 'pre', 'td', 'th',
                             'figcaption']
config.bind('v', 'hint para userscript caret-anchor')
```

Adjust the selector list to taste — it decides what counts as a "paragraph".

## How it works

A qutebrowser userscript receives the picked element through the `QUTE_*`
environment variables and can send commands back through `QUTE_FIFO`. This
script:

1. takes the element's tag name out of `QUTE_SELECTED_HTML`, and passes both the
   tag and the picked text to the page **base64-encoded** — page text can
   contain newlines, quotes or `$`, and inlining it into the command string
   would break qutebrowser's command parsing (the text would be executed as
   commands of its own),
2. re-finds that element by searching `document`, same-origin iframes and every
   *open* shadow root (the same container list qutebrowser's own hinting uses),
   preferring the innermost element whose text contains the picked text,
3. creates a **non-empty** selection whose *focus* end sits at the start of the
   paragraph (one character, selected backwards),
4. writes commands to `QUTE_FIFO`: the `jseval` that sets the selection,
   `mode-enter caret`, a `jseval` that collapses the selection, and
   `selection-toggle` to return to "move only" mode.

Step 3 is the whole trick. `caret.js`'s `setInitialCursor()` runs

```js
const len = window.getSelection().toString().length;
if (len === 0) positionCaret();          // caret jumps to the top of the page
...
if (len > 0) selectionState = NORMAL;    // selection mode is on
```

so a non-empty selection both keeps the caret where we put it *and* switches
selection mode on — no patch to qutebrowser required. A collapsed selection
(`len === 0`) would be treated as "no selection" and the caret would jump back
to the top of the document.

`Range` objects have no direction, so `createRange()` + `setStart(t, 1)` +
`setEnd(t, 0)` is normalized into a collapsed range and silently fails.
`Selection.setBaseAndExtent()` is the only DOM API that expresses which end is
the focus, hence the backwards selection.

## Limitations

Verified on qutebrowser 3.7.0 / QtWebEngine 6.11.2 / Qt 6.11.2. Tested on
Arch Wiki, ChatGPT and Gemini. It also works on pages whose text lives inside an
*open* shadow root (e.g. Bilibili comment threads) — see the caveats below, the
caret may not be visible there.

* **Paragraphs are matched by their text content.** The element picked by hints
  is re-found by looking for an element whose text contains the picked text,
  preferring the shortest match (innermost element). Pages with several
  paragraphs of identical text can resolve to the wrong one.
* **Only elements listed in `c.hints.selectors['para']` can be picked.**
* **The script briefly selects one character** to satisfy caret.js (see *How it
  works*) and collapses it again right away, so no text stays highlighted.
* **Only *open* shadow roots and same-origin iframes are searched.** This
  mirrors what qutebrowser's own hinting does (`find_css()` in
  `javascript/webelem.js`). Text inside a *closed* shadow root cannot be reached
  from a userscript at all — that is a browser-level restriction, not a
  limitation of this script.
* **In pages with a strict CSP, the caret itself may be invisible.** qutebrowser
  draws the caret by inserting a `<style>` element into the page; pages whose
  `Content-Security-Policy` restricts `style-src` without `'unsafe-inline'`,
  a nonce or a hash will refuse it (`Refused to apply inline style ...`). The
  anchor, the selection and caret movement all still work — only the blinking
  caret is missing. Use the selection highlight to see where you are.
* **Inside a shadow root, visually reversing the selection (`o`) can be off.**
  The logical selection reverses correctly, but the caret's on-screen position
  may point somewhere else, because qutebrowser computes the caret coordinates
  from the shadow *host*. Use `hjkl` to extend/verify the selection instead.
* Behaviour relies on `caret.js`'s current `setInitialCursor()` logic (see
  above). If upstream changes how it detects an existing selection, this script
  needs updating.

## License

GPL-3.0-or-later, same as qutebrowser itself.
