# Apple Shortcuts Integration

Create, install, run, and verify Apple Shortcuts programmatically on macOS, including generating .wflow files with correct plist serialization. All patterns below are empirically verified on macOS 26 (Shortcuts 2605/5114).

## Running shortcuts (CLI)

```bash
shortcuts list                       # list installed shortcuts
shortcuts run "Name"                 # run
shortcuts run "Name" -o /tmp/out.txt # run and capture output (most reliable verification channel)
shortcuts view "Name"                # open in the editor
shortcuts sign --mode anyone --input x.wflow --output x.shortcut
```

There is no `import` or `delete` subcommand. Notifications shown by a shortcut are NOT readable via the accessibility tree; clipboard writes from the background runner are unreliable. Use text output captured with `-o` for verification.

## Building a shortcut: the .wflow format

A `.wflow` is an XML plist with a top-level dict:

```xml
WFWorkflowClientVersion: "2605.0.5"
WFWorkflowMinimumClientVersion: 900
WFWorkflowMinimumClientVersionString: "900"
WFWorkflowIcon: {WFWorkflowIconGlyphNumber, WFWorkflowIconStartColor}
WFWorkflowTypes: ["NCWidget", "WatchKit"]
WFWorkflowImportQuestions: []
WFWorkflowInputContentItemClasses: [...]
WFWorkflowActions: [ {WFWorkflowActionIdentifier, WFWorkflowActionParameters}, ... ]
```

Shortcut name at import time = the `.shortcut` filename.

### Installing

Unsigned imports are BLOCKED ("Importing unsigned shortcut files is not supported"). Always sign first:

```bash
shortcuts sign --mode anyone --input x.wflow --output x.shortcut
open x.shortcut   # opens a preview window; press "Add Shortcut"
```

The preview window's Add button is `button 2 of scroll area 1 of group 1 of window 1` (System Events AXPress). Timing: ~1.2s after `open`; the preview can auto-close after ~4s, so press quickly. Importing a file with the same name does NOT reliably replace the existing shortcut (sometimes replaces, sometimes creates a duplicate) - verify via the database after every import.

### Serialization rules (each one verified the hard way)

- **Text with variables** (`WFTextTokenString`):
  ```
  {Value: {string: "prefix\uFFFCsuffix",
           attachmentsByRange: {"{<position-of-U+FFFC>,1}": {OutputName, OutputUUID, Type: "ActionOutput"}}},
   WFSerializationType: "WFTextTokenString"}
  ```
  The position is the character offset of the U+FFFC (object replacement) char. Do NOT use an `attachments` dict - the attachment gets silently stripped and only the plain string remains.
- **Standalone variable parameter**:
  ```
  {Type: "Variable", Variable: {Value: {OutputName, OutputUUID, Type: "ActionOutput"},
                                WFSerializationType: "WFTextTokenAttachment"}}
  ```
  `OutputName` is cosmetic; `OutputUUID` must point at the source action's `UUID`.
- **Headers on "Get Contents of URL" (`is.workflow.actions.downloadurl`) are ALWAYS silently dropped on import** (verified with a local echo server across 4 serialization variants). Use path-based auth in the URL instead of headers.
- **If comparison value must be a serialized `WFTextTokenString`**; a plain string is dropped and the If shows "Please choose a value for each parameter in this action".
- **The If input's type determines available operators**: a "Contents of URL" (dictionary/URL) input only offers "has any value" in the UI. Insert a Text action (`is.workflow.actions.gettext`) that embeds the variable, and use its output as the If input - then "contains" etc. work.
- **`WFCondition` values (empirical)**: 4 = "is greater than" (NOT equals!), 99 = "contains". Do not guess other values; check in the UI or test.
- **Grouping (Repeat/If)**: the same action identifier opens and closes the block, disambiguated by `WFControlFlowMode`: 0 = start, 1 = else, 2 = end. Children link via `GroupingIdentifier`. Repeat count: `WFRepeatCount` (number). Delay: `is.workflow.actions.delay` with `WFDelayTime` (real, seconds).
- **Notification** (`is.workflow.actions.notification`): `WFNotificationActionTitle/Body/Sound/Urgency` still work.
- **JSON responses are re-serialized WITHOUT spaces** (`{"active":true}` not `{"active": true}`): never match on pretty-printed JSON; match loose tokens (e.g. contains `false`).
- **"Exit This Shortcut" (`is.workflow.actions.exit`) does NOT propagate the previous action's output**: empty output + fast return means the exit branch was taken.
- **Get Dictionary Value** (`is.workflow.actions.getvalueforkey`): params `WFDictionaryKey` + `WFInput` work as expected.

### Useful action identifiers

| Action | Identifier | Key params |
|---|---|---|
| Get Contents of URL | `is.workflow.actions.downloadurl` | `WFURL`, `UUID` |
| Text | `is.workflow.actions.gettext` | `WFTextActionText` (WFTextTokenString) |
| Show Notification | `is.workflow.actions.notification` | title/body/sound/urgency |
| Repeat (count) | `is.workflow.actions.repeat.count` | `WFRepeatCount`, `WFControlFlowMode`, `GroupingIdentifier` |
| If | `is.workflow.actions.conditional` | `WFCondition`, `WFConditionalActionString` (token!), `WFInput`, `WFControlFlowMode` |
| Wait | `is.workflow.actions.delay` | `WFDelayTime` |
| Stop This Shortcut | `is.workflow.actions.exit` | - |
| Get Dictionary Value | `is.workflow.actions.getvalueforkey` | `WFDictionaryKey`, `WFInput` |
| Copy to Clipboard | `is.workflow.actions.setclipboard` | `WFInput` |

## Verifying what actually got imported

Ground truth lives in `~/Library/Shortcuts/Shortcuts.sqlite`:

```sql
SELECT Z_PK, ZNAME, ZACTIONCOUNT FROM ZSHORTCUT;
SELECT writefile('/tmp/actions.bin', ZDATA) FROM ZSHORTCUTACTIONS WHERE Z_PK = <pk>;
```

Then `plutil -convert xml1` the dumped `ZDATA` (a binary plist of the flat action array) to see exactly which parameters survived the import. Icons live in `ZSHORTCUTICON` (`ZBACKGROUNDCOLORVALUE`, `ZGLYPHNUMBER` per `ZWORKFLOW`) - copy those two values into `WFWorkflowIcon` to clone an existing shortcut's look.

## Environment gotchas

- The Shortcuts app can get into a bad state (preview windows stop opening, imports fail silently): restart it (`pkill -x Shortcuts`) - with user consent.
- Gallery grid tiles are nearly impossible to automate via AX (no names, no AXShowMenu); **deleting shortcuts must be done manually** by the user. Cmd+Delete and Edit > Delete do not work on tiles.
- The menu bar progress ring shows "percentage of actions completed" and is useless with Repeat loops (confirmed by Matthew Cassinelli / Jason Snell). Spinning = running; the position means nothing. Workarounds: notifications, One Thing, SwiftBar, or a custom menu bar app polling a status endpoint.
- The published "Apple Shortcuts Integration" skill ecosystem (YAML pseudo-docs) does not cover any of the wflow details above. Best real reference: github.com/joshfarrant/shortcuts-js (attachmentsByRange, aggrandizements like `WFDictionaryValueVariableAggrandizement` and `WFCoercionVariableAggrandizement`).
