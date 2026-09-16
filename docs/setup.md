# Setup

Wiring the three pieces to one brain repo. Budget an afternoon the first time.

## 0. The brain repo

Start it from [template-intelligence](https://github.com/agentmatik/template-intelligence) (use it as a GitHub template), or add to an existing brain:

- `docs/CURATION.md`: the tier contract. Copy it from the template and adjust the SENSITIVE list to what your company considers sensitive.
- `inbox/extracted/` with a `README.md` (the staging folder the tap writes to).
- `scripts/validate-frontmatter.sh` (the curator runs it before finishing).

The layout the curator expects: `customers/`, `competitors/`, `suppliers/`, `distributors/`, `people/`, `decisions/`, `meetings/`, `weekly/`, `templates/`.

## 1. GitHub token

One fine-grained personal access token, or a GitHub App installation token, scoped to the brain repo only:

- `contents: write` (the tap writes staging files; the approval workflow merges)
- `pull_requests: write` (the approval workflow merges and closes; the curator opens PRs with its own `gh` auth)

Put it in n8n as a credential you reference from the two Code nodes, or paste it in place of `REPLACE_WITH_GITHUB_TOKEN` if your n8n instance is private. Never commit an export that contains it.

## 2. The extract tap (n8n)

1. In your transcript pipeline, after the summary node, add a parallel branch: Agent node with a Structured Output Parser, then a Code node. Copy the prompt, the schema and the script from [`../n8n/knowledge-extract-tap.md`](../n8n/knowledge-extract-tap.md).
2. Point the Code node's first line at the node that carries `call_name`, `date` and the transcript in your pipeline.
3. Run one meeting through. Expected result: a commit `Stage extracted facts: <meeting>` on the brain repo and a file under `inbox/extracted/`.

## 3. The curator (scheduled cloud agent)

1. Create a scheduled agent that checks out the brain repo on each run. With Claude Code routines: a nightly cron, source = the brain repo, the prompt from [`../routines/curator-prompt.md`](../routines/curator-prompt.md) with `{{COMPANY_NAME}}` filled in.
2. The agent needs `git` push to `main` and `gh` auth able to open pull requests on the repo.
3. First run: trigger it by hand and read the run report. Expected: SAFE commits on `main`, and for anything sensitive a branch `bot/<slug>` with an open PR.

## 4. PR approval in Slack (n8n)

1. Create a Slack app with a bot token (`chat:write`) and Interactivity enabled. Invite the bot to the approval channel; without the invite, posting fails with `channel_not_found`.
2. Import [`../n8n/pr-approval-in-slack.json`](../n8n/pr-approval-in-slack.json). On the Slack node pick your Slack credential. Replace `REPLACE_WITH_SLACK_CHANNEL_ID` with the channel id, and in the two Code nodes replace `OWNER/REPO` and `REPLACE_WITH_GITHUB_TOKEN`.
3. Activate the workflow and copy its two production webhook URLs.
4. GitHub, brain repo, Settings, Webhooks: add the `.../brain-pr-events` URL, content type `application/json`, event `Pull requests` only.
5. Slack app, Interactivity and Shortcuts: set the Request URL to the `.../brain-approval-action` URL.
6. Test: open a PR from a branch named `bot/test` by hand. A card should appear in the channel; Approve squash-merges it, Reject comments and closes it. The card updates in place with who clicked.

## 5. Run one real meeting end to end

Transcript in, staging file the same day, commit or Slack card the next morning, approved change on `main`. If any step is silent, check in this order: the tap's `extract_error` field in the n8n execution, the curator's run report, the n8n execution log of the webhook workflow.
