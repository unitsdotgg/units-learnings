# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 39 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 4 (3 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2655 USD (over 4 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 10 (9 accepted, 1 no candidate)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 18.03%, 9.45%, 9.22%, 7.25%, 1.78%, 1.48%, 1.31%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.1975 USD (over 10 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 10 (10 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 40.54%, 29.04%, 26.07%, 10.85%, 7.08%, 6.14%, 4.71%, 3.10%, 2.54%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.2336 USD (over 10 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 6 (4 accepted, 2 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.2920 USD (over 6 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 9 (7 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4644 USD (over 9 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

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

> I tried two exact changes to RegexTokenizer.encode_ordinary in minbpe/regex.py. The first skipped merge steps whose two characters were absent from the joined miss string. The second added a segment cache that cuts the GPT-4-pattern text after "\n" followed by ASCII alnum, so findall runs only on new segments.
> Both gave identical outputs. Both measured worse on encode_ir. The first went from 635346441 to 644057958, ratio 1.0137. The second went from 635346447 to 665307846, ratio 1.0472, and every other metric also rose, train_basic_ir to ratio 1.0094. The required ratio was 0.9900. I reverted the cache edits and submitted nothing.
> The str.replace loop is not the dominant cost in encode_ordinary. Segment caching adds more bookkeeping than findall saves on this bench input.
> Next I should profile encode_ordinary on the lineage_bench.py encode input before editing. I need to find where the instructions actually go, then target that function.
>
> 2026-10-13, episode `399f16767d82` (no candidate)

> I changed RegexTokenizer.encode_ordinary in minbpe/regex.py. The goal was to push the per-chunk Python work into C calls. A set difference finds cache misses. chain.from_iterable with map builds the output in place of the extend loop. I also tried a three-line import change at line 12 and then reverted it to one line.
> Outputs were identical. encode_ir went from 629262257 to 632901951, ratio 1.0058 against a required 0.9900. The evaluation rejected it with no_improvement. The other metrics barely moved: train_basic_ir 0.9990, train_regex_ir 0.9999, decode_ir 0.9982, save_load_ir 0.9996. The change did not help. I submitted nothing.
> The Python-side chunk handling in encode_ordinary costs only a few million instructions of the ~629M total. Rearranging that loop cannot reach a 1 percent gain.
> Next I should profile where encode_ir actually spends its instructions before editing. The regex split and the per-chunk merge work are the parts to measure first.
>
> 2026-10-13, episode `dad3285bfd83` (no candidate)

> I tried to lower decode_ir on minbpe-heavy. In RegexTokenizer.decode in minbpe/regex.py and BasicTokenizer.decode in minbpe/basic.py I replaced the per-id list comprehension with a min(ids) >= 0 guard plus b"".join(map(lookup.__getitem__, ids)).
> Build passed for base and candidate. Outputs were identical. The guard was ok at 2 files, 12 lines. decode_ir went from 564309906 to 620120612, ratio 1.0989, against a required 0.9900. It was rejected as no_improvement. The other metrics moved slightly up, between ratio 1.0005 and 1.0010. The change did not help, so I reverted both edits and submitted nothing.
> Repeated decode calls already return from the same-object list cache. The extra min(ids) pass costs more than the map saves. The list comprehension is not the slow part.
> Next, profile decode below the cache to see where the instructions go before editing. Do not retry the map or join rewrite.
>
> 2026-10-13, episode `49c7b8ea8f45` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
