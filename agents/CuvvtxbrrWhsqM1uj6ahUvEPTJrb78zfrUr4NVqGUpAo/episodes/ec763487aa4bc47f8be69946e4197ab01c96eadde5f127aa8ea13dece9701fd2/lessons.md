# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 50 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 5 (4 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.72%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3032 USD (over 5 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 13 (10 accepted, 3 no candidate)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 18.03%, 9.45%, 9.22%, 7.25%, 2.19%, 1.78%, 1.48%, 1.31%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.2307 USD (over 13 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 12 (12 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 40.54%, 29.04%, 26.07%, 10.85%, 7.08%, 6.14%, 4.71%, 4.15%, 3.10%, 2.54%, 1.11%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.2475 USD (over 12 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 7 (4 accepted, 3 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3708 USD (over 7 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 13 (11 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4542 USD (over 13 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed RegexTokenizer.train in minbpe/regex.py twice, aiming at train_regex_ir. In the first variant the initial pass only builds the pair-to-chunk index, and pair counts come from a C-level weighted sum over each list. The second variant added byte values absent from the chunks as new token characters, so each chunk str stays one byte per character. I reverted both edits.
> Variant one measured train_regex_ir 362059278 to 361226417, ratio 0.9977. It was rejected because the required ratio is 0.9900. Variant two measured 1.0002, which is worse than the parent. Both gave identical outputs and passed the guards.
> The initial counting pass is not where train_regex spends most of its time. Making the chunk strs one byte per character did not help.
> Next, profile the merge loop in train itself, the per-merge updates through the pair-to-chunk index, before editing anything. Read lines 300 onward of regex.py first.
>
> 2026-10-13, episode `770ccb3efdf8` (no candidate)

> I tried to speed up training in minbpe/regex.py, targeting train_regex_ir. All edits stayed in that file, around the pair counting and the where index (lines 71-107 and 171-196). I read lineage_bench.py and basic.py for context.
> Four variants were measured. All had identical outputs, and all were rejected. Counting pairs as sum(map(weights)) over the where index, instead of updating stats per pair, gave 0.9962. Adding free byte codes for new tokens plus a dict.fromkeys dedupe of the where lists gave 1.0047, which is worse. Change (1) with heapq imports gave 0.9942, the best result but still short of 0.9900. The largest variant, at 39 guarded lines, gave 1.0106. Other metrics stayed within limits. regex.py is reverted to the parent, and nothing was submitted.
> These small rewrites of the counting loop move train_regex_ir by under 1%. The cost is not in how stats are summed.
> Next, profile train_regex_ir to find where instructions go before editing. Then try a structural change to merge selection, not another counting variant.
>
> 2026-10-13, episode `7bb7c20b8301` (no candidate)

> I tried to keep the token id string narrow in BasicTokenizer.train in minbpe/basic.py. It holds one character per id. New tokens used chr(256 + i), so after the first merge the str widened to 2 bytes per character. Every split then re-checked and narrowed parts, every join copied twice the bytes, and neighbour characters above U+00FF were new allocations. I changed new tokens to take byte values that do not occur in the text. That meant three edits in basic.py, 15 changed lines.
> The sandbox run was accepted with identical outputs. train_basic_ir went from 264603878 to 261683692, ratio 0.9890. The other metrics moved up slightly: train_regex_ir 1.0024, encode_ir 1.0043, decode_ir 1.0040, save_load_ir 1.0020. All guards were within limits.
> The gain is small. Str width is not the main cost in train. The pair counting and merge loop likely dominate.
> Next I will profile the counting step in train and look at the drift in encode_ir, which I did not touch.
>
> 2026-10-13, episode `34a655b9519b` (accepted)

> I tried two changes to the encode path in minbpe/regex.py, near lines 442-451. The first skipped str.replace for merges whose child characters were absent, checked against a set of characters present in the joined string. The second built ids with list(chain.from_iterable(...)) in place of the existing loop.
> The first variant measured encode_ir at 662379078 against a parent of 645140539, ratio 1.0267. It was rejected for no improvement. Outputs were identical. The second measured 652538001, ratio 1.0115, and was also rejected. Neither change helped, so I reverted both and regex.py is back to the parent. I submitted no candidate.
> Building set() over the UCS-2 joined string allocates a str for every character above U+00FF. The skip rarely fires, so the set costs more than it saves. The replace chain is already cheap per merge.
> Next time, profile _encode_chunk before editing. I should look for work that runs every time, not checks that guard against a rare case.
>
> 2026-10-13, episode `c17a2b88a238` (no candidate)

> I read minbpe/regex.py, minbpe/base.py, minbpe/basic.py and lineage_bench.py, looking for a change to decode_ir. I searched for the decode path, _decode_cache, the vocab equality check and module-level imports and compiles.
> I made no edits and ran no evaluations. There is no candidate. I stopped because I found no change I expect to reach the 1% minimum. Regex is already imported lazily. Load results are already cached. So decode pays very little fixed library cost. Cache hits happen in 11 of the 12 bench rounds. On each hit a vocab equality check runs over about 769 entries.
> I now believe the fixed costs in decode are already mostly removed. What remains per round is small, and the equality check on cache hits is the only part I found that clearly repeats.
> Next I would measure that equality check directly before changing it. One option is to key the cache on vocab identity or a stored hash instead of comparing all entries. If the measured share is under 1%, I should leave decode alone and look elsewhere.
>
> 2026-10-13, episode `85b09090563f` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
