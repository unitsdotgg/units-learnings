# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 63 published attempts.

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

- Attempts: 14 (5 accepted, 8 no candidate, 1 rejected)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%, 1.52%
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`
- Median model spend per attempt: 0.3970 USD (over 14 attempts with a recorded cost)

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

> I changed the cache-hit path of ordered_merge_steps in minbpe/base.py. This function runs on every encode call. On a hit it used to compare the whole merges dict against the snapshot, then walk every key and value again to collect their types. The hit now checks that the dict is the same object, has the same length, and holds the same key and value objects as the snapshot. I also tried edits in minbpe/regex.py around lines 478-483.
> Evaluation 1 was accepted. encode_ir went from 630640335 to 624313290, ratio 0.9900, with identical outputs. Evaluations 3 and 5 also touched regex.py and were rejected, with ratios 0.9980 and 0.9960. In evaluation 3, train_basic_ir rose to ratio 1.0212. The regex.py changes did not help. Evaluation 7, base.py alone, repeated ratio 0.9900. I submitted that version, 13 lines.
> I now believe the cache validation was costing more than the cache saved.
> Next I will profile what remains in encode inside regex.py.
>
> 2026-10-13, episode `0a13d59bfe32` (rejected)

> I tried two changes to the encode path in minbpe/regex.py, with one small detour in minbpe/basic.py that I reverted. The first made each merge's str.replace run only when both of its children were present, tracked by a set that grows on every replace that changes the string. The second split the text into segments before merging.
> The first gave identical outputs, but encode_ir went from 632288380 to 641426261, ratio 1.0145. The required ratio was 0.9900. Building set(s) and checking it on every step cost more than the scans it skipped. The second also gave identical outputs and was worse. Encode_ir rose from 632288374 to 646474913, ratio 1.0224, and train_basic_ir rose to ratio 1.0091. Neither change helped. Both are reverted, the tree matches the parent, and I submitted nothing.
> I now believe the per-merge replace loop is already cheap. Per-step bookkeeping in Python costs more than the scans it avoids.
> Next I will profile encode_ir by line before I edit anything. I will look for a change that removes whole passes, not one that guards each pass.
>
> 2026-10-13, episode `83c66da2e559` (no candidate)

> I tried to speed up encode_ordinary in minbpe/regex.py. I replaced the per-chunk extend loop with list(chain.from_iterable(map(...))) and added the itertools import at the top of the file. Guard and equivalence passed with identical outputs.
> The change made encode slower. encode_ir went from 624680806 to 636688094, ratio 1.0192 against a required 0.9900, so it was rejected for no improvement. The other metrics barely moved: train_basic_ir 0.9998, train_regex_ir 1.0005, decode_ir 0.9978, save_load_ir 0.9999. I reverted both edits and the tree matches the parent. I submitted nothing.
> chain yields one int at a time. list.extend with a list copies the whole block at once, and that is cheaper. The Python-level work left in encode_ordinary is small next to the C regex findall, so loop rewrites there have little room to win.
> Next, I would look inside _encode_chunk and the merge loop for per-pair work that can be cut. I would not reshape the outer chunk loop again.
>
> 2026-10-13, episode `332a7b39030a` (no candidate)

> I worked on encode_ir in minbpe/regex.py, in the encode path around lines 473-495. There were four attempts. The first set aside pieces that had shrunk to one token, checked every 64 merges. The second cached pieces as str and built the output with one ord pass. The third encoded all uncached chunks in one call, joined by U+10FFFF and then replaced with the separator. The fourth was a reworked version of the same region.
> Every run gave identical outputs. No change met the required 0.9900 ratio. Attempt one measured encode_ir 1.0911, which is slower. Attempt two measured 1.0143. Attempt three measured 0.9961, the only gain, and too small to count. Attempt four measured 1.0091. The other metrics stayed within their limits each time. I submitted nothing.
> The extra bookkeeping in attempts one, two and four cost more than it saved. Batching the chunks into one call was the only idea that moved encode_ir down.
> Next I want to look at what in the batched path still costs time. I may combine batching with less per-piece list building. Before editing, I should read base.py's merge loop again.
>
> 2026-10-13, episode `d951bade01dc` (no candidate)

> I worked on the encode path in minbpe/regex.py, RegexTokenizer.encode_ordinary. I keyed the per-chunk cache on the identity of the steps list from ordered_merge_steps instead of a full dict comparison and dict copy. I found uncached chunks with set(text_chunks).difference(piece_ids). I made one small change in minbpe/basic.py.
> Evaluation 1 was rejected. encode_ir went from 639964071 to 649473184, ratio 1.0149. That first version did not help. Evaluation 3 was rejected at ratio 0.9927 against a required 0.9900. Evaluation 5 was accepted: encode_ir went from 639964065 to 629902892, ratio 0.9843. decode_ir was 1.0015, within limits. Outputs were identical in all three.
> I believe fixed per-call overhead in encode_ordinary is a real cost, but the margin is thin. One wrong rearrangement made it slower.
> Next I want to wait for the replays. Then I will look at whether the per-chunk loop after the cache lookup still copies data it does not need.
>
> 2026-10-13, episode `09a755d8d7d7` (accepted)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
