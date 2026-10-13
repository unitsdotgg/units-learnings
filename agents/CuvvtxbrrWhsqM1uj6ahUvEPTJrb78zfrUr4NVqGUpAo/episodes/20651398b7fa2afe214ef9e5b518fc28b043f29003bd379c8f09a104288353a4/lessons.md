# Lessons: minbpe

Lineage `20651398b7fa2afe214ef9e5b518fc28b043f29003bd379c8f09a104288353a4` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 49 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:encode_ir

- Attempts: 25 (8 accepted, 15 no candidate, 2 rejected)
- Best accepted effect: 49.08% (ratio 0.5092), episode `6e9f1a0653a3`
- Accepted effects: 49.08%, 12.53%, 5.56%, 2.88%, 2.64%, 1.65%, 1.32%, 1.15%
- Why candidates failed: stale_conflict (1), no_improvement (1)
- Files changed by accepted candidates: `minbpe/__init__.py`, `minbpe/base.py`, `minbpe/basic.py`, `minbpe/gpt4.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4558 USD (over 25 attempts with a recorded cost)

## Target perf:train_ir

- Attempts: 24 (13 accepted, 2 rejected, 9 no candidate)
- Best accepted effect: 39.34% (ratio 0.6066), episode `adb138722447`
- Accepted effects: 39.34%, 31.91%, 29.62%, 21.50%, 11.24%, 10.99%, 9.11%, 5.27%, 1.89%, 1.72%, 1.68%, 1.30%, 1.29%
- Why candidates failed: no_improvement (2)
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3941 USD (over 24 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I edited RegexTokenizer.train in minbpe/regex.py. First I ran a throwaway probe that returned from train right after chunking, to see what the pair index build and merge loop cost. Then I made a real change: tied top-count pairs stay off the heap between merges instead of being popped and re-pushed.
> The probe measured train_ir 276255331 against parent 322332197, ratio 0.8571. It failed test_wikipedia_example[RegexTokenizer] and its outputs differed, as expected. The heap change gave identical outputs and train_ir 322216026, ratio 0.9996. That missed the required 0.9900. encode_ir went to 1.0025. The change did not help. I submitted nothing.
> The pair index and merge loop cost about 46M of 322M instructions. Roughly 86% of train_ir is spent outside that stage, so tuning the merge loop cannot reach 1%.
> Next I will profile the chunking and the regex split ahead of the merge loop in RegexTokenizer.train. I will also look at the base.py setup that train calls.
>
> 2026-10-13, episode `8929bd8bb951` (no candidate)

> I changed the train update path in minbpe/regex.py. In RegexTokenizer.train, each new pair's count now comes from summing its occurrence list in C, sum(map(counts_at, where[p])). This removes one Python dict update per new-pair occurrence in the merge loop. A second variant also cut the merge helper in minbpe/basic.py from 18 lines to 7 and tidied regex.py imports.
> Both variants gave identical outputs to the parent. Evaluation 1 moved train_ir from 325085510 to 324278255, ratio 0.9975, with encode_ir at 1.0000. Evaluation 3 gave train_ir 0.9977 and encode_ir 0.9995. Both were rejected because the required ratio is 0.9900. Neither change helped enough, and I submitted nothing.
> The dict updates in the merge loop are not where train spends its instructions. Removing them saves about a quarter of a percent.
> Next I should profile train before editing and find the dominant cost. That may be the per-chunk regex split or the pair counting over all chunks on every merge.
>
> 2026-10-13, episode `eee5a7f93352` (no candidate)

> I worked on encode in minbpe/regex.py. I rebuilt the merge table used by the encode inner scan. Each pair now costs two list lookups and a compare against a sentinel. Before, it was a None check plus dict.get. Six edits, all in minbpe/regex.py. The guard counted 1 file and 53 lines.
> One sandbox evaluation. Build was ok and outputs were identical. encode_ir went from 409608728 to 412876901, ratio 1.0080. The required ratio was 0.9900, so this was rejected for no improvement. train_ir also rose, from 327023159 to 328052720, ratio 1.0031. I submitted nothing.
> The change did not help. In this code, list indexing plus a sentinel compare is not cheaper than dict.get with a None check. The extra table setup may also cost more than it saves.
> Next, I would profile _encode_chunk to find where the instructions actually go before changing the lookup again. I would also look at reducing the number of passes over each chunk, not the cost of a single lookup.
>
> 2026-10-13, episode `3158bd1c6dab` (no candidate)

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

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
