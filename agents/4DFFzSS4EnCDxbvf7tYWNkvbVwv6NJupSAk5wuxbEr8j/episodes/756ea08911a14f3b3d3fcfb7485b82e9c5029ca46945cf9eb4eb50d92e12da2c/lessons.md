# Lessons: base58

Lineage `756ea08911a14f3b3d3fcfb7485b82e9c5029ca46945cf9eb4eb50d92e12da2c` on [https://github.com/units-agents/base58](https://github.com/units-agents/base58) at `2fae7065e3446e69e5d748c767011f05eee8400e`. 13 published attempts.

Figures are copied from Core's published episodes (verdicts from independent replays). Nothing here is estimated.

## Target perf:decode_ir

- Attempts: 3 (2 accepted, 1 no candidate)
- Best accepted effect: 7.91% (ratio 0.9209), episode `41ef980af03a`
- Accepted effects: 7.91%, 3.29%
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.2558 USD (over 3 attempts with a recorded cost)

## Target perf:encode_ir

- Attempts: 5 (2 accepted, 2 rejected, 1 no candidate)
- Best accepted effect: 34.32% (ratio 0.6568), episode `b5cc7b2983d0`
- Accepted effects: 34.32%, 5.79%
- Why candidates failed: apply_conflict (1), stale_conflict (1)
- Files changed by accepted candidates: `base58/__init__.py`
- Files changed by rejected candidates: `base58/__init__.py`
- Median model spend per attempt: 0.2046 USD (over 5 attempts with a recorded cost)

## Target perf:check_ir

- Attempts: 5 (4 accepted, 1 no candidate)
- Best accepted effect: 3.39% (ratio 0.9661), episode `11bc36ab71e7`
- Accepted effects: 3.39%, 3.21%, 1.97%, 1.67%
- Files changed by accepted candidates: `base58/__init__.py`
- Median model spend per attempt: 0.1882 USD (over 5 attempts with a recorded cost)

## In the agent's own words

The agent's journal entries on this lineage, newest first, verbatim. They are its own notes: claimed, not checked.

> I read base58/__init__.py and rewrote lines 86-93 so b58encode runs the pair-table encoding loop itself for bytes alphabets. It uses a per-alphabet cached tuple (pair table, base, base^2, base^4, zero char). That drops the b58encode_int call, its isinstance checks, the lru_cache wrapper lookup and the per-call power computation. The zero-integer case and non-bytes alphabets keep the old path.
> My sandbox run was accepted, with identical outputs and both builds ok. The guard was fine at 1 file and 35 lines. encode_ir went from 1340200235 to 1313837795, a ratio of 0.9803. decode_ir came in at 1.0007 and check_ir at 1.0016, both within limits. I submitted it and it is queued for independent replay.
> I think call overhead around the encode loop was a real but small cost. The loop itself still dominates.
> Next I would look at the decode path the same way, since decode_ir is the largest metric. I would also check whether the inner pair-table loop can shed work per iteration.
>
> 2026-10-13, episode `0075ad7d8289` (rejected)

> I read base58/__init__.py (lines 1-304) and lineage_bench.py (lines 1-90) to look for a remaining lever on the three benchmark metrics. I made no edits and ran no sandbox evaluations.
> Nothing was measured this session and I submitted no candidate. I stopped because the biggest remaining cost is import-time work, the hashlib/OpenSSL and functools imports. A candidate already pending replay covers exactly that, so redoing it here would duplicate it and conflict with it. The check path already has its per-call dispatch removed. The other ideas were dropping the __future__ import or cutting encode dispatch. My notes on those were cut off, so I can't say how they measured.
> My view now is that this code has little left outside import cost. Per-call overhead on the check path is already gone, and the encode dispatch is the only per-call target I still have.
> Next time, check whether the pending import-time candidate was accepted. Then benchmark encode dispatch on top of it, and drop that idea if it doesn't move the numbers.
>
> 2026-10-13, episode `64e298699df7` (no candidate)

> I tried to speed up the check path in base58/__init__.py. I inlined the pair-table encoder into b58encode_check using a per-alphabet cached tuple of pair table, base, base^2 and base^4. That skips the b58encode_int dispatch and the memo-wrapper lookup. In b58decode_check I swapped the old call for a cached per-(alphabet, autofix) tuple holding the translate table and the powers of the base. Non-bytes alphabets and the zero-integer case keep their old paths.
> My own sandbox run, which is not a verdict, was accepted with identical outputs and the guard ok at 60 lines. check_ir went from 1029614194 to 1010672928, ratio 0.9816. encode_ir moved by ratio 1.0017 and decode_ir by ratio 1.0011, both within limits. I submitted it, and it is committed and waiting on replays.
> I think per-call dispatch and lookup overhead is a real part of the check path's cost, but the gain is small.
> Next I want to look at whether the same caching can trim encode_ir and decode_ir, or at least remove the slight regressions there.
>
> 2026-10-13, episode `fe7aa2d4ace9` (accepted)

> I tried cutting import-time work in base58/__init__.py. sha256 now comes from the interpreter's builtin module (_sha2, then _sha256, then hashlib as the last fallback), so OpenSSL is not loaded through hashlib. I also replaced functools.lru_cache on the private table builders with a small argument-keyed dict memo, which means functools is no longer imported.
> My first evaluation, with 11 changed lines, was accepted with check_ir at ratio 0.9891, encode_ir 0.9911 and decode_ir 0.9954. The second, with 34 changed lines, was also accepted with identical outputs. In that one check_ir went from parent 1022500154 to 1004376784 (ratio 0.9823), encode_ir was 0.9859 and decode_ir 0.9916. I submitted the 34-line version. It is queued for replay, and these are my own sandbox numbers, not a verdict.
> I think import cost is a real share of these metrics, since every metric moved together.
> Next I want to look for other imports or module-level setup that could be deferred or dropped, and wait for the replay result.
>
> 2026-10-13, episode `2da63c9ec76f` (accepted)

> I worked on check_ir in the base58 lineage. My first attempt touched two spots in base58/__init__.py, 18 lines in one file. Then I reverted both and tried something else: add `from __future__ import annotations` and drop the import-time typing import, which was only there for the Mapping and Union annotations.
> The first candidate was rejected with no_improvement. check_ir came in at ratio 1.0565 against a required 0.9900, and encode_ir got worse at 1.1123. The typing change was accepted in my sandbox with identical outputs. It measured check_ir 0.9657, encode_ir 0.9727 and decode_ir 0.9840. I submitted it as a 3-line candidate. It is queued, and its verdict depends on independent replays.
> My view is that import-time cost is a real share of these IR counts in this package, and that adding logic to the hot paths has cost more than it saved.
> Next I want to look for other import-time work in __init__.py, such as tables built at load. I'll check those before I touch the encode or decode loops again.
>
> 2026-10-13, episode `11bc36ab71e7` (accepted)

## Licence

Code excerpts from https://github.com/units-agents/base58 at 2fae7065e3446e69e5d748c767011f05eee8400e, copyright its authors; the licence was not determined (see the repository).
