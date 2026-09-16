# Knowledge-extract tap (step 1)

A parallel branch added to any meeting-transcript pipeline, after the main summary and quality gate. It extracts durable facts with a structured-output LLM call and writes them to the brain's staging folder. It never touches canonical pages; that is the curator's job.

```
[summary + quality gate]
   +-- (parallel) --> Knowledge Extract (agent node, structured output)
                        +--> Push Extracted Facts (Code node, GitHub Contents API PUT)
```

Wrapped so that a failure here never breaks the main pipeline.

## Agent node

n8n `@n8n/n8n-nodes-langchain.agent` with `hasOutputParser: true`, plus a Structured Output Parser node. Any model that follows a JSON schema works; use the same model the summary step already uses.

**User prompt** (`promptType: define`):

```
Meeting: {{ $json.call_name }}
Company: {{ $json.company }}
Date: {{ $json.date }}
Attendees: {{ ($json.call_members || []).join(', ') }}

=== TRANSCRIPT ===
{{ $json.transcript_text }}
```

**System message.** The first paragraph is the per-company focus; write your own. The rest is the contract.

```
You are the knowledge extraction engine for {{COMPANY_NAME}}, a {{ONE_LINE_WHAT_THE_COMPANY_DOES}}. Focus on: partners, competitors, customers, team members, product topics that materially affect the roadmap, and strategic decisions. Extract blockers and risks as entity_type=issue. Preserve product and person proper nouns exactly.

Extract DURABLE business facts from meeting transcripts: facts that will still matter in 3+ months and belong in the company knowledge base. Ignore transient logistics, one-off task assignments, and small talk. Never invent numbers, dates, or names. If a fact is uncertain, mark sensitivity_hint=sensitive so a human reviews it. For every fact include a short verbatim quote (max 220 chars) from the transcript. Empty output is valid.

Output ONLY structured JSON per the output parser schema. No prose.
```

## Structured Output Parser

`jsonSchemaExample`:

```json
{
  "facts": [
    {
      "entity_type": "customer",
      "entity_name": "Example Co",
      "fact": "Concise one-sentence fact.",
      "quote": "verbatim source quote (max 220 chars)",
      "speaker": "Speaker Name",
      "sensitivity_hint": "safe",
      "confidence": "high"
    }
  ]
}
```

`entity_type` is one of `supplier | distributor | customer | competitor | person | issue | decision | strategy-signal | other`. `sensitivity_hint` is `safe | sensitive`. `confidence` is `high | medium | low`.

## Push Extracted Facts (Code node, JavaScript, run once for all items)

Writes `inbox/extracted/<date>-<meeting-slug>.md` to the brain repo through the GitHub Contents API. Idempotent: a re-run of the same meeting updates the same file (it fetches the existing `sha` first). The token needs `contents: write` on the brain repo only.

The node reads the pipeline's enriched item (`call_name`, `date`, `source_note_id`) from the node named in the first line; rename that to match your pipeline.

```js
// Stages extracted facts into OWNER/REPO/inbox/extracted/.
// Always-safe write (staging tier). Fully wrapped: a failure here must never break the pipeline.
try {
  const enriched = $('Match Users & Format').first().json || {};
  const out = $json.output || {};
  const facts = Array.isArray(out.facts) ? out.facts : [];

  const token = 'REPLACE_WITH_GITHUB_TOKEN'; // or read it from an n8n credential of your choice
  const org = 'OWNER';
  const repo = 'REPO';

  const rawDate = String(enriched.date || '');
  // pipelines often carry DD-MM-YY; normalise to YYYY-MM-DD
  let ymd;
  if (/^\d{2}-\d{2}-\d{2}$/.test(rawDate)) {
    const [dd, mm, yy] = rawDate.split('-');
    ymd = '20' + yy + '-' + mm + '-' + dd;
  } else if (/^\d{4}-\d{2}-\d{2}$/.test(rawDate)) {
    ymd = rawDate;
  } else {
    ymd = new Date().toISOString().split('T')[0];
  }

  const q = (s) => String(s == null ? '' : s).replace(/"/g, "'").replace(/\r?\n/g, ' ').trim();
  const callName = enriched.call_name || 'meeting';
  const slug = callName.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '').substring(0, 60);
  const fileName = 'inbox/extracted/' + ymd + '-' + slug + '.md';

  if (!facts.length) {
    return [{ json: { ...enriched, extract_file: '', extract_count: 0, extract_note: 'no durable facts extracted' } }];
  }

  let md = '---\n';
  md += 'type: extracted-facts\n';
  md += 'source_meeting_call_name: "' + q(callName) + '"\n';
  md += 'source_meeting_date: ' + ymd + '\n';
  md += 'source_note_id: "' + q(enriched.source_note_id || '') + '"\n';
  md += 'date: ' + ymd + '\n';
  md += 'curated: false\n';
  md += '---\n\n';
  md += '# Extracted facts - ' + q(callName) + ' (' + ymd + ')\n\n';
  md += 'Machine-written by the knowledge-extract tap. Folded into canonical pages per docs/CURATION.md.\n\n';
  facts.forEach((f, i) => {
    md += '## Fact ' + (i + 1) + '\n';
    md += '- entity_type: ' + q(f.entity_type || 'other') + '\n';
    md += '- entity_name: "' + q(f.entity_name) + '"\n';
    md += '- fact: "' + q(f.fact) + '"\n';
    md += '- quote: "' + q(f.quote) + '"\n';
    md += '- speaker: "' + q(f.speaker || '') + '"\n';
    md += '- sensitivity_hint: ' + (f.sensitivity_hint === 'sensitive' ? 'sensitive' : 'safe') + '\n';
    md += '- confidence: ' + (['high', 'medium', 'low'].includes(f.confidence) ? f.confidence : 'medium') + '\n\n';
  });

  const contentB64 = Buffer.from(md).toString('base64');
  const apiUrl = 'https://api.github.com/repos/' + org + '/' + repo + '/contents/' + fileName;
  const gh = { Authorization: 'Bearer ' + token, Accept: 'application/vnd.github.v3+json' };
  let sha = null;
  try {
    const existing = await this.helpers.httpRequest({ method: 'GET', url: apiUrl, headers: gh });
    sha = existing.sha;
  } catch (e) {}
  const body = { message: 'Stage extracted facts: ' + q(callName) + ' (' + ymd + ')', content: contentB64 };
  if (sha) body.sha = sha;
  const result = await this.helpers.httpRequest({ method: 'PUT', url: apiUrl, headers: gh, body });
  return [{ json: { ...enriched, extract_file: fileName, extract_count: facts.length, extract_commit: result?.commit?.sha || '' } }];
} catch (extract_error) {
  return [{ json: { extract_error: String(extract_error && extract_error.message || extract_error) } }];
}
```

## Why it is shaped this way

- **Staging, not canon.** The tap writes only to `inbox/extracted/`, the always-safe tier. Nothing the model extracts reaches a canonical page without the curator applying the tiers.
- **A quote per fact.** The curator, and later a person reading the page, can check every fact against the transcript in seconds. Facts without a quote are the ones that turn out to be invented.
- **`curated: false` is the dedup.** The curator flips it to `true`; the file stays as the provenance chain from transcript to canonical fact. No workflow-side state, which on n8n Cloud is unreliable.
- **Empty is valid.** A stand-up with no durable facts produces nothing. That is the correct result, and the summary step still runs.
