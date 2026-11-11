# zigkit: plan

Move every Zig project to **Zig 0.16.0**, upgrade its pinned dependencies,
pull the code copied between projects into **zigkit** (a public library, as
gokit is for the Go projects), and remove anything 0.17 would break that can
already be removed on 0.16. The aim is that the jump to 0.17 is small when it
comes.

Go big on scope, not timid: every project, every copied helper, every 0.17
hazard that can be removed now.

Written on dan-blade, 2026-11-11, from a research conversation. Execute on
dan-desktop. The user gave ultracode for this work: use workflows, and run
several adversarial review rounds one after another.

---

## 1. Facts as found (re-verify before acting)

### Projects

| Project | Zig pin today | How Zig is provisioned | minish | zwanzig | `@cImport` files | `@bitCast` uses |
|---|---|---|---|---|---|---|
| flong | 0.15.2 | Nix `zig_0_15` (flake), fixed-output `flong-<v>-zig-deps` | v0.1.0 | v0.15.1 | 0 (fdlint *bans* it) | **42** |
| quoin (`sudo/`) | 0.15.2 | Nix `zig_0_15` | - | - | 3 (`build.zig`, `abi.zig`, `lint.zig`) | 3 |
| capsper | 0.15.2 | `.mise.toml`, mise-action CI (Linux + macOS) | v0.1.0 | - | 6 (pipewire, opus, ORT, macOS audio, input ×2, build.zig) | 3 |
| zerocast | 0.15.2 | `.mise.toml`, cmake via zig-cc shims (libc++ ABI) | v0.1.0 | **v0.8.1** | 11 (wayland/macOS capture, input, SVT, session, terminal_share) | 3 |
| glim | 0.15.2 | `.mise.toml`, mise-action CI | - | - | 0 | 1 |
| fulcio-codesign | 0.15.2 | `mlugg/setup-zig@v2` in CI, macOS signing | - | - | 0 | 1 |
| browser-agent-os (`packages/shell`, `packages/coreutils`) | **already 0.16.0** | - | ? | ? | ? | ? |
| wasiglk (`packages/server`) | unknown, check | ? | ? | ? | ? | ? |

All repos are on `trunk` (wasiglk: `master`), clean, with an `origin` under
github.com/danielbodart.

### Copied code known so far (the inventory will find more)

- `flong/src/fd.zig` (1272 lines): a descriptor table of handles made of a
  slot plus a generation. Kinds are part of the type. Every open is
  O_CLOEXEC. It has `Held`, `AnyFd`, `Stdio` and `retainOnly`. It sits on
  `flong/src/sys.zig` and `errno.zig`. Its guards are `tools/fdlint.zig`
  (240 lines, the `raw-number` rule, which also bans `@cImport`), the
  compile-fail tests in `tests/zig/compile_fail/`, and
  `tests/zig/fd_props.zig` (minish properties).
- `quoin/sudo/fd.zig` (176 lines): a cut-down copy. It has no kinds,
  `capacity = 4`, and a **different error model**
  (`error{TableFull, Closed, Sys}` where flong uses `sys.Result(T)` with
  errno as a value).
- `quoin/sudo/lint.zig` looks like a relative of `fdlint.zig`.
- Check for other shared helpers between capsper, glim and zerocast
  (dist/run.ts layouts, platform/linux input, version stamping in
  build.zig, and so on).

### Dependencies

- **minish** (property testing, CogitatorTech/minish): v0.1.x targets
  0.15.2, v0.3/v0.4 target **0.16**, and v0.5.0 (3 Oct 2026) targets 0.17.0.
  Its README warns of breaking API changes between releases.
- **zwanzig** (static analyser, forketyfork/zwanzig): its version numbers
  are its own, not Zig's. v0.15.1 (27 Aug 2026) is the latest and has
  builds for both the **0.15.2 and 0.16.0** frontends. It has **no Zig
  0.17 support yet**. Its README says v0.18.0 will drop 0.15.2.
- **zig-protobuf** isn't used yet. It's for a later Mosh port (§6).
- nixpkgs has `zig_0_16` at 0.16.0.

### What 0.17 breaks (official notes: https://ziglang.org/download/0.17.0/release-notes.html)

- **Build system**: `build.zig` runs in a separate program from the one
  that executes the build. `b.build_root` becomes `b.root` (a Path). The
  `add*Arg` functions are replaced by `*Arg2`. `if (b.args)` becomes
  `run.addPassthruArgs()`. The build runner override is gone in favour of
  BSP, and **ZLS doesn't work with 0.17.0 yet**.
- **`@cImport` removed** (already deprecated in 0.16). The replacement is
  the **translate-c package** (`codeberg.org/ziglang/translate-c`), used
  from build.zig. `std.Build.Step.TranslateC` is deprecated.
- **`@bitCast`** now works on the value's logical bits, the same on any
  byte order. It can **change behaviour silently for arrays and
  vectors**. Casts involving an `extern struct` or `extern union` become
  compile errors (use `@ptrCast` or an extern union). Casting to an enum
  is now safety-checked.
- Removed: `errdefer |err|`, `void{}`, `i0`, `a ** n` (use `@splat`).
- `containsAtLeastScalar` swapped its parameters, so old calls still
  compile but are wrong.
- `StackFallbackAllocator` becomes `BufferFirstAllocator`.
  `DebugAllocator` is deprecated in favour of `SafeAllocator`.
- Deprecated, not yet removed: `std.builtin` (use `std.lang`),
  `@intFromEnum`/`@enumFromInt` (use `@backingInt`/`@fromBackingInt`),
  `@import("builtin").cpu/os/abi/object_format` (removed in 0.18).
- Minimum Linux is 5.10 and minimum macOS is 15.0.

A grep found no `errdefer |err|`, `containsAtLeastScalar` or
`stackFallback` in the six main projects.

---

## 2. Decisions

1. **zigkit** is public at `github.com/danielbodart/zigkit` and its
   license is Apache-2.0, as gokit's is. The repo already exists with this
   file. Its default branch is `trunk`, to match the other Zig projects.
   gokit uses `main`.
2. **zigkit targets Zig 0.16.0 only.** Pin it with mise (`mise.toml`), as
   gokit does, and use Nix where a consumer needs it.
3. **Expose each module separately**: `sys`, `errno`, `fd`, `lint` and so
   on, each with its own `b.addModule`. A project pulls in only what it
   names. Zig only compiles code that's referenced, so unused code costs
   nothing. Watch global state such as the fd table: it goes in BSS only
   when it's used.
4. **Settings that differ per program** are read at compile time from
   `@import("root").zigkit_options`, in the `std_options` pattern, with
   defaults (for example fd `capacity`: quoin 4, flong 1024).
5. **Descriptor kinds must be open to extension.** zigkit ships the common
   kinds and a program can add its own: either `Fd(Kinds, k)` with a
   program-supplied enum, or a base set plus extras. Choose in the design
   phase (§3, phase 1).
6. **One error model.** Reconcile flong's `sys.Result(T)` (errno as a
   value) with quoin's error set during design. Expect flong's to win,
   since it is the more complete.
7. **The guards move into zigkit and every consumer gets them**: fdlint
   (`raw-number`, the `@cImport` ban), compile-fail tests, and minish
   property tests. zwanzig runs in zigkit's CI.
8. **Remove 0.17 hazards on 0.16 wherever 0.16 allows it**:
   - `@cImport` becomes the translate-c package everywhere. It's deprecated
     in 0.16, so this is legal now.
   - Audit every `@bitCast`. Cast arrays and vectors explicitly, and use
     `@ptrCast` where an extern struct is involved. Pin each one with a
     test; a minish property where the shape suits it.
   - Already-deprecated APIs move to their replacements where 0.16 has
     them (`std.lang` and `@backingInt` exist only if 0.16 has them, so
     check first).
   - **The build API can't be moved early**: the `*Arg2` functions,
     `b.root` and `addPassthruArgs` are 0.17-only. Instead, keep each
     project's build.zig small and put build helpers in one place (zigkit
     `build` helpers if they turn out to be shared), so the 0.17 rewrite
     touches one place. Stop reading `b.args` for anything except passing
     arguments through.
9. **Versioning and auto-release follow gokit's scheme**: a `MAJOR` file,
   and CI tags a release only when a release path changed (see
   `gokit/.github/workflows/ci.yml` and the `release-due` mise task).
   Consumers pin zigkit by tag and hash in `build.zig.zon`.
10. Usual rules: trunk-based with no PRs. Each project is committed and
    pushed to its trunk and its CI watched (the
    `commit-commands:commit-push-trunk` skill). No timing assertions in
    CI.
    **Confirm with the user at kickoff that pushing each project's trunk
    is OK.** Permission was given explicitly only for zigkit.

---

## 3. Phases (one workflow each; read the result before starting the next)

### Phase 0: Inventory (Understand workflow)
One reader per project (all eight), searching in several ways:
- Which Zig files are copies or near-copies of other projects' files?
  Diff them.
- Every 0.17 hazard in §1 with file:line, every `std.Io` / 0.16-breaking
  call site, and everything in build.zig that depends on the build API.
- How Zig is provisioned, how deps are pinned and how CI runs (Nix
  fixed-output hashes, mise caches, cmake shims).
- A baseline for each project: does `zig build test` (plus lint/analyze)
  pass on 0.15.2 now? Record **release binary sizes** for the size
  comparison later.

Then a completeness critic: which project, file kind or hazard was missed?

Output: `docs/inventory.md` in zigkit (what to extract, from where, and
who uses it).

### Phase 1: zigkit design (judge panel)
3–4 independent designs for module boundaries, kind extensibility, the
options mechanism, the error model and the lint packaging. Score them
on: binary size, how small and clear each consumer's import is, how
much it costs to adopt 0.17, and keeping today's safety guarantees
(stale handle → panic, wrong kind → compile error, CLOEXEC). Combine the
best into `DESIGN.md`. Copy flong's descriptor-layer docs, minus the
flong history: C line references, quirk numbers and phases.

### Phase 2: Build zigkit on 0.16
- Extract `sys`, `errno`, `fd` and the lint tool, then anything else the
  inventory shows is shared by **two or more** projects.
- Add a **`udp` kind** (`bind`/`sendto`/`recvfrom` with the source
  address) so a future Mosh port can use it. Add the `pty_*` /
  `signalfd` / `pollEntry` kinds that flong already has.
- Tests: unit tests, compile-fail tests, minish v0.4 properties, zwanzig
  v0.15.1 (0.16 frontend) and fdlint over zigkit itself.
- CI: mise-action, `mise run check`, gokit-style release due-check and
  tagging. Write a README in gokit's style (one paragraph per module).
- Push, and tag v0.x.

### Phase 3: Move each project (Migrate workflow, one worktree per project, pipelined)
For each project, in order of risk: glim, fulcio-codesign,
browser-agent-os, quoin, capsper, zerocast, flong. (wasiglk is deferred and
stays on 0.15.2: see DESIGN.md §5.13. ten-vad-ggml was added at kickoff.) **flong last**
(most `@bitCast`, Nix-built, and the source of the fd code).
1. Bump the Zig pin (`.mise.toml` / flake `zig_0_16` / setup-zig
   version) and `minimum_zig_version`. Refresh Nix fixed-output dep
   hashes.
2. minish v0.1 → v0.4 (rewrite properties for the API changes), and
   zwanzig to v0.15.1 with the 0.16 frontend.
3. Fix the 0.16 breaking changes (`std.Io` threading and so on).
4. `@cImport` → translate-c package. Check generated types against the C
   ABI (struct sizes and offsets) with comptime asserts.
5. Audit and fix each `@bitCast`, with a test for each one.
6. Replace the copied code with zigkit imports. The fdlint/zwanzig
   checks keep running.
7. Gate: tests + lint + analyze green, release binary size ≤ baseline
   (explain any growth), CI green after pushing to trunk.

### Phase 4: Adversarial review (several rounds one after another, until a round finds nothing)
For zigkit and each migrated project, each round uses separate reviewers
looking at:
- behaviour change from the migration (compare old and new on the same
  inputs where possible),
- `@bitCast` / byte order / ABI of translated C,
- descriptor and resource safety (leaks across fork/exec, stale handles,
  CLOEXEC),
- 0.17 readiness (anything left that 0.17 removes),
- build/CI/release (cache keys, Nix hashes, macOS jobs).

Several reviewers try to refute each finding, and a finding survives
only if most fail. Fix the survivors, then start a fresh round
(deduplicate against everything already seen). Stop after **two rounds
in a row that find nothing**.

### Phase 5: Report
What moved into zigkit, per-project diffs, binary sizes before and after,
what 0.17 will still need (expected: the build.zig API rewrite only), and
any findings that were confirmed but deferred.

---

## 4. Risks
- **minish v0.1 → v0.4 API churn** could change property semantics, such
  as shrinking or seeds. Keep failing seeds reproducible.
- **translate-c output differs from `@cImport`'s**, for example in macro
  handling and opaque types. Check per project, especially zerocast's
  SVT and Wayland headers and capsper's pipewire macros.
- **Nix fixed-output hashes** (`flong-<v>-zig-deps`) need refreshing each
  time a dep pin moves.
- **zerocast's cmake zig-cc shims** depend on the Zig version's
  libc++. Rebuild libdatachannel and SVT and confirm the cache keys
  change.
- **ZLS**: fine on 0.16. On 0.17 it doesn't work yet, so don't rush 0.17.

## 5. Not in scope / next
- The Zig 0.17 move itself: wait for zwanzig to support it, or contribute
  the 0.17 frontend upstream. Its build already handles two Zig versions.
- Mosh port (§6).

## 6. Mosh port in Zig (later; this is why `udp` is in zigkit)
- Terminal state: **libghostty-vt** (Zig module) on both the server and
  the client.
- You write: the framebuffer diff that redraws the client
  (`Display::new_frame`), local echo prediction, and SSP transport.
- Crypto: `std.crypto.aead.aes_ocb` (AES-128-OCB3, 64-bit nonce counter
  plus direction bit).
- Wire format: zig-protobuf, which supports proto2 extensions as Mosh's
  `.proto` files need, or hand-written encoding of the roughly 8
  messages.
- Descriptors: zigkit `fd` (udp, pty_master/slave, tty_out, signalfd,
  and `retainOnly` when forking the shell).
- Auto-release from day one.
