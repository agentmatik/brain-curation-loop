# Design notes

What the loop is optimised for, and what we got wrong before it looked like this.

## The one constraint

An agent may add knowledge to the brain on its own. It may not change what the company is, decides or keeps private without a person clicking approve. Everything else follows from that.

## Decisions

**Tiers live in the brain repo, not in the agent.** `docs/CURATION.md` in the brain defines SAFE (timeline appends, new draft stubs, meetings, weekly notes) versus SENSITIVE (strategy, decisions, finance, verified pages, anything private). The curator reads the rules from the repo on every run. Moving a company's threshold is a pull request on its own brain, not a prompt change on our side, and the same curator serves every brain.

**A staging tier between the model and the canon.** The extract tap writes only `inbox/extracted/`. Nothing goes from a transcript straight into a customer page. The staged file keeps `curated: false` until the curator has folded it, then flips to `true` and stays forever as provenance. Two effects: the model's mistakes are contained in a folder a person can delete, and every canonical fact can be traced back to a quote in a transcript.

**One verbatim quote per fact.** It is the cheapest hallucination check there is. A fact the model cannot quote is a fact it made up.

**Append-only timelines, immutable decisions.** The curator never rewrites history. A wrong fact is corrected by a new dated line, and a superseded decision by a new decision that names the old one. This is what makes automatic writes safe to allow at all.

**Empty output is a valid output.** A stand-up with nothing durable produces no file. Forcing the model to always return something is how junk facts enter a brain.

**Approval where the approver already is.** Managers do not review pull requests. A Slack card with Approve and Reject, written in plain language, in the language the approvers read, gets clicked. The card updates in place with who clicked and what happened, so the channel is also the audit log.

**Names spelled the brand's way.** The curator carries the company's own spelling rules (capitalisation, diacritics). Brains are read by agents that will repeat whatever is written.

## What we got wrong first

- **Dedup in workflow state.** The first version tracked processed meetings in n8n static data. On n8n Cloud it silently failed and meetings were staged twice. The `curated` flag in the file, and a GitHub file as the dedup store, replaced it.
- **English PR cards for non-English approvers.** Nobody clicked. The curator now writes PR titles and bodies for the people who approve them.
- **Sensitive facts in the SAFE tier.** Ownership and compensation details appeared in a timeline line because the model rated them safe. The contract now names those topics explicitly, and the curator writes a neutral one-line note and routes them SENSITIVE regardless of the model's hint.
- **Forgetting to invite the bot.** A Slack channel the bot is not a member of returns `channel_not_found`, which reads like a config error. It is an invite.

## What it is not

It is not a chat interface to the brain, and it does not answer questions. For reading a brain with typed relationships (who owns an account, which decision replaced which), see [gbrain-company-brain](https://github.com/mattzimak/gbrain-company-brain).
