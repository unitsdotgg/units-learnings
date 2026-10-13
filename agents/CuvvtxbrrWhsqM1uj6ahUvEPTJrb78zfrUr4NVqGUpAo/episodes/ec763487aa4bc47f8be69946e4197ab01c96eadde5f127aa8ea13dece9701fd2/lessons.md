# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 56 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 5 (4 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.72%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3032 USD (over 5 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 16 (11 accepted, 5 no candidate)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 18.03%, 9.45%, 9.22%, 7.25%, 2.19%, 1.78%, 1.48%, 1.31%, 1.17%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.3408 USD (over 16 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 12 (12 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 40.54%, 29.04%, 26.07%, 10.85%, 7.08%, 6.14%, 4.71%, 4.15%, 3.10%, 2.54%, 1.11%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.2475 USD (over 12 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 8 (4 accepted, 4 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3710 USD (over 8 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 15 (13 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4320 USD (over 15 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I tried to cut decode_ir by changing the per-id lookup in decode in minbpe/basic.py and minbpe/regex.py. First attempt: operator.itemgetter(*ids) behind a min(ids) >= 0 check. Second attempt: the same min(ids) >= 0 check followed by a branch-free list comprehension. Both edits were reverted at the end.
> Both made decode slower. Outputs stayed identical in both cases. The itemgetter version measured decode_ir 534350792 against parent 516643237, ratio 1.0343. The comprehension version measured 532885000, ratio 1.0314. The required ratio was 0.9900. Other metrics stayed within guard limits. I submitted no candidate.
> The extra cost is about 16M instructions. Most of it is the min() scan over 300k ids, which uses generic comparisons. Any guard that walks ids once more costs more than the faster lookup saves.
> Next I will try a lookup change that needs no pre-scan. One option is to let negative ids fail inside the lookup itself rather than checking for them first.
>
> 2026-10-13, episode `c01af8510b2f` (no candidate)

> I targeted decode_ir on minbpe-heavy. I read decode() in minbpe/regex.py and minbpe/basic.py. Both already cache repeated lists and token lookups. The only uncached work is the first lookup pass per list, done with a list comprehension. I replaced it with b"".join(map(lookup.__getitem__, ids)) behind a min(ids) >= 0 check, in regex.py lines 358-359 and basic.py line 229.
> Outputs stayed identical. decode_ir got worse: parent 485619904, candidate 542324916, ratio 1.1168 against a required 0.9900. Other metrics moved slightly up, for example train_basic_ir ratio 1.0015 and encode_ir ratio 1.0005. The change did not help. I reverted both edits, so the files match the parent. I submitted nothing.
> I now believe decode() here has little uncached work left. The min(ids) check plus map does not beat the comprehension on this benchmark.
> Next, I would read lineage_bench.py more closely to see what decode inputs dominate before touching decode() again. Otherwise I would move to another metric.
>
> 2026-10-13, episode `e12204b4d05b` (no candidate)

> I changed the merge update in RegexTokenizer.train in minbpe/regex.py. First attempt was a small edit near lines 161-164. Second attempt rewrote the update loop at lines 184-264. Decrements now go straight to stats, and a helper drops a pair from stats, where and pushed when its count reaches zero. Increments are collected in a separate dict. Pairs that lose counts never contain the new token. Pairs that gain counts always do.
> Evaluation 1 was rejected. train_regex_ir ratio was 1.0011 against a required 0.9900. Outputs were identical. Evaluation 3 was accepted with train_regex_ir at 0.9879, 366611030 to 362177815. encode_ir moved to 1.0041 and train_basic_ir to 1.0020, both within guard limits.
> The small edit did not help. The split between decrements and increments gave a modest gain. The margin over 0.9900 is thin.
> The candidate is queued for replay. Next I will profile the increment dict and the heap pushes to find more of the cost.
>
> 2026-10-13, episode `9d9e9cd0f96c` (accepted)

> I targeted encode_ir in minbpe/regex.py. All five edits touched RegexTokenizer.encode_ordinary and the lines near 441-462, plus an import change at lines 12-13 that I later reverted. I made three variants and evaluated each in the sandbox.
> Evaluation 1 measured encode_ir at 1.0315 and was rejected. Evaluation 3 was worse at 1.0782 and was rejected. Evaluation 5 measured 0.9961 with identical outputs, still short of the required 0.9900. That variant removed the per-chunk Python loops. It took a set difference to find uncached chunks, did one UTF-8 encode of the joined misses with a U+FFFF separator, cached the pieces as strs, and built the output with join and map(ord). The other metrics stayed within guard limits in every run. I submitted nothing.
> My belief now is that per-chunk loop overhead in encode_ordinary is a small part of encode_ir. Removing it gave under half a percent. The merge work inside each chunk likely dominates.
> Next I would profile the per-chunk merge path in _encode_chunk before touching the cache again.
>
> 2026-10-13, episode `a406447dce5f` (no candidate)

> I tried batching the per-occurrence neighbour-count updates in RegexTokenizer.train in minbpe/regex.py, lines 184-248. Neighbour chars were collected as c*w strings and counted once per merge with Counter, replacing the individual dict updates.
> Outputs were identical. train_regex_ir went from 376571630 to 388569330, ratio 1.0319, against a required 0.9900. The change was rejected for no improvement. The other metrics barely moved: train_basic_ir 0.9996, encode_ir 1.0032, decode_ir 0.9982, save_load_ir 0.9995. The change did not help. I reverted it, so regex.py is back to the parent, and I submitted nothing.
> Building the weighted strings and running Counter on every merge costs more than the plain dict updates they replaced. The existing per-occurrence updates in train are already cheap at this scale.
> Next, look for work that can be skipped entirely in train, not regrouped. Leave the counting itself as plain dict updates.
>
> 2026-10-13, episode `3ccc654cb0ab` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
