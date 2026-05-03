---
name: daily-report-en
description: Fetches activities from the Acteedog MCP server and generates a structured daily report. Use for requests like "daily report", "today's recap", "what did I do yesterday", or any request to generate or save a daily activity summary.
---

# Daily Report Skill

Fetches activity data from the Acteedog MCP server and generates a structured daily report in markdown format.

## Workflow

### 1. Determine the Target Date

Identify the target date from the user's request. Default to the previous day (yesterday) if not specified.

### 2. Fetch Activities

Use `get_activities` to retrieve data for the target date.

- Data exists and is sufficient → proceed to step 3
- Data is missing or insufficient → use `trigger_activity_collection` to trigger jobs, then use `get_job_status` to confirm all jobs complete before re-fetching

#### Waiting for Jobs to Complete

```
1. Use trigger_activity_collection to get job IDs for all connectors
2. Wait ~15 seconds, then check all job statuses in parallel
3. If any jobs are still pending, wait and check again
4. Once all jobs complete, re-fetch with get_activities
```

### 3. Analyze Data

Activity data can be very large (tens of thousands to hundreds of thousands of bytes). **Always use code execution (Bash + python3) to analyze it.** Do not load it directly into context.

#### Response Data Structure from `get_activities`

`get_activities` returns the path to a cache file. The JSON inside has the following structure:

```jsonc
{
  "start_date": "2026-04-10",
  "end_date": "2026-04-10",
  "total_activities": 82,
  "groups": [              // Activities grouped by context
    {
      "activity_count": 11,
      "context_chain": [   // Hierarchy: source → channel/repo → thread/PR (elements may be null)
        {
          "id": "slack:source",
          "name": "slack:source",          // optional
          "description": "Activity source from Slack",  // optional, may be null
          "url": "https://yourorgname.slack.com"        // optional
        },
        { "id": "slack:channel:CXXXXXXXX", "url": "https://yourorgname.slack.com/archives/CXXXXXXXX" },
        { "id": "slack:thread:CXXXXXXXX:123456.789", "description": "First message in thread", "url": "..." }
      ],
      "activities": [      // Individual activities
        {
          "description": "Message content or action",  // may be null
          "timestamp": "2026-04-10T06:36:28.542649030+00:00",
          "url": "https://yourorgname.slack.com/archives/..."
        }
      ],
      "relations": {       // NOTE: dict, not list
        "activities": { "outbound": [], "referenced_by": [] },
        "contexts": {
          "outbound": [    // External contexts referenced by this group
            {
              "connector_id": "github",
              "id": "github:pull_request:yourorgname/reponame:123",
              "resource_type": "pull_request",
              "title": "PR#123: Fix issue with user authentication",
              "url": "https://github.com/yourorgname/reponame/pull/123"
            }
          ],
          "referenced_by": []  // References from other contexts
        }
      }
    }
  ]
}
```

#### Analysis Steps

**Step 1: Get an overview of the data**

```python
import json
from collections import Counter

with open(FILE_PATH) as f:
    data = json.load(f)

groups = data['groups']
print(f"Total activities: {data['total_activities']}, Total groups: {len(groups)}")

# Activity count by source
sources = Counter()
for g in groups:
    chain = g.get('context_chain', [])
    if chain and chain[0]:
        sources[chain[0].get('id', '?').split(':')[0]] += g.get('activity_count', 0)
for src, cnt in sources.most_common():
    print(f"  {src}: {cnt}")

# List groups sorted by activity count (descending)
for i, g in sorted(enumerate(groups), key=lambda x: x[1].get('activity_count', 0), reverse=True):
    chain = [c.get('name', c.get('id', '?')) for c in g.get('context_chain', []) if c]
    desc = (g['activities'][0].get('description') or '')[:100] if g.get('activities') else ''
    print(f"[{i}] ({g.get('activity_count',0)}) {' → '.join(chain)}")
    if desc:
        print(f"    {desc}")
```

**Step 2: Inspect important groups in detail**

Identify important groups using these criteria:

- High activity count (indicates depth of discussion)
- Groups involving deliverables like GitHub PR reviews
- Activities spanning multiple contexts

Information to extract from each group:
- Context chain (source → channel/repo → thread/PR)
- `description`, `timestamp`, `url` for each activity
- For PRs: `description` in `context_chain` (purpose and changes of the PR)
- Related contexts via `relations`

```python
important_indices = [...]  # Selected from Step 1 results

for idx in important_indices:
    g = groups[idx]
    print(f"\n{'='*80}")
    print(f"[Group {idx}] Activities: {g.get('activity_count', 0)}")

    # Context chain
    for c in g.get('context_chain', []):
        if c is None:
            continue
        print(f"  - {c.get('id','?')}: {c.get('name', '')}")
        desc = (c.get('description') or '')[:200]
        if desc:
            print(f"    desc: {desc}")
        if c.get('url'):
            print(f"    url: {c['url']}")

    # Relations (note: dict structure)
    rels = g.get('relations', {})
    if isinstance(rels, dict):
        for direction in ['outbound', 'referenced_by']:
            items = rels.get('contexts', {}).get(direction, [])
            for r in items:
                print(f"  relation({direction}): {r.get('title','')} - {r.get('url','')}")

    # Activities
    for a in g.get('activities', []):
        if a is None:
            continue
        desc = (a.get('description') or '')[:300]
        print(f"  [{a.get('timestamp','')}] {desc}")
        if a.get('url'):
            print(f"    url: {a['url']}")
```

**Step 3: Analyze cross-context relationships**

- Group activities that belong to the same project
- Match recurring meeting check-ins with corresponding Slack thread activity
- Link calendar events to their related Slack threads

### 4. Generate the Report

Use the following format:

```markdown
## yyyy-mm-dd

### Key Achievements

- **Drove [initiative]**: Specific outcome summary ([link](URL))
- ... (3–5 items)

### Project & Activity Details

#### [Project Name]

Summary of activity across this project.

- **Specific activity** ([thread](URL) / [PR](URL))
  - Concrete description of what you said or contributed
  - Technical perspective on the contribution

#### Cross-cutting & Other

- **Activity** ([link](URL)): Specific details
```

#### Guidelines on Level of Detail

The value of a daily report is not *what* you did, but *how* you contributed.

- **Bad**: "Reviewed PR#123" / "Discussed in Slack"
- **Good**: "Reviewed PR#123's feature flag logic, verified the toggle behavior matched the spec, and approved"
- **Good**: "Pointed out a technical constraint: having multiple classifications at the same level would break structural integrity guarantees"

Specifically:
- PR reviews → what the PR was about, what you checked during review, comments or feedback you gave
- Slack discussions → what arguments you raised, what proposals you made
- Mob programming → knowledge you shared, code you referenced, insights you surfaced

#### Grouping Related Contexts

Rather than listing isolated activities, group related ones:
- Calendar meeting + its corresponding Slack thread → describe together
- Recurring meeting plan + actual work thread → describe as a single flow

#### Include URLs

Include a URL for every cited source. Use the `url` field from activities and contexts: Slack threads, GitHub PRs, calendar events, etc.

### 5. Confirm and Save

Present the generated report to the user for review, then save it using `save_daily_report`.
