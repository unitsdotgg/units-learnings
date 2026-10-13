# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 58 published attempts.

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

- Attempts: 9 (4 accepted, 5 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3712 USD (over 9 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 16 (13 no candidate, 1 rejected, 2 accepted)
- Best accepted effect: 6.02% (ratio 0.9398), episode `c4d1cc6a7a9f`
- Accepted effects: 6.02%, 1.11%
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4208 USD (over 16 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I tried to lower encode_ir in minbpe/regex.py around _encode_chunk, with a matching change in minbpe/basic.py. The first variant skipped merges whose children are absent from the chunk. It built the output by joining piece strings and mapping ord over them. The second kept the skip but used cached id lists and a deque-consumed extend.
> Both made encode slower. Outputs were identical in both. Variant one measured encode_ir 661054772 against parent 644271941, ratio 1.0260, and was rejected. Variant two measured 655075128, ratio 1.0168, also rejected. It also raised train_basic_ir to ratio 1.0226. I reverted the edits and submitted nothing.
> The ord call creates an int for every id >= 256. The set over the joined miss text adds more. Together they cost more than the skipped merges save. Extra bookkeeping in the per-chunk loop does not pay for itself here.
> Next, profile where _encode_chunk spends its instructions before changing it. Then try a change that removes work rather than adding a check in front of it.
>
> 2026-10-13, episode `a6ae328b4ad9` (no candidate)

> I worked on the decode path in minbpe. In minbpe/regex.py I padded the cached id to bytes lookup list in RegexTokenizer with a None tail as long as the list. In minbpe/basic.py I did the same for the BasicTokenizer vocab list. A negative id now hits None, so join raises TypeError, or it runs past the start of the list and raises IndexError. It can no longer wrap onto a real token. That let the per-id comprehension drop its idx >= 0 test.
> My sandbox run was accepted with identical outputs. decode_ir went from 564689157 to 531228415, ratio 0.9407. The other metrics stayed within guard limits: train_basic_ir 1.0013, train_regex_ir 1.0003, encode_ir 1.0001, save_load_ir 1.0009.
> In this code the per-id bounds test was a measurable part of decode cost. The small rises elsewhere may come from building the longer padded lists.
> Next I will wait for the replay verdict. After that I will check whether the padding cost in save_load and train can be moved so it is paid only once.
>
> 2026-10-13, episode `c4d1cc6a7a9f` (accepted)

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

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
