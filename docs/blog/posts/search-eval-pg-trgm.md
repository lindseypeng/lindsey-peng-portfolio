---
date: 2026-07-13
authors:
  - lindseypeng
categories:
  - Data Science
  - Engineering
---

# Is PostgreSQL pg_trgm Good Enough for Fuzzy Search?

*How a simple data science habit—a tiny eval set and accuracy@k—helped engineers and PMs turn "this looks okay" into a defensible decision in one hour.*

---

## The problem

Our product lets users search a database of emission factor names. Think entries like "Copper products", "Crude petroleum and services related to crude oil extraction, excluding surveying", or "Aluminium and aluminium products". Users type short, messy queries: typos ("cupper"), partial names ("crude petroleum"), regional spellings ("aluminum" vs "aluminium").

We were migrating search off a dedicated search engine and onto plain Postgres. The question on the table:

> Can an out-of-the-box Postgres extension (`pg_trgm`) give us good enough search, or do we need something heavier: embeddings, a vector database, a Hugging Face model, a custom bag-of-words ranker?

In most teams, this conversation goes one way. Someone types three queries into a staging environment, squints at the results, and says "looks fine to me". Someone else tries a typo, gets nothing back, and says "hmm, not sure". The discussion ends on intuition, and the decision inherits that uncertainty.

<!-- more -->

Engineers and PMs are very good at building things. But "is it good enough?" is not a building question. It is an **evaluation** question, and evaluation is where a data science background pays off, even when there is no machine learning anywhere in sight.

## The approach: treat it like a model evaluation

I did not write a single line of application code for this. I did three things any data scientist does before trusting a model:

### 1. Build a tiny, representative eval set

I wrote down **10 test cases** against a 200-item public dataset (Exiobase sector names). Each case is a (query, expected result) pair, and each one stresses a different failure mode:

| # | Variation type | Query | Expected result | What it stresses |
|---|---|---|---|---|
| 1 | Exact match (control) | copper products | Copper products | Sanity check |
| 4 | Single-letter typo | weat | Wheat | Typos on short words |
| 5 | Common misspelling | cupper | Copper products | Vowel swaps |
| 6 | Regional spelling | aluminum | Aluminium and aluminium products | US vs UK spelling |
| 8 | Keyword in long name | crude petroleum | Crude petroleum and services related to... | Short query vs long target |
| 9 | Typo + partial | natual gas | Natural gas and services related to... | Realistic worst case |

That is it. Not 10,000 labeled examples. Ten rows in a Google Sheet, written in about an hour, covering the query patterns we actually expect from users.

### 2. Pick a metric everyone can understand: accuracy@k

For each query, I checked: **does the correct item appear in the top k results?**

- **Accuracy@1**: the right answer is the first result
- **Accuracy@5**: the right answer is somewhere in the top 5
- **Accuracy@10**: the right answer is somewhere on the first page

This is a standard information retrieval metric, but the reason I like it here is that it maps directly to product experience. Accuracy@1 is "the user sees the answer immediately". Accuracy@10 is "the user finds it without scrolling past page one". A PM does not need any statistics background to reason about those two sentences.

### 3. Run every candidate against the same set

Four candidates, all pure Postgres:

- **ILIKE**: the naive `WHERE name ILIKE '%query%'` substring match (our baseline)
- **pg_trgm whole-string similarity**: `similarity(name, query)`
- **pg_trgm word similarity**: `word_similarity(query, name)`, which matches the query against the best span inside a name
- **Combined**: take the best score of the two

## The results

| Method | Acc@1 | Acc@5 | Acc@10 | Avg query time |
|---|---|---|---|---|
| ILIKE (baseline) | 45% | 55% | 55% | 0.40 ms |
| pg_trgm whole-string | 82% | 91% | 100% | 1.30 ms |
| pg_trgm word_similarity | 73% | 100% | 100% | 1.87 ms |
| Combined | 73% | 100% | 100% | 3.17 ms |

Suddenly the vague debate has sharp edges:

**ILIKE fails every single typo.** Substring matching returns nothing for "cupper", "weat", "aluminum", or "natual gas". That caps it at 45% accuracy@1. Nobody needs to argue about whether the baseline is good enough. It is not, and here is the number.

**pg_trgm gets the right answer onto page one 100% of the time.** For roughly 1 extra millisecond per query. That single row is the answer to "do we need embeddings?": not for this use case, not yet.

**The two pg_trgm modes have opposite strengths, and the eval set shows exactly where.** Whole-string similarity wins on short typo queries. Word similarity wins when a short query targets a long name ("crude petroleum" vs the 80-character official entry). And it produces a fun failure: for "weat", word similarity ranks "Wearing apparel; furs" above "Wheat", because one missing letter destroys most of a four-letter word's trigrams. You only catch a failure like that if you have a test case designed to stress it.

**Combining both modes bought nothing.** Same accuracy@5 and @10 as word similarity alone, at more than double the query time. Without measurement, "combine both, best of both worlds" sounds obviously right. With measurement, it is obviously not worth it.

**And one honest limitation, stated up front:** none of this captures meaning. Searching "metal" will never return "Copper". Trigrams match letters, not concepts. That is the line where embeddings would actually earn their complexity, and now we know precisely where that line is instead of vaguely gesturing at it.

## The decision

The write-up that went to the implementation team was short:

1. **Ship pg_trgm.** Either mode works; the eval sheet shows which query patterns each one favors, so the team could pick based on which patterns matter most to the product.
2. **Set the similarity threshold to 0.1.** Even the threshold became a quantified trade-off rather than a magic number: a high threshold gives fewer, stronger matches but sometimes returns 1 or 0 results on a badly typed query; a low threshold reliably fills all 10 slots, at the cost of weaker matches at the bottom of the list.
3. **Skip embeddings for now.** We know exactly what we are giving up (semantic matching) and exactly what we are getting (100% accuracy@10 on realistic queries at millisecond latency, with zero new infrastructure).

Total effort: one afternoon and a spreadsheet.

## The takeaway

The valuable part here was not pg_trgm knowledge. Any engineer can read the Postgres docs. The valuable part was the reflex to ask: **what would convince us this is good enough, and can we measure it?**

You do not need an ML pipeline, a labeled dataset with thousands of rows, or an eval framework. You need:

1. **10 to 15 test cases** that represent real user behavior, including the ugly cases (typos, partial queries), written down *before* you look at results
2. **One metric that maps to user experience** (accuracy@k is hard to beat for search)
3. **The same test set run against every candidate**, baseline included

That is the whole method. It converts "this looks okay" into "82% of queries return the right answer as the top result, 100% land on page one, and here are the three cases where it fails and why". One of those statements ends a meeting. The other one starts a longer one.

As more teams evaluate whether simple tools can replace complex AI ones (and increasingly, whether AI-generated solutions actually work), this small evaluation habit might be the most transferable skill data scientists bring to engineering teams.
