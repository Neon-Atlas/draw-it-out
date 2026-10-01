---
name: draw-it-out
description: Answers "does my app do X?" with a status label and proof from your code, in plain English. Also keeps a one-screen cheat sheet and a hand-drawn notebook page of the app, and teaches it to you with quizzes, tours and meeting prep.
argument-hint: "[question] | quick <question> | scan | page | quiz | tour | prep <who>"
disable-model-invocation: true
allowed-tools:
  - Bash(git rev-parse *)
  - Bash(git status *)
  - Bash(git diff *)
  - Bash(git log *)
  - Bash(git ls-files *)
  - Bash(git grep *)
  - Bash(git show *)
  - Bash(git cat-file *)
  - Edit(~/.draw-it-out/**)
license: MIT
---

# draw-it-out

The user built this app, often with an AI agent, and needs to know what it really does, how it works, and what the people using it see. Tell them the truth, including the parts that aren't finished. The code is the source of truth. README, CLAUDE.md and docs describe intent, not shipped behaviour.

Request: $ARGUMENTS

## Route the request

| Request | Do |
|---|---|
| empty | [Summary](#summary) |
| `scan`, `refresh`, `update` | [Scan](#scan) |
| `page`, `link` | [Page](#page) |
| `quiz` | [Quiz](#quiz) |
| `tour` | [Tour](#tour) |
| `prep <who>` | [Prep](#prep) |
| `quick <question>` | [Ask](#ask), one sentence only |
| anything else | Treat it as a question: [Ask](#ask) |

Words like "for an investor", "for a developer" or "for my mum" set the audience for that answer.

## Files

- App name: the folder name of `git rev-parse --show-toplevel`. Not a git repo: the current folder name.
- Notes: `~/.draw-it-out/<app>.md`. Page: `~/.draw-it-out/<app>.html`. Resolve `~` to the absolute home path for file tools. If the notes' `repo:` is a different path, use `<app>-<parent folder>` as the name instead.
- The notes are the cheat sheet, the source for every answer, and the content of the page.
- Never write inside the user's repo.

## Writing

Everything you show the user or save in the notes follows [writing.md](writing.md): run it through the `humanizer` skill in embedded mode if it's installed, otherwise use the rules in that file.

## Freshness check

Run before everything except a scan.

1. Run `git rev-parse --abbrev-ref HEAD`, `git rev-parse --short HEAD` and `git status --porcelain`. You need `branch@commit` for every answer.
2. No notes: stop here.
3. Notes' `commit` equals HEAD and nothing changed: the notes are current.
4. Otherwise changed files = `git diff --name-only <notes commit>` plus untracked files from `git status --porcelain`. If git doesn't know that commit (`git cat-file -t <commit>` fails after a rebase or fresh clone), the notes need a full scan.
5. Not a git repo: freshness unknown. Verify against the code anything you state.

For Ask, re-check only what the question touches. For everything else, refresh first if source files changed: say "Code changed since the last scan (N files). Updating notes first." and run [Scan](#scan) in refresh mode.

## Status labels

Use exactly these, written as emoji plus bold word:

- ✅ **Yes**: a user can reach it (screen, button, link, public API) and it does the real work.
- 🟡 **Partly**: reachable but limited: TODOs, mock or hardcoded data, missing steps, missing on a platform where the others have it, or behind a switch you can't see (a feature flag, a database setting).
- 🔌 **Built, not switched on**: the code exists but nothing lets a user reach it (no route or link, flag off, commented out, unused).
- ❌ **No**: no code for it. If docs claim it, say "Docs claim it; the code doesn't."

Gated features (paid plan, admin only, invite only): label what the gated user gets, and name the gate in the answer ("✅ **Yes**, on paid plans"). Apps with several clients (web, iOS, Android): say which have it. A feature that's on one platform by design is ✅.

Labels describe the checked-out code, not the live site. Don't compare branches unless asked. If the project's own docs say a feature isn't released, isn't live or hasn't been tried on a real device, keep the label and say so in the answer ("✅ **Yes** in the code, not released yet").

## Summary

- Notes exist: freshness check, then show the notes' "On one screen" section, the page link (`page:`), and one line: "Try `/draw-it-out does it do X?`, `quiz`, `tour` or `prep investor`."
- No notes: ask with AskUserQuestion: "No notes for <app> yet. Scan it now? About 15 minutes for a small app, up to an hour for a big one." Options: "Scan now" (do [Scan](#scan)) and "Not now" (reply "Ask a question any time with `/draw-it-out <question>`; I'll check the code directly."). Without AskUserQuestion, ask the same in text and wait for the answer.

## Ask

1. Freshness check.
2. Look for the answer in the notes. If there are no notes, they don't cover it, or the area changed (a changed file is one of the feature's proof files or sits next to them): trace it in the code yourself, from entry point (screen, route, button, API) to handler to where the data goes. Find the entry point fast: `git grep -n -i` the feature's words in UI text, route paths and handler names, then follow the calls. Stop once the label is clear. Aim for under 3 minutes; if it would take longer, answer with what you've verified, say what you haven't, and suggest a scan.
3. Reply in this shape. The block stays under 120 words:

   ```
   <✅ **Yes** | 🟡 **Partly** | 🔌 **Built, not switched on** | ❌ **No**>: <one plain-English sentence>

   How it works:
   1. <what the user does or sees>
   2. <what happens behind the scenes, in plain words>
   3. <where the data ends up>

   Proof: <1 to 3 most telling path:line>
   Checked on `<branch>@<commit>`.
   Remember this: <one short takeaway worth memorising>
   ```

   Then, outside the block: "Say 'deeper' for under the hood." For a developer audience, give the technical version straight away.
   For `quick`, reply with the first line of the shape only: `<label>: <one plain-English sentence>`. The sentence carries any gate or "not released yet". Nothing before or after it, not even the closing lines. "more" then gives the full shape.
4. "deeper": the technical walk-through (files, functions, services, data flow), under 200 words, with proof.
5. If the notes exist, add what you verified (a feature row, a likely question) without changing `commit`; only a scan sets it. The page catches up on the next scan or `/draw-it-out page`. If they don't exist, add a last line: "No notes for this app yet. `/draw-it-out scan` builds the cheat sheet and page (about 15 minutes for a small app, up to an hour for a big one)."

## Scan

1. Tell the user: "Scanning <app>. This takes about 15 minutes for a small app and up to an hour for a big one." Then spawn one `general-purpose` subagent with this prompt, filled in:
   "Scan the app at <repo path> for draw-it-out. Mode: <full | refresh, changed files: ...>. Notes file: <notes path>. Follow ${CLAUDE_SKILL_DIR}/scan.md exactly."
   Use refresh mode when notes exist and the freshness check found changes; otherwise full.
2. When it finishes, read the notes, show their "On one screen" section, then do [Page](#page).
3. Close with the counts it reported (✅ 🟡 🔌 ❌) and one line: "Ask anything with `/draw-it-out <question>`, or try `quiz`, `tour`, `prep <who>`."

## Page

The page is a hand-drawn notebook: the notes rendered onto ruled paper, with the summary on a taped index card and the diagrams as pencil sketches. The design lives in [page.html](page.html); don't redesign it or write HTML yourself.

1. Notes missing: say so and offer a scan. Otherwise freshness check.
2. Build the page from the template and the notes with one Bash command (`<Name>` is the notes' `#` heading):

   ```bash
   { printf '<title>%s, drawn out</title>\n' "<Name>"; cat "${CLAUDE_SKILL_DIR}/page.html" "<notes path>"; printf '</textarea>\n'; } > "<page path>"
   ```

3. Publish `<page path>` with the Artifact tool:
   - No `page:` in the notes yet: publish with icon `notebook` and description "How <Name> works, checked against its code." Save the returned URL as `page:` in the notes frontmatter.
   - `page:` set: Artifact `read` that URL first, then publish the page file with `url` set to it. Don't pass `icon`.
   - Give the link. It stays private until the user shares it.
4. Without the Artifact tool: "Open <page path> in your browser."

## Quiz

1. Needs notes (offer a scan if missing). Freshness check.
2. Write 5 multiple-choice questions from the notes: 2 about feature status ("Can a user…?", the four labels as options), 1 about how something works, 1 about data or privacy, 1 about tech, cost or screens. Favour the notes' "Likely questions". If a prep just ran in this conversation, quiz on those questions instead.
3. Vary the position of the right answer. Never mark an option as recommended. Keep options short.
4. Ask with AskUserQuestion (up to 4 questions per call, so use two calls). Without it, number the questions with a to d options and ask for a reply like "1a 2c 3b 4d 5a".
5. Reply: "Score: N/5". Then each question: ✓ or ✗, the right answer, one line of why, proof path. End with "Remember this:" lines for up to two misses.

## Tour

1. Needs notes (offer a scan if missing). Freshness check.
2. Stops: the top features from "On one screen", then the other ✅ and 🟡 features, most important first. One stop per reply, under 120 words:

   ```
   Stop N of M: <feature> <label>
   What you see: <1 to 2 lines>
   What happens underneath:
   1. ...
   2. ...
   Remember this: <one line>
   ```

   End each stop with: "Say next, deeper, or stop."
3. "deeper": the technical version of the current stop, with proof. "stop": list the stops covered, one line each.

## Prep

1. Audience from the words after `prep` (customer, investor, client, partner, developer, press, friend). Unclear: ask one AskUserQuestion with those options.
2. Freshness check. Use the notes; without notes, check the likely questions in the code directly and suggest a scan at the end.
3. Reply, under 250 words:
   - **Pitch for <audience>:** 2 to 3 sentences they'd care about.
   - **They'll likely ask:** 5 questions, each with a status-first answer the user can say out loud.
   - **Tough one:** 1 hard question and a straight answer.
   - **Don't oversell:** 🟡 and 🔌 items they might ask about, each with wording that says only what works today.
4. End with: "Say 'quiz' to rehearse these."

## Rules

- Plain English first. Explain any technical term in a few words.
- Never invent costs, prices, user numbers, limits, dates, plans, or security guarantees ("secure", "GDPR compliant", "encrypted at rest"). Without proof in code or config, say "not verified".
- Never present a 🟡 or 🔌 feature as done. Overselling is the failure this skill exists to prevent.
- Proof is `path:line`.
- Never edit the user's repo. Write only the notes and page files.
