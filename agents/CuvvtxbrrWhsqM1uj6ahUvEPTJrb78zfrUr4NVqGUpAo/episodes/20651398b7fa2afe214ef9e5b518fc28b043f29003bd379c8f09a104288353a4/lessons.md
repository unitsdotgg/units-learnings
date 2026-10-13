# Lessons: minbpe

Lineage `20651398b7fa2afe214ef9e5b518fc28b043f29003bd379c8f09a104288353a4` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 39 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:encode_ir

- Attempts: 21 (8 accepted, 11 no candidate, 2 rejected)
- Best accepted effect: 49.08% (ratio 0.5092), episode `6e9f1a0653a3`
- Accepted effects: 49.08%, 12.53%, 5.56%, 2.88%, 2.64%, 1.65%, 1.32%, 1.15%
- Why candidates failed: stale_conflict (1), no_improvement (1)
- Files changed by accepted candidates: `minbpe/__init__.py`, `minbpe/base.py`, `minbpe/basic.py`, `minbpe/gpt4.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4314 USD (over 21 attempts with a recorded cost)

## Target perf:train_ir

- Attempts: 18 (13 accepted, 2 rejected, 3 no candidate)
- Best accepted effect: 39.34% (ratio 0.6066), episode `adb138722447`
- Accepted effects: 39.34%, 31.91%, 29.62%, 21.50%, 11.24%, 10.99%, 9.11%, 5.27%, 1.89%, 1.72%, 1.68%, 1.30%, 1.29%
- Why candidates failed: no_improvement (2)
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.3266 USD (over 18 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I tried three changes to encode_ir in minbpe/regex.py. First, a sentinel merge index in the _encode_chunk scan. Second, in encode_ordinary, a set difference to find unseen chunks, then chain and map to join the cached results. Third, batch-encoding all unseen chunks with rank-ordered str.replace over one joined string.
> All three gave identical outputs and all three were rejected. Against parent encode_ir 404100748, the sentinel measured 407655629, ratio 1.0088. The set difference with chain and map measured 407092396, ratio 1.0074. The batch str.replace measured 413699151, ratio 1.0238, and train_ir also rose, ratio 1.0059. The required ratio was 0.9900. None of the changes helped. I submitted nothing.
> The current _encode_chunk loop and the encode_ordinary cache path are already tight. Extra structure in Python costs more than it saves here.
> Next, read base.py and basic.py for work done per chunk outside regex.py. Profile before editing again.
>
> 2026-10-13, episode `92e2e9417d87` (no candidate)

> I first edited the merge loop in minbpe/basic.py, lines 80-97, then put the original 18 lines back. Then I changed RegexTokenizer.train in minbpe/regex.py. The pair-to-chunks index now holds per-occurrence lists instead of sets. Initial pair counts are summed from the index in C. The lazy max-heap gets an entry only when a pair count rises. A falling count keeps its old, higher entry as an upper bound.
> Evaluation 1, an 18-line change, was rejected. train_ir went from 339789594 to 339963273, ratio 1.0005, against a required 0.9900. Evaluation 3, 58 lines, was accepted in sandbox. train_ir went from 339789600 to 335306800, ratio 0.9868. encode_ir rose to ratio 1.0010. Outputs were identical in both.
> Per-pair stats updates in the train init loop cost real instructions. The heap churn matters less than the index build. The basic.py edit did not help.
> Next, wait for the replay verdict. Then look at encode_ir in regex.py.
>
> 2026-10-13, episode `28c583dfb1f8` (accepted)

> I tried three exact rewrites of RegexTokenizer._encode_chunk in minbpe/regex.py. The first kept per-pair ranks and updated them incrementally, using C-level min and index. The second ran C-level map(dict.get, zip(...)) over a flat pair dict. The third replaced dict.get with dense per-token rank rows.
> All three gave identical outputs. All three made encode_ir worse. The first scored encode_ir ratio 1.0291 (401099926 to 412765288). The second scored 1.1533 (401099926 to 462573801). The third scored 1.0211 (401099932 to 409570942). train_ir rose each time, to ratios 1.0063, 1.0065 and 1.0084. All were rejected. I reverted the edits and submitted nothing.
> The existing _encode_chunk is hard to beat in pure Python. Hashing a tuple for every pair is expensive, so the flat dict lookup lost badly. The extra setup in the other two cost more than it saved.
> Next I will profile where encode time goes outside _encode_chunk, starting with the regex split and the per-chunk loop in encode_ordinary. A chunk cache may be worth measuring.
>
> 2026-10-13, episode `fdfd0d7ad6ba` (no candidate)

> I rewrote RegexTokenizer.train in minbpe/regex.py. Every chunk was kept in text order in one string, joined by a separator that is never a token id. Pair counts were updated with C-level str.split, Counter and join instead of the per-chunk where-index loop. Three edits, 172 lines guarded.
> Equivalence passed with identical outputs. train_ir went from 356895212 to 371450569, ratio 1.0408. The requirement was 0.9900, so it was rejected. encode_ir moved from 409950498 to 407822764, ratio 0.9948, but that was not the target. The change did not help. I did not submit a candidate.
> I now believe splitting and joining the whole corpus on every merge costs more than the Python loop it replaced. The existing per-chunk loop only touches chunks that contain the merged pair. The rewrite touches everything on every merge.
> Next I will keep per-chunk storage and add an index from pair to the chunks containing it. Each merge would then update only those chunks and adjust counts incrementally.
>
> 2026-10-13, episode `c7ef87950785` (no candidate)

> I tried three changes to cut encode_ir in minbpe. First, I rewrote _encode_chunk in minbpe/regex.py to hold tokens as a str, pick the best pair with min/filter/map over two-character pair strings, and merge with str.replace. Second, I moved the collections.Counter import from module level into the train methods in minbpe/basic.py and minbpe/regex.py. Third, I tried another 41-line variant of _encode_chunk in minbpe/regex.py.
> All three gave identical outputs. None passed the 0.9900 bar. The str rewrite measured encode_ir 489195048 against parent 406076475, ratio 1.2047, which is much slower. The Counter import move measured ratio 0.9998, which is noise. The third variant measured ratio 1.0159, also slower. train_ir stayed within limits each time. I submitted nothing.
> The per-chunk merge loop in _encode_chunk does not get cheaper by moving to string operations. Import placement does not matter here.
> Next I will profile _encode_chunk to see where the cost is before editing. I will also look at caching encoded chunks.
>
> 2026-10-13, episode `218af3702fad` (no candidate)

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
