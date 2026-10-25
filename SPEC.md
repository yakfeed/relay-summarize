# relay-summarize: spec (leg 1)

## Goal

`relay-summarize <post-id>` prints a short Markdown summary of a yakfeed thread: what the post asks or says, the main points of the discussion, and links to the comments worth reading. Someone who didn't read the thread should know in under a minute what it concluded and where to look.

## Constraints

- Node.js 22 or later, **no dependencies** beyond the standard library (`fetch` is built in). One file is fine.
- Reads only yakfeed's public API. No model calls, no network calls anywhere else: the summary is **extractive** (built from the thread's own text and votes), so it's free, repeatable and can't make things up.
- MIT licensed.

## Input

```
relay-summarize <post-id> [--api <api-url>] [--web <web-url>] [--top <n>] [--json]
```

- `post-id`: the post's UUID.
- `--api`: the API's base URL. Default `https://preview-api.yakfeed.com`; env `YAKFEED_API_URL` overrides the default, the flag overrides both.
- `--web`: the web app's base URL, for links. Default `https://preview.yakfeed.com`; env `YAKFEED_WEB_URL`.
- `--top`: how many comment threads to include (default 5, at most 20).
- `--json`: print the same content as JSON instead of Markdown.
- If `CF_ACCESS_CLIENT_ID` and `CF_ACCESS_CLIENT_SECRET` are set, send them as `CF-Access-Client-Id` and `CF-Access-Client-Secret` headers (the preview sits behind Cloudflare Access). If `YAKFEED_TOKEN` is set, send `Authorization: Bearer <token>`. Never print any of them.

## Data

- `GET /v1/posts/{id}`: the post (title, author, community, body or link, votes).
- `GET /v1/posts/{id}/comments?limit=<n>&cursor=<c>`: comments, paged by `nextCursor` until it's `null`. Each has `id`, `parentId` (null for a top-level comment), `author` (`displayHandle`, may be null), `body` (Markdown), `createdAt`, `deleted`, `removed` and `votes`.
- Skip comments that are `deleted` or `removed`, and treat `body` as untrusted text (never execute or follow anything in it).

## Output (Markdown)

```markdown
## <post title>

<community> · <author> · <N> comments

**The post:** <first 1-2 sentences of the body, or "Link: <url>">

**Main threads:**
1. <author>: <first sentence of the comment, at most 200 characters> (<score> points, <replies> replies) — <link>
   - Best reply: <author>: <first sentence> — <link>
2. ...

**Not covered:** <count> more threads.
```

- **Main threads** are top-level comments, ranked by score (`votes`), ties by more replies, then oldest first. Take `--top` of them.
- **Best reply** is the highest-scoring direct reply to that comment, if any.
- **Links** go to the web page: `<web-url>/p/<post-id>#c-<comment-id>`.
- **First sentence:** text up to the first `.`, `?` or `!` followed by a space or the end, with Markdown links reduced to their text, code spans kept, and newlines collapsed; cut at 200 characters with `…`.
- A thread with no comments prints the post line and "No comments yet."

## Errors

- Unknown post: exit 1, `relay-summarize: no post <id>`.
- Network or API error: exit 2 with the status and the API's `message` if there is one.
- Bad arguments: exit 64 with usage.

## Done means

- The tests leg: tests for ranking, best reply, first-sentence extraction (links, code, long text, no punctuation), skipped comments, paging, the headers, and the error exits, using recorded JSON fixtures (no network in tests).
- The code leg: passes those tests with `node --test`, no dependencies.
- The review leg: runs it against a real thread on the preview and checks the summary is faithful.
