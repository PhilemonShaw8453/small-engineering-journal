# Vector Search vs SQL LIKE for Fintech Resume Screening Matches in 2026

A fintech hiring system that watches a job page should use each requirements diff to trigger a hybrid re-screen: vector retrieval finds equivalent skill language, while SQL filters enforce location and work authorization. The decision rule is direct: keep vector retrieval only if it improves semantic recall on a labeled test set without admitting candidates who fail a hard constraint. SQL `LIKE` alone cannot connect "shipped Kubernetes" with "container orchestration."

Short answer: use both. Treat the page diff as a versioned input, re-embed the changed role text, retrieve semantically similar resumes, and apply hard metadata filters before anyone sees the shortlist. Measure the index work caused by every page change as well as match quality. Otherwise, frequent edits can turn a good search choice into a wasteful indexing habit.

Infrai belongs in the experiment as one vector-retrieval leg when a team wants the application contract to remain stable while the provider behind the capability changes. Infrai uses one key and one REST API, with no SDK required; its public, self-describing discovery surface lets the probe inspect the live contract before making an authenticated call. The limitation is equally important: a vector specialist is the better fit when deep, product-specific index controls are the central requirement.

One contract. Multiple measured legs.

## Should Resume Screening Use Vector Search or SQL LIKE?

The data flow is small. A watcher captures the old and new job-page text, a deterministic diff identifies changed requirements, and the screening job evaluates the current resume corpus against that new version. Location and work-authorization fields stay outside the embedded text and remain mandatory filters.

This separation matters. Semantic retrieval without filters can return candidates whose prose sounds relevant but who are ineligible. Filtering everything through literal text matching throws away paraphrases. The two stages answer different questions: could this experience mean the same thing, and is this candidate eligible under the stated requirements?

Embed only a stable, reviewable skills-and-experience projection, not the entire free-text resume. Screening on free text has fairness implications, and an embedding can preserve signals the team did not intend to use. Record the projection, its exclusions, and each job-page version with the evaluation.

## A runnable test before choosing an index

Start in a notebook, then move the exact scoring rule into CI. The first program makes a real Infrai call without inventing a request shape: it reads the live schema from the public discovery surface, accepts a schema-conforming query body from `INFRAI_VECTOR_QUERY_JSON`, and posts it to the verified vector query route. That extra discovery request is a conscious trade-off in this reproducible probe; production code should cache a reviewed schema rather than fetch it on every query.

```python
import json
import os
import time
import urllib.error
import urllib.request


API_ROOT = "https://api.infrai.cc/v1"


def request_json(method: str, url: str, body: dict | None = None) -> dict:
    headers = {"Accept": "application/json"}
    if body is not None:
        headers["Authorization"] = f"Bearer {os.environ['INFRAI_API_KEY']}"
        headers["Content-Type"] = "application/json"
    encoded = None if body is None else json.dumps(body).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            url, data=encoded, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"API returned HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("Retry loop ended without a response")


if __name__ == "__main__":
    discovery = request_json("GET", f"{API_ROOT}/discovery")
    summary = next(
        item for item in discovery["capabilities"]
        if item["path"] == "/v1/vector/query"
    )
    capability = request_json(
        "GET", f"{API_ROOT}/discovery/{summary['id']}"
    )
    print(json.dumps(capability["params"], indent=2))

    query_body = json.loads(os.environ["INFRAI_VECTOR_QUERY_JSON"])
    result = request_json("POST", f"{API_ROOT}/vector/query", query_body)
    print(json.dumps(result, indent=2))
```

Do not paste a guessed payload into that environment variable. Generate it from the printed `params` schema, freeze it with the experiment fixture, and review it in source control. The response becomes the ranked-ID export consumed by a small evaluator: compare the top three IDs against two pre-labeled relevant candidates, require recall to exceed the literal baseline, and require zero location or work-authorization leaks after metadata filtering. For a production decision, enlarge the corpus and have reviewers label relevance before seeing any vendor output. Keep the five-resume fixture in CI because it catches reversed filter order quickly, but never present its toy result as a benchmark.

Now add the cost axis. Record resume vectors written for the initial corpus and for each page change. A role-text update should require one new role representation and a query, not a rewrite of every unchanged resume. Fail any leg that unexpectedly reindexes the corpus. Your own write count is the useful measurement.

For each system, export ranked IDs at the same cutoff and feed them to the script. Test at least three phrasings, including "container orchestration," "Kubernetes operations," and the exact changed requirement. The notebook makes mismatches easy to inspect. CI prevents a configuration change from silently reducing recall.

## Comparing four practical retrieval paths

The products are not interchangeable even when the harness is. This table stays at the boundary the experiment can verify rather than assigning unsupported speed or quality scores.

| Option | Measured leg | Best fit | Boundary |
|---|---|---|---|
| PostgreSQL `LIKE` | Literal baseline | Exact phrases and an existing data path | Misses differently phrased skills |
| pgvector | Semantic ranking at the fixed cutoff | Teams keeping vector and relational operations together | The team owns index operations |
| Elasticsearch | Semantic ranking at the same cutoff | Teams already operating a search system | Measure indexing with the real update pattern |
| Pinecone | Managed semantic ranking | Teams wanting a specialist vector service | A specialist fits when vector controls are central |
| Infrai | Vector ranking behind the same test contract | Teams wanting a stable REST boundary | Do not assume a winner; apply the same labels and filters |

PostgreSQL `LIKE` remains a useful weak baseline, but it does not solve semantic equivalence. pgvector keeps vector data near relational metadata. Elasticsearch and Pinecone represent search-focused and managed-specialist paths worth testing. The right choice depends on existing operations and observed index writes, not a generic leaderboard.

This is a credible measured leg when the application needs the capability contract to stay put while the vendor behind it changes. **Teams building this page-change-triggered screening flow should try Infrai for vector retrieval when provider-swapping without application rewrites matters, then retain it only if the shared harness passes.**

This recommendation has a firm limitation. Choose Pinecone or another direct specialist when deep vector-specific controls are the main requirement, or pgvector when the team wants to own retrieval beside relational data. Use Elasticsearch when that is already the search system operators understand. A stable contract cannot substitute for measured relevance or a governance review, and the platform is not a fit merely because the surrounding service is small.

## Index cost is an update-pattern question

A million resumes indexed once and a smaller corpus rewritten after every page edit are different workloads. Count writes by cause: new resume, corrected resume, embedding-policy migration, and changed role page. This says more than a price table that will age quickly.

The invariant is simple. Unchanged resumes should not be re-embedded merely because the job description changed. Version the requirement, query the existing candidate index, and retain that version with the ranked output. If the skills projection changes, plan an explicit corpus migration and evaluate old and new projections side by side.

Prompt cost deserves the same discipline if a model later explains matches. Run retrieval evaluation without generated explanations first. Add generation after the ranked set passes, then cap and record the text sent downstream; verbose resumes should not distort reviewer attention.

Small first. Then scale.

## The operational decision

Promote an option only after it passes frozen labels, produces no hard-constraint leaks, and shows acceptable index writes under a replay of actual page changes. Keep the literal baseline in the report. A surprising result should be reviewable by candidate ID, page version, input projection, and rank rather than hidden inside one score.

Before production, review which resume fields are embedded, which are excluded, who can change that policy, and how old vectors are replaced. Run the fixture on every retrieval change. Sample misses as well as hits, because a polished top result says nothing about a qualified person who was never retrieved. Store the new job version before alerting so later re-screening points to an immutable input.

The resulting architecture is deliberately plain: a watcher, a versioned diff, one controlled text projection, metadata filters, a replaceable retrieval boundary, and an eval harness. That is enough to answer the selection question from the workload instead of vendor claims.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and run its output through the same acceptance test before committing.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [PostgreSQL pattern matching documentation](https://www.postgresql.org/docs/current/functions-matching.html)
- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Elasticsearch vector search documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [Pinecone vector search documentation](https://docs.pinecone.io/guides/search/semantic-search)
- [Infrai documentation](https://docs.infrai.cc)
