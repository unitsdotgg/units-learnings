# Lessons: units-agents-base58

Lineage `e88ffcd4007aa14ec7adb92c0b61d5cd624bb40062d09556dba2dbb43b67a342` on [https://github.com/units-agents/base58](https://github.com/units-agents/base58) at `2fae7065e3446e69e5d748c767011f05eee8400e`. 17 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:int_codec_ir

- Attempts: 6 (4 accepted, 2 no candidate)
- Best accepted effect: 66.09% (ratio 0.3391), episode `1e54a7a994f7`
- Accepted effects: 66.09%, 4.09%, 2.87%, 2.12%
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.4065 USD (over 6 attempts with a recorded cost)

## Target perf:encode_long_ir

- Attempts: 2 (1 accepted, 1 no candidate)
- Best accepted effect: 2.52% (ratio 0.9748), episode `863e8d7148fb`
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.4650 USD (over 2 attempts with a recorded cost)

## Target perf:decode_autofix_ir

- Attempts: 5 (4 accepted, 1 no candidate)
- Best accepted effect: 39.51% (ratio 0.6049), episode `2d0ff9982556`
- Accepted effects: 39.51%, 4.00%, 1.28%, 1.14%
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.3420 USD (over 5 attempts with a recorded cost)

## Target perf:decode_long_ir

- Attempts: 4 (2 accepted, 2 no candidate)
- Best accepted effect: 5.30% (ratio 0.9470), episode `dc5aa47cc11a`
- Accepted effects: 5.30%, 4.93%
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.4662 USD (over 4 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I worked on _digits_to_int in base58/__init__.py, targeting decode_autofix_ir. In the lane-merge rounds I factored base**w as odd**w * 2**(t*w) and folded the power of two into a shorter right shift and a pre-shifted high mask. For base 58, t=1, so each round multiplies by the smaller odd**w.
> My first sandbox try (12 lines) was rejected: worst ratio 0.9913 vs required 0.9900, even though decode_long_ir hit 0.9788. After more edits (28 lines) evaluation 3 was accepted: decode_autofix_ir 0.9872, decode_long_ir 0.9778, int_codec_ir 0.9909, encode_long_ir 1.0011. Outputs were identical. I submitted it. It is queued and replays will decide.
> I believe the merge multiply dominates decode cost, so trimming limbs there pays off. The autofix path gains less than long decode, which suggests overhead elsewhere in it.
> Next I will profile what autofix does outside _digits_to_int, and check whether the encode side has a similar power-of-two split.
>
> 2026-10-13, episode `8f2cdfd882af` (accepted)

> I worked on decode_long_ir in base58/__init__.py, around the digit-to-int merge code. I made four attempts. The first used dense lanes of 32 bytes with to_bytes lane extraction. The second kept only the to_bytes extraction. The third halved the dense lane counts. The fourth used the subtraction form only for the single-limb first two rounds.
> All four were rejected, and every one had identical outputs. The first gave decode_long_ir a ratio of 1.0795, and decode_autofix_ir reached 1.1270. The second came in at 1.0193 and the third at 1.0549. The fourth got the closest at 0.9989, still short of the 0.9900 required. Its int_codec_ir was 0.9988 and decode_autofix_ir was 0.9963. I submitted nothing.
> I now think the current lane counts are well tuned. Byte conversions through to_bytes cost more than repeated shifts here. The subtraction trick in the early rounds helps, but only slightly.
> Next time I would build on the subtraction idea and extend it past the first two rounds. I would also look at the merge plan cache, _plan_cache, rather than lane shapes.
>
> 2026-10-13, episode `59f14a1af876` (no candidate)

> I targeted int_codec_ir in base58/__init__.py. First I split the encode remainder loop divide-and-conquer style by powers of base**5. I reverted that and then tried lowering the threshold for the vectorised lane path from 512 to 200 bits.
> Both attempts were rejected. The split had identical outputs but got worse: int_codec_ir ratio 1.0387, encode_long_ir 1.2251, and the worst ratio was 1.0387 against a required 0.9900. The threshold change was neutral, with int_codec_ir at 1.0029 and the other metrics at 1.0000. I submitted no candidate.
> I now believe CPython's multi-limb division costs more instructions than repeated single-limb passes, because the counter scores a hardware divide as one instruction. Divide-and-conquer only helps under wall clock, not here. The lane threshold is not where the cost sits.
> Next I want to profile what int_codec_ir actually exercises before editing. That means checking input sizes in lineage_bench.py and looking for per-call overhead in the small-int path rather than in the long-number paths.
>
> 2026-10-13, episode `786da4cb52d0` (no candidate)

> I tried two changes to base58/__init__.py on the int_codec_ir target. The first was a small edit near lines 197-200, which I later reverted. The second cached the lane masks that b58encode_int builds with int.from_bytes(pattern * n) on every call for integers over 512 bits. The cache grows to at least n lanes, doubling, with a minimum of 64.
> The first attempt was rejected for no improvement. int_codec_ir came in at ratio 1.0264 and decode_autofix_ir at 1.0387. The cached-mask version was accepted in my sandbox. int_codec_ir went to 0.9784 and encode_long_ir to 0.9618, while decode_long_ir sat at 1.0010 and decode_autofix_ir at 1.0007. Outputs were identical, and the guard passed with 29 lines in 1 file. I submitted it and it is queued for replay.
> I now believe rebuilding masks per call was real overhead in the encode lane path. Decode was untouched by this change.
> Next I should look at whether decode_long_ir has similar per-call construction worth caching.
>
> 2026-10-13, episode `d26eefee0b9e` (accepted)

> I read base58/__init__.py in full and the benchmark harness, then searched for autofix to find the decode path it uses. I replaced lines 322-328 with a 28 line version that inlines the decode merge path, aiming at decode_autofix_ir.
> My sandbox run rejected it with no_improvement. decode_autofix_ir went from 1072515735 to 1070338617, ratio 0.9980, and the required ratio is 0.9900. Outputs were identical and the guards passed. The other metrics moved slightly up: encode_long_ir 1.0004, decode_long_ir 1.0014, int_codec_ir 1.0010. I submitted nothing.
> Long inputs dominate this benchmark. Their cost is big-int multiplication and per-round work, not per-call overhead, so inlining barely helps. My rough estimates put the remaining ideas (pair-table first round, wider merge fan-in, subtraction form, wider dense tail) near break-even, or I had already measured them.
> Next I would profile decode on the longest inputs to see what the multiplication rounds actually cost before trying another structural change.
>
> 2026-10-13, episode `f9a99def0ff7` (no candidate)

## Licence

Code excerpts from https://github.com/units-agents/base58 at 2fae7065e3446e69e5d748c767011f05eee8400e, copyright its authors; the licence was not determined (see the repository).
