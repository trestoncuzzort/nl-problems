# nl-problems — natural-language programming problems, four public sources

**24,648 problems** collected from four public benchmark datasets, normalised to one
gzipped JSONL per split. Every file is one JSON object per line.

Assembled on `cs-rahman-dell` on 2026-08-27. Counts and checksums in
[`manifest.json`](manifest.json) are measured from these exact files, not copied from
upstream documentation.

## Files

| file | records | raw | gzipped |
|---|---:|---:|---:|
| `data/apps_raw_test.jsonl.gz` | 5,000 | 1,292.4 MB | 429.1 MB |
| `data/apps_raw_train.jsonl.gz` | 5,000 | 107.1 MB | 26.0 MB |
| `data/codecontests_train.jsonl.gz` | 13,328 | 28.3 MB | 8.1 MB |
| `data/codecontests_test.jsonl.gz` | 165 | 0.4 MB | 0.1 MB |
| `data/codecontests_valid.jsonl.gz` | 117 | 0.3 MB | 0.1 MB |
| `data/mbpp_test.jsonl.gz` | 500 | 0.3 MB | 0.1 MB |
| `data/mbpp.jsonl.gz` | 374 | 0.2 MB | 0.0 MB |
| `data/humaneval.jsonl.gz` | 164 | 0.2 MB | 0.0 MB |
| **total** | **24,648** | **1,429.2 MB** | **443.4 MB** |

`apps_raw_test` is 92% of the raw bulk on its own. That is not 5,000 unusually long
problem statements — it is the `input_output` field, which carries the full judge test
data (many problems ship hundreds of input/output cases). If you only need statements and
reference solutions, drop that one field and the split becomes small.

## Why gzip + Git LFS

GitHub refuses any single file over 100 MB through normal Git, and `apps_raw_test.jsonl`
is 1.29 GB. Both constraints are handled here: the files are gzipped (1.43 GB → 443 MB)
and tracked with Git LFS.

Gzip costs you nothing at read time — `gzip.open()` streams line by line, so you never
need to decompress to disk:

```python
import gzip, json

with gzip.open("data/apps_raw_train.jsonl.gz", "rt") as f:
    for line in f:
        rec = json.loads(line)
        print(rec["question"][:200])
```

To clone:

```bash
git lfs install
git clone <repo-url>
git lfs pull          # if the data/ files come down as pointer stubs
```

## Schemas

Fields are verified against the first record of every file; they are consistent within
each family.

**APPS** (`apps_raw_*`) — Codeforces, Kattis, AtCoder and similar, scraped with judge data.

| field | type | notes |
|---|---|---|
| `id` | int | index within the split |
| `question` | str | the problem statement |
| `solutions` | str | **JSON-encoded string** holding a list of reference solutions — parse it a second time |
| `input_output` | str | **JSON-encoded string** `{"inputs": [...], "outputs": [...]}` — the judge test data, and the reason this split is large |
| `difficulty` | str | e.g. `introductory`, `interview`, `competition` |
| `url` | str | the originating problem page, e.g. a `codeforces.com/problemset/...` link |
| `starter_code` | str | often empty |

Note the double encoding on `solutions` and `input_output`: they are strings containing
JSON, not nested objects. `json.loads(rec["input_output"])` is required.

**CodeContests** (`codecontests_*`) — DeepMind's competitive-programming set.

| field | type | notes |
|---|---|---|
| `name` | str | problem slug |
| `description` | str | the problem statement |
| `public_tests` | dict | `{"input": [...], "output": [...]}` — a real nested object here, not a string |
| `difficulty` | int | |
| `cf_rating` | int | Codeforces rating; `0` where unknown |

**HumanEval** (`humaneval.jsonl.gz`) — 164 records, which matches the canonical size.

| field | type | notes |
|---|---|---|
| `task_id` | str | e.g. `HumanEval/0` |
| `prompt` | str | signature + docstring, the model input |
| `canonical_solution` | str | reference body only, not the full function |
| `test` | str | a `check(candidate)` harness |
| `entry_point` | str | function name to call |

**MBPP** (`mbpp.jsonl.gz`, `mbpp_test.jsonl.gz`)

| field | type | notes |
|---|---|---|
| `task_id` | int | |
| `text` | str | one-sentence NL description |
| `code` | str | reference solution |
| `test_list` | list[str] | assert statements |

## Provenance and licensing — read before redistributing

All four are publicly released research datasets, each under **its own upstream licence**,
and several carry problem statements whose copyright rests with the original judges
(Codeforces, AtCoder, Kattis) rather than with the dataset authors.

**Those licences have not been reviewed here.** This repository is private and is a
working transfer between two people in the same lab. Check the upstream terms for each
source before republishing any of it, redistributing it outside the lab, or including it
in a public model release.

## Known limitations

- **No cross-source deduplication.** APPS, CodeContests, MBPP and HumanEval overlap in
  places; nothing here has been deduplicated against the others.
- **No decontamination.** These are widely-used benchmarks that appear in many pretraining
  corpora. If you evaluate on `humaneval` or `mbpp_test` after training on the rest,
  treat the result as contaminated until you have checked.
- **Nothing has been re-executed.** No solution in any split was run to confirm it passes
  its own tests. The files are as collected.
- **`apps_raw_test` vs `apps_raw_train`** are both 5,000 records; the size difference is
  entirely judge test data, not problem count.

## Integrity

`manifest.json` records, per file, the record count plus SHA-256 of both the raw `.jsonl`
and the shipped `.jsonl.gz`. To check a file after download:

```bash
gunzip -c data/mbpp.jsonl.gz | sha256sum     # compare to raw_sha256
sha256sum data/mbpp.jsonl.gz                 # compare to gz_sha256
```
