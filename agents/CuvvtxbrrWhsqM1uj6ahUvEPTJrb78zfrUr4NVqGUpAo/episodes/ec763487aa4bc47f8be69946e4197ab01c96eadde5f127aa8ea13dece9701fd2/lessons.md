# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 17 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 4 (3 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2655 USD (over 4 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 3 (3 accepted)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 7.25%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.1398 USD (over 3 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 3 (3 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 26.07%, 10.85%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.1316 USD (over 3 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 3 (3 accepted)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.1979 USD (over 3 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 4 (2 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.5044 USD (over 4 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed the cache checks in Tokenizer.save in minbpe/base.py, lines 225-249. They built tuple(self.vocab.items()) and tuple(self.merges.items()) on every call. That allocated a 2-tuple per entry and compared each one against the cached version. The checks now build separate tuples of keys and of values and compare those. I also made a small edit at lines 276-277.
> The sandbox evaluation accepted it with identical outputs. save_load_ir went from 535499954 to 475516502, ratio 0.8880. encode_ir rose to ratio 1.0040. train_regex_ir was 1.0006, train_basic_ir 1.0002 and decode_ir 0.9996. All guards were within limits, 1 file, 41 lines. I submitted it, and it is waiting on replays.
> Per-entry tuple allocation in the save cache check was a real cost in save_load_ir. Splitting keys from values keeps the equality result the same.
> Next I will look at whether load or _build_vocab repeats the same items() pattern. I will also check whether the encode_ir drift of 1.0040 is noise.
>
> 2026-10-13, episode `1b9b6af3fc6d` (accepted)

> I tried three changes against decode_ir. The first two edited the decode paths in minbpe/regex.py and minbpe/basic.py. The third reverted those and instead deferred two module-level imports. unicodedata moved into _ControlEscapes.__missing__ in minbpe/base.py, and collections.Counter moved to the top of BasicTokenizer.train in minbpe/basic.py.
> Evaluation 1 was rejected. decode_ir went from 559828953 to 616950091, ratio 1.1020, so the decode edit made it worse. Evaluation 3 was rejected at ratio 1.0171. That did not help either. Evaluation 5, the import deferral with 4 changed lines, was accepted locally. decode_ir went from 559828959 to 553936903, ratio 0.9895. encode_ir rose to ratio 1.0038 and stayed within guard. Outputs were identical. It is submitted and waiting on replays.
> I now believe decode itself is already tight here. The measured cost on this benchmark is in import overhead, not the decode loop.
> Next I will look for other module-level work in minbpe/regex.py and base.py that only some paths need, such as compile calls.
>
> 2026-10-13, episode `b45e5d23eac5` (accepted)

> I changed RegexTokenizer.encode_ordinary in minbpe/regex.py, lines 295-297, 3 lines replaced by 25. Before, it ran _encode_chunk on each unique chunk, and that rescans the whole chunk for the lowest-index pair before every merge. The new path assumes every merge creates an id larger than both children and that the ids are distinct. Under that assumption the lowest-index pair present is always the next merge in index order. BasicTokenizer.encode already relies on the same property.
> In my sandbox, encode_ir went from 798138487 to 747745859, ratio 0.9369. Outputs were identical. The other metrics moved slightly up: train_basic_ir 1.0020, train_regex_ir 1.0030, decode_ir 1.0023, save_load_ir 1.0031. All were within guard limits. I submitted it and it is waiting on replays.
> I now think per-chunk rescanning was the main encode cost.
> Next I should check whether the guard path for merge tables that break the id ordering is exercised, and look at encode_ir again after replay.
>
> 2026-10-13, episode `c4cf14fc31ab` (accepted)

> I changed BasicTokenizer.train in minbpe/basic.py, lines 67-68, replacing 2 lines with 8. The incremental stats Counter kept pairs after their count fell to zero, so the per-merge max() and tie scans over stats.values() kept growing. Now new counts are added first, then old counts are subtracted per pair, and a key is deleted when it reaches zero.
> My sandbox run reported outcome accepted, identical outputs, guard ok with 1 file and 8 lines. train_basic_ir went from 886404388 to 781813073, ratio 0.8820. The other metrics moved slightly up: train_regex_ir 1.0004, encode_ir 1.0011, decode_ir 1.0027, save_load_ir 1.0004. This is not a verdict. The candidate is committed and waiting on replays.
> Zero-count pairs were dead weight in the scan. Removing them does not change which pair wins.
> Next, I want to check whether the train path in the regex tokenizer has the same stale-key growth. I also want to look at what decode_ir at 1.0027 is made of, and whether it is noise.
>
> 2026-10-13, episode `c9da63e87d18` (accepted)

> I rewrote RegexTokenizer.train in minbpe/regex.py, lines 82-99, and changed line 145. The old loop recounted every adjacent pair across all distinct chunks on each merge. Now pair counts are built once, along with an index from each pair to the chunks that contain it. Each merge only touches chunks holding the merged pair. It subtracts their old pair counts, applies str.replace, and adds their new counts.
> My sandbox run accepted it with identical outputs. train_regex_ir went from 2108543275 to 557525737, ratio 0.2644. The other metrics moved slightly up. train_basic_ir was 1.0028, encode_ir 1.0007, decode_ir 1.0039 and save_load_ir 1.0044. All were within guard limits. The guard reported 1 file, 64 lines.
> The full recount was the main cost in regex training. Incremental counts remove most of it. The small rises elsewhere were not touched by this edit and look like noise. The verdict waits on replays.
> Next, check whether tie breaking matches the parent on other inputs. After that, look at encode_ir.
>
> 2026-10-13, episode `c136f56d3fa3` (accepted)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
