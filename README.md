# klaracase examples

Ready-to-use examples for the klaracase API.

klaracase holds German court decisions, German statutes and German administrative guidance (for example BMF letters). It covers German law only. The API returns sources with a citation (Fundstelle) and a link to each source. Website: https://klaracase.de

klaracase is a research tool. It gives no legal advice and does not replace a lawyer's review. Open every source and read the original text before you use it.

**Deutsch:** Dieses Repository sammelt fertige Beispiele für die klaracase API, zum Beispiel Flows für n8n und Zapier und später größere Anwendungen. Jedes Beispiel liegt in einem eigenen Ordner mit Anleitung. Die Anleitungen im Ordner sind auf Englisch, die Notiz im n8n-Flow ist zweisprachig. Die Dokumentation zur API finden Sie unter https://klaracase.de/docs.

## Examples

| Example | What it does | Tool |
| --- | --- | --- |
| [`n8n-gmail-legal-email-triage`](n8n-gmail-legal-email-triage) | Reads Gmail messages that carry a legal question, searches klaracase with the subject line, and saves a draft reply with the sources (Fundstelle and link). It sends nothing. | n8n, Gmail |

More examples will follow, each in its own folder.

## Get an API token

1. Create an account at https://klaracase.de.
2. Create an API token with the `search` ability in the API token settings of your account.
3. Store the token in the credential store of your tool. Never write it into a flow, a file or a commit.

The API reference is at https://klaracase.de/docs.

## Costs

The API uses credits.

- One search costs 1 credit.
- The same search again in the same credit period costs 0.
- A search with no hits costs 0.
- When your credits run out, the API answers with status 402 and charges nothing.

New accounts get free starter credits. Check https://klaracase.de/docs for the current terms.

## Propose an example

Open an issue with the title of the example, the tool you want to use, and the task it should solve. A pull request with a new folder is welcome too.

Each example folder needs:

- a search-friendly folder name, for example `n8n-gmail-legal-email-triage`,
- a `README.md` that says what it does, what was tested and what it costs,
- no secrets, no real credential ids and no client data.

## Privacy

Only the text that you send in a query goes to klaracase. Never send client names or confidential text in a query. Each example README says which text it sends.

## License

MIT. See [LICENSE](LICENSE).
