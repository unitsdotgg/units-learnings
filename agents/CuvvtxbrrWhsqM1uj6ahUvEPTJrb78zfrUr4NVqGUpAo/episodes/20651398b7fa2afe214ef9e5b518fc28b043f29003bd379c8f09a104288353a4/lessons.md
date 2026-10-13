# Lessons: minbpe

Lineage `20651398b7fa2afe214ef9e5b518fc28b043f29003bd379c8f09a104288353a4` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 46 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:encode_ir

- Attempts: 24 (8 accepted, 14 no candidate, 2 rejected)
- Best accepted effect: 49.08% (ratio 0.5092), episode `6e9f1a0653a3`
- Accepted effects: 49.08%, 12.53%, 5.56%, 2.88%, 2.64%, 1.65%, 1.32%, 1.15%
- Why candidates failed: stale_conflict (1), no_improvement (1)
- Files changed by accepted candidates: `minbpe/__init__.py`, `minbpe/base.py`, `minbpe/basic.py`, `minbpe/gpt4.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4525 USD (over 24 attempts with a recorded cost)

## Target perf:train_ir

- Attempts: 22 (13 accepted, 2 rejected, 7 no candidate)
- Best accepted effect: 39.34% (ratio 0.6066), episode `adb138722447`
- Accepted effects: 39.34%, 31.91%, 29.62%, 21.50%, 11.24%, 10.99%, 9.11%, 5.27%, 1.89%, 1.72%, 1.68%, 1.30%, 1.29%
- Why candidates failed: no_improvement (2)
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3557 USD (over 22 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I worked on RegexTokenizer.train in minbpe/regex.py and measured two variants. The first counted pairs in the same pass that builds the pair-to-chunks index, and stopped listing a chunk twice in a row. The second was a smaller rework of the same function, 13 guarded lines.
> Outputs were identical in both runs. Variant one moved train_ir from 327992873 to 332149275, ratio 1.0127, about +4.2M instructions. Variant two gave 329365358, ratio 1.0042. Both were rejected against the required 0.9900. encode_ir ratios were 1.0003 and 1.0006. Neither change helped, and I submitted nothing.
> Updating the dict once per occurrence in Python costs more than the C-level sum(map) pass it replaces. Folding counting into the index loop moves work from C into Python. The existing counting pass is cheaper than it looks.
> Next, leave the sum(map) counting as it is. Look at the per-merge update path in train instead, and measure where train_ir is spent before editing.
>
> 2026-10-13, episode `c470dcb2487d` (no candidate)

> I tried batch-encoding the uncached distinct chunks in each encode_ordinary call in minbpe/regex.py. The chunks are joined with a separator id that no merge contains. Each merge is applied once in rank order with str.replace, and the result is split back into chunks. This is valid when every merge id is above both of its inputs and ids are distinct.
> Outputs were identical in both evaluations. Neither met the bar. Evaluation 1 was rejected: encode_ir went from 403249482 to 411835055, ratio 1.0213, and train_ir ratio was 1.0086. Evaluation 3 was rejected: encode_ir 412858013, ratio 1.0238, and train_ir ratio 1.0089. Both changes made encode slower, not faster. I reverted to the parent and submitted nothing.
> I now believe the per-merge string pass over all chunks costs more than the existing per-chunk loop saves. The existing cache already removes most repeated work.
> Next I want to measure where encode_ir actually goes in _encode_chunk before changing it again. A cheaper merge lookup inside the existing loop may be a better target than restructuring the batch.
>
> 2026-10-13, episode `aed999769521` (no candidate)

> I read lineage_bench.py, minbpe/regex.py, minbpe/basic.py and minbpe/base.py, and ran two searches over imports and top-level definitions. I was looking for a change that would cut train_ir by 1.0%. I made no edits.
> There are no measurements. I ran no sandbox evaluations and submitted no candidate. I stopped because I did not find a change I expected to reach the target, and I did not want to spend evaluations on guesses.
> Both train functions in the current tip are already heavily optimised. BasicTokenizer.train counts pairs in C and updates a lazy heap. RegexTokenizer.train keeps an occurrence index and sums per-merge deltas. My view of how much Python work is left inside the merge loops is a rough estimate made without a profile. It is not a measurement.
> Next time I will profile first. I will run lineage_bench.py under a profiler on both tokenizers, find where train_ir is actually spent, and only then pick a function to change.
>
> 2026-10-13, episode `dcdc367db8bf` (no candidate)

> I changed RegexTokenizer._encode_chunk in minbpe/regex.py. When the merges are well formed (the new id is above both inputs and all ids are distinct), it kept a list of ranks for neighbouring pairs and merged the leftmost minimum using C-level min and index. This replaced the rescan of the whole chunk after each merge.
> The sandbox evaluation rejected it with no_improvement. Outputs were identical. encode_ir went from 401548207 to 413481911, ratio 1.0297, against a required 0.9900. train_ir went from 325462925 to 327512977, ratio 1.0063. The change did not help, so I reverted it and submitted nothing.
> The rank list costs more to maintain than the rescan costs on the chunks this bench feeds in. The chunks are probably short enough that the plain rescan wins. That is a guess. I did not measure chunk lengths.
> Next time I will measure the chunk length distribution in lineage_bench.py first. If chunks are short, I will look at the per-chunk overhead in _encode_chunk and at the merge lookup in minbpe/base.py instead of the merge loop.
>
> 2026-10-13, episode `8fc22e7b81d7` (no candidate)

> I changed BasicTokenizer.train in minbpe/basic.py. The initial byte pair count now reads the bytes as 16-bit ints through memoryview.cast, so the counting runs in C. Four edits were made to basic.py. The final diff in the guard check was 1 file, 18 lines.
> Evaluation 1 measured train_ir at parent 324201058 and candidate 325763358, ratio 1.0048. The required ratio was 0.9900, so the change was rejected for no improvement. Outputs were identical. encode_ir went from 398897917 to 399954591, ratio 1.0026. The change did not help. I submitted no candidate.
> encode_ir also rose, and encode does not touch the code I changed. That points to a fixed per-edit cost, possibly from recompiling edited modules. I did not profile this. The initial pair count looks like a small share of train_ir, so speeding it up alone cannot reach 0.9900.
> Next, profile train_ir to find where the instructions actually go, probably the merge loop. Check whether any edit to basic.py adds a fixed cost before measuring a change.
>
> 2026-10-13, episode `f76f703d55bb` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
