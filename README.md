# brain-curation-loop

Meetings in, a curated company brain out, with a human approving anything sensitive.

A company brain is a git repo of Markdown that AI agents read and update: one file per customer, competitor, supplier, person, decision and meeting, in the [template-intelligence](https://github.com/agentmatik/template-intelligence) layout. This repo is the loop that keeps such a brain current without a person typing into it, and without an agent ever changing what the company *is* on its own.

Running in production since July 2026 on one client brain and three company brains.

## The loop

```
meeting transcript (Granola, Fireflies, an upload form, ...)
        |
        v
 1. Knowledge-extract tap  (n8n)         structured facts with a verbatim quote each,
        |                                 staged as inbox/extracted/<date>-<meeting>.md
        v                                 with curated: false. Always-safe tier.
 2. Curator                (nightly cloud agent)
        |                                 folds staged facts into canonical pages.
        |                                 SAFE tier: commits to main with a citation.
        |                                 SENSITIVE tier: branch bot/<slug> + pull request.
        v
 3. PR approval in Slack   (n8n)         GitHub webhook -> Approve / Reject card ->
        |                                 squash-merge or close, from the chat.
        v
 canonical brain on main, every fact citing the meeting it came from
```

The rules that decide SAFE versus SENSITIVE live in the brain repo itself (`docs/CURATION.md` in the template), not in the agent's prompt, so the same curator can serve any brain and the company controls its own tiers.

## What is in this repo

| Path | Piece | Runs on |
|---|---|---|
| [`n8n/knowledge-extract-tap.md`](n8n/knowledge-extract-tap.md) | Step 1. The agent node's system prompt, the structured-output schema, and the drop-in Code node that writes the staging file through the GitHub API | n8n, any LLM node with structured output |
| [`routines/curator-prompt.md`](routines/curator-prompt.md) | Step 2. The curator's prompt: rules files first, fold, tier, mark provenance, weekly note | a scheduled cloud agent with `git` and `gh` |
| [`n8n/pr-approval-in-slack.json`](n8n/pr-approval-in-slack.json) | Step 3. Importable workflow (9 working nodes plus 3 notes): `pull_request` webhook, `bot/*` filter, Block Kit card, Slack action webhook, merge or close | n8n |
| [`docs/setup.md`](docs/setup.md) | Wiring the three pieces to a brain repo, a Slack app and a GitHub token | |
| [`docs/design.md`](docs/design.md) | Why the staging tier, why tiers live in the repo, why a quote per fact, what we got wrong first | |

## Install, short version

1. Give the brain repo the curation contract: copy `docs/CURATION.md` from template-intelligence (or start the brain from the template).
2. n8n: add the extract tap to your transcript pipeline after the summary step (`n8n/knowledge-extract-tap.md`); import `n8n/pr-approval-in-slack.json`, replace the `REPLACE_WITH_*` placeholders, register the two webhook URLs.
3. Schedule the curator with `routines/curator-prompt.md` on a nightly cron against the brain repo.
4. Run one meeting through. Expect a file in `inbox/extracted/`, then a commit or a Slack card the next morning.

Full steps with the exact GitHub and Slack settings: `docs/setup.md`.

## Related

- [template-intelligence](https://github.com/agentmatik/template-intelligence): the brain layout this loop writes to, including the curation contract.
- [gbrain-company-brain](https://github.com/mattzimak/gbrain-company-brain): a gbrain skillpack that loads the same brain into gbrain with its types and relationships intact, so agents can query it.

## License

MIT.
