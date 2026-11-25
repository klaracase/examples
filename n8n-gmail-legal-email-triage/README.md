# E-mail triage with Gmail and n8n for German legal questions

An n8n workflow that reads Gmail messages with a legal question, searches the klaracase API, and saves a draft reply with the sources. Each source has a citation (Fundstelle) and a link. The workflow sends nothing. You review the draft and open every source before you reply.

klaracase covers German court decisions, German statutes and German administrative guidance. It is a research tool. It gives no legal advice. It does not check whether a decision is still good law.

## What the flow does

```
Gmail Trigger -> Search klaracase -> Any hits? -> Gmail: create draft -> Gmail: add label
                                         |
                                         no hits: the flow ends, no draft
```

1. **Gmail Trigger** starts the flow for each mail with the Gmail label `rechtsfrage`.
2. **Search klaracase** calls `GET https://klaracase.de/api/v1/search` with the subject line and `limit=3`.
3. **Any hits?** stops the flow when the search found nothing, so no empty draft is made.
4. **Gmail: create draft** saves a reply draft in the same thread. The text lists each Fundstelle with its link.
5. **Gmail: add label** puts the label `klaracase-geprüft` on the mail.

The file also holds helper nodes. A `Build draft text` node writes the draft text. A `Mail` node gives both paths the same name to read mail fields from. A manual trigger and a `Sample mail (test)` node form a test path that needs no Gmail account. A sticky note explains the setup in English and German.

## Prerequisites

- n8n 2.x. Tested with 2.49.5.
- A klaracase account and an API token with the `search` ability. See the [repository README](../README.md#get-an-api-token).
- For the live path: a Gmail account and a Google OAuth2 credential in n8n.
- Two Gmail labels: `rechtsfrage` and `klaracase-geprüft`. Only mails with `rechtsfrage` start the flow. This controls what the flow reads and when it spends credits.

## Import

1. In n8n, create a new workflow. Open the menu and choose "Import from file". Select [`workflow.json`](workflow.json).
2. Create a credential of the type "Header Auth". Name the header `Authorization`. Set the value to `Bearer <your token>`. Select this credential in the node `Search klaracase`.
3. Test first. Select the node `Manual Trigger (test)` and run the workflow. The sample mail has a sample subject. The flow stops at `Build draft text`, because the Gmail nodes are disabled in the file.
4. To go live, add your Google OAuth2 credential to `Gmail Trigger`, `Gmail: create draft` and `Gmail: add label`. Enable these three nodes. Pick the label `klaracase-geprüft` in `Gmail: add label`. Disable `Manual Trigger (test)` and `Sample mail (test)`. Then activate the workflow.

The Gmail nodes are disabled in the template on purpose. This way the file imports without a Google credential and the test path runs first.

## Field settings

In n8n, an expression is a field in "Expression" mode. The code sits between two opening and two closing curly braces.

**Gmail Trigger**

- Event: "Message Received". Set the interval under "Poll Times".
- Filters, "Search": `label:rechtsfrage`
- Filters, "Read Status": "Unread and read emails". Without this, a mail you already opened does not start the flow.
- "Simplify" stays on. It returns `Subject`, `From`, `id` and `threadId`. It returns no mail body.

**Search klaracase** (HTTP Request)

- Method: GET. URL: `https://klaracase.de/api/v1/search`
- Authentication: "Generic Credential Type". Generic Auth Type: "Header Auth".
- "Send Query Parameters" is on.
- `query`: `{{ $json.Subject.slice(0, 500) }}`. This cuts the subject at 500 characters, the API limit.
- `limit`: `3`. The limit counts per section. `limit=3` gives up to 3 hits each for decisions, laws and guidance.

**Any hits?** (If)

- Condition, "Number", "is greater than" `0`, on `{{ Object.values($json.sections).flat().length }}`
- A search without hits returns empty lists. This step stops the flow there.

**Gmail: create draft** (resource "Draft", operation "Create")

- Subject: `Re: {{ $('Mail').item.json.Subject }}`
- Email Type: "Text"
- Message: the output of `Build draft text`, which is `{{ Object.values($json.sections).flat().map(h => '- ' + h.fundstelle + '\n  ' + h.corpus_url).join('\n') }}` under a short German line. A single dragged field would insert only the first hit. This expression lists all hits.
- Options, "Thread ID": `{{ $('Mail').item.json.threadId }}`
- Options, "To Email": `{{ $('Mail').item.json.From }}`
- After the HTTP Request, `$json` is the API answer. The mail fields come from the node `Mail` by its name. If you build the flow by hand, you can use `$('Gmail Trigger')` instead of `$('Mail')`.

**Gmail: add label** (resource "Message", operation "Add Label")

- Message ID: `{{ $('Mail').item.json.id }}`
- Label Names or IDs: pick `klaracase-geprüft` from the list. The label must exist in Gmail first.

## What the API returns

The answer has `sections.decisions`, `sections.laws` and `sections.guidance`. A section without hits can be missing. Each hit has `fundstelle` and `corpus_url`. The example is shortened. Values are placeholders.

```json
{
  "sections": {
    "decisions": [
      { "fundstelle": "<citation>", "corpus_url": "<link to the source>" }
    ],
    "laws": [
      { "fundstelle": "<citation>", "corpus_url": "<link to the source>" }
    ]
  },
  "credits": { "metered": true, "cost": 1, "credits_remaining": "<number>" }
}
```

## What was tested

- Tested: the flow ran end to end in n8n 2.49.5, with the manual trigger and a sample mail, against the klaracase API. It ran from the sample mail to the draft text.
- Not tested: the Gmail nodes. They follow the n8n node definition. They were not run against a live Gmail account. Run them once with a test mail before you rely on them.

## Costs

- One search costs 1 credit.
- The same search again in the same credit period costs 0. A search with no hits costs 0.
- The `credits` field in the answer shows the cost and the credits left.
- The trigger runs one search for each new mail with the label `rechtsfrage`. Use a token with the `search` ability only. If your organization allows it, set a credit limit for the token.

## Privacy

The flow sends only the subject line as the query. It sends no mail body and no sender.

Never send client names or confidential text in a query. A subject line can hold such text. Check your subject lines, or change the `query` expression so that it sends a neutral question only.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Status 402 | Your credits are used up. The API charges nothing and the flow stops. Wait for the next credit period or add credits. |
| Status 422 | The query is shorter than 2 characters or longer than 500. The API rejects it. The template cuts at 500, but it does not check for a very short subject. |
| Status 401 | The token is missing, wrong or without the `search` ability. Check the Header Auth credential. The value must start with `Bearer `. |
| No draft, flow ends at `Any hits?` | The search found nothing. This is a normal result and costs 0 credits. Try a subject with German legal terms. |
| A mail does not start the flow | Check that it has the label `rechtsfrage`. Check that "Read Status" is "Unread and read emails". A mail that was already read does not start the flow with the default setting. |
| The label step fails | Create the label `klaracase-geprüft` in Gmail first. Then pick it in `Gmail: add label`. |

## Before you use a draft

klaracase does not check whether a decision is still good law. Open every source with its link. Read the original text. Then decide.

## License

MIT. See [LICENSE](../LICENSE).
