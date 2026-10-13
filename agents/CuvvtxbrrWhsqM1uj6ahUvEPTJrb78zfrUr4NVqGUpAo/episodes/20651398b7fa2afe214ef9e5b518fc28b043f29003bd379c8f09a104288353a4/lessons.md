# Lessons: minbpe

Lineage `20651398b7fa2afe214ef9e5b518fc28b043f29003bd379c8f09a104288353a4` on [https://github.com/karpathy/minbpe](https://github.com/karpathy/minbpe) at `1acefe89412b20245db5a22d2a02001e547dc602`. 52 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:encode_ir

- Attempts: 27 (8 accepted, 17 no candidate, 2 rejected)
- Best accepted effect: 49.08% (ratio 0.5092), episode `6e9f1a0653a3`
- Accepted effects: 49.08%, 12.53%, 5.56%, 2.88%, 2.64%, 1.65%, 1.32%, 1.15%
- Why candidates failed: stale_conflict (1), no_improvement (1)
- Files changed by accepted candidates: `minbpe/__init__.py`, `minbpe/base.py`, `minbpe/basic.py`, `minbpe/gpt4.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/base.py`, `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4986 USD (over 27 attempts with a recorded cost)

## Target perf:train_ir

- Attempts: 25 (13 accepted, 2 rejected, 10 no candidate)
- Best accepted effect: 39.34% (ratio 0.6066), episode `adb138722447`
- Accepted effects: 39.34%, 31.91%, 29.62%, 21.50%, 11.24%, 10.99%, 9.11%, 5.27%, 1.89%, 1.72%, 1.68%, 1.30%, 1.29%
- Why candidates failed: no_improvement (2)
- Files changed by accepted candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Files changed by rejected candidates: `minbpe/basic.py`, `minbpe/regex.py`
- Median model spend per attempt: 0.4101 USD (over 25 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I changed RegexTokenizer.encode_ordinary in minbpe/regex.py. The new version joined the distinct uncached chunks with a separator character that no merge can contain. It then applied each merge once, in index order, as a str.replace, and split the result back into chunks. I later reverted both edited regions.
> One sandbox evaluation. Equivalence held and outputs were identical to the parent. encode_ir went from 409426275 to 414665831, ratio 1.0128, against a required 0.9900. train_ir went from 325891809 to 328635582, ratio 1.0084. The change did not help. It was rejected for no improvement, and I submitted no candidate.
> The per-chunk cache already removes most repeated work. One pass of str.replace per merge across the joined text costs more than it saves, and the extra join and split add to that. encode_ir does not reward a change to batching shape alone.
> Next, profile _encode_chunk on uncached chunks before editing anything. Measure how often the merge loop scans the pairs dict, and target that inner loop.
>
> 2026-10-13, episode `f664c42dcaf1` (no candidate)

> I tried two changes aimed at train_ir. In minbpe/regex.py I switched the where index and the per-merge delta in RegexTokenizer.train to defaultdict. In minbpe/basic.py I replaced neighbour counting in BasicTokenizer.train with Counter(map(itemgetter, filter(None, parts))). I reverted each edit after measuring it.
> Both builds passed and both gave identical outputs. The defaultdict version moved train_ir from 327347319 to 330320130, ratio 1.0091, which is worse. encode_ir was 0.9967. The Counter version moved train_ir from 327347319 to 327585638, ratio 1.0007. The bar was 0.9900. Neither change helped. I submitted no candidate.
> In this loop the __missing__ factory calls cost more than the dict.get calls they replace. Moving the neighbour counting into C-level builtins did not change the total in any useful way. The plain dict.get code is already close to as cheap as these small rewrites can make it.
> Next I would profile RegexTokenizer.train to find where the instructions actually go before editing anything. A structural change to the merge step is more likely to pay off than more swaps of one container for another.
>
> 2026-10-13, episode `8041d3bc5903` (no candidate)

> I worked on encode_ir in minbpe/regex.py, in the per-chunk merge loop. All edits were in that file.
> The first variant ranked every adjacent pair in C on each pass with map of merges.get over zip, then min, index and count. Outputs were identical. encode_ir went from 409171687 to 471139347, ratio 1.1514. train_ir ratio was 1.0074. Rejected. The second variant kept a rank list per chunk and updated only the neighbouring pairs after each in-place merge. Outputs were identical. encode_ir was 424476527, ratio 1.0374, train_ir ratio 1.0117. Rejected. Neither change helped. I submitted no candidate.
> I now think the chunks are too short for these approaches. The per-call overhead and the bookkeeping cost more than the plain Python scan they replace.
> Next I want to look away from the inner merge loop. Two options are caching encoded chunks so repeated chunks skip merging, and cutting setup cost per chunk. I should measure how often chunks repeat before writing code.
>
> 2026-10-13, episode `d5d19e26c76b` (no candidate)

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

## Licence

Code excerpts from https://github.com/karpathy/minbpe at 1acefe89412b20245db5a22d2a02001e547dc602, licensed MIT, copyright its authors; see https://github.com/karpathy/minbpe/blob/master/LICENSE.
