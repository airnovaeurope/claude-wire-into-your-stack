# NOTES - wiring Claude into course-api

Everything below is committed in the repo, so it reaches every teammate who clones it. There are no secrets anywhere, and `.env` / `.env.*` stay git-ignored and are also denied in permissions.

## 1. Server (MCP): `docs`, a filesystem server pointed at `./docs`
**What.** `.mcp.json` connects `@modelcontextprotocol/server-filesystem` at **project scope**, rooted at `./docs`. It needs no credentials. `.claude/settings.json` turns on just this server (`enabledMcpjsonServers: ["docs"]`), so it isn't a blanket "trust all project servers".

**Why it helps.** `docs/api.md` is the contract for this API. With the server, Claude can read and search it directly when it adds or reviews an endpoint, and it can't wander outside `docs/`.

**Permission rule.** `permissions.allow` lists only the read-only tools Claude actually needs: `mcp__docs__read_text_file`, `mcp__docs__list_directory` and `mcp__docs__search_files`. The write tools (`write_file`, `edit_file`, `move_file`, `create_directory`) are in `deny`, so the docs can only change through a normal reviewed edit, never through the server.

**Using it.** The first real task for the server is "check `docs/api.md` against `routes/users.js` for `PUT /users/:id`". I did that check by reading the files directly, because the cloud workspace couldn't start a live MCP session (see the honest note in section 5). `api.md` says the body may include `name`, `email` or both, with a `400` when neither is sent and a `404` for an unknown id. That matches the route. While checking, I noticed that `name: ""` is accepted and the docs don't say so. That is noted for a follow-up, not changed here. To see the server work in a fresh session, run `/mcp` and look for `docs` as connected, then ask *"read docs/api.md and list the users endpoints"*. Claude should call `mcp__docs__read_text_file` without asking for approval.

## 2. Skill: `.claude/skills/add-route/SKILL.md`
**What it captures.** It records the way every route in this project is written:
- The handler goes in `routes/<resource>.js`, with a `// METHOD /path — …` comment.
- Data access goes only through `db/store.js`, and new helpers are reset in `reset()`.
- Validation runs in order: `Number(id)`, then 404, then 400, always with an `{ error }` body.
- Each route gets supertest tests using `store.reset()` that cover success, 404 and 400.
- `docs/api.md` is updated.
- `npm test` and `npm run lint` must pass.

**How the description fires.** It names the action and the objects in the words people actually type: *add, create, or change an endpoint, route, or API method (GET/POST/PUT/PATCH/DELETE) in this project*. It also says what the skill does, so Claude can match it. It doesn't fire on unrelated work such as "fix the README" or "explain the store". A request like *"add a DELETE /users/:id endpoint"*, which doesn't name the skill, is exactly what its description covers.

## 3. Command: `/review-route <METHOD /path>` (`.claude/commands/review-route.md`)
**What.** It reviews one endpoint against a five-point checklist (store-only data, 404/400 shapes, status codes, tests, docs) and asks for PASS/FAIL with file:line evidence plus the single most important fix. It is report-only and doesn't touch files. `$ARGUMENTS` takes the endpoint as one free-form value, for example `/review-route PUT /users/:id`.

**Why it's worth a shortcut.** This review comes up on every PR that touches `routes/`, and typing the checklist each time is how items get skipped.

**Running it once on `PUT /users/:id`** gave these results:
1. PASS: data goes through `store.updateUser`.
2. PASS: an unknown id gives 404, and a non-numeric id becomes `NaN` and gives 404. 400 when neither field is sent.
3. PASS: success returns 200.
4. FAIL (`tests/users.test.js`): no test covers the 400 "neither field" case.
5. PASS: the docs match.

Top fix: add the 400 test. That result was useful enough to keep the prompt exactly as it is.

## 4. Hook: PostToolUse on `Write|Edit` - lint the file Claude just changed
**What.** The hook lives in `.claude/settings.json` at project scope. On **PostToolUse**, with matcher `Write|Edit`, it runs `.claude/hooks/lint-edited-file.sh`. The script reads the edited `file_path` from the hook's JSON on stdin and ignores anything that isn't `.js`. It then runs ESLint on that one file. On errors it exits `2`, so ESLint's output goes back to Claude and Claude fixes the file before continuing.

**React or prevent.** It **reacts**: the edit has already happened, and the hook holds the standard that no `.js` file is left failing lint. I chose PostToolUse because linting needs the written file. Blocking risky commands is already handled by the deny list.

**Triggered on purpose.** I fed the hook the payload for an edit to a bad file (an undefined `foo()` and an unused variable). It printed ESLint's 1 error and 1 warning and exited `2`. For a clean `routes/users.js` it exited `0`, and for `docs/api.md` it was skipped with exit `0`. It runs path-relative to `$CLAUDE_PROJECT_DIR`, with no hardcoded absolute paths.

## 5. Headless run with a scoped `--allowedTools`
**Task.** Add the missing 400 test that `/review-route` found, then run the tests.

```
claude -p "Add a test in tests/users.test.js for PUT /users/:id returning 400 when neither name nor email is sent, following the existing tests, then run npm test." \
  --allowedTools "Read,Edit(tests/**),Bash(npm test:*)" \
  --max-turns 6
```

**What I allowed and why.**
- `Read`, to read the route and the existing tests.
- `Edit`, limited to `tests/**`. It may add a test but can't change the route or anything else.
- `Bash(npm test:*)`, to check its own work.

Nothing else is pre-approved: no other Bash, no network and no git. With nobody watching, any other action has no approval and simply doesn't happen. `--max-turns` caps how long it can run. The PostToolUse lint hook also fires on its edit.

**Honest note.** The Claude Code CLI in the cloud workspace I used for this project couldn't sign in, so I couldn't run this headless command, the live MCP session, or a fresh-checkout hook trigger there. I checked the hook with the stdin payloads above. All the files are in place for the reviewer's fresh checkout. Running the command above from the repo root reproduces the headless step.
