# GTM sessionStorage Read & Write

Read and write `sessionStorage` from Google Tag Manager using Sandboxed JavaScript Custom
Templates — with only **one** inline script (and therefore only one CSP SHA-256 hash) in the
entire container.

> Full write-up with screenshots-free deep dive: *sessionStorage in GTM via Sandboxed JavaScript
> — Read & Write With Just One CSP Hash* (blog post link in repo description).

## Why this exists

A number of published community GTM templates manipulate `localStorage` directly, since it's a
natural fit for a "utility" template. `sessionStorage` exposes the same API shape —
`getItem`, `setItem`, `removeItem`, `clear`, `key`, `length` — but a Sandboxed JavaScript
template can't reach either storage API on its own. Templates run inside GTM's locked-down
sandbox with a fixed set of permitted built-in functions; there is no `require('sessionStorage')`.

The usual workaround is a Custom HTML tag with an inline `<script>` block. That works, but on any
site enforcing CSP with SHA-256 source hashes, every inline script needs its own hash — and the
hash has to be recomputed and redeployed every time the script's text changes by even one byte.

This recipe splits the problem in two:

1. **One** Custom HTML tag (`sessionStorage Listener`) defines a small window-scoped helper,
   `window.gtmSessionStorage`, wrapping the native `sessionStorage` API. This is the only tag
   with an inline script, and therefore the only artifact that ever needs a CSP hash.
2. **Two** Sandboxed JavaScript Custom Templates — a variable (`sessionStorage Read`) and a tag
   (`sessionStorage Write`) — call that helper via `callInWindow`, gated by the `access_globals`
   permission. Template code runs inside GTM's own sandbox, is not injected as an inline script,
   and needs no CSP hash at all — including every future edit to the read/write logic.

## What's in the container

File: [`gtm-session-storage-read-write.json`](./gtm-session-storage-read-write.json)

| Component | Name | Type | Notes |
|---|---|---|---|
| Folder | `Process sessionStorage` | Folder | Holds everything below |
| Tag | `sessionStorage Listener` | Custom HTML | Priority 99, fires early, Once Per Event. Defines `window.gtmSessionStorage`. **The only inline script in the container.** |
| Tag | `sessionStorage Write` | Custom Template (`sessionStorage Write`) | Fires on the `sessionStorage Write` trigger. Sets `{{sessionStorage Name}}` to `{{sessionStorage Value}}`. |
| Trigger | `sessionStorage Write` | Initialization | Fires only when `{{sessionStorage Read}}` matches `^(undefined\|null\|0\|false\|NaN\|)$` — i.e. the key isn't already set. |
| Variable | `sessionStorage Read` | Custom Template (`sessionStorage Read`) | Action: `getItem`, Key: `{{sessionStorage Name}}` |
| Variable | `sessionStorage Name` | Constant | `sessionTest` |
| Variable | `sessionStorage Value` | Constant | `something-written-2-sessionStorage` |
| Custom Template | `sessionStorage Read` | MACRO (Variable) | Actions: Get Item, Get Key By Index, Get Length |
| Custom Template | `sessionStorage Write` | TAG | Actions: Set Item, Remove Item, Clear All |

## How the Listener tag works

```js
<script>
(function() {
  if (window.gtmSessionStorage) return; // avoid re-init on multiple tag fires

  window.gtmSessionStorage = {
    setItem: function(key, value) {
      try {
        sessionStorage.setItem(key, value);
        return true;
      } catch (e) {
        console.error('gtmSessionStorage.setItem failed:', e);
        return false;
      }
    },
    getItem: function(key) {
      try {
        return sessionStorage.getItem(key);
      } catch (e) {
        console.error('gtmSessionStorage.getItem failed:', e);
        return null;
      }
    },
    removeItem: function(key) {
      try {
        sessionStorage.removeItem(key);
        return true;
      } catch (e) {
        console.error('gtmSessionStorage.removeItem failed:', e);
        return false;
      }
    },
    clear: function() {
      try {
        sessionStorage.clear();
        return true;
      } catch (e) {
        console.error('gtmSessionStorage.clear failed:', e);
        return false;
      }
    },
    key: function(index) {
      try {
        return sessionStorage.key(index);
      } catch (e) {
        console.error('gtmSessionStorage.key failed:', e);
        return null;
      }
    },
    length: function() {
      // exposed as a function, not a property — callInWindow calls functions
      try {
        return sessionStorage.length;
      } catch (e) {
        console.error('gtmSessionStorage.length failed:', e);
        return 0;
      }
    }
  };
})();
</script>
```

Every native call is wrapped in try/catch so a browser that blocks storage access (private
browsing edge cases, storage partitioning, etc.) fails safely instead of throwing.

**This is the only script that needs a CSP hash.** Fire it once, early, and every Read/Write
template call after it talks to `window.gtmSessionStorage` through GTM's own sandbox.

## sessionStorage Read (Variable template)

| Action | Parameter | Calls | Returns |
|---|---|---|---|
| Get Item | Key (text) | `window.gtmSessionStorage.getItem(key)` | Stored string, or `null` |
| Get Key By Index | Index (non-negative number) | `window.gtmSessionStorage.key(index)` | Key name at that index, or `null` |
| Get Length | — | `window.gtmSessionStorage.length()` | Number of stored items |

```js
const callInWindow = require('callInWindow');
const queryPermission = require('queryPermission');
const makeString = require('makeString');
const makeNumber = require('makeNumber');
const logToConsole = require('logToConsole');

const action = data.action;

if (action === 'getItem') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.getItem')) {
    return callInWindow('gtmSessionStorage.getItem', makeString(data.key));
  }
  logToConsole('Session Storage: Read - missing execute permission for gtmSessionStorage.getItem');
  return undefined;
}

if (action === 'key') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.key')) {
    return callInWindow('gtmSessionStorage.key', makeNumber(data.index));
  }
  logToConsole('Session Storage: Read - missing execute permission for gtmSessionStorage.key');
  return undefined;
}

if (action === 'length') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.length')) {
    return callInWindow('gtmSessionStorage.length');
  }
  logToConsole('Session Storage: Read - missing execute permission for gtmSessionStorage.length');
  return undefined;
}

logToConsole('Session Storage: Read - unknown action: ' + action);
return undefined;
```

## sessionStorage Write (Tag template)

| Action | Parameters | Calls |
|---|---|---|
| Set Item | Key (text, required), Value (text) | `window.gtmSessionStorage.setItem(key, value)` |
| Remove Item | Key (text, required) | `window.gtmSessionStorage.removeItem(key)` |
| Clear All | — | `window.gtmSessionStorage.clear()` |

```js
const callInWindow = require('callInWindow');
const queryPermission = require('queryPermission');
const logToConsole = require('logToConsole');
const makeString = require('makeString');

const action = data.action;

if (action === 'setItem') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.setItem')) {
    callInWindow('gtmSessionStorage.setItem', makeString(data.key), makeString(data.value));
    data.gtmOnSuccess();
  } else {
    logToConsole('Session Storage: Write - missing execute permission for gtmSessionStorage.setItem');
    data.gtmOnFailure();
  }
} else if (action === 'removeItem') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.removeItem')) {
    callInWindow('gtmSessionStorage.removeItem', makeString(data.key));
    data.gtmOnSuccess();
  } else {
    logToConsole('Session Storage: Write - missing execute permission for gtmSessionStorage.removeItem');
    data.gtmOnFailure();
  }
} else if (action === 'clear') {
  if (queryPermission('access_globals', 'execute', 'gtmSessionStorage.clear')) {
    callInWindow('gtmSessionStorage.clear');
    data.gtmOnSuccess();
  } else {
    logToConsole('Session Storage: Write - missing execute permission for gtmSessionStorage.clear');
    data.gtmOnFailure();
  }
} else {
  logToConsole('Session Storage: Write - unknown action: ' + action);
  data.gtmOnFailure();
}
```

## Permission model

Both templates request `access_globals`, scoped to exactly six named window paths:
`gtmSessionStorage.getItem`, `.setItem`, `.removeItem`, `.clear`, `.key`, and `.length` — each
with `read`, `write`, and `execute` granted individually. The template never touches
`sessionStorage` directly; it calls a named function on `window` that the Listener tag already
defined, and GTM's permission system whitelists that exact function name at template-approval
time. Anyone reviewing the container can see precisely which six window paths the templates are
allowed to touch — nothing broader.

## Demo wiring included in the container

- `sessionStorage Read` checks for a key named `sessionTest` (from the `sessionStorage Name`
  constant).
- The `sessionStorage Write` trigger fires only when that variable currently resolves to an
  empty/falsy value — i.e. the key hasn't been set yet in this session.
- When the trigger fires, `sessionStorage Write` sets `sessionTest` to
  `something-written-2-sessionStorage` (the `sessionStorage Value` constant).
- On any later page in the same tab/session, `sessionStorage Read` resolves to that string, the
  trigger condition is no longer met, and the write tag does not fire again.

## Import instructions

1. Download [`gtm-session-storage-read-write.json`](./gtm-session-storage-read-write.json) from
   this repo (or the [Google Drive mirror](https://drive.google.com/file/d/1JXsNHQcouM7u1PzAKXcvPgIrjU9MaLfg/view?usp=drivesdk)).
2. In GTM, go to **Admin → Import Container**.
3. Select the JSON file, choose the workspace, and pick **Merge** (not Overwrite) if the
   container already has other tags.
4. If prompted about conflicting Custom Templates, choose **Rename** to keep both, or overwrite
   for a clean container.
5. Confirm the import — you should see the `Process sessionStorage` folder with the
   `sessionStorage Listener` and `sessionStorage Write` tags, the `sessionStorage Write` trigger,
   and the three variables.
6. Check **Templates** in the left nav for the two new Custom Templates: `sessionStorage Read`
   (Variable) and `sessionStorage Write` (Tag).
7. If your site enforces CSP with SHA-256 sources, compute the hash for the
   `sessionStorage Listener` tag's script block and add it to `script-src`. No other tag in this
   recipe requires a CSP change.
8. Open GTM Preview mode and load a page. Confirm `sessionStorage Listener` fires first, then
   `sessionStorage Write` fires on the first page of a new session.
9. In DevTools, run `sessionStorage.getItem('sessionTest')` and confirm it returns
   `something-written-2-sessionStorage`.
10. Reload the page — the write trigger should not fire again, since `sessionStorage Read` now
    resolves to a truthy value.
11. Publish the container version once everything checks out in Preview.

## Repo contents

```
.
├── gtm-session-storage-read-write.json   # GTM container export (import this)
└── README.md                              # This file
```

## License

Use, fork, and adapt freely. Attribution appreciated but not required.
