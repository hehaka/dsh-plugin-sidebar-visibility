> Fork of [Robert-Wang-08/dsh-plugin-sidebar-visibility](https://github.com/Robert-Wang-08/dsh-plugin-sidebar-visibility) with an English UI.

# dsh-plugin-sidebar-visibility

Enhancement plugin for the DSH Web GUI (`dsh web`, default `http://127.0.0.1:3080`). Eleven features share one package:

| Feature | Entry | Behavior |
| --- | --- | --- |
| Visual workspace hiding | "Hide" button at the end of a workspace row in the sidebar | Collapses the workspace out of the tree; restore it in one click from the "Hidden workspaces" section at the bottom; session data is untouched |
| Workspace collapse | The ▾/▸ arrow on the workspace title row, clicking the title itself, or "Collapse all / Expand all" at the top of the tree | Collapses the group's session rows; the title row stays in place; the collapsed state is persisted with preferences; independent of hiding and archiving |
| Workspace drag-and-drop reordering | Drag a workspace title row | Dropping in the upper / lower half of the target group inserts before / after it; the drop position shows a 2px primary-color line; the order is written back to the Host via `IWorkspaces.insertBefore` and persists after refresh |
| Directory tree view | Top of the tree: "View → By directory tree" | Registered workspaces are automatically nested under their nearest strict ancestor by normalized directory (12px indentation, same rules as rc.2); collapsing a parent group collapses its entire subtree; hiding a parent moves its child workspaces up to the nearest visible ancestor; dragging **only reorders among siblings** (parent/child relationships are determined by path, so items cannot be dragged across levels); unregistered directories are not fabricated and Host ownership is not changed |
| New session | "+" button at the end of a workspace row | Starts the new-session flow in that workspace; if the group is collapsed, it is expanded first |
| Session row actions | Hovering a session row shows a vertical stack of "Rename / Fork / Archive" **below** the title row | Rename edits inline (Enter to submit, Escape to cancel); Fork creates a child session from the last complete turn; archived sessions can be restored under "Settings → Archived sessions" |
| Completed-unread highlight | Small dot on the right side of a session row, left of the pin button | Appears when a session finishes (running true→false) while observed in the browser and is not the current session; shown as a primary-colored dot with a soft glow; opening the session clears it; the state is persisted under the same localStorage key; runs that finish while the page is closed are not marked unread (the list has no run-completion timestamp, so `updatedAt` is not used as a guess) |
| Session favorites + "Favorites only" filter | ☆/★ at the right end of a session row's title row (always visible) + "Favorites only" toggle at the top of the tree | Favorites are a purely front-end derived view and do not write to Host data; archived and subagent sessions are neither shown nor favoritable |
| Running indicator + timer + model color | Colored spinner ring and model badge plus elapsed minutes, to the left of the session title | While running, the spinner ring and badge background use the **provider badge color of the session's selected provider** (user-chosen color > default color table > palette; same source as the input-box badge, color table not duplicated); the badge shows elapsed `Nm` for this run to its right; a session already running when its row first appears only shows a lower bound `≥Nm` (the rc.2 list has no run start time, so `updatedAt` is not used as a stand-in); disappears when the turn ends. The color reflects the **session's selection**, not the actual model executing after a runtime failover |
| Session list mode switch | "Session list: Plugin list / Raw list" button in the sidebar footer | Hot-switches in one click between the **plugin-enhanced tree** and the **official raw WorkspaceBrowser**, without restarting dsh; the choice is persisted under the same localStorage key |
| Provider suspend/resume | Toggle on the provider card in Settings → Models | Targets the card's own `settingsPath`: if the user layer has this provider's configuration, it is stashed and that subtree is unset; if it exists only in the composition layer, `models: []` is written as an override; other providers in the same family are unaffected; resuming writes it back |

All state is persisted in a single browser localStorage key, `dsh.sidebar-visibility.v1`, following the same convention as the official sidebar expansion state.

## Installation

```powershell
cd dsh-plugin-sidebar-visibility
pnpm install        # or npm install
pnpm run bundle     # produces lib/index.js + lib/client.js
```

**Build outputs are committed**: `lib/index.js`, `lib/client.js` and the two `.d.ts` files are committed together with the source. Machines registered with `link:` therefore get the new version via `git pull` + restarting `dsh web` + refreshing the browser; there is no need to run `dsh plugin add` again. After changing the source, run `pnpm run bundle` before committing, otherwise the old build outputs get committed.

### Tests

```powershell
node scripts/test-hierarchy.mjs   # pure directory-hierarchy functions (26 checks)
node scripts/test-runtime.mjs     # run timer + badge display-bridge contract (15 checks, including the real cross-plugin contract)
node scripts/test-bundle.mjs      # build-output smoke test (run bundle first)
```

The tests import `.ts` files directly using Node's built-in type stripping, so no test framework or extra installs are required.

## Mounting into dsh web

1. In the `cordis.patch.yml` of the mount profile (or `$DSH_HOME/cordis.patch.yml`), add a line to the browser roster section, following `cordis.patch.example.yml`:

   ```yaml
   - id: sidebar-visibility
     name: '@local/dsh-plugin-sidebar-visibility'
   ```

2. Make `dsh-client-modules` able to resolve this package: link it in the DSH runtime directory's dependencies (`pnpm link` / `file:` dependency), or add this directory to dsh's plugin search path.
3. Restart `dsh web`, then open the GUI and verify:
   - The sidebar session tree is replaced by this tree (a "Favorites only" toggle appears at the top);
   - Each provider card under Settings → Models shows a "Suspend provider" button.

During development, run `pnpm run dev:web` (inside a DSH checkout) to hot-reload changes to this plugin; for day-to-day work, `pnpm run watch` rebuilding the outputs is enough.

## Usage

- **Hide a workspace**: click "Hide" on the workspace title row → the whole workspace (including its sessions) disappears from the tree; expand "Hidden workspaces (N)" at the bottom of the tree and click "Restore" to bring it back.
- **Collapse a workspace**: click the ▾/▸ arrow on the left of the title row, or click the title directly → the group's session rows collapse / expand, while the title row always stays in place; when collapsed, the session count of the group is shown to the right of the title. Collapsed and hidden states are independent and persist across refreshes.
- **Directory tree view**: in the tree's "View" menu, choose "By directory tree" → registered workspaces are automatically nested into multiple levels by their nearest strict directory ancestor (rc.2 rules: Windows drive letters / UNC normalized to forward slashes, **case-sensitive**, compared by full path segments); group headers show a folder outline icon and indentation; collapsing a parent group collapses its **entire subtree**; when hiding a parent workspace, its visible child workspaces move up to the nearest visible ancestor (the bottom collapsed section only keeps the restore entry). Dragging only reorders within the **same parent**; a cross-level drop shows a "not allowed" cursor and does not write to the Host; the "Ungrouped" pseudo-group is always appended at the end of the tree. The existing "By workspace" and "Single list" modes are still available, and old preferences are unaffected.
- **Collapse all / Expand all**: the button on the right of the view options row at the top of the tree; collapses or expands all currently visible groups at once. It shows "Collapse all" while any group is expanded, and "Expand all" once all groups are collapsed. Hidden groups do not take part (they have no collapse arrow to begin with), and their collapsed state is kept as-is; the button is not shown in flat mode.
- **Workspace drag-and-drop reordering**: press and drag a workspace title row → drop it on the **upper half** of the target group to insert before it, or the **lower half** to insert after it; a 2px primary-color line marks the drop position. Releasing writes the order to the Host, and it persists across refreshes. The "Ungrouped" pseudo-group does not take part in dragging and cannot be a drop target; dragging back to the original position sends no request.
- **New session**: click "+" at the end of a workspace row → starts the new-session flow in that workspace. If the group is collapsed, it is expanded first, matching the behavior of the official project-row plus button.
- **Session row actions**: hover over a session row and three vertically stacked buttons, "Rename / Fork / Archive", appear **below** the title row; ☆/★ is always visible at the right end of the title row.
  - Rename: click "Rename" → the title turns into an input box in place → Enter to submit, Escape to cancel; if the Host rejects the change, the error message appears below the title.
  - Fork: click "Fork" → creates a new session from the last complete turn of this session and opens it.
  - Archive: click "Archive" → the session disappears from the tree; it can be restored under "Settings → Archived sessions".
- **Completed-unread**: when a session finishes running (spinner disappears) and you have not opened it yet, a small primary-colored dot appears on the right side of the row (left of the pin button); opening the session clears it. Only completions observed **while the page is open** are recorded; they do not reappear after a refresh or on another device. A session you are watching when it finishes is not counted as unread.
- **Favorite sessions**: click ☆ on a session row to turn it into ★; checking "Favorites only" at the top of the tree shows only favorited rows across the whole tree, and the header shows the favorite count.
- **Running indicator and timer**: while a session is running, a **colored** spinner ring + model badge + elapsed minutes (e.g. `[a1gpt] 3m`) appear to the left of its title. The color comes from the read-only display bridge of provider-badge (same color resolution as the input-box badge, including your custom color). **Sessions that are not currently open** are colored using the provider and the solid color table of the badge from the session projection, so you do not need to open the session first; hovering shows the provider's full name, model name, and timing basis. The timer only counts **continuous running intervals actually observed by the browser**: a session already running when its row first appears shows a `≥Nm` lower bound, and the last-updated time is not used as a stand-in start; it disappears automatically when the turn ends. The semantics match the official ongoing status, and it applies equally to grouped views and search result rows.
- **Switching session list mode**: click the "Session list: Plugin list / Raw list" button in the sidebar footer to hot-switch between the plugin-enhanced tree and the official raw list without restarting dsh. When switching to "Raw list", this plugin's occupant of `sidebar.workspaces` is unmounted and the official WorkspaceBrowser takes over immediately; switching back to "Plugin list" re-registers it, and all enhanced features (hide / collapse / favorites / drag-and-drop reordering, etc.) are restored. The footer button is always present and visible in both modes.
- **Suspend provider**: in Settings → Models, click "Suspend provider" on the target card → the provider's configuration is stashed and its models disappear from the model picker; clicking "Resume provider" in the same spot writes it back. **The granularity is the card's own `settingsPath`**; other providers in the same namespace are not affected. While suspended, new sessions cannot select this provider; existing sessions pinned to it are expected to enter a non-routable disabled state, which is cleared automatically after resuming (this point is item 2 of the First-run checklist).

## Verified facts (the basis for all of this version's code)

- Seat registration: `ctx.slots.inject(seat, () => ctx.slots.register(options, Component))`, where options contain `name / key / children / store / inject / locale` (precedent: `dsh-client-ui-workspace/lib/client.js`).
- Seat shadowing: for a single seat, the later registrant shadows the official occupant (priority comment in `dsh-client-ui-renderer` registry.d.ts), so the official tree does not need to be disabled.
- Keyed seats: registration carries a `key`, with no wildcard; this version covers the two built-in namespaces `llm-deepseek` / `llm-pi-ai`.
- Data hooks: `useWorkspaces` → `WorkspaceSnapshot { items, archivedSessionIds }`; `useSessions` → `SessionListState { ids, byId, current }`; `WorkspaceView { workspaceId, title, path, sessionIds }`; `SessionSummary { id, displayTitle, origin, blank, running, updatedAt }`.
- Action service: `ctx.uiWorkspace` (`openSession / startSession / openWorkspace / connectWorkspace / forkSession / archiveSession`), cordis inject name `uiWorkspace`.
- Session rename: the official `renameSession` goes through `ctx.get('sessions').binding(id)?.session.rename(title)` and returns `RemoteResult<{ title, seq }>`; both unbound sessions and rejected results are thrown as errors by the caller (precedent: `browserInjected()` in `dsh-client-ui-workspace/lib/client.js`). This plugin reuses the same path, and error messages are shown in place below the title.
- Official session-row menu: `SessionNodeItem` in `dsh-client-ui-workspace` provides three items, `rename / fork / archive` (workspace rows additionally have `rename / delete`, which this plugin does not implement); `rowActions` is normally hidden and shown on row hover. This plugin replicates that behavior with `.dshsv-rowActions` + an inline `<style>`, and moves the three actions to a vertical stack below the title row to avoid covering session names in the narrow sidebar.
- Official new-session button: the plus icon on the right of a workspace row (`IconPlusOutline16`) calls `setGroupExpanded(key, true)` and then `startSession(workspaceId)` in its `onClick`; this plugin implements the same semantics, so clicking while collapsed first expands the group.
- Workspace drag-and-drop reordering: goes through the **pure Workspace Controller**, not `uiWorkspace` — cordis inject name `workspaces`, method `insertBefore(workspaceId, beforeWorkspaceId?)`; omitting `beforeWorkspaceId` moves the item to the end (same path as `insertWorkspaceBefore` in the official `browserInjected()`). Which half the drop lands in is determined by whether the pointer is in the upper half of the target group's height (official `workspaceGroupHalf`); the two no-change cases, its own position and "already directly before the anchor", return immediately without sending a request.
- Escaping pitfall with inline `<style>` (hit twice): React converts `"` `'` `&` `<` `>` in text into HTML entities, but `<style>` is a raw text element that does **not decode** entities, so rules silently fail. Therefore these characters must not appear in the CSS string; for example `content:""` becomes `content:&quot;&quot;`. Draw lines with `box-shadow` instead (no quotes, takes no layout space).
- Sidebar button size: the browser's default `button` styling is too large in the narrow sidebar, so this plugin uses `.dshsv-btn` across the board to reduce it to 11px / 16px line height / `1px 6px` padding, with a light background on hover.
- Seat dispatch granularity (a pitfall we hit): `settings.models.provider-card` is dispatched by `settingsNs`. A single registration receives **every** card in that namespace (saved, newly added, and hand-declared ones all count), and each card carries its own `settingsPath` (official slot-contract text: an adapter family's companion plugin registers one entry under the family's namespace and receives every card of that family). Therefore suspend/resume must target by `settingsPath`; acting on the whole namespace would also clear other providers of the same family. The official provider deletion also uses `{ op: 'unset', path: [...settingsPath] }`.
- Root-level provider (empty `settingsPath`, e.g. `deepseek-official` within `llm-deepseek`) owns the entire namespace: suspending performs a key-by-key unset of the whole section in the user layer, and resuming must write it back as a whole-section merge via `update(ns, section)`. An empty path cannot address the namespace root with `mutate set path: []`.
- Settings read/write: `ctx.remote.settings.describe() → { user?, revision }`; `update / replace / mutate(ns, [{ op: 'set'|'unset', path, value? }], expectedRevision)` (precedent for op: `pathOps()`); the provider card's owner props carry `settingsPath`, which gives the user-layer addressing path directly.
- Running data source: `SessionSummary.running` (the official 0.2.0-rc.2 and this repo's 0.1.6-alpha.2 have the same shape). The official tree derives the `ongoing` status dot from `status?.running ?? summary.running`; this plugin reads `byId[id].running` directly to show the spinner ring.
- Footer seat: `sidebar.footer.action` (`kind: 'list'`, `scope: 'root'`, owner props `{ wide }`), declared by the official ui-sidebar's `sidebar` entry and rendered persistently in `SidebarRoot`; list registrations carry `id` / `order`.
- API basis for hot switching: `ctx.slots.register(...)` and `ctx.slots.inject(...)` both return idempotent disposers; calling one removes the occupant. For a single seat, the first live entry in ascending priority order wins (`dsh-client-ui-slots` renderer: first live entry per cell in priority order), so after unmounting the plugin occupant at priority -1, the official occupant at priority 0 takes over immediately; re-injecting restores it.

## First-run checklist (confirm in order during the first live run)

1. Once this tree shadows the official tree, the official interactions still missing are: session-level drag-and-drop reordering, workspace rename / delete, and new workspace (workspace-level drag-and-drop reordering / new session, and session-level rename / fork / archive are already covered).
2. Actual behavior of existing sessions after suspension (expected: a non-routable disabled state).
3. Whether routes are hot-unregistered after a user-layer unset on `llm-pi-ai` (`llm/adapters-updated` broadcast).
4. Cross-account interference with the single localStorage key (the official code derives per-account keys; aligned in iteration 2).
5. Provider card registration for user-defined namespaces (currently only the two built-in keys are covered).
6. Behavior of rename on unbound sessions (the official same path throws `unknown session`; if non-current sessions in the list are commonly unbound, switch to the rename RPC of `sessionController`).
7. Hot switching of session list mode: after clicking the footer button to switch to "Raw list", whether the official WorkspaceBrowser fully takes over; after switching back to "Plugin list", whether all plugin features are restored; and the display of the footer button in sidebar rail mode.

## Iteration plan

- Iteration 1 (current): get mounting working + a minimal closed loop of seven features (hide / collapse / workspace drag-and-drop reordering / new session / session row actions / favorites / Provider suspend).
- Iteration 2: strip the subagent `allowedModels` snapshot; secondary filtering of Host content search by the in-tree search box and "Favorites only"; align the account key; fill in the missing official tree interactions (session-level drag-and-drop reordering, workspace rename / delete).
- Iteration 3: evaluate proposing `hiddenWorkspaceIds` / `favoriteSessionIds` / `collapsedWorkspaceIds` upstream (following the `archivedSessionIds` pattern into the Host baseline).

## Host compatibility (reviewed 2026-09-23: dsh 0.1.5-rc.3 + dsh-permission-compat)

After the host was upgraded `0.1.5-rc.2 → rc.3` (npm latest) and the global `dsh-permission-compat` sandbox patch (modifying `dsh-sandbox` + three tool guards) was applied, this was re-checked: **this plugin works with zero changes**:

- Unrelated to the patch: the source and build outputs (`lib/index.js`, `lib/client.js`) do not reference privilege-escalation APIs such as `sandbox_permissions` / `justification` / `approveEscalation` (grep returns zero matches).
- Unrelated to rc.3: compared with rc.2, `dsh-client-ui-sidebar/lib/client.js` in rc.3 differs only by a one-line change to the brand badge version string inside `localBuildVersion()`; there are no API or behavior changes, and this plugin does not depend on that value.
- Known build status (not introduced this time): `src/index.ts` imports `@deepseek-ai/dsh-api-settings-controller`, but the built `lib/index.js` is a 199-byte placeholder stub with no imports. This has no impact under rc.3; when next changing this plugin, check `tsdown.config.ts`.

Review record: see the 2026-09-23 entry in the vault file `70-WorkSpace/dsh工具维护日志.md`; reinstall notes: see `70-WorkSpace/dsh复装说明.md`.
