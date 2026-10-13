# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 60 published attempts.

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

- Attempts: 11 (5 accepted, 6 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%, 1.52%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3787 USD (over 11 attempts with a recorded cost)

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

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
