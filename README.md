# Arbitrary Code Execution in Obsidian Templater via Autocomplete (No Template Execution Required)

**Affected software:** [Templater](https://github.com/SilentVoid13/Templater) — a template plugin for Obsidian
**Affected versions:** all versions before `2.23.1` (last affected: `2.23.0`)
**Fixed version:** `2.23.1` (released 2026-07-03)
**Vulnerability type:** Arbitrary Code Execution / Code Injection (CWE-94)
**Severity:** High — arbitrary code execution, but requires a shared/synced vault and a victim edit at the trigger location.
**CVE:** [CVE-XXXX-XXXXX — pending / to be assigned]
**Reported by:** jprodriguez33

---

## Summary

Templater's "User Scripts" feature lets a `.js` file in a configured folder be
called from templates, e.g. `tp.user.myscript()`. To do this, Templater loads
the file and runs it with `eval()`.

That same `eval()` call also gets triggered from a second, unrelated place:
Templater's editor autocomplete. If the cursor is positioned right after text
like `tp.user.myscript.` in **any** open note, Obsidian's autocomplete tries to
show a helpful suggestion of what functions the script has. To generate that
suggestion, Templater loads and **fully executes** the script file — even though
no template ran and no suggestion was ever accepted.

In short: a script's code can run just because someone edits text next to a
string that looks like `tp.user.something.`

## Impact

Execution reaches Node.js with `require` available, so it is real code
execution, filesystem access, process access, and system commands are all
reachable. On a desktop Obsidian install this means an attacker-controlled
script can run arbitrary code in the victim's user context.

## Exploitability

Through a shared or synced Obsidian vault, an attacker can plant:

1. A `.js` file containing harmful code in the configured user-scripts folder.
2. A settings file (`data.json`) pointing Templater at that folder.
3. A note containing text like `tp.user.evil.` somewhere in it.

None of this requires the victim to approve anything. Templater already shows
confirmation popups for a few settings it considers dangerous
(`trigger_on_file_creation`, `enable_startup_templates`,
`enable_system_commands`), but the user-scripts-folder setting is not one of
them — and it syncs automatically with the vault via `data.json`.

The one required user action is editing text at the trigger location.
Realistically, this can be engineered by disguising the trigger text as
something a victim would naturally edit — for example, a fill-in-the-blank field
in a template.

## What triggers execution (confirmed by testing)

| Action | Triggers execution |
| --- | --- |
| Opening the vault | No |
| Opening/viewing the note (Reading mode) | No |
| Clicking elsewhere in the note | No |
| Arrow-keying past the trigger text | No |
| Editing text elsewhere in the same note | No |
| **Editing (typing or deleting a character) right after `tp.user.<name>.`** | **Yes** |

## Technical detail

Two code paths both reach the same `eval()`:

```
Normal use (template execution):
  Templater.parse_template() -> ... -> UserScriptFunctions.generate_object()
    -> load_user_script_function()   [eval runs here]

The bug (autocomplete):
  Autocomplete.onTrigger()   [fires on cursor/edit events matching the regex]
    -> Autocomplete.getSuggestions()
      -> TpDocumentation.get_user_script_object_documentation()
        -> load_user_script_function()   [same eval, same effect]
```

`load_user_script_function()` (in `UserScriptFunctions.ts`) does roughly:

```js
const wrapping_fn = window.eval(
    "(function anonymous(require, module, exports){" + file_content + "\n})"
);
wrapping_fn(req, mod, exp);
```

This runs the entire top-level body of the script file, and because `require`
is in scope, it is full Node.js code execution.

**Relevant files:**
- `src/UserScriptFunctions.ts` — `load_user_script_function`, `generate_user_script_functions`
- `src/TpDocumentation.ts` — `get_user_script_object_documentation`
- `src/Autocomplete.ts` — `onTrigger`, `getSuggestions`

## Proof of concept

Place the following file in the vault's configured user-scripts folder as
`_scripts/poc.js`:

```js
require('fs').writeFileSync(
  require('os').homedir() + '/templater-poc-fired.txt',
  'RCE confirmed: ' + new Date()
);
```

**Steps to reproduce:**

1. Set Templater's user-scripts folder to `_scripts`, containing the file above
   as `poc.js`.
2. In any note, type `tp.user.evil.`
3. Click right after the `.` and type or delete one character.
4. Check your home directory — `templater-poc-fired.txt` will be present,
   confirming code execution.

The PoC deliberately only writes a marker file; it demonstrates execution
without carrying a harmful payload.

## Remediation

Fixed in Templater `2.23.1`. Users should update to `2.23.1` or later.

The underlying issue was addressed by no longer fully executing user-script
files solely to generate autocomplete suggestions. Two mitigations were
suggested during disclosure, either of which resolves the issue:

1. Apply the same confirmation-popup treatment to the user-scripts-folder
   setting that Templater already applies to its other dangerous settings, and
   ensure it cannot be silently set from a synced `data.json` without
   confirmation.
2. Stop executing script files to build autocomplete suggestions — parse the
   file to find its exported names without running it, instead of `eval()`-ing
   the whole thing.

## Disclosure timeline

- **2026-07-02** — Reported privately to Zachatoo (the current maintainer) by
  email; he promptly fixed the issue and released it. 
- **2026-07-03** — Fix released in Templater `2.23.1` (previous release `2.23.0`, 2026-06-23).
- **2026-07-04** — Public advisory published.

## Credit

Discovered and reported by jprodriguez33.

## References

- Fix commit: https://github.com/SilentVoid13/Templater/commit/9a0806047af3a7288a8b2acb4869e37f0af6ea29
- Fixed release: https://github.com/SilentVoid13/Templater/releases/tag/2.23.1
- Vulnerable code (last affected release): https://github.com/SilentVoid13/Templater/blob/2.23.0/src/UserScriptFunctions.ts
