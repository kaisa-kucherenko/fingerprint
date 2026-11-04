# Browser fingerprint page

A static page that reads what the browser it is opened in reports about itself and saves it as one JSON dump.
It works in any browser on any device: desktop and mobile, Chrome, Firefox, Safari, Edge and others. When an API
is missing or throws, the dump records that, and the page keeps going.

Live page: https://kaisa-kucherenko.github.io/fingerprint/

## Taking a dump

1. Open the page in the browser you want to capture. It collects everything on load, which takes a few seconds.
2. Press **Save JSON**. The file downloads as `fingerprint_<timestamp>_<type>_<os>_<browser><major>.json`.
   **Copy JSON** puts the same text on the clipboard.
3. Send the file as is.

Automation reads the same dump without saving a file. Once collection finishes, the element
`#fingerprint-json` holds it as one line of JSON:

```js
document.getElementById('fingerprint-json').textContent
```

`#fingerprint-json` stays empty until every check has finished. A script should wait for it to be non-empty.
A dump is about 300–350 KB.

## Dump format

The dump is one flat JSON object: key → value.

- **Base keys** are the original sections (`navigator.*`, `screen.*`, `chrome.*`, the iframe and worker checks and so
  on). Their names and order do not change. Some of them hold the page's own verdict, not a browser value:
  a string ending in `→ CONSISTENT` or starting with `PASS:`, or the object `{"_fail":true}` / `{"_warn":true}`
  for a failed or blocked check (the verdict text is not kept). A thrown base check is `"ERROR: <message>"`. The
  page itself also adds `_timestamp`, `_url` and `_userAgent`, always the last three keys.
- **`ks.*`** keys:
  - the keys of `window.chrome` and the members of `chrome.app`;
  - `"callPhantom" in window` and `"Cypress" in window`;
  - `Date()` and `new Date(0).toString()`;
  - legacy `performance.timing`;
  - `Function.prototype.toString` called on `null` and `{}`;
  - the error text of six short code snippets;
  - `Object.getOwnPropertyNames(X.prototype)` for every own property `X` of `window` that is a function, has a
    `prototype` and starts with a capital letter;
  - the `typeof` of every other own property of `window`, in one key.

  These lists, and `inj.Object.keys(window).length`, include the page's own functions. Compare only dumps taken
  from the same version of the page.
- **`inj.*`** keys:
  - `Object.getOwnPropertyDescriptor(owner, name)` for a fixed list of getters and methods. For a function the
    stored value has its `name`, `length` and own keys, and its `toString` called both here and through an
    iframe's `Function.prototype.toString`. For a getter it also has what the getter returns when called on `{}`.
    A non-function data value is stored as is, an object value as its `typeofValue`;
  - the plugin and MIME type lists;
  - `canPlayType` answers of a new `<video>` element, and `matchMedia` answers;
  - parameters and extensions of a new canvas's `webgl` and `webgl2` context. The unmasked renderer and vendor keys
    are present only when `WEBGL_debug_renderer_info` is available;
  - `Intl` options and locale formatting;
  - viewport sizes;
  - the navigation timing entry.

Every `ks.*` and `inj.*` key is the expression that produced the value. For example,
`ks.Object.keys(window.chrome)` holds exactly what `Object.keys(window.chrome)` returned. These keys hold only what
the browser returned. The values are encoded like this:

| browser returned | stored as |
|---|---|
| string, finite number, boolean, `null` | as is |
| array or object | its `JSON.stringify` text (so inside it `undefined` and functions drop out and `NaN` becomes `null`); `String(value)` when that fails |
| `undefined`, `NaN`, `Infinity`, a function | `String(value)`, e.g. `"undefined"` |
| an exception | `"ERROR: <name>: <message>"`, with the message in the engine's own words |
| an exception in a `ks.error: <snippet>` or `ks.Function.prototype.toString.call(…)` key | `"<name>: <message>"`, without the prefix: the text itself is the value |

In `inj.*`, a key that is just an owner (`inj.NetworkInformation.prototype`) means the page could not reach that
owner. Its value is `"undefined"` when the owner does not exist, or `"ERROR: …"` when reaching it threw. In that case
there are no descriptor keys for the owner.

The page adds three fixed strings of its own:

- `"no throw"` — an error snippet that did not throw;
- `"no answer within 1000 ms"` — a promise or callback (`chrome.app.installState`, `getHighEntropyValues`) that
  did not settle in time;
- `"undefined"` — a descriptor's `get` or `set` that does not exist.

## File name

The file name starts with `fingerprint_<Date.now()>`, as it always has. The page then adds
`_<type>_<os>_<browser><major>`, taken from three base keys. The dump does not contain this tag.

| part | from | rule |
|---|---|---|
| os | `navigator.userAgent`, `navigator.platform`, `navigator.maxTouchPoints` | iPhone / iPad / iPod in the UA, or platform `MacIntel` with more than 1 touch point (iPadOS requesting desktop sites) → `ios`; then `android`, `windows`, `chromeos`, `macos`, `linux` by UA, else `unknown` |
| type | the same keys | iOS: iPhone / iPod → `mobile`, else `tablet`. Android: `Mobile` in the UA → `mobile`, else `tablet`. X11 Linux with touch points (a phone in desktop mode; a Linux touchscreen laptop gets the same tag) → `pcmode`. Anything else → `desktop` |
| browser | `navigator.userAgent` | the first match of Edge, Opera (`OPR` / `OPT`), Samsung Internet, Yandex, Firefox, Chrome (`CriOS` on iOS), Safari (`Version/<n> … Safari`); the number is the major version from the same token. No match → `unknown` without a number (e.g. an in-app browser on iOS) |

The file name is a label for people. Anything that needs certainty should read the raw keys inside the dump.
