# Question format

Each card is one document in the `questions` collection of the docket's
database. The page reads every field below; anything else is ignored.

## Fields

| Field | Type | Written by | Notes |
| --- | --- | --- | --- |
| `title` | string | you | The question as a decision: "Whether…", "When…", "Which…". |
| `short` | string | you | Two to four words for the index. Falls back to `title`. |
| `priority` | `"now"` \| `"security"` \| `"review"` \| `"later"` | you | Sort order and pill colour. `now` means work is blocked on it. |
| `due` | string | you | Optional pill text, e.g. "Decide before Friday's deploy". Without it the pill reads "Blocks work", "Security", "Your review · asked …" or "No deadline · asked …". |
| `area` | string | you | A short label shown beside the pills ("Billing", "Infra"). |
| `asked` | string | you | When you asked, shown in the default pill. |
| `order` | number | you | Tiebreak within a priority; lower comes first. |
| `context` | array | you | See below. |
| `options` | array | you | `{key, label, detail?, cost?, gain?, recommended?}`. `recommended: true` shows a "Recommended" tag; a string shows that string instead. |
| `recommendation` | string | you | Shown in a callout as "My recommendation: …". |
| `ifNothing` | string | you | Shown as "If you do nothing: …". |
| `items` | array | you | `{key, label, what?, source?, recommend?, default?}`: a list where each row gets its own choice. |
| `itemChoices` | array | you | `{key, label, tone?}` for every item row; `tone: "danger"` colours that choice red. Default: Keep, Hide, Cut. |
| `calls` | array | you | `{call, what}` rows for an FYI card of decisions you already made. |
| `status` | `"open"` \| `"answered"` | both | Create as `"open"`. The page sets `"answered"`; "Change decision" sets it back to `"open"`. |
| `answer` | object | the page | `{choice?, label?, items?, note, at, by?}`. `note` is free text and may qualify the choice; `by` is the recorder's opaque user id. |

### `context`

An array rendered top to bottom:

- a plain string is a paragraph;
- `"### What happened"` starts a section with that uppercase label;
- `"- 14 charges failed"` is a bullet (consecutive bullets form one list);
- `{"table": {"head": ["Week", "Failed"], "rows": [["41", "14"]], "num": [1]}}`
  is a table, where `num` lists the columns to right-align as numbers.

Any string field may use `` `code` ``, `**bold**` and bare `https://` links.

A card can have `options`, `items`, both, or neither. With neither, the
page shows a free-text answer box, which suits "review" cards ("Read the
draft and tell me what's wrong").

## A decision card

```json
{
  "order": 1,
  "priority": "now",
  "due": "Decide before Friday's deploy",
  "area": "Billing",
  "asked": "Oct 20",
  "short": "Charge retries",
  "title": "Whether failed charges retry automatically",
  "context": [
    "Card charges that fail today stay failed until the customer updates their card. Last week 14 of 2,310 charges failed; 9 of them were soft declines that usually succeed on a second try.",
    "### The numbers",
    {"table": {"head": ["Decline kind", "Count", "Usually succeeds on retry"], "rows": [["Soft (insufficient funds, issuer busy)", "9", "Yes"], ["Hard (stolen, closed)", "5", "No"]], "num": [1]}},
    "### Why it's your call",
    "- Retrying charges a customer's card again without asking, which some customers notice and complain about.",
    "- It changes the dunning emails support already sends."
  ],
  "options": [
    {"key": "soft3", "label": "Retry soft declines up to three times over five days", "cost": "A few confused customers; support updates one email.", "gain": "Recovers most of the 9 soft declines a week (about $1,100 a month).", "recommended": true},
    {"key": "all3", "label": "Retry every decline three times", "cost": "Hard declines never succeed and raise fraud flags with the processor.", "gain": "Simplest rule."},
    {"key": "none", "label": "Keep today's behaviour", "cost": "The soft-decline revenue stays lost.", "gain": "No change for customers or support."}
  ],
  "recommendation": "Retry soft declines only. The processor's codes tell them apart, and retrying hard declines costs us standing with the processor for nothing.",
  "ifNothing": "Nothing retries, and Friday's deploy goes out without the retry job.",
  "status": "open"
}
```

## A list to sort

```json
{
  "order": 3,
  "priority": "later",
  "area": "Settings screen",
  "asked": "Oct 20",
  "short": "Settings to keep",
  "title": "Which settings stay on the account screen",
  "context": ["The sweep found 6 settings on the account screen. Each row is set to my recommendation; change any you disagree with and record once."],
  "itemChoices": [{"key": "keep", "label": "Keep"}, {"key": "advanced", "label": "Move to Advanced"}, {"key": "cut", "label": "Cut", "tone": "danger"}],
  "items": [
    {"key": "tz", "label": "Time zone", "what": "Used by every scheduled email.", "recommend": "keep", "default": "keep"},
    {"key": "beta", "label": "Beta features", "what": "Only two accounts have it on.", "source": "added in the March spike", "recommend": "cut", "default": "cut"}
  ],
  "status": "open"
}
```

The answer comes back as `answer.items`, e.g. `{"tz": "keep", "beta": "advanced"}`.

## A calls card (FYI)

```json
{
  "order": 9,
  "priority": "review",
  "area": "This week",
  "asked": "Oct 20",
  "short": "Calls I made",
  "title": "Whether the calls I made without you this week stand",
  "context": ["These couldn't wait for an answer. Each one is reversible; tell me which to reopen."],
  "calls": [
    {"call": "Pinned the queue library to 4.2", "what": "4.3 broke retries in staging; pinned until upstream fixes it."},
    {"call": "Renamed the `exports` bucket", "what": "The old name collided with the analytics team's bucket."}
  ],
  "options": [
    {"key": "keep", "label": "Keep all", "recommended": true},
    {"key": "reopen", "label": "Reopen some", "detail": "Say which in the note."}
  ],
  "status": "open"
}
```

## Reading answers back

```json
{"action": "query", "url": "<docket url>", "collection": "questions",
 "query": {"where": [["status", "==", "answered"]]}}
```

Each result carries its `version`; delete with that version as `if_version`
once you have acted and logged the ruling.
