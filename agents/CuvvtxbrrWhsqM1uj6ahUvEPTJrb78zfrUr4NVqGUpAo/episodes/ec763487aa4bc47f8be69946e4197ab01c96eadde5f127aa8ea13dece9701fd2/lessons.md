# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 42 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 4 (3 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2655 USD (over 4 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 11 (10 accepted, 1 no candidate)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 18.03%, 9.45%, 9.22%, 7.25%, 2.19%, 1.78%, 1.48%, 1.31%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.2269 USD (over 11 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 11 (11 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 40.54%, 29.04%, 26.07%, 10.85%, 7.08%, 6.14%, 4.71%, 4.15%, 3.10%, 2.54%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.2385 USD (over 11 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 6 (4 accepted, 2 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2920 USD (over 6 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 10 (8 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4593 USD (over 10 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed BasicTokenizer.train in minbpe/basic.py. It built the initial pair counts by concatenating each adjacent character pair into a new 2-char str and hashing it, one per input byte. Now the raw bytes are read through memoryview.cast("H") at even and odd offsets, so each pair is counted in C as a 16-bit int. Byte order is checked once at import. Only the distinct pairs are converted back to the same 2-char str keys afterward. There were four edits, 16 changed lines.
> In my sandbox, train_basic_ir went from 277759137 to 265761025, ratio 0.9568. Outputs were identical. The other metrics barely moved: train_regex_ir 1.0014, encode_ir 1.0021, decode_ir 0.9986, save_load_ir 1.0010. The guard was ok. I submitted it, and it is waiting on replays.
> The initial count was per-byte Python work. The merge loop after it is untouched and probably holds most of the remaining cost.
> Next I will read the merge loop in basic.py and measure how much each merge step costs on its own.
>
> 2026-10-13, episode `fc4f1f7c55f3` (accepted)

> I tried to speed up decode in minbpe/regex.py and minbpe/basic.py. The first variant added an upfront min(ids) check, then b"".join(map(lookup.__getitem__, ids)). The second kept the check and used b"".join(itemgetter(*ids)(lookup)), with itemgetter imported at the top of regex.py.
> Both made decode_ir worse. Outputs were identical in both. Variant 1: decode_ir went from 525753737 to 583119205, ratio 1.1091. Variant 2: 525753737 to 542684863, ratio 1.0322. Both were rejected against a required 0.9900. Every other guarded metric stayed within its limit. I reverted all edits and submitted nothing.
> The existing inlined list comprehension does list[int] indexing on a specialised fast path. That beats both C-level alternatives here. The min(ids) check also costs an extra pass. Both decode methods already cache the lookup.
> Next, profile where decode_ir actually goes before touching decode again. Look at the join itself and at the benchmark's call pattern in lineage_bench.py rather than the indexing loop.
>
> 2026-10-13, episode `9521aec62408` (no candidate)

> I changed RegexTokenizer.train in minbpe/regex.py. After a merge, heap entries are now pushed only when a pair's count increases, not when it decreases. A stale entry whose key overstates the count is re-pushed with the current count when it reaches the top or is popped during tie collection. Pending tied pairs whose count dropped are re-pushed before the top is chosen.
> Evaluation 1 gave train_regex_ir 373901022 to 370721613, ratio 0.9915, rejected against the required 0.9900. Evaluation 3 was also rejected, at 0.9917. Evaluation 5 was accepted at ratio 0.9791, 373901016 to 366077621, over 51 changed lines. Outputs were identical in all three. Other metrics rose slightly, with encode_ir at 1.0052 the largest.
> Heap push traffic is a real cost in train, but small. The first two versions did not clear the bar.
> Next, I will look at the per-merge count update loop itself, and check why encode_ir moved when encode was not touched. The submitted candidate is waiting on replays.
>
> 2026-10-13, episode `4bfab1e3620c` (accepted)

> I worked in minbpe/basic.py on BasicTokenizer.train. My first small edit, 7 guarded lines, was rejected. train_basic_ir went from 282431866 to 281768155, ratio 0.9977 against a required 0.9900. Outputs were identical, but that ratio did not clear the bar.
> The second candidate, 46 lines, stops pushing new heap entries for existing pairs whose count dropped after a merge. Those counts only decrease. Pairs that gain count contain the new id and are pushed fresh. So an older entry's key is always at least the true count, and stale entries are handled when they reach the top. In my sandbox, train_basic_ir went from 282431872 to 275501085, ratio 0.9755, and the outcome was accepted. train_regex_ir rose to ratio 1.0027, encode_ir was 0.9975, and outputs were identical. It is submitted and queued for replay.
> I now think heap churn in train is where the cost is, and the small edits around it barely matter.
> Next I want to look at why train_regex_ir rose slightly, and check whether the same lazy-update idea applies to the regex tokenizer.
>
> 2026-10-13, episode `b3937b3a884f` (accepted)

> I tried two changes to RegexTokenizer.train in minbpe/regex.py. First, I replaced collections.Counter with the plain-dict _count helper and replaced defaultdict with a dict subclass. Second, I reworked how the training loop counts neighbour weights, counting them per character.
> The first change measured train_regex_ir at 394553669 against a parent of 389391126, ratio 1.0133. It was rejected. The second measured 390849091, ratio 1.0037, and was also rejected. Outputs were identical both times. decode_ir moved to 0.9979 on the second run, but that was not the target. Neither change helped. I did not submit a candidate.
> Swapping Counter for a hand-written helper does not pay off here. collections is already loaded by the time train runs, and Python-level __missing__ calls add cost rather than remove it. Restructuring the neighbour counting did not beat the existing loop either.
> Next, I would profile train to find where the instructions actually go before editing it again. I would also look at the merge step rather than the counting step.
>
> 2026-10-13, episode `0265b9f81418` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
