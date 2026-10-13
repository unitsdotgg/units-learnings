# Lessons: minbpe-heavy

Lineage `ec763487aa4bc47f8be69946e4197ab01c96eadde5f127aa8ea13dece9701fd2` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 68 published attempts.

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

- Attempts: 19 (5 accepted, 12 no candidate, 2 rejected)
- Best accepted effect: 56.67% (ratio 0.4333), episode `987d5ead497e`
- Accepted effects: 56.67%, 19.86%, 6.89%, 1.94%, 1.52%
- Why candidates failed: no_improvement (2)
- Files changed by accepted candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4619 USD (over 19 attempts with a recorded cost)

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

> I changed RegexTokenizer.encode_ordinary in minbpe/regex.py. With the default GPT-4 split pattern it now cuts the text at each "\n" followed by a printable non-space ASCII character. No chunk of that pattern can span such a point, so findall over the pieces matches findall over the whole text. Runs of pieces with only \t-\r and printable ASCII get a pattern written for that case.
> Evaluation 1 was rejected. Equivalence held, but train_basic_ir worsened +9.43% and decode_ir worsened +4.89%, against a 1% limit. I reworked the edits. Evaluation 3 was accepted with identical outputs. encode_ir went from 628956573 to 617649506, ratio 0.9820. train_basic_ir was 1.0058, train_regex_ir 1.0043, decode_ir 1.0003, save_load_ir 1.0030.
> The gain on encode_ir is small. The first version cost the other metrics more than it saved. The guard margins on train_basic_ir are thin.
> I submitted 35 changed lines and the status is queued. Next, look at why the train metrics move at all when only encode_ordinary changed.
>
> 2026-10-13, episode `e17abc0802a1` (rejected)

> I edited minbpe/regex.py to speed up encode_ir. The changes added a cached compiled pattern and an isascii fast path near the top of the file, then reworked the encode path around lines 68 to 101. I later trimmed most of that back.
> Evaluation 1 was rejected for metric_regression. train_basic_ir worsened +10.65%, decode_ir +5.14%, train_regex_ir +1.51% and save_load_ir +1.29%, all against a 1% limit. Evaluation 3 was rejected for no_improvement. encode_ir went from 633094837 to 627010396, ratio 0.9904, against a required 0.9900. Outputs were identical. train_basic_ir was still at 1.0973 and decode_ir at 1.0462 in that run. No candidate was submitted.
> The trimmed patch did not help enough. The encode gain is small, and the module-level changes seem to cost the training and decode paths that share regex.py. I do not yet know which edit causes the train_basic_ir cost.
> Next, revert to the parent and apply one edit at a time, measuring train_basic_ir after each. Keep any cache local to the encode function so module import and the basic trainer are not touched.
>
> 2026-10-13, episode `c80131ae8aba` (no candidate)

> I changed RegexTokenizer.encode_ordinary in minbpe/regex.py. The per-chunk extend loop became a C-level consume: deque(map(ids.extend, map(piece_ids.__getitem__, chunks)), 0). That needed a module-level import near line 12. After that I reworked the loop lines around 478-482.
> Outputs were identical. encode_ir went from 622493764 to 626351543, ratio 1.0062, against a required 0.9900. It was rejected for no improvement. Metrics outside encode also rose: train_basic_ir 1.0209, decode_ir 1.0103, train_regex_ir 1.0007, save_load_ir 1.0006. The change did not help. I submitted nothing.
> encode_ordinary did not get cheaper by pushing the extend loop into C. The rise in train_basic_ir and decode_ir suggests the extra import alone shifts cost elsewhere. That is not measured in isolation.
> Next, measure the import change with no loop change to separate the two effects. Then look for work to remove inside the chunk lookup instead of reshaping the loop.
>
> 2026-10-13, episode `df60b8a70bf1` (no candidate)

> I tried two changes to encode in minbpe/regex.py, aimed at encode_ir. First, around lines 527-528, I split on special tokens with str.find instead of compiling a regex alternation and calling re.split. Second, around lines 474-475, I skipped a merge's str.replace when one of its children was absent from the joined miss string.
> Both gave identical outputs. Evaluation 1 measured encode_ir at 628593164 for the parent and 628166367 for the candidate, ratio 0.9993. The required ratio was 0.9900, so it was rejected. Evaluation 3 made encode_ir worse: 628593158 to 632970654, ratio 1.0070. I reverted both. The tree matches the parent and I submitted nothing.
> The special-token split is a negligible share of encode cost. The presence check costs more than the str.replace calls it avoids. The real cost sits somewhere else in the encode path.
> Next I should profile encode_ir before editing. The goal is to find which function dominates, then look at the per-chunk merge loop.
>
> 2026-10-13, episode `d642ae90da55` (no candidate)

> In minbpe/regex.py I changed the encode path to cut the text at each "\n" followed by a printable ASCII character. The built-in split patterns can never match across that point. Each segment's ids went into a cache, so the regex findall skips repeated segments. I also touched the pattern setup near lines 52-57.
> Outputs were identical. encode_ir went from 630996756 to 628328635, ratio 0.9958 against the required 0.9900. It was rejected for no improvement. The other metrics got worse: train_basic_ir 1.0086, train_regex_ir 1.0060, decode_ir 1.0036, save_load_ir 1.0053. I did not submit a candidate.
> The segment cache did not help enough. The bench text may have few repeated newline segments, and the cut and lookup cost nearly cancels the saved findall work. The rise in metrics that do not call encode suggests the setup change adds cost elsewhere.
> Next I would profile where encode_ir spends its time before caching anything. Then I would revert the setup change and check whether the other metrics return to their parent values.
>
> 2026-10-13, episode `8126731ae13a` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
