# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 12 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 3 (2 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2277 USD (over 3 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 2 (2 accepted)
- Best accepted effect: 25.49% (ratio 0.7451), episode `a87e2622e23f`
- Accepted effects: 25.49%, 7.25%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.1336 USD (over 2 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 2 (2 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 26.07%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.1851 USD (over 2 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 2 (2 accepted)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.1705 USD (over 2 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 3 (2 no candidate, 1 rejected)
- Why candidates failed: no_improvement (1)
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4744 USD (over 3 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I first tried changing the decode paths in minbpe/regex.py and minbpe/basic.py directly. Evaluation 1 rejected it. decode_ir went from 504573091 to 523153717, ratio 1.0368, against a required 0.9900. Outputs were identical. The change made decode worse, so I reverted both edits.
> Next I deferred two module imports. In minbpe/base.py, unicodedata moved into _ControlEscapes.__missing__, its only user. In minbpe/basic.py, collections.Counter moved into BasicTokenizer.train. Evaluation 3 accepted it with 4 changed lines. decode_ir went from 504573097 to 498919340, ratio 0.9888. encode_ir was 1.0005 and the other guarded metrics stayed within limits.
> What I believe now: in this bench, import cost at load counts toward decode_ir more than the decode loop does. Rewriting the loop added instructions.
> Next I will look for other module-level imports that the load and decode path does not need. The submitted candidate is waiting on independent replays.
>
> 2026-10-13, episode `a6d9f324e332` (rejected)

> I moved the collections import in minbpe/basic.py into train, since only train uses it. I moved the unicodedata import in minbpe/base.py into render_token. I also tried edits to the decode paths in regex.py and basic.py, then reverted them.
> Evaluation 1 on decode_ir gave parent 550479262 and candidate 545536663, ratio 0.9910. The required ratio is 0.9900, so it was rejected with no_improvement. Outputs were identical. The other metrics stayed near 1: train_basic_ir 0.9997, train_regex_ir 1.0003, encode_ir 0.9992, save_load_ir 1.0001. The lazy imports did not help enough. I submitted no candidate.
> The decode paths already cache their results through _decode_vocab_list_cache and _decode_lookup_cache. On repeat calls the remaining cost is the exact list and dict equality checks, and those are near their floor. Import time is a small share of decode_ir.
> Next, I will look for a cheaper cache validity check than full equality, such as an identity or version check. It must keep outputs identical.
>
> 2026-10-13, episode `c42216955eab` (no candidate)

> I changed BasicTokenizer.encode in minbpe/basic.py, lines 127-128, replacing 2 lines with 22. The old loop rescanned the whole id list for the lowest-index pair before every merge. If every merge creates an id larger than both children and merge ids are distinct, the lowest-index pair present is always the next merge in index order. A merge only creates pairs containing the new token, and those have higher indices. So encode now walks the merges in order.
> My sandbox evaluation accepted it with identical outputs. encode_ir went from 1844887567 to 798908769, ratio 0.4330. train_basic_ir was 1.0015, train_regex_ir 1.0007, decode_ir 1.0010 and save_load_ir 1.0025. All guards were within limits. This is not a verdict. Replays decide.
> I believe encode correctness rests on that id-ordering assumption, which minbpe training satisfies. Loaded merges that break it would be wrong.
> Next I will check RegexTokenizer encode in minbpe/regex.py for the same pattern. I will also check whether the assumption needs a guard on load.
>
> 2026-10-13, episode `987d5ead497e` (accepted)

> I tried to cut decode_ir on minbpe-heavy. The decode paths already cache their results, so the only real work left is the first list-comprehension pass over each 100k-id list. In minbpe/regex.py and minbpe/basic.py I replaced that pass with a min() check plus b"".join(map(lookup.__getitem__, ids)).
> Outputs were identical and guards were ok, 2 files, 12 lines. decode_ir went from 513195468 to 569549027, ratio 1.1098 against a required 0.9900. The change did not help. It was rejected for no improvement. The other metrics barely moved: train_regex_ir 0.9989, encode_ir 1.0015. I reverted both edits and submitted nothing.
> I now believe CPython's specialised list comprehension is already cheaper than a C map that goes through a method-wrapper. The decode loop is close to its floor in pure Python.
> Next I would look at whether the per-call lookup construction in decode costs anything, or I would move to a different metric.
>
> 2026-10-13, episode `0e7dd78e6982` (no candidate)

> I changed BasicTokenizer.train in minbpe/basic.py, lines 35-51, plus one line at line 13. Before the change it recounted every adjacent pair of the whole sequence on each merge. Now the counts are built once, keyed by the two-character string. After each merge only the pairs touched by that merge are updated. Runs of non-overlapping occurrences are found with str.find, scanning left to right the same way str.replace does.
> My sandbox evaluation was accepted, with identical outputs. train_basic_ir went from 2102615613 to 861430062, ratio 0.4097. The other metrics barely moved: train_regex_ir 1.0007, encode_ir 1.0006, decode_ir 1.0030, save_load_ir 1.0031. The guard passed on 1 file, 39 lines. This is my measurement only. Replays decide the verdict.
> I now think the full recount per merge was the main cost in basic training.
> Next, I want to try the same incremental counting in the regex tokenizer's train, which this change did not touch.
>
> 2026-10-13, episode `bab37e70b75e` (accepted)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
