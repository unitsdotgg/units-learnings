# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 53 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:save_load_ir

- Attempts: 5 (4 accepted, 1 no candidate)
- Best accepted effect: 12.33% (ratio 0.8767), episode `9671a1690038`
- Accepted effects: 12.33%, 11.10%, 4.72%, 4.64%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3032 USD (over 5 attempts with a recorded cost)

## Target perf:train_regex_ir

- Attempts: 15 (10 accepted, 5 no candidate)
- Best accepted effect: 74.51% (ratio 0.2549), episode `c136f56d3fa3`
- Accepted effects: 74.51%, 25.49%, 18.03%, 9.45%, 9.22%, 7.25%, 2.19%, 1.78%, 1.48%, 1.31%
- Files changed by accepted candidates: `minbpe/regex.py`
- Median model spend per attempt: 0.3237 USD (over 15 attempts with a recorded cost)

## Target perf:train_basic_ir

- Attempts: 12 (12 accepted)
- Best accepted effect: 59.45% (ratio 0.4055), episode `bab37e70b75e`
- Accepted effects: 59.45%, 40.54%, 29.04%, 26.07%, 10.85%, 7.08%, 6.14%, 4.71%, 4.15%, 3.10%, 2.54%, 1.11%
- Files changed by accepted candidates: `minbpe/basic.py`
- Median model spend per attempt: 0.2475 USD (over 12 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 8 (4 accepted, 4 no candidate)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3710 USD (over 8 attempts with a recorded cost)

## Target perf:decode_ir

- Attempts: 13 (11 no candidate, 1 rejected, 1 accepted)
- Best accepted effect: 1.11% (ratio 0.9889), episode `b45e5d23eac5`
- Why candidates failed: no_improvement (1)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`
- Median model spend per attempt: 0.4542 USD (over 13 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I targeted encode_ir in minbpe/regex.py. All five edits touched RegexTokenizer.encode_ordinary and the lines near 441-462, plus an import change at lines 12-13 that I later reverted. I made three variants and evaluated each in the sandbox.
> Evaluation 1 measured encode_ir at 1.0315 and was rejected. Evaluation 3 was worse at 1.0782 and was rejected. Evaluation 5 measured 0.9961 with identical outputs, still short of the required 0.9900. That variant removed the per-chunk Python loops. It took a set difference to find uncached chunks, did one UTF-8 encode of the joined misses with a U+FFFF separator, cached the pieces as strs, and built the output with join and map(ord). The other metrics stayed within guard limits in every run. I submitted nothing.
> My belief now is that per-chunk loop overhead in encode_ordinary is a small part of encode_ir. Removing it gave under half a percent. The merge work inside each chunk likely dominates.
> Next I would profile the per-chunk merge path in _encode_chunk before touching the cache again.
>
> 2026-10-13, episode `a406447dce5f` (no candidate)

> I tried batching the per-occurrence neighbour-count updates in RegexTokenizer.train in minbpe/regex.py, lines 184-248. Neighbour chars were collected as c*w strings and counted once per merge with Counter, replacing the individual dict updates.
> Outputs were identical. train_regex_ir went from 376571630 to 388569330, ratio 1.0319, against a required 0.9900. The change was rejected for no improvement. The other metrics barely moved: train_basic_ir 0.9996, encode_ir 1.0032, decode_ir 0.9982, save_load_ir 0.9995. The change did not help. I reverted it, so regex.py is back to the parent, and I submitted nothing.
> Building the weighted strings and running Counter on every merge costs more than the plain dict updates they replaced. The existing per-occurrence updates in train are already cheap at this scale.
> Next, look for work that can be skipped entirely in train, not regrouped. Leave the counting itself as plain dict updates.
>
> 2026-10-13, episode `3ccc654cb0ab` (no candidate)

> I edited the chunk setup near lines 63-67 of minbpe/regex.py, inside RegexTokenizer.train, three times. The goal was a lower train_regex_ir.
> Evaluation 1 measured train_regex_ir at 256004648 against parent 367610906, ratio 0.6964. Evaluation 3 gave 274402386, ratio 0.7464. Both were rejected. Outputs differed from the parent and tests.test_tokenizer::test_wikipedia_example[RegexTokenizer] failed. The gains came from changing behavior, so they do not count. I submitted nothing.
> My probes of RegexTokenizer.train put fixed cost (interpreter start, imports, pattern compile) at about 256M of 367.6M. findall is about 18M. Chunk setup, the pair index and the merge loop take about 93M together. Reaching 1% would mean saving about 4% of that last stage. Earlier small rewrites of it gave about 0.25% each.
> Next, any edit must keep outputs identical. Applying count decreases straight to stats is my only idea left, and it looks like another 0.25% change. A different target may be the better use of time.
>
> 2026-10-13, episode `499acec9a611` (no candidate)

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

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
