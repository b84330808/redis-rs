# Rework of redis-rs/redis-rs#2514 (cache: multi-key commands)

## Status: commit ready, NOT pushed

| | |
|---|---|
| New branch | `fix-cache-multi-key-commands-v2` (one commit on upstream `main` `6015a72`) |
| Commit | `5f44b04f9633f51bbdad2bc186d89f3d2ecf10ca` "Invalidate cached replies when any key they read changes" |
| Author / committer | `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |
| Files | `redis/src/caching/{cache_manager,cmd,sharded_lru}.rs`, `redis/src/commands/mod.rs`, `redis/tests/test_cache.rs` (+420 / -76, about 250 of the added lines are tests) |

**Why it stopped before pushing:** `git push` to `b84330808/redis-rs` returned 403. The GitHub
account connected to the cloud session is `wilson-li-appier`, and it has no push access to the fork.
So neither `fix-cache-multi-key-commands-v2` nor `cloud/report-redis-rs` is on GitHub. I didn't
push through the GitHub API either, because that would have committed as `wilson-li-appier`. The
PR branch `fix-cache-multi-key-commands` is untouched (still `bbaa508`), and nothing was posted
upstream.

**How to get the exact commit locally** (both ways give SHA `5f44b04`):

```bash
# from the patch, on top of upstream main
git fetch upstream main
git switch -c fix-cache-multi-key-commands-v2 upstream/main
GIT_COMMITTER_NAME=b84330808 GIT_COMMITTER_EMAIL=b84330808@gmail.com \
  git am --committer-date-is-author-date 0001-invalidate-cached-replies-when-any-key-changes.patch
git rev-parse HEAD   # 5f44b04f9633f51bbdad2bc186d89f3d2ecf10ca
```

```bash
# or from the bundle, which holds both branches; it needs upstream main (6015a72) and the
# fork's main (cfd757b) to be in the clone already
git fetch upstream main && git fetch origin main
git fetch redis-rs-rework.bundle \
  fix-cache-multi-key-commands-v2:fix-cache-multi-key-commands-v2 \
  cloud/report-redis-rs:cloud/report-redis-rs
```

To let a cloud session push next time, connect the `b84330808` GitHub account at
https://claude.ai/connect-github and make sure the Claude GitHub App is installed on
`b84330808/redis-rs`.

## Design

The reviewer asked for two things: (1) keep caching multi-key commands, and evict the reply when
**any** of its keys is invalidated; (2) find each command's keys by position instead of assuming
the first argument.

1. **Key positions live in the table the cache already uses.** `command_properties` in
   `redis/src/commands/mod.rs` already lists the cacheable commands (`Properties::ReadOnlyCacheable`)
   and is shared by the cluster code (`is_readonly_cmd`) and the cache. The variant now carries a
   `KeyPositions`: `First` (GET, HGET, ...), `FirstTwo` (LCS), `All` (EXISTS, SDIFF, SINTER,
   SUNION, MGET), `AllButLast` (JSON.MGET), or `AfterKeyCount` (SINTERCARD, ZDIFF, ZINTER,
   ZINTERCARD, ZUNION). Each command is still listed once, so there's no second table.
   `is_cachable_cmd` became `cacheable_key_positions`. `command_keys()` in `cache_manager.rs`
   turns positions into keys. If they can't be found (key count not a number, zero, or larger
   than the number of arguments, or LCS with one key), the command is not cached and goes to the
   server unchanged.

2. **Why not the cluster routing's key lookup.** `cluster_handling/routing.rs` is only compiled
   with the `cluster` feature, and `cache-aio` doesn't depend on it. It also only finds the key
   that picks the slot: SDIFF/SINTER/SUNION/LCS are `RouteBy::FirstKey`, and ZUNION and the
   other key-count commands are `SecondArgAfterKeyCount`, which reads only the first key after
   the count. Routing never needs the full key list, except for the MGET/EXISTS split. So
   routing is unchanged.

3. **The reply is stored under every key it reads.** `SingleCachedCommand` now holds
   `keys: Vec<TrackedKey>` (each key with its shard's invalidation count, as before) instead of
   one `redis_key`. On a miss the pipeline is `CMD, PTTL k1, ..., PTTL kn`. Each key gets its
   own expiry from its PTTL. The reply is inserted under each key with that key's
   in-flight-invalidation check. Invalidating, expiring, evicting or epoch-bumping any one key
   drops that key's copy. This reuses the existing per-key machinery, and `ShardedLRU` itself is
   unchanged apart from lookup.

4. **Lookup checks every key: `ShardedLRU::get_from_all_keys`.** A hit needs the entry under
   every key, and all copies must be **equal**. The equality check matters because the
   in-flight check uses a per-shard counter. A later request can be refused under `key_1`
   because a different key in that shard was invalidated, while it is stored under `key_2`.
   Then `key_1` still holds an older reply that was computed from `key_2`'s old value. If all
   copies are equal, each key has been unchanged since the newest of those requests ran, so the
   reply is current. Hits and misses are counted once per command, not once per key. `get()` is
   now `get_from_all_keys(&[key])`, with the shared lookup factored into `find()`.

5. **The command now goes before the PTTLs, for every cached single command.** In opt-in mode
   the client sends `CLIENT CACHING YES` first, and Redis applies it only to the **next**
   command (`resetClient` clears `CLIENT_TRACKING_CACHING`). With `PTTL k1` first, the server
   tracked only `k1`. I checked this: with the old order, the new integration test fails in
   opt-in mode (`["EXISTS","str_1","str_2"] after ["DEL","str_2"]`, left `int(2)`, right
   `int(1)`). Default (non-opt-in) mode passed with either order. The nextest run with the old
   order: `4 tests run: 2 passed (arg_2_false = default mode), 2 failed (arg_1_true = opt-in)`. For single-key commands the
   swap changes nothing, because the command now gets tracked instead of the PTTL.

6. **Unchanged:** MGET and JSON.MGET still go through the split path (one `GET key` entry per
   key, shared with plain GET), so a change to one key already evicts only that key's value.
   Single-key commands behave as before, apart from the pipeline order.

## New PR description (plain text)

```
Cache replies of multi-key commands under every key they read

A cacheable command's reply was stored under its first argument and evicted only when the
server invalidated that key. EXISTS, LCS, SDIFF, SINTER and SUNION read other keys too, and
SINTERCARD, ZDIFF, ZINTER, ZINTERCARD and ZUNION start with the number of keys, so their
reply was stored under that number. After any other key changed, the old reply kept being
served until the client-side TTL ran out.

As suggested in review, the list of cacheable commands now also records where each command's
keys are (first argument, first two, all, all but the last, or after a key count), and the
reply is stored under every key the command reads. It is served only while all of them hold
the same reply, so invalidating any one of them evicts it. MGET and JSON.MGET still cache
each key separately.

The command is now sent before the PTTL of its keys. In opt-in mode CLIENT CACHING YES only
applies to the next command, so with the PTTL first only the first key was tracked.

I didn't reuse the cluster routing's key lookup: it is only built with the cluster feature,
and it only finds the key that picks the slot (the first one for SINTER or LCS). The positions
are in command_properties instead, which the cluster code and the cache already share.

Tests: unit tests in cache_manager.rs (eviction by any key, key positions, keys holding
different replies) and an integration test in test_cache.rs for both cache modes, run
against Redis 7.0.

I used Claude Code to write this change.
```

## Tests

Environment: Redis 7.0.15 (`/usr/bin/redis-server`), rustc 1.97.0, cargo-nextest 0.9.100,
`REDISRS_SERVER_TYPE=tcp`, nextest profile `tcp`, `--all-features`.

New tests:
- `cache_manager.rs`
  - `test_reply_is_evicted_when_any_key_it_reads_is_invalidated`: 10 commands × invalidating
    `key_1` or `key_2` = 20 cases. Each case checks the reply is cached, misses after the
    invalidation, and is cached again.
  - `test_reply_is_not_served_when_its_keys_hold_replies_of_different_requests`
  - `test_command_keys`: 10 cases, including `ZUNION 2 key_1 key_2 WEIGHTS 1 2` giving
    `[key_1, key_2]`, and bad key counts giving `None`.
- `test_cache.rs`
  - `test_cache_evicts_reply_when_any_key_it_reads_changes`: tokio and smol × opt-in on/off.
    For each of the 10 commands it reads twice (must be one hit), changes one key from another
    connection, then checks the reply equals a fresh uncached read and is a miss. ZUNION changes
    its first key (`zset_1`, at position 2); the others change their second key. It needs Redis
    7.0 (LCS, SINTERCARD, ZINTERCARD) and is skipped below that.

For the before runs I transplanted only the new tests into the other trees. `test_command_keys`
can't compile without the new code, and the mixed-replies test needs APIs the PR base doesn't
have. Each tree's `redis` crate was rebuilt from scratch with `cargo clean -p redis -p redis-test`
before its run. Filter: `--lib --test test_cache -E 'test(/^caching::/) | binary(test_cache)'`,
`--retries 0`, `PROTOCOL=RESP3`.

### Fail before: current PR head `bbaa508` (the commands aren't cached at all)

```
Summary [   6.197s] 72 tests run: 48 passed (1 slow), 24 failed, 380 skipped
FAIL evicted_when_any_key_invalidated {EXISTS,LCS,SDIFF,SINTER,SINTERCARD,SUNION,ZDIFF,ZINTER,ZINTERCARD,ZUNION} x (invalidate key_1 | key_2)   -> 20/20 fail
  assertion `left == right` failed: ["SINTER", "key_1", "key_2"] must be served from the cache
    left: int(2)
   right: int(1)
FAIL test_cache_evicts_reply_when_any_key_it_reads_changes::case_{1_tokio,2_smol}::arg_{1_true,2_false}   -> 4/4 fail
  assertion `left == right` failed      (assert_hit!: 0 hits, expected 1)
    left: 0
   right: 1
```
All 24 failures are new tests. Every existing test passed.

### Fail before: upstream `main` `6015a72` (stale replies)

```
Summary [   6.530s] 88 tests run: 68 passed (2 slow), 20 failed, 383 skipped
FAIL evicted_when_any_key_invalidated EXISTS|LCS|SDIFF|SINTER|SUNION (invalidate key_2)
FAIL evicted_when_any_key_invalidated SINTERCARD|ZDIFF|ZINTER|ZINTERCARD|ZUNION (invalidate key_1 and key_2)
PASS evicted_when_any_key_invalidated EXISTS|LCS|SDIFF|SINTER|SUNION (invalidate key_1)
  assertion `left == right` failed: ["ZUNION", "2", "key_1", "key_2", "WITHSCORES"] must not be served from the cache after key_1 changed
FAIL test_reply_is_not_served_when_its_keys_hold_replies_of_different_requests
FAIL test_cache_evicts_reply_when_any_key_it_reads_changes (all 4)
  assertion `left == right` failed: ["EXISTS", "str_1", "str_2"] after ["DEL", "str_2"]
    left: int(2)
   right: int(1)
```
All 20 failures are new tests.

### Pass after: `fix-cache-multi-key-commands-v2` `5f44b04`

```
Summary [   6.804s] 98 tests run: 98 passed (2 slow), 383 skipped      (same filter: 42 caching unit + 56 test_cache)
--lib, RESP3:                    419 tests run: 419 passed, 2 skipped
--test test_cache, RESP3:        56 tests run: 56 passed (3 slow), 4 skipped
--test test_cache, RESP2:        56 tests run: 56 passed (3 slow), 4 skipped
```
The 4 skipped tests are the RedisJSON-module tests (no module installed).

Static checks on the commit:
- `cargo fmt --all -- --check`: clean.
- `cargo clippy --workspace --all-targets --all-features -- -D clippy::all -D warnings` (the
  repo's `make lint`): clean.
- `RUSTFLAGS="-D warnings" cargo check -p redis --no-default-features` with `--features cluster`,
  with `--features cache-aio,tokio-comp`, with `--features cluster-async,cache-aio,tokio-comp`,
  and with no features: all clean.

**Not run:** the cluster integration suites (`test_cluster*`), the TLS and unix-socket profiles,
the RedisJSON module tests, servers other than Redis 7.0.15 (including Valkey), `make
flag-frenzy`, and benchmarks.

## Things the reviewer might push back on

1. **The reply is stored once per key.** A 3-key SUNION keeps 3 copies. The simplest fix is to
   share one `Arc<Value>` across the copies and compare with `Arc::ptr_eq` instead of `==`. That
   saves the memory and makes the check O(1), but it changes `CacheCmdEntry.value` and
   `insert`'s signature, so I left it out.
2. **Value comparison on hits.** For commands with more than one key, each hit compares the
   reply under every other key (O(reply size) per key). It is needed for correctness because
   the in-flight counter is per shard, not per key (Design 4). Alternatives: per-key counters,
   a request id stored in each entry, or the `Arc` above.
3. **One extra PTTL per key on every miss.** These are pipelined in the same round trip, and
   MGET already does this. They keep per-key server expiry correct.
4. **Pipeline order changed for all cached single commands** (CMD, then PTTL). This is needed
   for opt-in tracking and was shown by the experiment in Design 5. A reviewer might want it
   split into its own commit.
5. **Routing isn't changed to use `KeyPositions`.** The reviewer said to track positions "like
   the cluster routing", and might prefer routing to derive `SecondArgAfterKeyCount` from the
   same data. That is a natural follow-up, but it touches routing for non-cacheable commands
   (LMPOP, ZMPOP, ...), so I kept it out.
6. **`#[cfg_attr(not(feature = "cache-aio"), allow(dead_code))]` on the
   `ReadOnlyCacheable(KeyPositions)` field.** Cluster-only builds never read it, and without the
   attribute that build fails with `-D warnings`. The alternative is a separate cache-only match,
   which would be the second table the review asked to avoid.
7. **Small allocation on the hit path.** `command_keys` returns a `Vec`, even for single-key
   GETs. It is not benchmarked (the repo has `benches/bench_cache.rs`). A `SmallVec` or a
   single-key fast path would avoid it.
8. **Statistics.** When one key of a multi-key reply is invalidated, only that key's copy is
   dropped right away. The copies under the other keys stay until those keys change, are
   evicted, or the next fetch overwrites them. `invalidate` counts each copy as it is dropped.
   `hit`/`miss` count once per command.
9. **Key order.** `command_keys` sorts and dedups keys, so `EXISTS a a` sends one PTTL, and the
   order of PTTLs doesn't follow the argument order. It doesn't matter for correctness, since
   every key holds the full reply.
10. **Commands sent in lowercase are still never cached.** The table matches upper-case names
    only. This is pre-existing and not touched here.
11. **On MGET (mentioned in the review):** it was already cached per key and already evicted
    per key. This change leaves it as it is.

## Housekeeping

- Remote `upstream` = `https://github.com/redis-rs/redis-rs` was added and `main` fetched.
- Nothing was opened, edited or commented upstream. The upstream PR page was only read.
- The PR branch `fix-cache-multi-key-commands` was not touched.
