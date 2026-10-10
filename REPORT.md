# Report: Defect Intake Form to Shared Defect Register

## Status per part

Global n8n MCP server config: DONE
  evidence: `~/.config/kilo/kilo.jsonc` -> type remote, url https://ranjitha23.app.n8n.cloud/mcp-server/http, enabled true, Authorization header present.
  `n8n_list_workflow_tags` -> `{"data":[],"count":0,"totalCount":0}` (native tools live; no restart required)

Workflow created in n8n: DONE
  evidence: `n8n_create_workflow_from_code` -> `{"workflowId":"RuM6RA3HxoAJQfbm","nodeCount":6}`
  URL: https://ranjitha23.app.n8n.cloud/workflow/RuM6RA3HxoAJQfbm

Graph and connections: DONE
  evidence: `n8n_get_workflow_details` -> 6 nodes; trigger -> "Steps and Severity Provided?" -> true: "Tidy Defect Text" -> "Append Defect to Register" -> "Show Defect Logged"; false: "Show Missing Fields Error"

Per-node schema validation: DONE
  evidence: `n8n_validate_node_config` -> all 6 nodes `valid: true`

Full-graph validation: DONE
  evidence: `n8n_validate_workflow` -> `{"valid": true, "nodeCount": 6}`

Whitespace tidy correctness: DONE
  evidence: first stored value was `replace(/s+/g, " ")` (backslash eaten). After `n8n_update_workflow`, `n8n_get_workflow_details` shows `replace(/\s+/g, " ")`.

Create register spreadsheet + end-to-end run: BLOCKED
  evidence: `n8n_list_credentials` with query "google" -> `{"data":[],"count":0}`
  `n8n_test_workflow` -> execution 2 status error: `WorkflowHasIssuesError: The 'Append Defect to Register' node has issues: Parameter "Document" is required. Parameter "Sheet" is required.`

Publish / activate workflow: NOT REQUESTED (workflow is inactive, so the production form URL does not serve yet)

## What broke and how I fixed it

1. Whitespace regex silently corrupted. The SDK `expr()` string treated `\s` in `'.../\s+/g...'` as a JS escape and stored `/s+/g`, which would replace runs of the letter "s" and never collapse whitespace. Found by reading the stored node back after creation (not by assuming). Fixed with `n8n_update_workflow` (updateNodeParameters, replace) and re-read the workflow to confirm `/\s+/g` is stored.

2. End-to-end test could not run. Pre-flight blocked on the Google Sheets node's Document and Sheet being empty. This is correct behaviour given no connection exists; it is not a workflow defect.

## Claims ledger

- Global config contains the n8n MCP server, token redacted: PROVEN (`Get-Content` + ConvertFrom-Json on kilo.jsonc).
- Native n8n MCP tools are loaded: PROVEN (`n8n_list_workflow_tags` returned a real payload).
- Workflow exists with 6 nodes and the intended wiring: PROVEN (`n8n_get_workflow_details`).
- Workflow code passes full validation: PROVEN (`n8n_validate_workflow` -> valid true).
- Tidy expressions use `\s`: PROVEN (stored value re-read after update).
- Text is actually tidied at runtime: UNVERIFIED (could not execute; blocked on Document/Sheet).
- Invalid row is kept out of the register and tester sees the error page: UNVERIFIED (could not execute).
- A new "Shared Defect Register" spreadsheet exists: NOT DONE (no Google Sheets credential to create it).

## What I would tell the next person

- The workflow is built and wired, not yet runnable, because n8n has no Google Sheets credential. Creating one needs a human Google sign-in.
- After the credential exists: create the register spreadsheet with a header row `Module | Steps | Expected | Actual | Severity | SubmittedAt`, then open the workflow and set the "Append Defect to Register" node's Document and Sheet (both are currently "From list", intentionally left empty rather than a fabricated ID).
- Then publish the workflow to get the production form URL, and submit one good and one bad defect to confirm both branches.

---

# Report: Scheduled Node news-digest agent (DailyQANewsDigest)

## Status per part

digest.js (Node, stdlib only): DONE
  evidence: `node digest.js` -> printed "Daily Digest - software testing (Bengaluru, India)", 5 stories with source+date, banner "AI summaries are missing: key.txt is empty (no key pasted)"; last line "Wrote C:\Users\User\Downloads\fwai-starter\digest.html"

key.txt (empty, for the member to paste the key): DONE
  evidence: Get-ChildItem -Filter key.txt -> Name key.txt, Length 0

digest-task.xml (Windows Task Scheduler, 08:00 daily, StartWhenAvailable): DONE
  evidence: `schtasks /Create /TN DailyQANewsDigest_TEST /XML ... /F` -> "SUCCESS: The scheduled task "DailyQANewsDigest_TEST" has successfully been created."; `/Query /FO LIST` -> Next Run Time 03-10-2026 08:00:00, Status Ready; `/Query /XML` -> <StartWhenAvailable>true</StartWhenAvailable>; temp task deleted after.

AI summaries through the real API: UNVERIFIED
  evidence: cannot be run until the member pastes a key into key.txt. The branch is coded (key starting sk-or- -> OpenRouter, otherwise DeepSeek) but no live call was made.

Runs after a missed 8am (machine asleep): PARTIALLY PROVEN
  evidence: <StartWhenAvailable>true</StartWhenAvailable> in the registered task XML. Not observed end-to-end across a real sleep/wake.

## What broke and how I fixed it

1. Schedule XML rejected: `schtasks /Create /XML` -> "The task XML is malformed. (1,40)::ERROR: unable to switch the encoding". Cause: the file was written UTF-8; schtasks needs UTF-16. Fix: re-wrote the same XML with PowerShell `Set-Content -Encoding Unicode` and set the declaration to encoding="UTF-16"; registration then succeeded (proof above).

## Claims ledger

- Script fetches the India edition (hl=en-IN, gl=IN, ceid=IN:en) and writes digest.html: PROVEN (node digest.js output above).
- Repeats dropped and newest first: PROVEN (5 unique items, dates descending 2026-09-29 .. 2026-09-02).
- Schedule registers for 08:00 daily with StartWhenAvailable: PROVEN (schtasks register + /Query + /XML export above).
- AI call reaches OpenRouter/DeepSeek and returns two lines per story: UNVERIFIED (no key yet).
- The scheduled run actually fires at 8am: UNVERIFIED (trigger not waited for).

## What I would tell the next person

- The three plan docs in this folder (PRD.md / TECH-STACK.md / IMPLEMENTATION-PLAN.md) describe the Release Defect Chase Board, not this digest agent. This work is a separate deliverable built from the chat brief; the conflict was flagged, not silently merged.
- The model sits on one line at the top of digest.js: `const MODEL = "deepseek-chat";`. If the key is an OpenRouter key, change it to an OpenRouter model id (e.g. deepseek/deepseek-chat).
- key.txt is a secret. This folder is not a git repo, so nothing is committed; if it ever becomes one, add key.txt to .gitignore before the first commit.
- The news query words are `software testing` + `India` (trade = software testing/QA, market = Bengaluru, India).
- Kept the form fields optional at the form level on purpose: the workflow's own IF check is the gate, so the error branch is reachable (a form-level "required" would block submission and make the error page dead code).

---

# Report: ChaseBoard QA one page website

## Status per part

index.html (7 sections, inline CSS): DONE
  evidence: chrome DevTools-protocol screenshots with device emulation -> 1440 sw=1440 cw=1440; 375 sw=375 cw=375 (no horizontal overflow); all 4 images natural size > 0.

images/ (the member's photos): DONE
  evidence: gh api repos/Ranjitha0923/chaseboard-qa-site/contents/images -> profile.jpg, shot-1.jpg, speaking.jpg, working.jpg

GitHub repo created and pushed: DONE
  evidence: gh repo view Ranjitha0923/chaseboard-qa-site -> {"url":"https://github.com/Ranjitha0923/chaseboard-qa-site","visibility":"PUBLIC","defaultBranchRef":{"name":"main"}}
  gh api contents -> root: WEBSITE-RULES.md, images, index.html

## Website rules checklist (WEBSITE-RULES.md part 6)

- [x] Exactly the seven sections, in order: hero, proof, what you get, how it works, price, enquiry, footer.
- [x] H1 64 to 88 at 1440 (clamp = 71.6px), 40px at 375, not cut off.
- [x] Six of seven sections are palette blocks; no two touching sections share a colour. Amber fills the hero and the enquiry block.
- [x] Every text and background pair ratio is in WORKLOG.md line 23 and passes AA.
- [ ] Five or more real photos: only FOUR are available (the member's own). Gap listed below.
- [x] No horizontal scroll at 375 (sw=cw=375). The phone bottom bar holds the call button (WhatsApp is off, see below).
- [x] Section spacing 96/72/56; nothing under 14px.
- [x] Two fonts loaded (Outfit, DM Sans); headline font is not Inter, Roboto or Arial.
- [x] Every contact link is real (booking and email). No placeholder text in any link.
- [x] No invented number, name, testimonial or guarantee.
- [x] No photo used twice. No em dash or en dash in the page text.
- [x] Looked at screenshots at 1440 and 375 and fixed the icon sizes after looking.

## Gaps and what the member still needs to give

- No logo files (logo.*, logo-dark.*). Used the wordmark "ChaseBoard QA" in the headline font, as the rules allow.
- Only four photos exist, so "five or more real photos" is not met. No stock photos were fetched: the rules forbid office, laptop and team cliches, and the offer has no plain-noun photo subject. The two service tiles without a photo are colour panels, as the rules allow.
- WhatsApp was left off per the member's answer, so the rules' WhatsApp-first button and phone bar are the call button and email instead.
- The member answered the bracket slots: years = "Just starting", releases covered = 1, result = "I catch defects before they reach customers", timeline = IST 7:30 pm. No square brackets are left on the page.
- "How long does a pack take" has no answer in the brief ([YOUR TIMELINE] unwritten), so that question is not on the page.
- The member said they have no client quote or numbers, so no testimonial appears. The line under the hero is the member's own description of the client's problem, not a client quote.
- The brief asked for five FAQ details; the website rules require three one-line answers under the price, and the rules win. Three are on the page; the other two are noted here.

## What broke and how I fixed it

1. The first 375px screenshot showed text cut off on the right. Cause: this Chrome enforces a minimum window width of 489px, so the screenshot cropped a 489px layout down to 375. This was a test fault, not a page fault. Fixed the test: measured the page inside a 375px iframe (CW=360 SW=360, no overflow), then captured with device emulation (375 sw=cw=375). The page has no horizontal overflow.

## Claims ledger

- Page has no horizontal overflow at 375: PROVEN (CDP evaluate sw=375 cw=375).
- All four photos load: PROVEN (naturalWidth x naturalHeight 1376x768, 1024x1024, 896x1200, 896x1200).
- Contrast pairs pass AA: PROVEN (ratios in WORKLOG.md line 23).
- Repo exists and contains the site: PROVEN (gh api contents -> WEBSITE-RULES.md, images, index.html).
- Vercel deploy: UNVERIFIED (not deployed here; the member imports the repo next).

## What I would tell the next person

- The repo is public. Flip it to private in GitHub settings if preferred before the site launches.
- When the member supplies the named picture set (logo, logo-dark, profile, working, speaking, shot-1 to shot-6), drop them into images/ and they fill the existing slots.
- WEBSITE-RULES.md is saved in the repo and wins over the brief on look, colour, type and section order.
