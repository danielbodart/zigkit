# zigkit Phase 0 inventory

This is the Phase 0 output of [PLAN.md](../PLAN.md): what zigkit should
extract, where it comes from, who would use it, and what each project faces
on the way to Zig 0.16 and later 0.17.

It was assembled on 2026-11-11 from one reader report per project, a
cross-project copy report, an upstream research report, a gokit report, and
nine follow-up investigations. Where reports disagreed, the disputed lines
were rechecked on disk; section 7.1 lists every contradiction and which
version this document believes.

Heads inspected: flong c72db3b, quoin f87f25e, capsper 4f4bb26, zerocast
9b95b86, glim 05cbed5, fulcio-codesign 00141af, browser-agent-os c6cb09c,
wasiglk 786bf48, ten-vad-ggml edc621c. **No repo was changed.**
`git status --porcelain` was empty in every repo before and after, except
for changes that were already there and not ours (wasiglk
`packages/server/tests/adventtest`; data-lab, partner-gateway, waydriver,
yardstick). All builds and trials ran in copies under `/tmp/claude-1000/`.

**Warning before Phase 3:** four local clones are behind the tags their
consumers pin (quoin, capsper, glim, frisket), and so is the gokit clone.
See section 7.2.

---

## 1. Summary

### 1.1 The headline facts

- **Every project except browser-agent-os is on 0.15.2. browser-agent-os is
  already on 0.16.0** and compiles there with no errors. ten-vad-ggml pins
  nothing and has no build at all.
- **minish v0.1.0 (and its dependency chilli v0.2.2) does not build on
  0.16**, and once it is on disk it breaks *every* `zig build` step, not
  just the tests (section 5.6). It must move to v0.4.0 in the same commit as
  the Zig bump in flong, quoin, capsper and zerocast. zerocast's zwanzig
  v0.8.1 also fails on 0.16; v0.15.1 builds on 0.16.
- **0.16 fetches packages into a project-local `zig-pkg/`.** No project's
  `.gitignore` lists it, and quoin has no `.gitignore` at all.
- **The Nix fixed-output dependency pattern survives 0.16 unchanged**
  (`ln -s ${deps} "$ZIG_GLOBAL_CACHE_DIR/p"`). Only the hash moves: quoin's
  becomes `sha256-Gzt+mB0nGWjlgIdaP20G0gL336igSyE/xNEQVOkJFqo=` after the
  minish bump.
- **No `@bitCast` in any project changes meaning under 0.17** on the
  little-endian targets we ship. 58 sites in all; one involves an array
  (zerocast, debug log only).
- **wasiglk is blocked on 0.16** for 7 interpreters (setjmp with
  `+exception_handling` crashes the bundled wasi-libc build). 0.17.1 does
  not fix it either, but both versions have a small, tested workaround
  (section 2.8).
- **The 0.16 std.Io rewrite is the bulk of the work**: `std.os.argv`,
  `std.posix.*` wrappers, `std.fs`, `std.time`, `std.Thread.Mutex/sleep`,
  `std.net`, `std.process.Child`. The largest ports are capsper (~120
  removed posix calls, std.http.Server) and zerocast (~120 posix calls,
  every Timer/Mutex, fork/exec).
- **Two shared libraries with no `main`** (quoin's darwin sudo plugin and
  wasiglk's `wasi_glk`) need an `Io` built by hand or libc calls instead.
- **flong's start code purity** ("no syscall before main") is at risk:
  0.16 calls `sigaltstack` before main unless
  `std_options.signal_stack_size = null`. The ~9x growth of a trivial
  0.16 binary comes from the default panic handler pulling in
  `Io.Threaded`; flong already uses a custom panic, so its shipped binary
  should not grow from this.
- **Copies are fewer than PLAN.md expected, but real.** Confirmed copy
  groups that span two or more projects: the fd table (flong → quoin ×2),
  the lint (flong → quoin), the doc-comment harvester (flong ↔ capsper),
  build.zig step wiring (flong ↔ quoin), the `-Dversion` option (capsper,
  zerocast, glim), D-Bus SASL (flong ↔ glim, independent), ioctl encoding
  (capsper, glim), and linux input-event-codes (capsper, zerocast). No file
  is byte-identical across projects.

### 1.2 Projects at a glance

| Project | Zig pin | Provisioned by | minish | zwanzig | `@cImport` sites | `@bitCast` | `a ** n` | Baseline on its pin | Shipped binary (bytes) |
|---|---|---|---|---|---|---|---|---|---|
| flong | 0.15.2 | mise (`mise.toml:10`) + Nix `zig_0_15` (`native.nix:35`, `flake.nix:255`), FOD `flong-0-zig-deps` | v0.1.0 | v0.15.1 | 0 (fdlint bans it; 4 test-only `addTranslateC`) | 42 | 49 lines | PASS, except `mise run check` 341/342 (env size, section 2.1) | 1,349,296 (`.#default`), 154,280 (`.#seccomp`) |
| quoin (`sudo/`) | 0.15.2 | Nix `zig_0_15` only | v0.1.0 | v0.15.1 | 1 (`abi.zig:8`) | 3 | 1 | PASS | `quoin_sudo.so` 56,328; `quoin-sudo-pam` 30,616; `quoin-sudo-check-plugin` 31,656 (Linux). Darwin never built or published |
| capsper | 0.15.2 | `.mise.toml:2`, mise-action CI (Linux + macos-15), Nix `zig_0_15` | v0.1.0 | - | 5 | 3 | 20 (+1 comment) | PASS, 718/718 tests | Linux 9,558,216 (v0.420.300 release; local 4f4bb26 build 9,566,632); macOS 1,764,992; Nix 1,621,296 |
| zerocast | 0.15.2 | `.mise.toml:2`, bootstrap.sh, cmake via zig-cc shims | v0.1.0 | **v0.8.1** | 11 files | 3 | 57 | PASS, 143/143 | ReleaseSmall 5,898,544 |
| glim | 0.15.2 | `.mise.toml:2`, mise-action CI, Nix `zig_0_15` | - | - | 0 | 1 | 9 | PASS, 19/19 | 95,952 |
| fulcio-codesign | 0.15.2 | `mlugg/setup-zig@v2` in CI only (`ci.yml:21-23`) | - | - | 0 | 1 | 2 | tests PASS 20/20; release build needs macOS | 585,568 (v0.8.9 release, aarch64 Mach-O) |
| browser-agent-os | **0.16.0** | `mise.toml:2`, mise-action@v2 CI | - | - | 0 | 0 | 0 | 62/64 bun tests (2 need Chromium) | 17 wasm files, 29,557–46,883 |
| wasiglk | 0.15.2 | `mise.toml:6` ("0.16 breaks it"), bootstrap.sh | - | - | 0 | 5 | 2 | PASS in a clone (66 Zig, 7 regtests, 76 client) | 18 wasm, ~6.0 MB total; tads3 1,354,690 |
| ten-vad-ggml | none (de facto 0.15.2) | none; no build.zig | - | - | 3 blocks in 2 files | 0 | 74 occurrences on 71 lines | `zig test` per file passes (36 tests); `zig fmt` fails | none shipped |

### 1.3 Module × project matrix

Key: **S** = source of the code; **C** = confirmed copy or twin that would be
replaced; **c** = candidate consumer (same need, code written
independently); blank = no use found.

| zigkit module | flong | quoin | capsper | zerocast | glim | fulcio | b-a-os | wasiglk | ten-vad |
|---|---|---|---|---|---|---|---|---|---|
| `sys` (raw syscalls, `Result(T)`, ioctl encoder, sockaddr_un) | S | c (inline `linux.*` in fd.zig) | c (`_ioc`, `extern "c" ioctl`, inotify) | c (~120 posix calls) | c (~30 removed posix calls, `iocWR`, sockaddr ×3) | | | | |
| `errno` | S | c | | | | | | | |
| `fd` (descriptor table) | S | C (fd.zig, fd_darwin.zig) | c (inotify hotplug) | | c (events, dbus, uleds) | | | | |
| `lint` (fdlint) | S | C (lint.zig) | | | | | | | |
| `build` helpers (lint/fmt/analyze/compile-fail/prop-test steps, `Modules.get`, `-Dversion`, translate-c wrapper) | S | C | C (`-Dversion`, prop-tests, docs gen) | C (`-Dversion`, zwanzig, prop-tests) | C (`-Dversion`) | c (`b.args`) | | | c (needs a build.zig) |
| `docharvest` (doc comments → generated table) | S | | C | | | | | | |
| `dbus` (wire format + SASL EXTERNAL) | S | | | | C (independent) | | | | |
| `input` codes (pure-Zig linux input-event-codes) | | | c (`@cImport` at input.zig:13) | S/C (keymap.zig hand-written; wayland/input.zig:16 `@cImport`) | | | | | |
| `macos/cf` (hand-written CoreFoundation externs) | | | c (audio.zig:17 `@cImport`) | c (macos/input.zig:7 `@cImport`) | | S (security.zig) | | | |
| minish property-test wiring | S (7 files) | C (4 calls) | C (24 calls) | C (2 calls) | | | | | c (lost capsper properties) |
| `msg`/panic (custom `FullPanic`, direct-write) | S | | | | | | | | |

Non-Zig shared assets (not zigkit modules, but worth one canonical copy):
`scripts/version.sh` (flong = quoin), TypeScript `version()` (capsper,
glim, zerocast, wasiglk), dist install/update/rollback scripts (capsper,
zerocast), CI workflows (capsper ≈ glim; flong ≈ quoin), bootstrap.sh
(zerocast, wasiglk).

---

## 2. Per project

### 2.1 flong

`/home/dan/Projects/flong`, trunk, c72db3b. One `build.zig` (2006 lines,
100 KB), one `build.zig.zon` (24 lines). 37,137 lines of Zig.

**Shared core** (native.nix:134-140 calls it `shared`): sys, msg, errno,
num, fd.

| File | Lines | Purpose |
|---|---|---|
| src/sys.zig | 1181 | Syscall layer, `Result(T)` |
| src/fd.zig | 1293 | Descriptor table (PLAN.md says 1272: stale) |
| src/errno.zig | 223 | glibc strerror and names |
| src/msg.zig | 419 | stderr and dying |
| src/num.zig | 174 | Numbers parsed as the C did |
| src/proc.zig | 430 | fork, clone3, Spawn |
| src/sig.zig | 200 | signalfd, awaitFd |
| src/dbus.zig | 547 | D-Bus wire format |
| src/manager.zig | 472 | systemd over D-Bus |
| src/tty.zig | 915 | Terminal |
| src/syscalls.zig | 72 | Per-arch syscall list |

Also mount 798, names 30, cgroup 1057, record 1157, ns 540, passwd 151;
programs main.zig 461, launch.zig 641 + 19 launch/*.zig (4,872 lines),
init 559, sweeper 52, check 1019, layer 459, spec 794, decl 1762,
decl_docs 1104, seccomp/* (main 81, compile 402, expand 338, project 419,
render 299, resolve 120, scmp 144); fixtures bpfdump 603, probe 248,
swapper 56, ioctl_probe 121; build/gen_decl_docs.zig 141, build/schema.zig
25, tools/fdlint.zig 240.

#### Provisioning and pins

- **Zig:** mise.toml:10 `zig = "0.15.2"` (dev loop and CI `check`).
  native.nix:35 `zig = pkgs.zig_0_15` (0.15.2,
  `/nix/store/harnmjs9…-zig-0.15.2`); flake.nix:255 devShell
  `pkgs.zig_0_15`; tests/integration.nix:36 and native.nix:14 comments.
  build.zig.zon:22 `minimum_zig_version = "0.15.2"`. The locked nixpkgs
  (nixos-26.05, c56a87d) has `zig_0_16` = 0.16.0 and bare `zig` = 0.16.0.
- **Deps** (both lazy, requested only under `-Ddev=true`):
  - minish, build.zig.zon:10-14,
    `https://github.com/CogitatorTech/minish/archive/refs/tags/v0.1.0.tar.gz`,
    `minish-0.1.0-SQtSTWHkAQACfz3xGuWHU8zbx320vK_47r2yto3Pq0Rf`. Pulls chilli
    v0.2.2 (`chilli-0.2.2-c19PrvIyAQAgsL0VkYCKq0DuxU7rZUpRuH991zsyOkYQ`)
    from minish's own zon (lines 6-8).
  - zwanzig, build.zig.zon:15-19,
    `https://github.com/forketyfork/zwanzig/archive/refs/tags/v0.15.1.tar.gz`,
    `zwanzig-0.15.1-oiXZliOBGQA1fkoEDA6UxFdOv1ezRF5Jc9YY7uZnHjKe`.
  - **native.nix:126 is wrong**: chilli is not "zwanzig's own"; zwanzig
    v0.15.1's zon has `.dependencies = .{}`.
- **Nix FOD:** native.nix:128-131 `deps = zigDeps { pname = "flong"; hash =
  "sha256-GicN77r9Oh9xPqlIP5CS0/y2hSenxlM6thAiv1bjBW8="; }`
  (`flong-0-zig-deps`), made by `zig.fetchDeps { fetchAll = true; }` at
  native.nix:116 over build.zig and build.zig.zon; linked by `ln -s ${deps}
  "$ZIG_GLOBAL_CACHE_DIR/p"` at native.nix:93. Used by native-test-debug,
  native-test-release and native-analyze (`-Ddev=true` at native.nix:266,
  :300) and tests/integration.nix:53, 71. Recomputed by
  scripts/zig-deps.sh:54-69.
- **Hook flags:** nixpkgs zig setup-hook `zigDefaultOptimizeFlag=--release=safe`;
  every set builds ReleaseSafe with `$zigDefaultCpuFlag` (native.nix:98).
  build.zig:51 `preferred_optimize_mode = .ReleaseSafe`.
- **Install sets:** seccomp (native.nix:145-152), fixtures (161-188),
  launcher (211-251, with -Dtini, -Dbwrap, -Dpasta, -Dnewuidmap,
  -Dnewgidmap, -Dcache, -Dseccomp, -Dself). Checks: native-test-debug and
  -release (274-275), native-lint (280-289), native-analyze (292-302),
  cross-aarch64 (310-367, greps abi plant messages at 360-363).
- **mise tasks** (checked on disk; the flong report's line numbers were
  wrong, see 7.1): `[env] RELEASE_PATHS` mise.toml:19; `check` :21-23
  (`zig build test lint compile-fail fmt -Ddev=true`); `build` :25-27
  (`nix build .#default .#seccomp`); `release-due` :31-37, which prints
  `false` **and exits 1** when nothing changed (:36).
- **CI** (`.github/workflows/ci.yml`, all ubuntu-latest, no macOS):
  `check` (46-66: apt libseccomp-dev, `jdx/mise-action@v5`, `mise run
  check`, `actions/cache@v6` of `~/.cache/zig` with key `zig-${{
  hashFiles('build.zig.zon','mise.toml') }}`, restore-key `zig-`, :60-64);
  `list` (68-90); `nix` (92-163, matrix of checks, Cachix push allow-list
  including `flong-[0-9]+-zig-deps` at :159); `eval` (168-180, `nix flake
  check --no-build`); `release` (194-236, `scripts/version.sh`, `gh release
  create`); `cache` (241-264, pushes `.#default` and `.#seccomp`).
- **update.yml** weekly (cron `23 21 * * 0`): `nix flake update`, mise
  tools to latest patch (336-343), `scripts/zig-deps.sh` (347), which only
  takes tags whose `minimum_zig_version` is no newer than current Zig (its
  comment at line 11 names minish 0.2.0 as needing 0.16).
- **Version-tied text:** comments citing std line numbers (build.zig:51
  `generic.nix:158-160`, :56 `build_runner.zig:370`, :450
  `Build/Step/Fmt.zig:16`, :1928 `start.zig:545-578`; native.nix:143
  `NativePaths.zig:17-72`, :57 `setup-hook.sh:16-41,108-110`).
  tests/zig/abi.zig hard-codes bundled uapi headers 6.13.4 (also
  DESIGN.md:2311). DESIGN.md:1747, 1769-1773, 1956-1974, 2029, 2332 name
  0.15.2/zig_0_15. `ZIG.md` is cited 10 times in build.zig and in src but
  **does not exist**.
- mise.toml is not trusted on this host (`MISE_TRUSTED_CONFIG_PATHS` used).

#### Baseline on 0.15.2

- Nix checks (all substituted from Cachix or already in store, so CI's
  derivations, not fresh builds): native-test-debug, native-test-release,
  native-lint, native-analyze, cross-aarch64, launcher, seccomp, golden,
  decl-options-fresh, reference-fresh, `.#default`, `.#seccomp`: all exit
  0. VM checks (basic-a/b, rootless-a/b, parity, native, integration,
  assertions-*) not run.
- Local `nix develop -c zig build <steps> -Ddev=true`: test+lint+compile-fail+fmt
  pass (15 s), test-libc (6 s), analyze (39 s), abi, cross (4 s), `test
  -Drelease=true` (83 s): all pass.
- **`nix develop -c mise run check`: FAIL, every time, 341/342**, at
  tests/zig/wrapper_test.zig:506 (`\nenv workspace=/w\n` not found).
  Cause confirmed by follow-up:
  - fakecmd's `show` mode echoes every env entry as `env K=V`
    (tests/zig/fakecmd.zig:88-91, 64 KiB buffer at :62); the test passes the
    whole process environment (wrapper_test.zig:489); `workspace`/`binds`
    are appended after the inherited entries (src/launch/cmd.zig:66-82,
    :82); `drain` stops silently when the 16384-byte `out_buf`
    (wrapper_test.zig:73) fills (:56-66).
  - Output = 88 + S bytes, S = Σ(len(entry)+5). Thresholds: S ≤ 16303
    passes; 16304–16312 fails at :508; 16313–16334 at :507; ≥ 16335 at
    :506; with a 100000-byte variable the fake's `bufPrint` fails at :505.
  - Measured S: plain shell 10589 (105 entries), `mise exec` 10981,
    `mise run` 11679, `nix develop` **16161 (142 bytes from failing)**,
    `nix develop` + `mise run` **17188** (fails at :506).
  - Fix options (none applied): pass a fixed base env at :489; make `drain`
    fail when full; make `out_buf` ≥ 64 KiB.
  - Without `nix develop`, `mise run check` does not reach the tests on
    NixOS (5 compiles fail: no `libseccomp`).

#### Release sizes (ReleaseSafe, `-Dcpu=baseline`)

| Artifact | Bytes | Command |
|---|---|---|
| flong, x86_64, static no-libc, stripped (`.#default`) | 1,349,296 | `nix build --no-link --print-out-paths .#default` → `/nix/store/3734ykb3…-flong-launcher-0-unstable-2026-11-10/bin/flong` |
| flong-seccomp, glibc + libseccomp (`.#seccomp`) | 154,280 | `nix build … .#seccomp` → `/nix/store/kwf6arin…/bin/flong-seccomp` |
| flong (`checks.launcher`, version 0) | 1,349,216 | `nix build --no-link .#checks.x86_64-linux.launcher` |
| flong-seccomp (`checks.seccomp`) | 154,232 | `nix build --no-link .#checks.x86_64-linux.seccomp` |
| flong, local, dummy `/nix/store/x` paths | 1,348,608 | `nix develop -c zig build install -Dcpu=baseline --release=safe -Dset=launcher -Dself=/nix/store/x/bin/flong -Dtini=/nix/store/x/tini -Dbwrap=/nix/store/x/bwrap -Dpasta=/nix/store/x/pasta -Dnewuidmap=/run/wrappers/bin/newuidmap -Dnewgidmap=/run/wrappers/bin/newgidmap -Dcache=/nix/store/x/flong-cache -Dseccomp=/nix/store/x/flong-seccomp --prefix $SCRATCH/prefix/launcher` |
| flong-seccomp, local | 154,176 | `nix develop -c zig build install -Dcpu=baseline --release=safe -Dset=seccomp -Dself=/nix/store/x-flong-seccomp --prefix $SCRATCH/prefix/seccomp` |
| bpfdump (fixtures, glibc+libseccomp) | 43,720 | `… -Dset=fixtures --prefix $SCRATCH/prefix/fixtures` |
| syscall-probe / swapper / ioctl-probe (static) | 22,024 / 20,520 / 19,016 | same fixtures set |
| flong aarch64 (cross step; same size in Nix cross-aarch64) | 1,198,856 | `nix develop -c zig build cross -Dcpu=baseline --release=safe --prefix $SCRATCH/prefix/cross` |
| syscall-probe / swapper / ioctl-probe aarch64 | 21,984 / 20,272 / 18,744 | `zig build cross` |

The shipped size is printed but not gated (native.nix:249).

#### 0.16-breaking call sites

Trial in `/tmp/claude-1000/inventory-flong` with 0.16.0.

*build.zig configure* (4 kinds of fix, then `zig build --help` lists every step):
- `NativePaths.detect` takes 4 args: build.zig:278, 910, 1008 →
  `(b.allocator, b.graph.io, &target.result, &b.graph.environ_map)`.
- `std.fs.realpathAlloc`: build.zig:865, 1573 →
  `std.Io.Dir.realPathFileAbsoluteAlloc(b.graph.io, …)` (one copy) or
  `std.Io.Dir.cwd().realPathFileAlloc(io, zig_exe, alloc)` (another copy;
  both configured).
- `.expect_term = .{ .Exited = 1 }` → `.exited`: build.zig:361, 375, 435,
  444, 508.
- `captureStdOut()` → `captureStdOut(.{})`: build.zig:530, 533.

*Dependencies:* minish v0.1.0/chilli v0.2.2 fail in their own build.zig
(`std.fs.cwd` gone). With minish v0.4.0
(`minish-0.4.0-SQtSTf4pAwD_cAeVB-_rEHWmgovGY-yaCGjX9hpVKxKH`) the build
runs. zwanzig v0.15.1 builds from source ("Zig frontend 0.16.0").

*Full trial* (`install -Dset=seccomp lint compile-fail fmt test-libc abi
cross test analyze -Ddev=true`): 86/226 steps OK, 56 failed, 62/62 tests
passed. Distinct errors:

- **argv/env:** `std.process.argsAlloc` gone: build/gen_decl_docs.zig:38,
  tools/fdlint.zig:113, build/schema.zig:13. **`std.os.argv`/`environ`
  gone: src/sys.zig:83, 89, 105, 114** (`sys.argv`, `sys.environ`,
  `sys.argvSlots`, `sys.environSlots`, used by every root). Roots:
  src/main.zig:258, src/seccomp/main.zig:45, fixtures,
  tests/zig/{fakecmd,fakebwrap,procdriver,launchdriver,ttydriver,walker}.zig.
  Direct uses: tests/zig/fakecmd.zig:60, 89, 90, 113, 117, 151, 173, 176,
  195, 198, 202-204; fakebwrap.zig:21; proc_props.zig:418;
  procdriver.zig:155; prologue_test.zig:534; wrapper_test.zig:246, 250.
- **`linux.E.init` gone (use `linux.errno`): 95 sites in 17 files**,
  including src/sys.zig:119 (core of `result()`). prologue_test 18,
  wrapper_test 12, procdriver 9, fakecmd 8, proc_props 8, probe 5,
  launchdriver 5; bwrap_test, fd_props, childpid_test, pasta_hook_test,
  spec_test 4 each; sys 3; fakebwrap 3; swapper 2; ioctl_probe, bpfdump 1.
- `linux.Stat` gone: src/sys.zig:50, src/fixtures/bpfdump.zig:462,
  tests/zig/pasta_hook_test.zig:224, 385. `linux.Statx.__pad2` renamed:
  comptime assert at src/sys.zig:592 fails. `linux.waitid` takes a 5th arg
  (`?*rusage`): src/sys.zig:741. `linux.kill`/`pidfd_send_signal` take the
  `SIG` enum: src/sys.zig:760, 1102; src/fixtures/probe.zig:70, 165.
- `std.mem.trimLeft/trimRight` gone (11): src/passwd.zig:47,
  src/launch/resolv.zig:155, src/seccomp/project.zig:284,
  build/gen_decl_docs.zig:138, tools/fdlint.zig:123, 127,
  tests/zig/launchdriver.zig:175, 184, 289, proc_props.zig:67,
  prologue_test.zig:113.
- `std.net` gone: src/decl.zig:699-700. `std.crypto.random` gone:
  tests/zig/libc_hookenv.zig:102, libc_num.zig:141. `std.fs.openDirAbsolute`
  gone: tests/zig/childpid_test.zig:572 (likely also fuzz.zig:264,
  launch_test.zig:122, 378).
- **`std.Options.keep_sigpipe` gone at 11 roots**: src/main.zig:64,
  src/seccomp/main.zig:30, 4 fixtures, fakecmd, fakebwrap, procdriver,
  launchdriver, ttydriver. 0.16 start code no longer touches SIGPIPE;
  tests/zig/walker.zig:36 never set it, so walker goes from "ignored" to
  the default action.
- zig fmt 0.16 rejects src/num.zig:136 (wants `"٣",` on its own line).
- abi.zig:59: 0.16 bundles uapi **6.19.0** (LINUX_VERSION_CODE 398080),
  not 6.13.4.
- compile-fail not reached; its 15 expected messages (build.zig:327-342)
  quote exact compiler wording (e.g. `'fd.Handle(.file,.owned)'`).

*Not reached but likely affected:* std.posix fchmodat
(tests/zig/launch_test.zig:55), flock (450-451), getenv (fakecmd.zig:147),
getcwd (wrapper_test.zig:395, 424); `std.Thread.sleep`/`spawn`
(launch_test.zig:362, 416); `std.fs.cwd()` users build/gen_decl_docs.zig:44,
91, build/schema.zig:15, tools/fdlint.zig:130, 135, 157, 161,
tests/zig/bwrap_test.zig:47, proc_props.zig:28, prologue_test.zig:513,
524, wrapper_test.zig:238, launch_test.zig:80, 115, 352-353;
`std.fs.max_path_bytes` (bwrap_test:42, proc_props:23, prologue_test:508,
519, wrapper_test:233); `std.process.exit` (gen_decl_docs:41, 48, 89;
schema:22; fdlint:165). `std.mem.indexOf`/`lastIndexOf` (152 and 18 lines)
are deprecated aliases and still compile. `std.Io.Writer`, `std.json`,
`std.Random.DefaultPrng` already use the new API.

*Analyze step on 0.16:* the flong report said the zwanzig Run step
"failed". A follow-up **could not reproduce** this: `zig build analyze
-Ddev=true` passes on 0.16.0 (5/5 steps, exit 0, three fresh caches, under
a pty, with `-j1`). What 0.16 prints is a warning block: a ` w` marker and
a red `failed command:` line, because the step succeeded but wrote stderr
with no stderr check (Run.zig:2721-2729, build_runner.zig:1170-1173,
1516-1521). quoin shows the same block. The original failure has no log;
it may predate the copy's build.zig edits. Adding an `expect_stderr_match`
(for example `"analysis complete"`) to the planted-bugs run silences it.

#### Start code on 0.16

Static no-libc, single-threaded, stripped `main() noreturn` calling
exit_group, `-O ReleaseSafe --stack 0` (files in
`/tmp/claude-1000/inventory-flong-start`):
- 0.15.2: 22,240 bytes; strace shows `rt_sigaction(SIGPIPE)` before main.
- 0.16.0: 196,144 bytes; **`sigaltstack(…, ss_size=262144)` before main**
  (`std.options.signal_stack_size`, default 1<<18, std.zig:125), even with
  `enable_segfault_handler=false`.
- With `.signal_stack_size = null`: no syscall before main, 195,760 bytes.

Why it grows (follow-up): `size -A` text 11706 → 114377, rodata 6880 →
48488, data 64 → 19056, bss 73 → 262184 (the signal stack is only bss).
The default ReleaseSafe panic (`debug.defaultPanic`, debug.zig:489) writes
via `lockStderr` through `std.Options.debug_io` =
`Io.Threaded.global_single_threaded.io()` (std.zig:207-221;
Io/Threaded.zig:1693, 1704), linking the whole `Io.Threaded` vtable
(74,032 bytes: processSpawnPosix 8005, lookupDns 7007, netLookup 5185, …)
and `Progress.global_progress` (18,152).

| Variant (0.16) | Bytes |
|---|---|
| base | 196,144 |
| + `signal_stack_size = null` | 195,760 |
| + `allow_stack_tracing = false` | 195,760 |
| + `networking = false` | 155,328 |
| `panic = std.debug.simple_panic` | 180,128 |
| **`panic = std.debug.FullPanic(f)`, f calls exit_group** | **1,848** |
| `panic = std.debug.no_panic` | 1,680 |
| custom panic touching only `debug_io.vtable.crashHandler` | 2,168 |
| custom panic using `std.debug.print` | 191,264 (150,816 with `networking=false`) |
| `std_options_debug_threaded_io = null` + default panic | compile error (std.zig:221) |
| 0.15.2 with custom `FullPanic` | 2,400 |

flong already installs a direct-write panic (src/main.zig:74,
src/seccomp/main.zig:36, fixtures, via `msg.onPanic`, src/msg.zig:143-150)
and grep finds no `std.debug.print`, `lockStderr`, `Io.Threaded` or
`std.log.` use in src/ or launcher/. So this growth should not reach the
shipped binary unless the port adds a `std.Io`/`debug_io` user (not
confirmed by a full 0.16 build). tests/zig/fakecmd.zig:47 sets no panic and
will grow. flong's design depends on "no syscall before main"
(build.zig:1654-1658, quirk 20): **every root needs
`.signal_stack_size = null`.**

#### Build API uses that 0.17 changes

- Options: `b.option` 58, 59, 62, 67, 1619-1625; standard options 47, 51.
- `b.step` 88, 245, 325, 380, 451, 457, 517, 538, 546, 568, 926, 954;
  `addFail` 75, 80, 84, 90, 459; lazy deps 91, 460, 793, 969.
- `addRunArtifact` dozens (96–1844); `addSystemCommand` with
  `b.graph.zig_exe` (compile-fail) 345, 368.
- `addArgs` 346-358, 370, 397, 406, 441; `addPrefixedFileArg` 347-359,
  369-373; `addFileArg` 407, 442, 465, 475, 476, 1845; `addArg` 464, 474,
  532; `addDirectoryArg` 466; `addOutputFileArg` 1846.
- Run checks (expect_stderr_match, expect_stdout_match, expect_stdout_exact,
  expect_term) 360-361, 374-375, 434-444, 505-508; `setCwd(b.path("."))`,
  `has_side_effects`, `stdio = .inherit` 394-396, 468-469.
- `addFmt` 452; `addUpdateSourceFiles` + `addCopyFileToSource` +
  `captureStdOut` 529-533; `addInstallArtifact` with `dest_dir` "cross"
  539, 540, 552, 562, 955, 1050; `installArtifact` 73, 78, 82; `addOptions`
  with `addOption`/**`addOptionPath`** (0.17 splits it) many sites;
  `addTranslateC` 4 sites.
- `b.graph.host` 385, 460, 519, 524, 599, 1840; `resolveTargetQuery` 547,
  576; `linkSystemLibrary("seccomp")` 103, 298, 1935, 1999;
  `exe.stack_size = 0` 1049, 1347, 1517, 1549, 1661, 1870, 1897, 1939,
  1949, 2002.
- Not used: `b.args`, `b.build_root`, `b.run`, custom `makeFn`, runner
  override.

#### @cImport / translate-c

No real `@cImport`. The only hit is the planted tests/zig/lint/bad.zig:31;
fdlint bans it at tools/fdlint.zig:216. `addTranslateC`, all test-only:

| Site | Header | Notes |
|---|---|---|
| build.zig:267 | tests/zig/scmp.h (seccomp.h) | NativePaths at :278 |
| build.zig:577 | tests/zig/abi.h | uapi for x86_64 and aarch64 musl |
| build.zig:903 | tests/zig/fcntl.h | O_TMPFILE; NativePaths at :910 |
| build.zig:1001 | tests/zig/termios.h | NativePaths at :1008 |

#### @bitCast (42 sites, none changes under 0.17)

| Class | Sites | Risk |
|---|---|---|
| packed struct(u32) `linux.O` → u32 (5) | src/fd.zig:663, 686; src/init.zig:286; tests/zig/libc_launch.zig:10; tests/zig/abi.zig:334 | none (packed already logical bits) |
| same-size int↔int (37) | src/sys.zig:146, 232, 241, 349, 442, 491, 495 (×2), 505 (×2), 509, 514, 605, 637, 660, 825, 891, 1030, 1044, 1054, 1061, 1067, 1075, 1081, 1087, 1096 (232/241 u64→i64, 1096 u32→i32); src/proc.zig:91 (i32→u32); src/launch/subid.zig:194 (u64→i64); src/fixtures/probe.zig:105, 109, 117; src/fixtures/bpfdump.zig:264, 283, 421, 422, 572 (result into an extern struct's i32 field, cast itself scalar), 580; tests/zig/launch_test.zig:107; tests/zig/fd_props.zig:187 | none |

No int↔float, array, vector, extern struct/union or enum target.

#### Other 0.17 hazards

- None: `errdefer |`, `void{}`, `i0`/`u0`, `containsAtLeast`,
  `StackFallbackAllocator`/`stackFallback`. `DebugAllocator`/GPA only in
  planted tests/zig/lint/bad.zig:58-59 and the fdlint `alloc` rule
  (tools/fdlint.zig:206).
- **`a ** n`: 49 lines** (a follow-up counted 48 non-comment lines). Non-test
  src: src/init.zig:555, 557; src/fixtures/bpfdump.zig:362, 575;
  ioctl_probe.zig:34; probe.zig:125, 129, 138, 147, 177, 218;
  src/decl.zig:1159; src/record.zig:1154. src test blocks: dbus.zig:530,
  534; manager.zig:469; check.zig:789; launch/lookup.zig:184; msg.zig:375,
  400; decl.zig:1742, 1744; names.zig:28-29; spec.zig:741. tests/zig:
  childpid_test 366, 368; libc_num 42, 126; libc_fixtures 47; spec_test 343,
  344, 396, 422, 440, 504, 825, 836; init_test 98-99; inputs 38, 39, 97,
  106, 122. Most are string repetition (`"a" ** 255`), which `@splat`
  cannot replace.
- `std.builtin.OptimizeMode` 29× in build.zig; `std.builtin.Endian`
  src/dbus.zig:177.
- `@intFromEnum` 28, `@enumFromInt` 13 (errno.zig 14, libc_scmp 8, probe 5,
  dbus 2, one each in gen_decl_docs, decl, manager, fd, bpfdump, scmp,
  bwrap_test, msg, launch_test, abi, sys, libc_errno).
- `@import("builtin")`: `.cpu.arch` src/fd.zig:1067 (`.endian()`),
  src/check.zig:186, 988, src/fixtures/probe.zig:72, 220, 247,
  src/sys.zig:382, 806, 1118, src/syscalls.zig:24, tests/zig/abi.zig:56,
  67; `.os.tag` tests/zig/lint/good.zig:14; `builtin.output_mode`
  src/sys.zig:278; `builtin.mode` childpid_test.zig:41, fuzz.zig:41.
- **`builtin.mode == .Debug` breaks on 0.17**: tests/zig/childpid_test.zig:41,
  tests/zig/fuzz.zig:41. Safe forms elsewhere: build.zig:53, 460, 519,
  525, 1841.
- **Reflection (`.fields` / `std.meta.fields`) breaks on 0.17.** 22 listed
  sites: build.zig:1104, 1408, 1683; src/decl.zig:1155 (needs defaults,
  plus `** fields.len` at :1159), 1217, 1242, 1291, 1292, 1321, 1326;
  src/decl_docs.zig:171 (defaults), 394, 799; src/layer.zig:169, 201;
  src/main.zig:408, 451; src/msg.zig:222 (tuple; `fields[a].name` at 236);
  src/syscalls.zig:48; tests/zig/abi.zig:85, 95, 247 (`std.meta.fields`).
  Plus capture sites: src/check.zig:214; src/decl.zig:1121, 1312-1314
  (`supported()` takes field structs); src/decl_docs.zig:237, 274, 326
  (defaults), 422, 441, 453, 498 (defaults), 751. And `.decls` at
  tests/zig/abi.zig:272 (0.17: `decl_names`). Defaults needed at 4 places.
- `@hasDecl`: 17 sites (src/fd.zig:406, 407, 510, 544, 704;
  launch/hook.zig:104; launch/pasta.zig:90; decl.zig:593, 1308;
  decl_docs.zig:114, 115, 135; layer.zig:154, 159; tests/zig/abi.zig:275,
  368, 376). Every target decl is `pub`, so 0.17's pub-only rule changes
  nothing.
- Minimum OS: flong already needs Linux 6.13 (DESIGN.md:2291-2309). Linux
  only.

#### Tests and guards

- 350 `test` declarations; `test` step runs 342 (spec_test only with a
  /nix/store zig, build.zig:860-884).
- minish in 7 files (fd_props 299, proc_props 884, props 54, fuzz 367,
  spec_test 1094, childpid_test 611, tty_test 278): 19
  `minish.check(alloc, gen, fn, .{ .num_runs, .seed })`; `gen.int` 18,
  `gen.list` 8+2, `gen.tuple` 2+2, `gen.intRange` 3, `gen.string` 1
  (`.charset = .custom`). **`gen.tuple` (the no-argument helper) is removed
  in minish v0.4.0**; check whether these are `tuple2`/`tuple3` or that
  helper.
- Compile-fail: 14 cases at build.zig:327-342 plus decl_undocumented at
  366-376, exact stderr matches.
- fdlint (tools/fdlint.zig:13-42) over src and tests/zig (build.zig:393-403);
  25 planted findings in bad.zig (410-433), none in good.zig (440-445).
- zwanzig: `.zwanzig.json` enables store-violations-engine,
  unreachable-code-engine, empty-catch-engine; 28 open models, 4 close
  models (close, await, reapNow, release); runs over src/ (461-470) and
  must find exactly 27 planted bugs B1-B28 in tests/zig/analyze/bugs.zig
  (472-509).
- test-libc against glibc/libseccomp; abi against bundled headers, planted
  mismatches in cross-aarch64 (native.nix:355-364).

#### Risks

1. argv/environ: `sys.argv/environ/argvSlots/environSlots`
   (src/sys.zig:80-116) feed every root; `argvSlots` mutates the kernel's
   pointer slots (flong init). Whether `std.process.Init.Minimal`'s slots
   are still writable kernel memory is **unknown**.
2. Start-code purity: `signal_stack_size = null` in every root;
   `keep_sigpipe` gone (walker changes behaviour).
3. errno plumbing: 95 `E.init` sites, `SIG` enum, `Stat`/`Statx` layout
   asserts (sys.zig:50, :592).
4. minish v0.4 API changes; analyze warning block.
5. Nix: `zig_0_15` → `zig_0_16` (native.nix:35, flake.nix:255), recompute
   hash (native.nix:130); `zig-pkg/` not in .gitignore.
6. abi pins 6.13.4 → 6.19.0; plant message wording (native.nix:360-363).
7. Compile-fail expectations re-checked on 0.16 and again on 0.17.
8. Build API: ~4 configure fixes now; 0.17 rewrites every add*Arg and
   `b.graph` use. Moving translate-c helpers out of build.zig reduces churn.
9. 0.17 language: `**`, enum builtins, `std.builtin`, `builtin.cpu.arch`,
   reflection rewrite.
10. Doc drift (DESIGN.md, native.nix, missing ZIG.md).
11. The wrapper_test environment-size bug hides regressions under mise.

### 2.2 quoin (`sudo/`)

`/home/dan/Projects/quoin`, trunk, f87f25e (v0.85.17). **Local clone is
behind**: consumers pin v0.100.28 (6af125cc), not present locally (7.2).
One Zig package at `sudo/`; the rest is Go and Nix. 4,973 lines of Zig.
**quoin has no `.gitignore`**, so in-place builds leave `sudo/.zig-cache`,
`sudo/zig-out` and, on 0.16, `sudo/zig-pkg`.

| File | Lines | Role |
|---|---|---|
| sudo/build.zig | 487 | install, test, lint, fmt, analyze; Linux (:7-149), macOS `darwin()` (:316-381) chosen at :16 |
| sudo/build.zig.zon | 21 | `.quoin_sudo`, fingerprint 0x2841faf8f85912db |
| sudo/abi.zig | 21 | the only `@cImport` |
| sudo/fd.zig | 176 | Linux handle table cut from flong (cap 4) |
| sudo/fd_darwin.zig | 144 | same table over std.c |
| sudo/chain.zig | 191 | process chain above sudo (/proc, sysctl) |
| sudo/wire.zig | 379 | wire v2 to the Go gate |
| sudo/agent.zig | 311 | ssh-agent client + test server |
| sudo/ids.zig | 232 | drop/restore euid/egid/groups |
| sudo/args.zig | 97 | sudo getopt mirror |
| sudo/quoin_sudo.zig | 321 | Linux audit plugin |
| sudo/quoin-sudo-pam.zig | 50 | pam_exec helper |
| sudo/quoin-sudo-check-plugin.zig | 30 | dlopen check |
| sudo/quoin_sudo_darwin.zig | 369 | macOS plugin |
| sudo/quoin-sudo-pam-darwin.zig | 79 | macOS PAM module |
| sudo/quoin-sudo-check-plugin-darwin.zig | 62 | macOS pre-flight |
| sudo/sshkey.zig | 237 | key lines, P-256/Ed25519 verify |
| sudo/sudomsg.zig | 115 | quoin-sudo-v1 marshal |
| sudo/lint.zig | 143 | fdlint subset |
| plugin_test 490, remote_test 557, darwin_test 391, misread_test 31, planted_type 6, analyze/bugs.zig 32, .zwanzig.json 6 | | |

C: sudo/oracle/oracle.c, sudo/darwin/quoin-sudo-load.c.

#### Provisioning and pins

- Zig only from Nix: `pkgs.zig_0_15` at nix/sudo-zig.nix:5,
  **nix/sudo-zig-darwin.nix:9** (the report said :98; the file is 77
  lines), flake.nix:165 (devShell). No mise, no setup-zig. flake.lock
  nixpkgs b1b875982b17dabde9b4a37f3e229e74913e6db3 (nixos-unstable) has
  zig_0_15 0.15.2 and zig_0_16 0.16.0.
- build.zig.zon:19 `minimum_zig_version = "0.15.2"`; :18 fingerprint.
- Deps (lazy, `-Ddev` only; :4-5 says "pinned to flong's tags"): minish
  :7-11 v0.1.0 and zwanzig :12-16 v0.15.1, **same URLs and hashes as
  flong**. PLAN.md's `-` is wrong.
- Nix FOD: nix/sudo-zig.nix:35-44 `zigDeps = zig.fetchDeps { fetchAll =
  true; hash = "sha256-GicN77r9Oh9xPqlIP5CS0/y2hSenxlM6thAiv1bjBW8="; }`
  (same as flong's); update recipe at :33-34; linked by `ln -s ${zigDeps}
  "$ZIG_GLOBAL_CACHE_DIR/p"` (:22-24, line 23) for the check derivation only
  (deps=true at :87).
- nix/sudo-zig.nix:28: `TERM=dumb zig build ${steps} -j$NIX_BUILD_CORES
  $zigDefaultCpuFlag --release=safe --prefix $out`. The report also cites
  nix/sudo-zig-darwin.nix:134 (`-Dsudo-include`, `-Dpam=${openpam}`),
  :151-158 (arm64e loader patch) and :1-5 (Apple sudo 1.9.13p2); **these
  line numbers cannot be right** for a 77-line file and were not rechecked.
- Release mode ReleaseSafe; build.zig:9 `preferred_optimize_mode`.
- Flake outputs: packages x86_64/aarch64-linux `quoin-sudo-plugin`
  (flake.nix:151), aarch64-darwin (:143); checks `sudo-zig` +
  `quoin-sudo-plugin` (Linux :201-202, darwin :174-178). nix/sudo.nix:11
  rebuilds the plugin against the host's sudo, so **every NixOS host that
  enables quoin sudo compiles this Zig**. tests/sudo-gate.nix:17,
  tests/sudo-remote.nix:12 use passthru variants misread and logIds.
- CI: one ci.yml; job `check` matrix with leg `sudo-zig` (:22) running
  `nix build .#checks.x86_64-linux.<leg>` (:79); cachix/install-nix-action@v31,
  **no binary cache** (:36-39), so no cache keys. **Darwin and aarch64-linux
  never run in CI.** `release` (:89-119) creates a release with notes only
  (`gh release create "v$V" --title … --notes-file notes.md --target
  "$GITHUB_SHA"`, :119); **no binaries attached**; none of 19 releases has
  an asset.
- Version from scripts/version.sh (VERSION=0), not stamped into Zig.
- docs/signer-and-sudo-plan.md:402 names "0.15.2 from nixpkgs' zig_0_15".

#### Baseline (0.15.2)

- `nix build --no-link -L .#checks.x86_64-linux.sudo-zig
  .#checks.x86_64-linux.quoin-sudo-plugin`: **PASS**, 8m02s cold (runs `zig
  build test lint fmt analyze -Ddev=true -Doracle=<sudo 1.9.17p2 oracle>`);
  minish 2000, 1000, 2000, 10000 runs pass; zwanzig no issues on 8 files,
  3 planted findings + "Found 3 issue(s)" + exit 1 match. Output
  `/nix/store/8axizd3q8cwnl2cfqhg710nlyi5fjmq1-quoin-sudo-zig-check-0`.
- Local copy: `zig build lint`, `fmt` exit 0; install builds.
- Darwin checks not run (no Mac); aarch64-linux not built.

#### Release sizes (ReleaseSafe)

| Artifact | Bytes | Command |
|---|---|---|
| lib/quoin_sudo.so (Linux x86_64, Nix) | 56,328 | `nix build .#checks.x86_64-linux.quoin-sudo-plugin` (= `zig build install -j$NIX_BUILD_CORES $zigDefaultCpuFlag --release=safe -Dsudo-version=1.9.17p2`, then strip -S); `stat -c %s /nix/store/5v33kw8bqbm8865gb67ngqnwvai0kjf3-quoin-sudo-plugin-0/lib/quoin_sudo.so` |
| bin/quoin-sudo-pam (Nix) | 30,616 | same |
| bin/quoin-sudo-check-plugin (Nix) | 31,656 | same |
| quoin_sudo.so / pam / check-plugin, local 0.15.2, native CPU, no strip | 57,104 / 31,872 / 31,992 | `cd /tmp/claude-1000/inventory-quoin/sudo15 && NIX_CFLAGS_COMPILE='-isystem <sudo-1.9.17p2-dev>/include -isystem <linux-pam-1.7.2>/include' NIX_LDFLAGS=-L<linux-pam>/lib zig build install -Dsudo-version=1.9.17p2 --release=safe --prefix outlinux` |
| aarch64-macos cross, 0.15.2 (NOT shipped: Linux sudo 1.9.17p2 header, unsigned, no arm64e loader): quoin_sudo.so / quoin_sudo_pam.so / check-plugin | 262,752 / 71,352 / 126,280 | `zig build install -Dtarget=aarch64-macos --release=safe --prefix outmac -Dsudo-include=<sudo-1.9.17p2-dev>/include -Dpam=<openpam-20230627>` |

No shipped darwin baseline exists: nothing builds or publishes one.

#### 0.16-breaking call sites (trial, in order)

1. build.zig:146, :307 `.Exited` → `.exited`.
2. minish 0.1.0/chilli 0.2.2 fetched into `sudo/zig-pkg/`; their build.zig
   fails (minish build.zig:29, chilli build.zig:68) and **breaks every
   step, including analyze and darwin install**.
3. analyze passes with minish removed (zwanzig v0.15.1, 0.16 frontend);
   prints the cosmetic `failed command:` block.
4. fmt passes unchanged.
5. lint.zig:43 `std.process.argsAlloc` gone.
6. Linux Debug install: fd.zig:37, :67, :80, :128 `linux.E.init` gone
   (`linux.errno(rc)`, os/linux.zig:592); wire.zig:200 `std.meta.intToEnum`
   gone (`std.enums.fromInt`); quoin-sudo-pam.zig:11,
   quoin-sudo-check-plugin.zig:9 `std.os.argv`.
7. **Linux ReleaseSafe (the shipped mode): the `@cImport` of glibc unistd.h
   fails** in the fortified `getgroups` wrapper ("expected type 'c_int',
   found 'bool'" in `__builtin.object_size(..., __USE_FORTIFY_LEVEL > 1)`),
   reached from ids.zig:40. Debug does not hit it. Fix:
   `_FORTIFY_SOURCE=0`/`#undef`, or translate-c in build.zig.
8. Darwin install (cross): quoin-sudo-check-plugin-darwin.zig:21
   `std.os.argv`; quoin_sudo_darwin.zig:126, :271 `std.fs.cwd`; :210
   `std.crypto.random`; :332 `std.time.timestamp`.
9. Darwin test (cross, compile only): agent.zig:188, fd_darwin.zig:128,
   ids.zig:217 `std.posix.unlink`; sshkey.zig:173, darwin_test.zig:259
   `KeyPair.generate()` takes `io`; "failed to parse TBD file" (expected
   on Linux).

The Nix follow-up (0.16 in Nix, minish v0.4.0) also hit: plugin_test.zig:213
`posix.close`; remote_test.zig:81/266 `fs.cwd`/`openDirAbsolute`;
plugin_test.zig:466 wrong argument count; check overall 21/50 steps, 12
failed.

Other sites from grep (0.16 std confirmed): getenv quoin-sudo-pam.zig:11-12;
`std.process` lint.zig:43, :88 (exit still exists, process.zig:854);
plugin_test.zig:424-436 Child; quoin-sudo-check-plugin-darwin.zig:50-53
Child/spawnAndWait/`.Exited`. `std.fs`: lint.zig:28, :55, :60, :62, :80,
:84; args.zig:94; quoin-sudo-check-plugin.zig:26;
quoin-sudo-check-plugin-darwin.zig:30, :58; quoin_sudo_darwin.zig:126,
:268, :271-288, :297, :325-328; plugin_test.zig:140, :416/:447, :463-466;
remote_test.zig:78, :81, :266; darwin_test.zig:57-58, :244, :263-264,
:274. `std.posix` socket family (mostly test helpers): fd.zig:157-166,
fd_darwin.zig:128-138, ids.zig:217-223, quoin-sudo-pam.zig:42-48,
agent.zig:180-225, plugin_test.zig:76-135, :213-229, :272,
remote_test.zig:107-182, :429-459, darwin_test.zig:151-237, :293-356;
**production** quoin-sudo-check-plugin-darwin.zig:40-41 `fstatat`/`S.IFMT`.
`std.net.Address.initUnix`: fd.zig:159, fd_darwin.zig:130, ids.zig:221,
quoin-sudo-pam.zig:46, agent.zig:190, plugin_test.zig:133,
remote_test.zig:110, darwin_test.zig:235. `std.Thread.spawn` (still
present): agent.zig:195, plugin_test.zig:84, remote_test.zig:146,
darwin_test.zig:158. `KeyPair.generate()`: sshkey.zig:173-174,
darwin_test.zig:259-260, :266 (generateDeterministic unchanged:
sudomsg.zig:73, sshkey.zig:190). `std.mem.trimRight`: lint.zig:51, :54.

**Design issue:** the plugins are shared libraries loaded into sudo, with
no `main` and so no `Io`. Linux needs none (raw syscalls); the darwin
plugin uses std.fs, crypto.random and time.timestamp
(quoin_sudo_darwin.zig:126, :210, :271, :325, :332), so it must build an
`Io.Threaded` inside sudo's process or use std.c calls through fd_darwin.

#### Build API (0.17)

`b.option` :10-14, :317-319; standard options :8, :9; install :18, :22,
:328, :330 (`addInstallFileWithDir`), :23-24, :331; `addFail` :26, :31, :33,
:124; `b.step` :29, :105, :119, :122, :343, :367, :379; lazyDependency :34,
:125 (`zw.artifact` :126, `minish.module` :56, :92); addTest :39, :43, :48,
:67, :84, :345, :347, :349; addRunArtifact :40, :43, :63, :80, :100, :111,
:128, :136, :303, :345, :347, :365, :372; addOptions :45-46, :156-163,
:384-389; `abi.linkSystemLibrary("pam")` :187, :410; addIncludePath :405,
:407; addLibraryPath :408; setCwd :112, :373; has_side_effects :113, :132,
:374; stdio inherit :114, :133, :375; addArgs :115, :305, :376; addArg
:129, :137; addFileArg :130-131, :138-139, :304; addCheck :144-146,
:306-307; b.graph.host :109, :125, :370; addLibrary(.dynamic) :219, :284,
:444, :475; addExecutable :107, :244, :261, :331, :368; addFmt :120, :380;
b.pathJoin :407-408; `linker_allow_shlib_undefined` :470;
`std.os.linux.getuid()` inside build.zig :35. Not used: `b.args`,
`b.build_root`, addOutputFileArg.

#### @cImport

One site, sudo/abi.zig:8-19: `@cDefine("_GNU_SOURCE", {})` (:9);
unistd.h (:10), grp.h (:11), sudo_plugin.h (:12), security/pam_appl.h
(:13); on macOS (:14) also security/pam_modules.h (:15), sys/sysctl.h
(:16), dlfcn.h (:17). PLAN.md's "3 files" is wrong: build.zig:186 is a
comment, lint.zig:125 the lint string.

Used translations: Linux SUDO_API_VERSION_MAJOR/MINOR, SUDO_AUDIT_PLUGIN,
SUDO_POLICY_PLUGIN, SUDO_CONV_ERROR_MSG, struct_audit_plugin,
sudo_printf_t, sudo_conv_t, PAM_* constants, pam_start/authenticate/
acct_mgmt/end, struct_pam_conv/message/response, pam_handle_t,
getresuid/getresgid/getgroups/setgroups/setegid/seteuid, gid_t. macOS
struct_kinfo_proc (kp_proc.p_comm, p_un.__p_starttime,
kp_eproc.e_ppid/e_tdev/e_pcred), CTL_KERN, KERN_PROC, KERN_PROC_PID,
sysctl, getpid, geteuid/getegid, dlopen/dlsym/dlclose,
RTLD_NOLOAD/RTLD_LAZY, pam_get_item, PAM module API. Macro arithmetic
`SUDO_API_VERSION_MAJOR << 16` (abi.zig:21, misread_test.zig:25).

#### @bitCast (3, none changes)

| Site | Cast | Class |
|---|---|---|
| chain.zig:70 | `l.ppid = @bitCast(kp.kp_eproc.e_ppid)` | pid_t i32 → u32 (translate-c field) |
| chain.zig:72 | `@bitCast(kp.kp_eproc.e_tdev)` | dev_t i32 → u32 (translate-c field) |
| chain.zig:130 | `.tty = @bitCast(tty)` | i32 → u32 |

#### Other 0.17 hazards

`**` 1 (sudomsg.zig:111 `"/" ** 200`, test); `std.builtin.OptimizeMode`
9 in build.zig (:183, :218, :242, :260, :280, :316, :403, :443, :474);
`builtin.os.tag` abi.zig:6 (:14), chain.zig:5 (:7), ids.zig:6 (:18, :25);
`@intFromEnum` 4 (wire.zig:62, :67, :165, fd_darwin.zig:38);
`@enumFromInt` 0; reflection build.zig:176 (names only). None of the
others. lint.zig:115 bans the two allocators. Minimum OS: Linux ≤ 5.10
needs; macOS has no deployment target (default 13.0 on 0.16, **15.0 on
0.17**); the target Mac's macOS version is unknown.

#### Tests and guards

59 test blocks (fd 3, fd_darwin 2, chain 4, wire 7, agent 4, ids 3, sshkey
5, sudomsg 2, pam 2, pam-darwin 3, plugin_test 9, remote_test 9,
misread_test 1, darwin_test 5). Linux `test` (build.zig:29-103) runs 42
blocks plus 5 check-plugin cases (295-301); darwin `test` (343-365) 28.
minish 4 calls, all `minish.check(allocator, minish.gen.list(u8,
minish.gen.int(u8), 0, N), fn, .{ .num_runs, .seed })`: plugin_test.zig:273
(2000, seed 0x5d0), :468 (oracle, 10000), remote_test.zig:460 (2000), :556
(1000). Goldens: wire.zig:267, sudomsg.zig:95. No compile-fail. lint rules:
extern (outside abi.zig), export, raw-number, handle-guts,
catch-unreachable (`// proven:`), alloc; skips `.zig-cache/`, `zig-out/`
(lint.zig:66), **not `zig-pkg/`** (gave 10 false handle-guts findings in
minish). zwanzig: open=connectUnix, close=close; 8 files (build.zig:127)
plus planted bugs (:140-146); darwin files not analysed.

#### Risks

1. minish blocks everything; bump in the same commit as zig_0_16; hash
   becomes `sha256-Gzt+mB0n…` (verified in Nix).
2. ReleaseSafe `@cImport` fortify failure (shipped mode).
3. `zig-pkg/` and no .gitignore; lint.zig:66 must skip it.
4. Darwin plugin has no Io; darwin untested in CI and here.
5. Test harnesses lean on removed std.posix/std.net (four test servers,
   tmpDir, symLink, Child): bulk of the work; a shared test helper.
6. zigkit fd must keep quoin's needs: error set used at
   quoin_sudo.zig:116, :120, :189, :192; ids.zig:85; quoin-sudo-pam.zig:19;
   peerUid, readFile, MSG_NOSIGNAL, darwin backend, capacity 4 in BSS.
   zwanzig models name `connectUnix`/`close`; bugs.zig line:col checks
   (build.zig:141-143) break if those move.
7. Every NixOS host rebuilds the plugin; rerun VM tests.
8. 0.17 build API: addFileArg and Run checks only.
9. macOS 13 → 15 on 0.17; Apple sudo 1.9.13p2.

### 2.3 capsper

`/home/dan/Projects/capsper`, trunk, 4f4bb26 (v0.410.293). **Local clone
is behind**: nix-config pins v0.420.300 (95aa2568). One package
(build.zig 493 lines, zon 25). `vendor/libopus` submodule (xiph/opus
503d81b, v1.6.1-68) built by build.zig. 20,074 lines of Zig. Artifacts:
`capsper` exe, host tool `gen-config-docs`, 18 pure test roots plus opus,
linux/input, linux/lid roots, `prop-tests` exe. C/ObjC helpers:
src/platform/linux/pw_helpers.c, src/platform/macos/input_helpers.c,
mic_permission.m, src/backend/coreml/helpers.m.

Main files: src/main.zig 616; build/gen_config_docs.zig 172;
src/backend/ort/{ort_c 46, init 138, pipeline 570, vad 250};
src/backend/coreml/{init 71, pipeline 379, vad 47};
src/platform/linux/{input 1983, pipewire_c 201, audio 367, detect 512, lid
181, sink 186, mic_level 65, output_duck 35}; src/platform/macos/{audio
561, input 175}; src/shared/{config 1123, config_docs 619, http_server
1301, server 661, meeting_runner 1149, meeting 750, room_runner 411,
session 716, source 697, webvtt 916, ogg 386, opus 332, nemo_mel 263,
nemo_mel_state 301, tokenizer 194, context_graph 335, utils 585, auto_gain
102, vad 203, recorder 233, status 248, keep_awake 198, settings_form 540,
asr_types 20, prop_tests 481}.

#### Provisioning and pins

- `.mise.toml:2` `zig = "0.15.2"`; also bun 1.3.10, shellcheck 0.11.0, cosign
  3.0.5, `github:danielbodart/fulcio-codesign` `latest` on macOS only
  (`.mise.toml:10-12`). `run.ts:1` shebang `mise exec -- bun`.
  build.zig.zon:19 `minimum_zig_version = "0.15.2"`.
- Nix: `zig_0_15` at nix/package.nix:11, :50 (zig build at :69, cache dir
  :66). nixos-26.05 rev c25784012c99 has zig_0_16 0.16.0 and onnxruntime
  1.24.4. **No FOD**: nix passes `-Dprop-tests=false` (:73), dropping the
  only dependency. devShell has no Zig (flake.nix:57-75); run.ts re-enters
  `nix develop` on NixOS (run.ts:394-402).
- Deps (build.zig.zon:4-17): minish v0.1.0 (same URL/hash as flong),
  `.lazy = true`; chilli 0.2.2 transitively. No zwanzig. libopus submodule;
  ORT prebuilt `dist/linux/lib/libonnxruntime.so` via Git LFS, headers in
  dist/linux/include/onnxruntime.
- No mise tasks; run.ts commands (run.ts:370-380).
- CI (ci.yml, push to trunk): `linux` (ubuntu-latest, :9-50:
  checkout@v4 with submodules + fetch-depth 0, LFS cache
  `lfs-${hashFiles('dist/linux/lib/*.so')}`, jdx/mise-action@v4, Zig cache
  `~/.cache/zig` key `zig-linux-${hashFiles('build.zig.zon','.mise.toml')}`
  restore `zig-linux-`, `./run ci` = shellcheck, `zig build test`, build
  ReleaseSafe `-Dcpu=x86_64_v3`, dist, sign, `gh release create`); `macos`
  (macos-15, :52-76, key `zig-macos-…`, NO_CREATE_RELEASE, fulcio-codesign
  + cosign); `nix` (:83-96, DeterminateSystems/nix-installer-action@main,
  `./run nix`). The tag v0.420.300 workflow differs (release job at
  :155-156, `gh release create` at :203).
- libopus flags (build.zig:325-351): `-std=c99 -DOPUS_BUILD -DVAR_ARRAYS
  -DHAVE_LRINT -DHAVE_LRINTF`. dist runs an AVX-512 objdump scan
  (run.ts:224).

#### Baseline (0.15.2)

`./run test` PASS (4m09s): 47/47 steps, 718/718 tests in 21 runs, prop
tests 24 properties × 500. shellcheck PASS. `nix build --no-link
.#capsper` PASS (1m30s); `checks.x86_64-linux.zon-renderer` built. Not
run: `nixos-module` VM check, bun integration tests, macOS job.

#### Release sizes

| Artifact | Bytes | Command |
|---|---|---|
| capsper Linux x86_64 dist binary (ReleaseSafe, unstripped, static libopus), local 4f4bb26 | 9,566,632 | `nix develop . --command mise exec -- zig build --prefix <scratch>/capsper-rel -Dversion=0.0.0-baseline -Doptimize=ReleaseSafe -Dcpu=x86_64_v3` (= run.ts:134; existing dist/linux/bin/capsper also 9,566,632) |
| capsper Linux, shipped v0.420.300 `capsper-linux-x86_64.tar.gz` (asset 14,020,011) → `bin/capsper` | 9,558,216 | `gh release download`; sha256 verified |
| capsper macOS arm64, shipped v0.420.300 `capsper-macos-arm64.tar.gz` (asset 1,137,029) → `bin/capsper` | 1,764,992 | same |
| Nix `libexec/capsper` (system libopus, nixpkgs onnxruntime 1.24.4, stripped) | 1,621,296 | `nix build --no-link .#capsper` → `/nix/store/inmcaci31a5ynav4vmaaq077wsqk929g-capsper-0.410.20261031133026/libexec/capsper` |

The 8,416-byte gap (−0.088%) between the local and shipped Linux binaries
spans ten tags, so it means nothing yet. Compare like for like (same
commit, `-Dcpu`, unstripped).

#### 0.16-breaking call sites

Trial in `/tmp/claude-1000/inventory-capsper` (cascade; true total larger).

- build.zig: `linkLibC` gone :80, :148, :162 (→
  `root_module.link_libc = true`); `linkFramework` :259, :260, :285-289 (→
  `root_module.linkFramework(name, .{})`); `linkSystemLibrary` :303, :327;
  `linkSystemLibrary2` :275 (→ `root_module.linkSystemLibrary(name,
  opts)`); `each_lib_rpath` (:282) still exists.
- minish breaks the build even with `-Dprop-tests=false` **when minish is
  already on disk** (in `zig-pkg/` or the global cache). From a clean
  state, `-Dprop-tests=false` neither fetches nor compiles it, online or
  offline, so the Nix build (fresh `$TMPDIR/zig-cache`, explicit fileset)
  does not need the minish bump (follow-up, section 5.6). With the configure
  fixes, a clean build then fails at build/gen_config_docs.zig:35.
- Source: build/gen_config_docs.zig:35 argsAlloc; src/main.zig:51
  nanoTimestamp; **src/shared/config.zig:74 `@Type` gone in 0.16** (→
  `@Enum(u8, .exhaustive, names, values)`, tested on 0.16 and 0.17;
  config.zig:61-79 builds `Channel` with `std.builtin.Type.EnumField` at
  :63); `ArrayListUnmanaged(T){}`/`.{}` needs `.empty`: utils.zig:248, 258,
  269, 280; tokenizer.zig:55, 86, 140, 153, 165; context_graph.zig:55;
  nemo_mel_state.zig:27; meeting.zig:211; webvtt.zig:65, 252, 354;
  http_server.zig:1112, 1142; source.zig:215; ogg.zig:64, 190, 371 (~13
  explicit plus ~30 `.{}`); `std.io` gone: utils.zig:543, 556, 566;
  webvtt.zig:596-609; source.zig:413; `ArrayList.writer(gpa)` gone:
  webvtt.zig:125, meeting_runner.zig:100, 138, prop_tests.zig:196;
  `anytype` writers (utils.zig:155-202, webvtt.zig:175) need
  `*std.Io.Writer`; `std.Thread.Mutex`: status.zig:36, linux/input.zig:488,
  492; `posix.openZ`: linux/input.zig:1766, 1867; `posix.pipe`:
  session.zig:617; fs `makePath/realpath/createFile/readFileAlloc`:
  lid.zig:165, 178; recorder.zig:187, 204, 219; session.zig:654;
  keep_awake.zig:148; http_server.zig:1169, 1229; opus.zig:226, 251, 272,
  300, 321; source.zig:489 (`std.fs.Dir`); nemo_mel.zig:136 (`std.fs.cwd`);
  `std.process.Child.init`: keep_awake.zig:39.
- Not yet reached: removed posix close (51), write (16), pipe (9), openZ
  (7), waitpid (4), socket/bind/listen/accept/getsockname/geteuid/dup/
  fchmodat; heaviest users linux/input.zig (69), server.zig (26, :295-312),
  session.zig (21), keep_awake.zig (18), detect.zig (17),
  meeting_runner.zig (12), linux/audio.zig (11), macos/audio.zig (11),
  room_runner.zig (8). `std.net`: server.zig:19, 294, 304; http_server.zig:97,
  138, 163 and `std.http.Server` (:174 on). `std.time.*Timestamp`:
  main.zig:51, 589, 603; server.zig:56, 534, 631; ort/pipeline.zig:155,
  199; coreml/pipeline.zig:135, 144; meeting_runner.zig:192, 554, 666, 721;
  room_runner.zig:169, 223, 300; recorder.zig:90, 174; http_server.zig:263;
  detect.zig:343, 346, 480, 483; linux/input.zig:697. `std.Thread.sleep`:
  main.zig:543; room_runner.zig:122; meeting_runner.zig:274;
  linux/sink.zig:104; macos/input.zig:73; keep_awake.zig:81, 130, 139, 186,
  195; http_server.zig:856, 1233; linux/input.zig:578, 1124, 1846.
  `std.Thread.spawn` stays (main.zig:506, 516; server.zig:278, 320;
  http_server.zig:118, 154; linux/input.zig:552, 553, 1711;
  macos/input.zig:134). argsAlloc main.zig:58-59; `getEnvVarOwned`
  config.zig:800, 804, 818, 1108; `selfExeDirPathAlloc` main.zig:185, 575,
  meeting_runner.zig:283 (→ `std.process.executableDirPathAlloc(io,
  gpa)`); `std.fs.cwd()/openFile/…` main.zig:250, 265, 329, 364, 580;
  room_runner.zig:369; source.zig:358; meeting_runner.zig:70, 891;
  config.zig:778; recorder.zig:53; http_server.zig:488, 808, 826, 935, 991;
  lid.zig:50; input.zig:1782; keep_awake.zig:115; `openDirAbsolute`
  input.zig:822, 1819; stdout writer main.zig:113, 613; `writeAll` opus.zig:202,
  meeting_runner.zig:101, 126, 140, 1031, 1120, recorder.zig:146, 165,
  room_runner.zig:409, http_server.zig:836; `makePathStatus`
  meeting_runner.zig:83; `bufPrintZ` input.zig:833, 992, 1826; mains
  main.zig:48, gen_config_docs.zig:30, prop_tests.zig:398.
- **GeneralPurposeAllocator is gone in 0.16**: main.zig:53
  (`enable_memory_limit = true`), prop_tests.zig:28, 58, 73, 118, 188, 224,
  399; `std.heap.ThreadSafeAllocator` main.zig:55 also gone.

#### Build API (0.17)

`b.option` :11, :25, :26, :32, :39, :183; addOptions :17 (+ :18-19);
`b.graph.host` :56; generator run :60 with **`addFileArg` :61,
`addOutputFileArg` :62** into `addAnonymousImport` :75, :133;
installArtifact :81; installFile :84, :89; run step :92-98 with
**`if (b.args)` :94-96**; steps :97, :101, :176; addTest loops :104-173;
lazyDependency :185; addRPathSpecial :79, :249, :252; addLibraryPath
`.cwd_relative` :248; addIncludePath :268, :270, :331-334;
addCSourceFile(s) :261, :290, :294, :304, :336; each_lib_rpath :282.
Optimize comparisons safe (:57).

#### @cImport (5 sites)

| File:line | Headers | What it uses |
|---|---|---|
| src/shared/opus.zig:16-18 | opus.h | integer macros `OPUS_GET_LOOKAHEAD_REQUEST`, `OPUS_SET_BITRATE_REQUEST`, `OPUS_OK`, `OPUS_APPLICATION_AUDIO`; variadic `opus_encoder_ctl` (:45, :83); create/encode/destroy; opaque `OpusEncoder` |
| src/backend/ort/ort_c.zig:5-7 | onnxruntime_c_api.h | `ORT_API_VERSION` macro, `ORT_LOGGING_LEVEL_WARNING`, `ONNX_TENSOR_ELEMENT_DATA_TYPE_*`, `OrtApi` fn-pointer table, opaque handles, `OrtGetApiBase` |
| src/platform/linux/pipewire_c.zig:1-5 | pipewire/pipewire.h, pipewire/stream.h, spa/param/audio/raw.h | `PW_KEY_*` string macros (:134-138), `PW_ID_ANY`, `PW_STREAM_FLAG_*`, `PW_DIRECTION_INPUT`, `enum_pw_stream_state`, `SPA_AUDIO_CHANNEL_*` incl. `SPA_AUDIO_CHANNEL_START_Aux`, `struct_pw_properties`, ~20 functions (variadic/SPA-pod work in pw_helpers.c) |
| src/platform/linux/input.zig:13-15 | linux/input-event-codes.h | integer macros only: ~222 `ev.KEY_*`, `ev.LED_CAPSL`; structs hand-written (:17-52) |
| src/platform/macos/audio.zig:17-20 | AudioToolbox/AudioToolbox.h, CoreAudio/CoreAudio.h | ~60 symbols, `k*` constants, `noErr`, CFString functions |

build.zig:154 and input.zig:17 are comments (PLAN's "input ×2, build.zig"
is wrong). No `@cDefine`, no `addTranslateC`. Other externs:
linux/input.zig:113-115 (`ioctl`, `inotify_init1`, `inotify_add_watch`),
pipewire_c.zig:53-201, macOS mic_permission and CoreML helpers.

#### @bitCast (3, none changes)

status.zig:74 (f32→u32 init), :77 (f32→u32 store), :80 (u32→f32 load), all
in `AtomicF32`.

#### Other 0.17 hazards

- `**` 20 real sites: utils.zig:233, 238; opus.zig:230, 274, 304, 325;
  settings_form.zig:278; session.zig:631, 638, 660; ogg.zig:257, 270, 285;
  config_docs.zig:150, 219; coreml/pipeline.zig:81, 247;
  ort/pipeline.zig:78; linux/input.zig:139, 472 (pipewire_c.zig:61 is a
  comment). ~6 repeat strings.
- `std.builtin` config.zig:63. `@intFromEnum` 0; `@enumFromInt` 3:
  linux/sink.zig:175 (runtime C value into enum(u8) {0,1,2}: 0.17
  safety-checked), config.zig:733 (comptime), build/gen_config_docs.zig:80.
- `builtin.os.tag` only: main.zig:2, 148, 150, 434; platform/*.zig:1-2;
  config.zig:19, 180; keep_awake.zig:145; prop_tests.zig:6, 445;
  nemo_mel_state.zig:195, 244; build.zig:8 uses `target.result.os.tag`.
- Reflection: config.zig:732 (names + values), 742; config_docs.zig:135,
  152; settings_form.zig:90, 147, 260, 280.
- Minimum OS: Linux uinput UI_DEV_SETUP (4.5+), nothing > 5.10; macOS
  `kAudioObjectPropertyElementMain` (12+) at macos/audio.zig:137, 161, 177,
  391, 406; no deployment target set.

#### Tests and guards

323 `test` declarations (linux/input 58, utils 45, meeting 34, webvtt 31,
session 26, config 21, ogg 13, http_server 13, vad 11, config_docs 11,
settings_form 9, source 8, opus 6, auto_gain 6, status 6, tokenizer 5,
context_graph 4, keep_awake 4, lid 4, recorder 3, nemo_mel_state 3,
nemo_mel 2). prop_tests.zig: 24 `minish.check(allocator, gen, fn, .{
.num_runs = runs })`, `gen.string(.{ .min_len, .max_len, .charset =
.custom, .custom_chars })` (:13, :285), `gen.intRange` (:20, :237, :281,
:283); no seed control; Linux-only guarded at :445. No compile-fail, Zig
lint or zwanzig; shellcheck only. Integration via bun tests (`ci()` does
not run them).

#### Risks

1. minish v0.1.0 does not build on 0.16; add `zig-pkg/` to .gitignore.
2. Threading `Io` through a heavily threaded, latency-sensitive program
   (.claude/rules/input-latency.md); or a zigkit `sys` layer for raw fds.
3. `@Type` → `@Enum` (config.zig:74); Channel names feed ZON and the
   home-manager renderer (flake.nix:84-136).
4. translate-c for string macros, variadic ctl, ORT table, macOS frameworks
   (untestable on Linux CI).
5. GPA with `enable_memory_limit` + ThreadSafeAllocator.
6. Nix `zig_0_16` (package.nix:11, :50); no FOD.
7. macOS only verifiable on macos-15 CI.
8. Size: compare like for like.
9. 0.17: `b.args`, addFileArg/addOutputFileArg, codegen step.

### 2.4 zerocast

`/home/dan/Projects/zerocast`, trunk, 9b95b86. Not consumed by any other
repo. **No submodules checked out in the real repo** (`git submodule
status` shows `-` for libdatachannel, svt-av1, wlroots) and no
.zig-cache; the baseline used a clone at
`/tmp/claude-1000/inventory-zerocast/base`.

Build files: build.zig 500 (dispatches on the **host** OS,
build.zig:5-9), build_shared.zig 117 (`linkDatachannel` :32-40, cmake steps
:44-74, :78-117), build_linux.zig 168, build_macos.zig 99, build.zig.zon 21
(`.paths` lists non-existent "src", :19). Zig sources in
packages/client/src: 13,007 lines including build files (main 24, cli 305,
daemon 747, control 610, session 1108, session_recorder 264,
terminal_share 544, encoder 361, svt_backend 315, encoder_contract_test
608, sw_integration 182, input_protocol 357, viewer_state 432, osc_parser
223, ivf 190, yuv 265, debounce 175, clock 99, codec 17, prop_tests 38;
linux gpu_detect 499, keymap 329, wayland/{compositor 491, compositor_test
196, app_share 711, input 259, cuda 381, nvenc 1071, nvenc_backend 76,
frame_download 107}; macos {app_share 518, frame_download 114, input 170,
keymap 337}). Committed prebuilt libs/wlroots/libwlroots.a (Ubuntu GCC
13.3) and stray ELF files test_egl_import (7.9 MB) and test_wlr_import
(9.9 MB) at the root (commit 3e3ac27).

#### Provisioning and pins

- `.mise.toml:1-5` `zig = "0.15.2"`, `bun = "1.2"`. bootstrap.sh installs
  mise into `tools/`, `ZIG_GLOBAL_CACHE_DIR=$MISE_DATA_DIR/zig-cache`
  (bootstrap.sh:9). No flake, no setup-zig.
- build.zig.zon: `minimum_zig_version = "0.15.2"` (:16); minish v0.1.0
  (:5-8, same hash as flong; chilli transitive); **zwanzig v0.8.1**
  `https://github.com/forketyfork/zwanzig/archive/refs/tags/v0.8.1.tar.gz`,
  `zwanzig-0.8.1-oiXZlsqhFQB0jB-80jJNdP5CH0bgBIGjhWeNaQgHpqZP` (:9-12);
  fingerprint `0xef481bcf95b8e141`. No Nix.
- Submodules: libdatachannel (danielbodart fork @44caa290, v0.24.1-31;
  json, libjuice v1.7.0, libsrtp, plog, usrsctp); svt-av1 @b33dcc56
  (v2.3.0-386, reports v3.1.2); wlroots fork branch zerocast-linear-gbm
  @7447d157 (not built; prebuilt .a used).
- mise tasks: deps (apt/brew), submodules, zig-cc-wrappers,
  libs:cache-from-main, libs:datachannel, libs:svt-av1, libs, build
  (ReleaseSafe, `-Dcpu=x86_64_v3`, `--prefix dist`), test, lint (`zig build
  analyze` + shellcheck bootstrap.sh), clean, dev, sw-integration,
  compositor-test, integration, dist, install, setup, version,
  ci:build-small (ReleaseSmall), ci:server-*, ci, server:*.
- CI: single job `ci` on ubuntu-latest (push to trunk), recursive
  submodules, `./run.ts ci`. No macOS CI. Cache keys: `tools`
  `mise-tools-${hashFiles('.mise.toml')}`; `.zig-cache/cmake`
  `libdatachannel-${runner.os}-${submodule sha}-${hashFiles('.mise.toml')}`;
  `packages/svt-av1/Bin` `svtav1-${runner.os}-${sha}-${hashFiles('.mise.toml')}`.
  A pin bump invalidates all three.
- **zig-cc shims:** scripts/zig-cc-wrappers.ts:7-15 writes
  `.zig-cache/bin/zig-cc`/`zig-c++` as `exec <absolute zig> cc|c++
  -march=x86_64_v3 "$@"` (observed
  `/home/dan/.local/share/mise/installs/zig/0.15.2/bin/zig cc
  -march=x86_64_v3`). cmake gets them through build_shared.zig:46-47,
  61-62, 80, 103 (`b.build_root.path`). Outputs `.zig-cache/cmake/*.a`
  (linked at build_shared.zig:33-36) and
  `packages/svt-av1/Bin/Release/libSvtAv1Enc.a` (build.zig:144, 326, 349).
- libc++ from Zig's clang: 0.15.2 = clang 20.1.2, 0.16.0 = clang 21.1.0.
  mise `libs:*` sources (.mise.toml:43, 54) do **not** include the Zig
  version. scripts/dist.ts:6-22 checks RUNPATH and no AVX-512.

#### Baseline (0.15.2, clone, inside `nix-shell -I nixpkgs=flake:nixpkgs -p
cmake pkg-config openssl wayland pixman libGL mesa libgbm libdrm
libxkbcommon libva shellcheck`, cmake 4.1.6)

`mise run libs` PASS 72 s; `mise run test` PASS 36/36 steps, 143/143
tests, 2 minish properties × 200; `mise run lint` PASS (zwanzig "34
file(s) … 18 rule(s) … No issues found"); `zig build sw-integration` PASS
("60 frames buffered, 1 keyframes, 10716 bytes total"). compositor-test and
integration need a GPU, not run. `mise run deps` not run (sudo apt).

#### Release sizes (`-Dcpu=x86_64_v3`)

| Build | Bytes | Command |
|---|---|---|
| ReleaseSmall (CI ship mode, `ci:build-small`, .mise.toml:150-160) | 5,898,544 | `nix-shell … --run 'zig build --prefix out-ReleaseSmall -Dversion=0.192.20261111170421 -Doptimize=ReleaseSmall -Dcpu=x86_64_v3'` |
| ReleaseSafe (dev `mise run build`) | 48,850,888 | same with `-Doptimize=ReleaseSafe` |

Built in nix-shell, so binaries carry a /nix/store RUNPATH and may differ
slightly from Ubuntu CI; libc++ static; NEEDED ssl, crypto,
wayland-server/client, pixman, EGL, GLESv2, gbm, drm, xkbcommon, m, c.
macOS not built.

#### 0.16-breaking call sites

Trial in `/tmp/claude-1000/inventory-zerocast/z16`:
- build.zig:208 `exe.linkLibC()`; `Compile.addObjectFile`,
  `linkSystemLibrary`, `linkLibCpp` (build_shared.zig:33-39) → root_module
  forms.
- Dependency build.zigs fail: minish-0.1.0/build.zig:29,
  chilli-0.2.2/build.zig:68, **zwanzig-0.8.1/build.zig:85, 130**
  (`std.fs.cwd`). `zig-pkg/` not in .gitignore.
- With deps stubbed: 35 `error:` lines (lower bound). First errors:
  clock.zig:35 nanoTimestamp; control.zig:136, 158, 168, 204, 549, 558
  `std.io`; control.zig:215 getEnvVarOwned; control.zig:317 crypto.random;
  daemon.zig:33 Thread.Mutex; daemon.zig:261, 372, 452 getEnvVarOwned;
  daemon.zig:718 fs.cwd; encoder_contract_test.zig:482 Child.run;
  input_protocol.zig:72 meta.intToEnum; ivf.zig:13 fs.Dir; ivf.zig:97, 136,
  167 fs.cwd; session.zig:950, 964 std.io; viewer_state.zig:147
  Thread.Mutex. **`@cImport` still compiles on 0.16.**
- Removed posix functions used: socket, connect, bind, listen, accept,
  write, close, fork, waitpid, open, access, execveZ, execvpeZ, shutdown,
  fcntl, dup, getuid, exit.
- std.time: nanoTimestamp clock.zig:35, encoder.zig:250,
  session_recorder.zig:72, :255; timestamp terminal_share.zig:447; Timer
  encoder.zig:181, :199, session_recorder.zig:80, :183, linux
  app_share.zig:136, 194-200, 343, 351, 352, 491, macos app_share.zig:57,
  75-80, 252, 261, 264, 267, 357-362, terminal_share.zig:45, 109, 281, 434,
  436; Instant.now encoder.zig:259, 264, 271, 289.
- std.Thread: Mutex daemon.zig:33, viewer_state.zig:147, session.zig:432,
  terminal_share.zig:62; ResetEvent daemon.zig:252; sleep macos
  app_share.zig:133, 322, 387, 408, terminal_share.zig:413; spawn
  daemon.zig:303, :414.
- std.io/fmt: fixedBufferStream (14) cli.zig:63, 103, 129; control.zig:136,
  148, 158, 168, 204, 549, 558; session.zig:950, 964;
  terminal_share.zig:212, 475; fmt.format (7) control.zig:186, 190;
  terminal_share.zig:216, 478, 489; session_recorder.zig:104, 128;
  ArrayListUnmanaged.writer session_recorder.zig:35, 103, 127;
  sw_integration.zig:160, 172; bufPrintZ session.zig:864 (deprecated).
- std.fs: cwd daemon.zig:75, 104, 631, 718, 719; ivf.zig:10, 97-170;
  terminal_share.zig:444, 452; session_recorder.zig:53;
  compositor_test.zig:69; File/Dir ivf.zig:6, 13; terminal_share.zig:433;
  session_recorder.zig:20.
- std.process: args main.zig:10, cli.zig:9; ArgIterator cli.zig:50, 99,
  125; getEnvVarOwned control.zig:215, daemon.zig:261, 273, 372, 452,
  terminal_share.zig:527, 529; Child.run encoder_contract_test.zig:482,
  532.
- std.posix (~120): daemon.zig:92-180, 631-650, 665-678 (sigaction),
  718-744; cli.zig:156-167 (execvpeZ ×3), 196-245; linux app_share.zig:237-238,
  370, 479-480, 544-566 (fork/open/dup2/execveZ/exit);
  compositor_test.zig:96-148; gpu_detect.zig:181, 206, 210, 340;
  terminal_share.zig:277-278, 295, 315, 358.
- crypto.random control.zig:317, terminal_share.zig:124;
  meta.intToEnum input_protocol.zig:72, 84, 93; GPA prop_tests.zig:30,
  sw_integration.zig:127. `std.c.environ` still exists (std/c.zig:10633);
  std.atomic.Value (33) and c_allocator (13) stay.

#### Build API (0.17)

standard options :12-13; `b.option` :16; addOptions/createModule :17-19;
~40 createModule; module include/object/C/link calls build.zig:63,
143-144, build_linux.zig:25-40, 89, 102-109, build_macos.zig:21, 41-43,
63-79; `.cwd_relative = "/usr/include/pixman-1"` build_linux.zig:29, 105;
addExecutable :196, 410, 453; installArtifact :210; addRunArtifact :216,
245, 260, 272, 283, 292, 309, 329, 351, 369, 386, 402, 423, 445, 469, 480;
**`if (b.args)` :218-220**; steps :221, 225, 424, 444, 470, 474, 491, 494,
497; addTest ×15; `b.dependency` minish :405/:417, zwanzig :475/:479;
zwanzig `addArgs` :481-482, **`addDirectoryArg` :483**; **`b.build_root.path`
build_shared.zig:46, 47, 80**; `addSystemCommand` cmake build_shared.zig:49,
64, 82, 105 (relative `-S`/`-B`, depends on cwd); `configure.addArg(b.fmt(…))`
build_shared.zig:61, 62, 103.

#### @cImport (11 files)

| File | Headers | Macros / notes |
|---|---|---|
| shared/svt_backend.zig:19-21 | EbSvtAv1Enc.h | |
| shared/session.zig:2-5 | rtc/rtc.h | `@cDefine("RTC_STATIC", {})` |
| shared/terminal_share.zig:4-17 | util.h (macOS) / pty.h + utmp.h (Linux) via comptime `builtin.os.tag` (:5); unistd, stdlib, signal, fcntl, sys/ioctl, sys/wait | forkpty, execlp, _exit, ioctl, TIOCGWINSZ, TIOCSWINSZ, O_NONBLOCK, sigprocmask, SIG_SETMASK, SIGTERM, WNOHANG, waitpid, setenv, getenv, kill, struct_winsize |
| linux/wayland/app_share.zig:3-5 | stdlib.h | setenv, unsetenv, environ |
| linux/wayland/frame_download.zig:25-27 | GLES2/gl2.h | |
| linux/wayland/compositor_test.zig:8-11 | stdlib.h, GLES2/gl2.h | |
| linux/wayland/input.zig:7-18 | wayland-server-core, wlr seat/keyboard/pointer (+interfaces), xkbcommon, linux/input-event-codes.h, time.h | `@cDefine("WLR_USE_UNSTABLE", "")`; BTN_* macros |
| linux/wayland/compositor.zig:5-26 | wayland-server-core; wlr backend, headless, allocator, renderer, gles2, egl, compositor, output, output_layout, scene, subcompositor, xdg_shell, seat, data_device, dmabuf, log; gles2_helper.h; time.h | `WLR_USE_UNSTABLE`; 68 distinct `c.` symbols incl. static-inline `wl_signal_add` |
| macos/frame_download.zig:16-18 | screen_capture.h | |
| macos/app_share.zig:19-22 | screen_capture.h, virtual_display.h | |
| macos/input.zig:7-11 | CoreGraphics, ApplicationServices, screen_capture.h | bound as `cg` |

No TranslateC; none in build.zig. `/usr/include/pixman-1` does not exist
on NixOS (did not break the build).

#### @bitCast (3)

| Site | Cast | Risk under 0.17 |
|---|---|---|
| linux/wayland/app_share.zig:600 | `const fmt: [4]u8 = @bitCast(dmabuf.format);` (u32 → array) | Involves an array. The upstream probe shows `[4]u8 ↔ u32` gives the same result on 0.16 and 0.17.1 on x86_64 (byte-multiple elements); it only changes on big-endian. Only a `log.debug` FourCC. Safer: `std.mem.toBytes`. **Low** |
| shared/encoder.zig:250 | u128 ← i128 (nanoTimestamp) | none (line rewritten for 0.16 anyway) |
| shared/clock.zig:35 | u128 ← i128 | none |

#### Other 0.17 hazards

- `**` 57: nvenc.zig 35 (extern-struct field defaults: :131, 177, 191,
  243, 245, 273, 325, 326, 353-355, 378, 412, 415, 417, 434, 435, 458, 460,
  478, 479, 508, 509, 511, 512, 521, 522, 552, 558, 559, 577, 578, 593,
  595, 651), yuv.zig 11 (:157, 164, 169, 183, 184, 198, 204, 216, 222,
  236, 237), ivf.zig 3 (:18, 105, 174), session.zig 3 (:762-764),
  gpu_detect.zig 2 (:177, 403), daemon.zig 2 (:32, :690 string),
  session_recorder.zig 1 (:222). nvenc layouts are ABI-critical: assert
  `@sizeOf` before and after.
- `std.builtin.OptimizeMode` 4 (build_linux.zig:8, :160; build_macos.zig:8,
  :90). `@intFromEnum` 11 (input_protocol 4, session 4, nvenc 2,
  gpu_detect 1); `@enumFromInt` 0.
- `builtin.os.tag` build.zig:5, 264, 275, 287, 429 (host OS), cli.zig:147,
  terminal_share.zig:5; `.mode` main.zig:6, daemon.zig:13 —
  **`builtin.mode == .Debug or … == .ReleaseSafe` breaks on 0.17**
  (main.zig:6, daemon.zig:13; `std.debug.runtime_safety` is simpler).
- Reflection: session.zig:1095 (names only). `@hasDecl`: 0.
- Packed structs: `packed struct(u32)` at nvenc.zig:157, 207, 289, 397 (all
  explicit; pointer fields at :408-409 are outside them).
- Minimum OS: macOS 14+ for CGVirtualDisplay (virtual_display.h:1,
  virtual_display.m:96, private API) → 15 on 0.17; ScreenCaptureKit 12.3+.
  Linux: no explicit floor.

#### Tests and guards

143 Linux tests (control 29, input_protocol 23, osc_parser 14, gpu_detect
12, viewer_state 10, yuv 10, linux keymap 10, encoder_contract 8, session
7, debounce 6, daemon 5, clock 4, ivf 3, svt_backend 1, terminal_share 1).
Not run on Linux: macos/keymap.zig 10, compositor_test 1;
**linux/wayland/app_share.zig:683-707 has 5 tests that never run**; cli.zig
0 (compile check only, build.zig:297-309). prop_tests.zig (build.zig:404-426):
`gen.string(.{min_len,max_len,charset=.ascii})`, `check(…, .{ .num_runs =
200 })`, 2 properties. No compile-fail, no custom lint, no `.zwanzig.json`.
zwanzig v0.8.1 (build.zig:473-484) over packages/client/src with `--do
store-violations-engine --do unreachable-code-engine`; both names still
exist in v0.15.1.

#### Risks

1. Deps block the upgrade (minish + zwanzig); port prop_tests.
2. C++ ABI goes stale silently: shims bake the 0.15.2 path
   (scripts/zig-cc-wrappers.ts:13-14); delete `.zig-cache/cmake`,
   `.zig-cache/cmake-svt`, `packages/svt-av1/Bin` when bumping locally (CI
   keys are safe).
3. Large std.Io rewrite in thread-heavy code with C callbacks on foreign
   threads (`peers_mutex`, session.zig:432); std.c/pthread may be
   pragmatic; clock.zig's vtable is a good Io seam.
4. fork/exec paths (app_share.zig:544-566, compositor_test.zig:96-116,
   cli.zig:156-167, terminal_share forkpty); setenv after fork in a
   multithreaded process is a pre-existing bug.
5. 11 `@cImport` files; terminal_share's comptime OS branch must move to
   build time.
6. Build script removals; 0.17 `b.build_root`, `b.args`,
   addArg/addDirectoryArg; cwd-dependent cmake steps.
7. `zig-pkg/`.
8. Host-OS dispatch (build.zig:5).
9. macOS floor 14 → 15; no macOS CI.
10. 57 `**`, ABI-critical in nvenc.
11. Stale `.paths`, CLAUDE.md VA-API, 17.8 MB stray ELF files, GCC-built
    libwlroots.a.
12. Real repo has no working tree for the libs.
13. 5 dead tests.

### 2.5 glim

`/home/dan/Projects/glim`, trunk, 05cbed5 (v0.4.3). **Local clone is
behind**: nix-config pins v0.13.10 (3b479b91). One package, 1697 lines of
Zig: build.zig 28 (no libc, strips unless Debug, `build_options`),
build.zig.zon 11 (no dependencies, fingerprint 0x4144efe960b4dd19),
src/main.zig 433, dbus.zig 442, hidraw.zig 318, razer.zig 225, events.zig
154, uleds.zig 67, notify.zig 30. Linux x86_64 only.

#### Provisioning and pins

- `.mise.toml:1-4` zig 0.15.2, bun 1.3.10, shellcheck 0.11.0; no tasks;
  entry `./run` (run.ts). build.zig.zon:5 min 0.15.2.
- Nix (checked on disk): `zig_0_15` argument at nix/package.nix:8, in
  `nativeBuildInputs` at :29; buildPhase :33-41 (`zig build install --prefix
  "$out" -Doptimize=ReleaseSafe -Dcpu=baseline -Dversion=${version}` at
  :38-39, cache dir :35); `doCheck` :43; checkPhase :46-51 (`zig build test
  --summary all` under dbus-run-session, :48-49); installPhase :53-65 (`glim
  udev-rules` at :63); `platforms = [ "x86_64-linux" ]` :71. flake.nix:54
  nixos-26.05 (rev c25784012c99…), where `zig_0_16` = 0.16.0; flake.nix:53
  comment "any nixpkgs with Zig 0.15 will do" (stale after the bump);
  checks.nixos-module VM test flake.nix:79-105; system x86_64-linux only
  (:61). No FOD.
- CI (push to trunk, `contents: write`): `linux` (checkout@v4 fetch-depth
  0, mise-action@v4, cache@v4 `~/.cache/zig` key `zig-${{
  hashFiles('build.zig.zon', '.mise.toml') }}` restore `zig-`, ci.yml:17-21;
  `./run ci` = lint, test, build, dist, `gh release create` on every push,
  run.ts:69-80; upload-artifact); `nix` (mise-action@v4,
  DeterminateSystems/nix-installer-action@main, `./run nix` = `nix flake
  check`).

#### Baseline

`./run lint` PASS; `./run test` PASS 4/4 steps, 19/19 (real-bus test ran);
`nix build .#glim --no-link` PASS (~5 min, 19/19 in checkPhase,
`/nix/store/0bxlqp7gnz8yw6k1dfzvqgvkrqnknzip-glim-0.4.20261026034120`);
`nix flake check --print-build-logs` PASS incl. VM test (log
`/tmp/claude-1000/inventory-glim-flakecheck.log`).

#### Release sizes (ReleaseSafe, baseline CPU, static, stripped)

| Build | Bytes | Command |
|---|---|---|
| `./run build` (CI release binary) | 95,952 | `zig build -Doptimize=ReleaseSafe -Dtarget=x86_64-linux -Dcpu=baseline -Dversion=0.4.20261111165541` |
| same, `-Dversion=0.0.0` | 95,936 | `mise exec -- zig build … -Dversion=0.0.0 --prefix /tmp/claude-1000/inventory-glim-rs` |
| Nix store | 95,872 | `nix build .#glim --no-link` |
| ReleaseSmall / ReleaseFast / Debug (reference only) | 52,320 / 94,480 / 9,517,837 | same flags, other `-Doptimize` |

#### 0.16-breaking call sites

build.zig compiles unchanged. First error src/main.zig:59 `std.os.argv`
(also `parseRunOptions(args: []const [*:0]u8)` at main.zig:75). Removed
posix (checked against the 0.16.0 std):
- getenv notify.zig:18, dbus.zig:65, :413 (test)
- socket notify.zig:27, events.zig:31, :91, dbus.zig:72
- close main.zig:300; hidraw.zig:27, :224; uleds.zig:25, :38; events.zig:32,
  :48, :65, :84, :92, :128, :143; dbus.zig:73, :88
- open uleds.zig:24, hidraw.zig:118; write uleds.zig:30, dbus.zig:181;
  sendto notify.zig:29; send events.zig:78; bind events.zig:33; listen :34;
  accept :56; connect events.zig:93, dbus.zig:74; unlink events.zig:27
  (switches on `error.FileNotFound`); clock_gettime(.MONOTONIC)
  main.zig:363.
- `std.Thread.sleep` main.zig:373, hidraw.zig:49 (800 µs EC wait);
  `std.time.milliTimestamp` dbus.zig:121, :135 (wall clock; Io.Clock would
  make it monotonic).
- `std.fs`: main.zig:408 stdout `writerStreaming`; hidraw.zig:125, :137
  (Dir), :138 (Iterator), :141 (openDirAbsolute), :129, :158 (readFile);
  uleds.zig:58 (openFileAbsolute), :60 (writeAll); events.zig:119, :121
  (max_path_bytes), :117 (testing.tmpDir), :120 (realpath).
- **`std.os.linux.fchmodat` at events.zig:39 passes 4 args; 0.16 takes 3**
  (linux.zig:1652). Keep mode 0o666.
- `std.mem.trimRight` events.zig:106 (→ `trimEnd`, mem.zig:1191).
  indexOf/indexOfScalar (dbus.zig:102, :197) deprecated, still compile.
- Unchanged: std_options log_level (main.zig:28), std.log.scoped,
  `std.process.exit` (main.zig:95, :380), `std.os.linux.ioctl`
  (hidraw.zig:75), `getuid` (dbus.zig:66, :95), posix.poll (main.zig:269,
  :313; dbus.zig:138), posix.read.

#### Build API (0.17)

standardTargetOptions :4, standardOptimizeOption :5, addOptions :7,
`b.option` :8, createModule :12-13, `.strip` :18 (**`.strip = optimize !=
.Debug` breaks on 0.17**), root.addOptions :20, addExecutable :22,
installArtifact :23, addTest :25, `b.step` :26, addRunArtifact :27. No
`b.args`, `b.build_root`, add*Arg.

#### @cImport / @bitCast / other

- `@cImport` 0 (uleds.zig:12-15 `UserDev`, hidraw.zig:68-72 `iocWR`
  hand-written).
- `@bitCast` 1: dbus.zig:237 `uint32(@bitCast(v))` i32 → u32, no change;
  covered indirectly by dbus.zig:363/:393, no negative value tested.
- `**` 9, all `[_]T{x} ** N`, all can become `@splat` today:
  notify.zig:19, events.zig:22, :110, dbus.zig:198, uleds.zig:27,
  hidraw.zig:43, :50, razer.zig:47, :193 (test).
- Enum builtins 7 (dbus.zig:254, :262, :311, :320; razer.zig:66, :194,
  :202), all non-exhaustive u8 enums.
- No allocator at all, no `std.builtin`, no `@import("builtin")`,
  `@hasDecl` 0.
- Minimum OS: uleds, hidraw ioctls, SOCK_SEQPACKET, fchmodat; all pre-5.10;
  no stated minimum.

#### Tests and guards

19 test blocks (razer 6, hidraw 5, dbus 3, main 2, events 2, uleds 1).
dbus real-bus test (dbus.zig:409) needs GLIM_TEST_BUS; events test
(events.zig:116) needs a tmp path < 108 bytes. No minish, compile-fail,
lint, zwanzig; shellcheck; Nix VM test.

#### Risks

~30 removed posix calls in every I/O file (best early zigkit `sys`/`fd`
adopter: no allocator, no threads); main signature and env plumbing to
notify.zig and dbus.zig; monotonic timing paths (hidraw.zig:49,
main.zig:362, dbus.zig:121); fchmodat arity; lazy analysis reveals errors
file by file; Nix name bump only; CI cache key moves naturally.

### 2.6 fulcio-codesign

`/home/dan/Projects/fulcio-codesign`, trunk, 00141af (= v0.8.9).
macOS-only tool. build.zig 44, zon 11 (no deps, fingerprint
`0xebd78ba318617269`, version 0.1.0), src/main.zig 282, security.zig 206,
keychain.zig 140, crypto.zig 205, asn1.zig 233, pem.zig 172, fulcio.zig
255, bundle.zig 100, signer.zig 119, scripts/sigstore-token.sh 78.

#### Provisioning and pins

- **Only CI pins Zig**: ci.yml:21 `uses: mlugg/setup-zig@v2`, :23
  `version: 0.15.2` (reports cited :16-19, :19, :20-23; disk wins).
  build.zig.zon:5 min 0.15.2; README "Zig 0.15.2+". No mise, no flake.
- CI: one job `build` on macos-15, `contents: write`, `id-token: write`:
  checkout@v7; `zig build test`; `zig build -Doptimize=ReleaseSmall`;
  version `0.<rev-count>.<run_number>`; trunk-only Fulcio self-sign
  integration + `codesign --verify`; package
  `fulcio-codesign-${VERSION}-aarch64-apple-darwin.tar.gz` (:53) + sha256;
  tag, push, `gh release create "$TAG" "$TARBALL" "$TARBALL.sha256"`
  (:76-79). No cache keys (setup-zig caches by default).
- Downstream: capsper `.mise.toml:10-12` takes `latest` on macOS (no lock),
  run.ts:291 and scripts/test-bundle-tcc.sh:81, 157. The CLI flags and
  tarball name are a contract.

#### Baseline

`mise exec zig@0.15.2 -- zig build test --summary all`: 9/9 steps, 20/20
(asn1 7, pem 8, fulcio 4, bundle 1). `zig build -Doptimize=ReleaseSmall`
fails on Linux (no Security/CoreFoundation framework; with
`-Dtarget=aarch64-macos` too). Native build-exe fails at link
(`SecKeychainDelete`).

#### Release size

| Artifact | Bytes | Source |
|---|---|---|
| shipped `fulcio-codesign` (aarch64 Mach-O, ReleaseSmall), v0.8.9 asset `fulcio-codesign-0.8.9-aarch64-apple-darwin.tar.gz` (270,497) | **585,568** | `gh release download v0.8.9`, sha256 verified |
| `main-0152.o` object only (not shipped) | 699,248 | `mise exec zig@0.15.2 -- zig build-obj -OReleaseSmall -target aarch64-macos -lc src/main.zig -femit-bin=main-0152.o` |

The tarball also ships `sigstore-token.sh` (3,086).

#### 0.16-breaking call sites

- build.zig:15 `linkFramework` → `root_module.linkFramework("Security",
  .{})` (Module.zig:392); build.zig:17 `linkLibC` → `link_libc = true`
  (Module.zig:234).
- Then `zig build test`: 4/9 steps, 2 failed, 11/11 tests: src/pem.zig:60,
  :167 `calcSizeUpperBound` no longer an error union (base64.zig:318);
  src/bundle.zig:67 `fs.cwd`.
- build-obj stops at src/main.zig:36 GPA (lazy analysis hides the rest).
- Expected: main.zig:35-38 → `main(init: std.process.Init)`; main.zig:174
  `std.process.args()` → `init.minimal.args.iterate()` (Args.zig:456);
  parser relies on `[:0]const u8` (main.zig:25, 27, 32, 177, 179, 184),
  whether 0.16's iterator yields `[:0]` is unknown; getenv main.zig:157,
  :162, fulcio.zig:29, :31; files main.zig:104, :142, :146, :258-259,
  bundle.zig:67-69, :86, :89; keychain.zig:23 nanoTimestamp (→
  `Io.Timestamp.now`, i96; better `io.randomSecure` since the keychain
  path and **password** (keychain.zig:33-36) come from a time-seeded
  DefaultPrng); **`http.Client` needs an `io` field** (fulcio.zig:135,
  :161; Client.zig:30) and requests (:140, :166, :147, :175, :177, :178,
  :150, :181, :201, :209) are unverified; ArrayListUnmanaged fulcio.zig:203
  (deprecated alias); bundle.zig:17 unchanged, :56/:63 reach into
  `writer.buffer`. json and `{}` formatting fine.

#### Build API (0.17)

build.zig:4, :5, :7-14, :15-16, :17, :18, :20, :21, **:22-24 `if
(b.args)`**, :25, :28, :29-43 (inline-for over 4 test roots). No option,
build_root, add*Arg.

#### Hazards

`@cImport` 0 (security.zig:52-146 hand-written externs, SPI at :133-146,
deprecated SecKeychain* at :102-107); `@bitCast` 1 (keychain.zig:23
`@truncate(@as(u128, @bitCast(std.time.nanoTimestamp())))`, i128 → u128,
no change, rewritten for 0.16 anyway); `**` 2 (pem.zig:88, :105, tests);
GPA main.zig:36 (gone in 0.16); nothing else. macOS-only; CI macos-15 meets
the 0.17 floor; behaviour on macOS 26 unknown.

#### Tests and guards

20 tests; roots build.zig:29-34; main, crypto, keychain, signer, security
never semantically checked on Linux. bundle.zig:84 writes a fixed
`/tmp/fulcio-codesign-test-bundle.json` (parallel race). Integration test
trunk-only on macOS.

#### Risks

No macOS here (only build-obj); verification needs a push to CI, and SSH
push is broken; Io plumbing through http.Client; lazy analysis hides
errors (add an aarch64-macos build-obj or refAllDecls check step); weak
randomness; downstream contract; pin lives in ci.yml, zon and README
(consider a mise.toml); 0.17 small (`b.args`).

### 2.7 browser-agent-os

`/home/dan/Projects/browser-agent-os`, trunk, c6cb09c. Bun/TS monorepo;
two wasm32-wasi Zig packages, **already on 0.16.0 APIs**, 1897 lines.
packages/coreutils (build.zig 45, 16 bins, target hard-coded at :27-30;
zon 12, `.coreutils`, fingerprint 0x3bcfe8c59865d1f5, min 0.16.0 at :5,
`.dependencies = .{}` at :11) and packages/shell (build.zig 20, **no
build.zig.zon**). Shell src/main.zig 608; coreutils cp 246, rm 180, wc
146, tail 123, head 103, ls 96, mkdir 67, cat 65, touch 55, mv 49, rmdir
38, echo 20, env 16, pwd 10, true/false 5.

#### Provisioning

mise.toml `zig = "0.16.0"` (:2), `bun = "latest"` (:3), `node = "latest"`
(:4, unpinned); tasks install:deps (:6), install:playwright (:10),
build:coreutils (:21, `zig build -Doptimize=ReleaseSmall`),
optimize:coreutils (:26), build:shell (:32), optimize:shell (:37),
build:bins (:43), check (:47), build (:51), test (:56), dev (:61),
dev:browser (:66), clean (:71). scripts/wasm-opt.ts:16-27 runs `wasm-opt
-Oz` with bulk-memory, sign-ext, mutable-globals,
nontrapping-float-to-int, reference-types (binaryen ^129.0.0), coupled to
Zig's default wasm features. CI: one job `build` on ubuntu-latest in
`mcr.microsoft.com/playwright:v1.59.1-jammy` (:21-22), 15 min, checkout@v4,
`jdx/mise-action@v2` (:27), install:deps, build. No release workflow;
artifacts served by host-bun/src/app.ts:7-8, host-browser/src/main.ts:7-8.
mise.toml is untrusted on this host. Nothing consumes the repo.

#### Baseline (0.16.0, bun 1.4.3)

`mise run build`: Zig builds and wasm-opt OK, `bun tsc --noEmit` OK, `bun
test` **62 pass, 2 fail** of 64 in 19 files (5.3 s); both failures are
missing Playwright Chromium (opfsFs.test.ts:20,
host-browser.smoke.test.ts:29). `zig fmt --check packages/` fails on cat,
head, wc, rmdir, rm, mv, tail, touch, cp, shell (not enforced). No Zig
tests.

#### Release sizes (ReleaseSmall + `wasm-opt -Oz`; command `MISE_TRUSTED_CONFIG_PATHS=$PWD mise run build`)

| wasm | Bytes (pre-opt) |
|---|---|
| cat | 43,132 (52,509) |
| cp | 34,463 (52,429) |
| echo | 31,494 (47,586) |
| env | 30,803 (46,824) |
| false | 29,557 (45,439) |
| head | 42,199 (54,394) |
| ls | 31,746 (48,199) |
| mkdir | 31,512 (47,826) |
| mv | 31,968 (48,196) |
| pwd | 30,930 (46,980) |
| rmdir | 31,525 (47,844) |
| rm | 34,153 (51,357) |
| tail | 43,275 (55,595) |
| touch | 31,734 (47,893) |
| true | 29,567 (45,449) |
| wc | 42,805 (55,115) |
| sh (shell) | 46,883 (66,544) |

#### Hazards and findings

- 0 of every hazard: `@cImport`, `@bitCast`, `**`, allocators, `std.builtin`,
  enum builtins, `@import("builtin")`. Build API uses none of the 0.17
  changes (`resolveTargetQuery`, `standardOptimizeOption`, `createModule`,
  `addExecutable`, `installArtifact` only).
- **Verified root cause of the documented `cp` "Zig 0.16 wasm codegen bug"**
  (docs/ROADMAP.md:224, cp/main.zig:117-122, cp tests excluded at
  packages/host-bun/test/coreutils/cp.test.ts:38-39): it is local. The
  hand-written `wasi_snapshot_preview1` externs use `u32` where std uses
  packed u16 and `errno_t` enum(u16) (std lib/std/os/wasi.zig:36-68), so
  LLVM FixFunctionBitcasts replaces calls with `unreachable` stubs. Debug
  build stubs: cp 6 stubs/33 calls, rm 5/10, ls 3/6, mkdir/touch/mv/rmdir
  stubs, sh fd_close ×4, fd_read ×2, path_open ×2; cat (std only) 0.
  **Debug builds of ls/rm/cp/mkdir/mv/rmdir/touch/sh trap on the first raw
  WASI call**; ReleaseSmall works only because the optimiser folds the
  stubs. Fix now: use `std.os.wasi` directly.
- **Latent bug:** `@ptrCast(@alignCast(...))` to an 8-aligned `Dirent` from
  a `[4096]u8` buffer at ls:83, rm:139, cp:209 panics in safety-checked
  builds.
- `@intCast` 31 sites at the extern boundary, harmless.
- 0.17 stability of `std.process.Init`/`std.Io` is unknown.

#### Risks

Extern/std mismatch depends on the optimiser (could trap under a new
LLVM); misaligned dirent cast; no Zig tests; wasm-opt flags coupled to
Zig's default features (pin `-mcpu`); bun/node `latest`; shell has no
zon.

### 2.8 wasiglk

`/home/dan/Projects/wasiglk`, **master**, 786bf48. Origin
`git@github.com:bodar/wasiglk.git` (**org `bodar`, not danielbodart**).
All 7 submodules uninitialised; four (garglk, git, glulxe, hugo) use
relative URLs that resolve to SSH, so they **cannot be fetched while SSH
is broken** (the clone was pointed at https). An earlier 0.16 attempt is in
`docs/2026-07-04-zig-016-upgrade-research.md` (status: blocked).

One build file, packages/server/build.zig (1551 lines); **no
build.zig.zon** anywhere. One Zig object `wasi_glk` (root src/root.zig,
build.zig:141-151) linked into 17 C/C++ interpreters plus agt2agx.
packages/server/src: 7311 lines including build.zig (root 56, types 199,
state 209, protocol 1391, stream 695, window 567, event 557,
unicode_case_tables 418, resources 265, unicode 239, fileref 206, gestalt
191, datetime 130, graphics 118, startup 115, style 113, dispatch 89,
sound 84, blorb 74, garglk 44). C shims fizmo_compat, fizmo_glk, gi_blorb,
gi_dispa, git_compat, tads2_glk, tads_compat, tads3_glk.cpp, glk.h,
glkstart.h; agt2agx/wasi_compat.

Targets: plain C glulxe :194, git :221, hugo :291, agility :804, jacl
:876, level9 :927, magnetic :966, fizmo :1418; setjmp advsys :1004, alan2
:1055, alan3 :1114, scare :338, tads2 :454, tads3 :613; Scott family scott
:1250, taylor :1313, plus :1357 (libs c64diskimage :1183, unp64 :1201);
agt2agx :840; bocfel :415 (native only, :113); test :121-137.

#### Provisioning

- mise.toml: bun/node/wasmtime `latest`, `wasi-sdk = "33"`, `zig = "0.15.2"
  # pinned: build.zig uses 0.15 APIs ...; 0.16 breaks it` (:6). Tasks
  install, buildZig, optimize, testZig, testServer, build.
- bootstrap.sh: `MISE_DATA_DIR=./tools`, **`MISE_GLOBAL_CONFIG_FILE=/dev/null`
  (:7) is rejected by current mise** (2026.10.17 and 2026.11.6: "unknown
  config file type: /dev/null"); worked around with an empty toml.
- run.ts: buildZig (:34-38, `zig build --build-file
  packages/server/build.zig --prefix packages/server/zig-out
  -Doptimize=ReleaseSmall`; `args.includes('-Doptimize=')` bug duplicates
  a user flag); optimize (:46-77, binaryen npm wasm-opt needs node; mise
  `node = latest` failed to install on NixOS); testZig (:100-102);
  testServer (:110-113); `ci` (:241-250) ends with **`publish()` = real
  `bunx jsr publish`; never run `./run ci` locally**.
- CI build.yml: one job, ubuntu-latest, checkout@v6 recursive submodules,
  cache@v4 path `tools` key `mise-tools-${{ hashFiles('mise.toml') }}`
  (:24-27), `./run ci` (:29-30); `id-token: write` for JSR.
- build.zig:273, :382, :1406 fallback to `~/.local/share/mise/installs/wasi-sdk/27`
  (stale; 33 pinned). LLVM-tied flags `-mllvm -wasm-enable-sjlj -mllvm
  -wasm-use-legacy-eh=false` and `featureSet(&.{.exception_handling})` at
  :228, :344, :460, :619, :1009, :1060, :1119.
- Consumer: talebrary/package.json:8 `"@bodar/wasiglk":
  "npm:@jsr/bodar__wasiglk@0.127.53"` (manual bump; prebuilt wasm). wasiglk
  has no git tags; version `0.<revisions>.<run#>` (run.ts:7-14).

#### Baseline (0.15.2, clone `/tmp/claude-1000/inventory-wasiglk/repo`)

testZig 66/66; buildZig all 18 wasm (~23 s); optimize pass (after node on
PATH); testServer 7/7 (glulxe, fizmo, scott, hugo only, **no setjmp
interpreter**); check clean; bundle + testClient 76/76 in 9 files. Not run:
`ci`, testE2E. Native (`-Dplatform=native -Doptimize=ReleaseSafe`) fails to
link tads2 (undefined `t3sprintf`, `t3strlwr`,
`CResLoader::open_res_file`; pre-existing); bocfel native 15,123,920, tads3
native 13,668,408.

#### Release sizes (ReleaseSmall + wasm-opt -Oz; `bun run.ts buildZig && bun run.ts optimize`)

| opt/*.wasm | Bytes |
|---|---|
| advsys | 147,964 |
| agility | 366,403 |
| agt2agx | 93,571 |
| alan2 | 211,620 |
| alan3 | 256,877 |
| fizmo | 278,505 |
| git | 261,723 |
| glulxe | 237,601 |
| hugo | 219,210 |
| jacl | 251,167 |
| level9 | 243,874 |
| magnetic | 205,470 |
| plus | 227,161 |
| scare | 461,864 |
| scott | 365,899 |
| tads2 | 693,074 |
| tads3 | 1,354,690 |
| taylor | 264,010 |
| bin/tads3.wasm raw (before wasm-opt) | 1,544,622 |
| bin/agt2agx.wasm raw, 0.16.0 trial (0.15.2 raw 111,822) | 106,408 |

#### 0.16-breaking call sites

- build.zig: every Compile linking method is gone (`addCSourceFiles` 45,
  `addCSourceFile` 1 (:329), `addIncludePath` 36, `addObject` 1 (:1538),
  `addObjectFile` 3 (:280, :389, :1413), `linkLibrary` 6 (:409, ~:1308,
  :1395, …), `lib.addCSourceFiles` 3, `lib.addIncludePath` 3,
  `unit_tests.addCSourceFiles` (:131), `unit_tests.addIncludePath` (:135),
  `linkLibC` 4 (:189, :871, :1196, :1550), `linkLibCpp` 6 (:449, :604,
  :791, :1245, :1308, :1352), `linkSystemLibrary(
  "wasi-emulated-process-clocks")` (:921)) → `root_module.*`.
  `std.process.getEnvVarOwned` :269, :379, :1403 (6 calls incl. HOME at
  :272, :381, :1405) → `b.graph.environ_map.get(K)` (Build.zig:121). With
  both patches agt2agx (pure C) builds on 0.16.
- Sources (`zig build test` on 0.16: 10 errors): datetime.zig:13, :21
  `std.time.timestamp`; fs.cwd fileref.zig:175, :188, stream.zig:68, :71,
  resources.zig:203; `std.fs.File` state.zig:89 plus stream.zig seek/
  getPos/write/read/close :313-315, :342, :442, :446, :450, :577, :625,
  :674, :73, :101, :248, resources.zig :206, :208, :204, OpenFlags
  stream.zig:60; `std.posix.write` protocol.zig:16 (`read` stays, :23);
  ArrayList `.{}` fileref.zig:31, protocol.zig:234, resources.zig:145,
  stream.zig:36; "unsupported OS" std/posix.zig:403.
- **Design issue:** a C-ABI library with no Zig `main`; needs a
  process-global Io (lazily created `std.Io.Threaded`, research doc
  :44-54) or libc calls.
- **Hard blocker (re-verified):** with `+exception_handling`, 0.16 compiles
  its bundled wasi-libc `libc-top-half/musl/src/setjmp/wasm32/rt.c` and
  clang 21.1.0 crashes: `undefined tag symbol cannot be weak`
  (codeberg.org/ziglang/zig#31436, open, no fix). A minimal setjmp program
  builds on 0.15.2 and runs under `wasmtime -W exceptions=y`.
- **0.17.1 does not fix it.** LLVM 22 (llvm cac54a8, #159143) requires the
  `__c_longjmp` tag to be defined externally; upstream wasi-libc added it
  in 2fc32bc81b9f (#772, 2026-03-11), not in 0.17.1's bundle. On 0.17.1:
  libc.a builds, then link fails (`undefined symbol: __c_longjmp`; with
  wasi-sdk's libsetjmp.a, `duplicate symbol: __wasm_setjmp`, …).
  `-lsetjmp` is silently folded into libc on both (target.zig:376 / :401)
  and does not help.
- **Workarounds tested (both run "ok"):** (a) 0.16 and 0.17.1: compile the
  setjmp C as an object with the EH target and `-mllvm` flags, link the exe
  on a plain wasm32-wasi target plus wasi-sdk `libsetjmp.a` (a structural
  build.zig change); (b) 0.17.1 only: keep EH, drop libsetjmp.a, add one C
  file `__asm__(".globl __c_longjmp\n.tagtype __c_longjmp i32\n__c_longjmp:\n");`.
  Per-file `-mexception-handling` does not work (setjmp.h:12-13 `#error`).
- Affected: **7** interpreters (git :228, scare :344, tads2 :460, tads3
  :619, advsys :1009, alan2 :1060, alan3 :1119). The research doc's "9 or
  10" also listed scott, taylor and plus: **they use no setjmp** (0 matches,
  no EH target, no `-mllvm` flags; their C compiles on 0.16: scott 31/31,
  taylor 15/15, plus 18/18). On 0.16 they fail only on the shared
  `wasi_glk` Zig code (11 errors), like every interpreter.

#### Build API (0.17) and hazards

`b.option` :13; target options :17, :19, :125; :21; install steps :49-51,
:63-65, :72-117; addTest/addRunArtifact :122, :136;
`addWriteFiles().addCopyFile` :325; `.cwd_relative` :280, :389, :1413;
`b.allocator` + allocPrint :273, :278, :382, :387, :1406, :1411;
**`b.path("../<submodule>")` reaches outside the build root everywhere**
(check 0.17 path rules). `@cImport` 0; `@bitCast` 5, none changes
(datetime.zig:15 i64→u64, :100 i32→u32, stream.zig:560 u32→i32,
event.zig:198, :199 i32→u32); `**` 2 (resources.zig:231 test,
protocol.zig:438); `std.builtin.OptimizeMode` in 23 build.zig signatures;
`@intFromEnum` datetime.zig:35; `@import("builtin").is_test`
startup.zig:112; extern types in types.zig and blorb.zig, no packed.
Internal duplicates: libsetjmp lookup ×3 (build.zig:264-283, 378-391,
1401-1416); `glk_exit` twice (event.zig:550, gestalt.zig:176).

### 2.9 ten-vad-ggml

`/home/dan/Projects/ten-vad-ggml`, trunk, one commit edc621c. **No build
system**: 4 loose Zig files plus Python. `.gitignore` lacks `.zig-cache/`
and `zig-out/`. zig/ten_vad_ggml.zig 585 (`TenVadGgmlCtx` :227, init :245,
process :469, reset :500, deinit :507, buildGraph :546; no tests),
conv.zig 235 (4 tests), dsp.zig 445 (14), pitch_est.zig 492 (4).
1757 Zig lines, 1062 Python. The 4 Zig files are **byte-identical to
capsper `a70c56c^:src/*.zig`**, deleted from capsper in a70c56c; this is
the only live copy. External C inputs not in the repo: ggml headers and
library; TEN-VAD `src/fftw.{h,c}` (README says submodule; none exists).
Model `ten-vad-ggml.bin` (302,778) on the HF Hub, gitignored.

#### Provisioning

None. `zig` in the repo: "No version is set for shim: zig". capsper built
these with `addIncludePath("ten-vad/src")`, `addCSourceFile(
"ten-vad/src/fftw.c")` and cmake ggml from the whisper.cpp submodule
(`-DGGML_NATIVE=OFF`); minish property tests at `a70c56c^:src/prop_tests.zig:744-860`
(celtLpc, computeBandEnergy, BiquadFilter) were not carried over.

#### Baseline (0.15.2 by full path)

conv 4/4, dsp 14/14, pitch_est 18/18 (4 + dsp's 14); ten_vad_ggml "All 0
tests passed" (nothing analysed). `zig fmt --check zig/` fails (conv.zig:5,
dsp.zig:23, :343, ten_vad_ggml.zig:312-314) on 0.15.2 and 0.16.0.

#### 0.16 trial

Nothing breaks. Tests pass on 0.16. A harness (refAllDecls plus explicit
`&init/&process/&reset/&deinit`) against ggml HEAD cd6d2b75f5d3 (v0.29.0)
and TEN-VAD fftw.h gives zero semantic errors on both versions; a
ReleaseFast driver links on both. Runtime not run (sandbox blocked
executing third-party C). Harness-only sizes (not shipped):

| Artifact | Bytes | Command |
|---|---|---|
| drv015 | 9,608,488 | `~/.local/share/mise/installs/zig/0.15.2/zig build-exe src/driver.zig tenvad/fftw.c -I tenvad -I ggml/include obj/libggml.a -lc -lc++ -O ReleaseFast -femit-bin=drv015` |
| drv016 | 13,018,304 | same with 0.16.0 and `obj016/libggml.a` (libggml rebuilt with `zig c++` 0.16) |

The ~35% growth is mostly libggml/libc++; not analysed.

#### Hazards

- `@cImport` 3 blocks: ten_vad_ggml.zig:13-18 (`ggml`: ggml.h,
  ggml-alloc.h, ggml-backend.h, ggml-cpu.h; types, typedefs,
  `GGML_TYPE_F32`, `GGML_STATUS_SUCCESS`, ~40 functions), :19-21 (`fft`:
  fftw.h), pitch_est.zig:8-10 (`fft_c`: fftw.h again). No macros, no
  `@cDefine`.
- `@bitCast` 0; `@ptrCast` 7 (ten_vad_ggml.zig:251, 258, 386, 387, 388,
  393, 583), unaffected.
- `**`: 74 occurrences on 71 lines (one more `**` token at :301 is the
  pointer type `**ggml.ggml_tensor`), all zero-fills: conv.zig 39 (tests,
  137-147, 158-168, 178-188, 199-207); dsp.zig 11 (67, 70 nested, 128, 261,
  272, 301, 324, 354, 366, 377, 387); pitch_est.zig 20 (62, 63, 67, 70, 71,
  74 ×2, 78, 81 ×2, 82 ×2, 90, 98, 99, 123, 397, 402, 403, 449; nested at
  74, 81, 82); ten_vad_ggml.zig 4 (105, 131, 134, 396).
- `[*c]` at ten_vad_ggml.zig:301, 311, 332, 361, 444, 445.
- Nothing else.

#### Latent bugs (not version-related)

Double `fclose` (errdefer at :247 plus fclose at :421, so errors at :423,
:442, :453, :458, :460, :462, :464 close twice); leaks on init errors (from
:291, `meta_buf` from :458); unchecked `name_len` (:403) and `n_dims`
(:393); `readI32` (:581) ignores short reads; `extract()` with < 256
samples reads undefined (`:147-161`); `std.debug.print` in init (:265).
Weak tests: dsp.zig:343-345, conv.zig:154.

#### Risks

Needs a new build.zig/zon (min 0.16.0), mise pin, gitignore, a decision on
ggml (zon dep + addCSourceFiles, pin a commit) and fftw (vendor or zon dep);
3 `@cImport` → one translate-c module each; 74 `**`; no runtime parity
guard (add a golden test with the HF model); fmt fails.

### 2.10 Downstream consumers and release order

No Zig project consumes another through Nix; capsper, glim, quoin and flong
are independent leaves. All involved nixpkgs revisions have zig_0_15
0.15.2 and zig_0_16 0.16.0.

| Consumer: file:line | Takes | Tag | Compiles Zig in the consumer? |
|---|---|---|---|
| nix-config/flake.nix:248-251 | capsper | v0.420.300 | yes (package.nix with consumer's nixpkgs) |
| nix-config/flake.nix:291-294 | glim | v0.13.10 | yes (dan-blade only) |
| nix-config/flake.nix:189-195 | chase | v0.242.148 | yes, through flong (chase module.nix:377, 379; flong module.nix:245) |
| nix-config via chase | quoin | v0.100.28 | yes: Linux plugin per host (nix/sudo.nix:11); dan-mac `quoin-sudo-plugin`; the server (hourly autoUpgrade) |
| chase/flake.nix:22-25 | flong | v0.164.130 | yes in chase checks |
| chase/flake.nix:42-45, go.mod:10 | quoin | v0.100.28 | unknown whether a check builds `nixosModules.sudo` |
| frisket flake.nix:13-16 / :24-27 | flong v0.162.128 (test-only), quoin v0.100.28 | | flong in frisket checks |
| capsper .mise.toml:10-12 | fulcio-codesign | latest (macOS) | no |
| talebrary/package.json:8 | wasiglk | 0.127.53 (JSR) | no |

Pins move by chase/scripts/pin.sh (needs network), pin checks
(chase/flake.nix:172-188, frisket-pin :360, quoin-pin :367; frisket
:156-168), chase update.yml (Sundays 21:23, :87-92), frisket update.yml
(:87-89), and nix-config follow-releases.yml (hourly at :17, no build gate,
:39-56).

Order: (1) capsper and glim independently, then nix-config bumps
(flake.nix:249, :292) or follow-releases; rebuild workstations by hand.
(2) quoin. (3) flong. (4) frisket optional (pin.sh flong + quoin). (5)
chase (pin.sh flong, quoin [, frisket]), `nix flake check`, release —
**after both quoin and flong**. (6) nix-config flake.nix:190; the server
takes the 0.16 sudo plugin on its next autoUpgrade with no human gate.
(7) fulcio-codesign release (capsper takes "latest"); wasiglk publish then
manual talebrary bump; browser-agent-os and zerocast nothing.

---

## 3. Copies

Method: 291 tracked `*.zig` files (flong 107, capsper 60, zerocast 38,
quoin 24, wasiglk 21, browser-agent-os 19, fulcio-codesign 10, glim 8,
ten-vad-ggml 4); whitespace-normalised hashes (**no identical file across
or within projects**); line-set Jaccard and containment for files > 30
lines; function-level Jaccard (≥ 0.4 across, ≥ 0.6 within); shared names;
scaffolding. Scripts in the session scratchpad (sim.py, fns.py,
fns_within.py, ci.py).

| J | containment | shared lines | pair |
|---|---|---|---|
| 0.56 | 0.73 | 49 | flong/build/gen_decl_docs.zig ~ capsper/build/gen_config_docs.zig |
| 0.36 | 0.65 | 64 | flong/tools/fdlint.zig ~ quoin/sudo/lint.zig |
| 0.11 | 0.21 | 6 | fulcio build.zig ~ browser-agent-os coreutils build.zig (zig init boilerplate) |
| 0.09/0.07/0.06 | 0.64/0.75/0.15 | 18/21/28 | zerocast, capsper, fulcio build.zig pairwise |
| 0.05 | 0.18 | 43 | flong/build.zig ~ quoin/sudo/build.zig |
| 0.03 | 0.11 | 26 | flong/src/decl_docs.zig ~ capsper/src/shared/config_docs.zig |
| 0.02 | 0.15/0.16 | 15/14 | flong/src/fd.zig ~ quoin fd.zig / fd_darwin.zig (rewrite) |

Function matches: gen docs `main` 0.71, `containerOf` 0.59, `docComment`
0.50; fdlint `main` 0.53, `lint` 0.42; fd `adopt` 0.50 (flong fd.zig:160 vs
quoin fd.zig:25, fd_darwin.zig:26).

### 3.1 Descriptor table — flong fd.zig vs quoin fd.zig vs quoin fd_darwin.zig

Lineage: quoin/sudo/fd.zig:1 "flong's src/fd.zig cut down to the gate
connection"; fd_darwin.zig:1-3 is a libc port of it (~80% the same).
Shared core: `Slot{raw,gen}`, `@splat(.{})` slots, `live()` panics "stale
descriptor handle" (flong :146-151, quoin :19-23), `adopt()` closes when
full (flong :160-171, quoin :25-34), `isLive`, `close` bumps `gen +%= 1`,
`connectUnix` with SOCK_CLOEXEC (flong :571-585, quoin :105-116).

Divergences that matter:

| Aspect | flong | quoin |
|---|---|---|
| Error model | `Error = error{TableFull}` (:126); opens return `Error!sys.Result(Fd(k))`, errno as a value (:173-179, :571) | `error{ TableFull, Closed, Sys }` (fd.zig:7, fd_darwin.zig:8); errno discarded (`failed()` → `error.Sys`, fd.zig:36-38, 70, 86, 96, 107, 110, 114). `Closed` is declared but never produced or matched |
| Kinds | 21 kinds (:56-78: file, dir, path, tree, fsctx, userns, mntns, netns, cgroupns, pipe_r, pidfd, cgroup, inotify, pipe_w, signalfd, inherited, record, pty_master, pty_slave, tty_out, bus); runtime check in `live` (:149), comptime `need()` per op (:263-268, e.g. `read` :272) | none; one `Sock` type ≈ flong `.bus` |
| Capacity / slot width | 1024, `u16` (:124, :182) | 4, `u8` (fd.zig:9, 42) |
| Only in flong | `reserved` slots (:132), `Held`, `AnyFd`, `Stdio`, `retainOnly`, `closeUntracked`, `Reservation`, `pollEntry`, `readAll`, `selfPath`/`pidPath`, inotify iteration, `liveCount`/`snapshot` (:181-1090) | |
| Only in quoin | | `Sock.sendAll` MSG_NOSIGNAL (fd.zig:63-73), `recvFull` (:76-90), `peerUid` SO_PEERCRED (:93-98), `readFile(path, buf) ?[]u8` outside the table (:120-138). flong `.bus` has only `read`/`writeAll` without NOSIGNAL (:272, :283-285) |
| Darwin | none | std.c, `c._errno()` (:38); socket without SOCK_CLOEXEC then `fcntl` FD_CLOEXEC (:96-99): **non-atomic CLOEXEC window**; SO_NOSIGPIPE (:100-101); no peerUid/readFile |
| Syscalls | through sys.zig | `std.os.linux` directly |
| Guards | fdlint raw-number/handle-guts, 14 compile-fail cases (build.zig:325-358), zwanzig model covering every open, 27 planted bugs (build.zig:470-503, bugs.zig:1-321), minish fd_props | lint raw-number/handle-guts, 3 planted bugs (analyze/bugs.zig:1-32, build.zig:138-147), open list a subset of flong's; no compile-fail, no property test |

0.17 note: flong/src/fd.zig:1067 `@import("builtin").cpu.arch.endian()`
(removed in 0.18). capsper duplicates the kernel `InotifyEvent` struct
(linux/input.zig:93-98 vs flong fd.zig:1034) and would fit fd's `.inotify`
kind (fd.zig:592, :1049-1065).

### 3.2 Doc-comment harvester and docs walker — flong vs capsper

- Build tool flong/build/gen_decl_docs.zig:1-141 vs
  capsper/build/gen_config_docs.zig:1-172 (flong:1-4 says it harvests "as
  capsper's … harvests"). Both `std.zig.Ast` → `Field{container,name,doc}`.
  `docComment`: flong keeps line breaks (Markdown, :127-141), capsper joins
  with spaces and returns "" early (:157-172). `containerOf` identical logic
  (flong :99-125, capsper :115-150). capsper calls `ast.deinit`, flong uses
  an arena. Both use `readFileAllocOptions`, `writeFile`, `argsAlloc` (0.16
  port needed).
- Runtime walker flong/src/decl_docs.zig vs capsper/src/shared/config_docs.zig
  (flong :1-3 says "as capsper's … walks"): `containerName` identical
  (flong :75-79, capsper :45-49); `docFor` same apart from names, eval
  quota (200_000 vs 100_000) and compileError text (flong :90-103, capsper
  :61-76); `equal` differs (flong recurses into slices and unions,
  :741-759; capsper `std.meta.eql`, no union, :126-140); `writeFields`
  same name, different output (Nix schema vs annotated ZON; flong :169-198,
  capsper :142-171).
- Wiring: flong/build.zig:1832-1845 (`declFieldDocs`), capsper/build.zig:46-75,
  :131-133, both `addAnonymousImport`.
- These files are also the heaviest **0.17 reflection** users (section 2.1,
  2.3), so one shared implementation means one reflection shim.

### 3.3 Token lint — flong fdlint vs quoin lint

quoin/sudo/lint.zig:1 "flong's fdlint with the rules quoin-sudo takes". 71
non-trivial lines identical including `main()` (arena, argsAlloc, --skip,
walk, sort, stdout writer). Shared tokenizer loop, `Tok`, output
`"NAME:LINE:COL: RULE: why"` (flong :168-178, :236-239; quoin :91-101,
:137-139).

| | flong | quoin |
|---|---|---|
| Rules | 10 (:54-64): raw-namespace, posix, extern, raw-number, argv, handle-guts, adopt-foreign, debug-output, catch-unreachable, alloc | 6 (:7-13): extern, export, raw-number, handle-guts, catch-unreachable, alloc |
| Shared detection | raw-number, handle-guts, alloc, `@extern`/`@cImport` ban, catch-unreachable `// proven: ` (flong :196-223, quoin :111-132) | |
| Per token | up to 2 rules (`rules: [2]?Rule`, :194) | 1 |
| Scope | full relative paths, lists for syscall layer and roots (:47-52, :90-107); tests/ get only extern | basenames `fd.zig`, `fd_darwin.zig`, `abi.zig`, `*_test.zig` (:27-37): any fd.zig anywhere is exempt |
| CLI | `--as LABEL FILE` (:124-126), self-test with planted bad/good (build.zig:403-446) | no `--as`, no self-test |
| Walk | relies on `--skip`; joins with root (:142); sorts per directory | skips `.zig-cache/`, `zig-out/` (:66), not `zig-pkg/`; names relative to walked dir (:70); sorts once |

zigkit needs exemptions supplied by the consumer (config, CLI args, or a
compiled-in `applies` callback). Both port `argsAlloc`, `std.fs.cwd()` and
stdout writer (flong :113-161, quoin :43-84).

### 3.4 sys.zig and errno.zig — flong only

flong/src/sys.zig (1181, `Result(T)` at :40-42, wrappers on `std.os.linux`
alone) and errno.zig (223). No other project has a sys or errno layer.
quoin uses `linux.E.init(rc)` inline in fd.zig and libc through
`@import("abi").c` in ids.zig:16-53. Name overlaps with different code:
flong `sys.setgroups` (:294) vs quoin `Libc.setgroups` (ids.zig:45);
`sys.connectUnix` (:902) vs quoin fd.zig:105-116; `sys.Iovec` (:70) vs
browser-agent-os WASI `Iovec` (cp/main.zig:31, shell/src/main.zig:53).
Other raw ioctl callers: glim/src/hidraw.zig:74-75,
capsper/src/platform/linux/input.zig:113-120. They go to zigkit as the base
under `fd` (PLAN decision 3), not because they were copied.

### 3.5 ioctl number encoding — capsper, glim (independent)

capsper input.zig:67-104 `_ioc(dir,typ,nr,size)` (EVIOCG*, UI_*,
length-dependent `EVIOCGNAME(len)`); glim hidraw.zig:66-72 `iocWR(nr, len:
u14)` (tested :287-290); flong sys.zig:975-986 hard-codes TCGETS, TIOCGPTN
etc. std's `IOCTL.IOR/IOW/IOWR` (0.15.2 lib/std/os/linux/ioctl.zig:41-49)
take `comptime T: type`, so cannot express length-parameterised numbers.
Recommendation: a ~10-line `ioctl.encode(dir, type, nr, len)` in `sys`.

### 3.6 D-Bus — flong dbus.zig + manager.zig vs glim dbus.zig (independent)

Line overlap only 13 generic lines. **SASL EXTERNAL is the same code**:
decimal uid, hex-encoded, `"\x00AUTH EXTERNAL {x}\r\n"`, require `OK `,
`BEGIN\r\n` (flong/src/manager.zig:268-286, glim/src/dbus.zig:91-110).
Writer helpers share names (pad, byte, uint32, string, signature, field):
flong allocator-backed and public (dbus.zig:45-100), glim private fixed
buffer (:205-250). Divergences: flong reads either byte order, glim
little-endian only; glim owns its fd via `std.posix` and logging, flong
goes through fd `.bus` and msg; glim does address discovery
(`$DBUS_SESSION_BUS_ADDRESS` + /run/user fallback, :62-66); glim has an
int32 variant (`@bitCast` at :237), flong arrays, structs, variants.
glim adopting it is a behaviour change (allocator, fd) — a design
decision, not a plain extraction.

### 3.7 build.zig step wiring — flong vs quoin (plus capsper, zerocast, glim)

Near verbatim: `Modules` struct with `get()` over `@typeInfo` fields
(flong :1667-1688, quoin :166-181; `get` identical); `fn modules(b,
target, optimize)` (flong :1690, quoin :183-216); `-Ddev` gate `b.option(bool,
"dev", "Enable the steps that need lazy dependencies: test, analyze")`
(flong :58, quoin :10) with `addFail("needs -Ddev=true")` and
`lazyDependency("minish")` (flong :88-91, quoin :29-34); per-module test
loop (flong :93-97, quoin :37-41); lint step (flong :379-401, quoin
:105-117); fmt step (flong :448-452, quoin :119-120); analyze step with
planted bugs, `"Found N issue(s):\n"`, `expect_term Exited=1` (flong
:454-511, quoin :122-148). zerocast runs zwanzig non-lazily with `--do`
flags (build.zig:473-484); capsper (:175-205) and zerocast (:404-424) wire
minish prop-test executables. 0.17 hazards inside: addFileArg,
addDirectoryArg, addPrefixedFileArg (flong :368-372, :407, :341-355; quoin
:130, :138-139); `Modules.get` uses `@typeInfo(...).fields` (0.17
reflection change); quoin build.zig:35 calls `std.os.linux.getuid()`.

### 3.8 Version stamping

- flong and quoin `scripts/version.sh`: identical except one comment line
  (:5). MAJOR from ./VERSION, MINOR `git rev-list --count HEAD`, PATCH
  `GITHUB_RUN_NUMBER` or timestamp, refuses shallow clones. Used at
  flong ci.yml:229, quoin ci.yml:112; shellchecked at flong/flake.nix:120,
  quoin/flake.nix:451. frisket and galley also have one.
- TypeScript `version()` `0.<rev-list --count branch>.<run#|timestamp>`, 4
  drifted copies: capsper/run.ts:102-109 (no nothrow), glim/run.ts:13-20
  (`.nothrow()`, `|| "0"`), zerocast/scripts/lib.ts:21-29 (+ version.ts:1-5),
  wasiglk/run.ts:7-15. All count a branch name, unlike version.sh (HEAD;
  flong/scripts/version.sh:17-18 explains why).
- `-Dversion` option identical in 3 projects: capsper/build.zig:11-18, :74;
  zerocast/build.zig:16-19; glim/build.zig:7-8, :20.
- `if (b.args) |args| run_cmd.addArgs(args)`: capsper/build.zig:94,
  zerocast/build.zig:218, fulcio-codesign/build.zig:22.

### 3.9–3.11 Not Zig (not zigkit)

- run.ts: glim/run.ts:1-116 is a cut-down capsper/run.ts:1-422 (J=0.13, 37
  lines): `version()`, command set, `commands` record dispatch, `gh release
  create v${ver} … --notes $(git log -1 --format=%B)`.
- dist scripts capsper vs zerocast (after renaming): install.sh 472 vs 373
  (J=0.30, 95 lines; zerocast also shares 47 with capsper
  install-common.sh); update 172 vs 106 (68); apply-update 167 vs 45 (23);
  rollback 59 vs 52 (32). zerocast swaps with atomic `mv -T`; capsper does
  `rm -f current; mv current.tmp current` (a moment with no link).
- CI: capsper ≈ glim (J=0.50); flong ≈ quoin (J=0.24). Action versions:
  mise-action v4 (capsper, glim), v5 (flong), v2 (browser-agent-os);
  checkout v4 (capsper, glim, zerocast, browser-agent-os), v6 (wasiglk), v7
  (flong, quoin, fulcio); cache v4 mostly, v6 flong; setup-zig@v2 fulcio
  only. Nix `export ZIG_GLOBAL_CACHE_DIR="$TMPDIR/zig-cache"` in capsper
  (package.nix:66) and glim (:35); flong uses an FOD (native.nix:49-108).

### 3.12 Platform input — not copies

capsper evdev/uinput (1983 lines, `_ioc`, extern ioctl/inotify, `@cImport`
input-event-codes.h :13) vs zerocast wlroots seat/xkbcommon (259 lines,
`@cImport` :7-17); same-named `InputEvent` (capsper extern struct :19,
zerocast union :21) and `drainEvents` (capsper :1678, zerocast :148) are
unrelated. macOS: capsper CGEventTap via C helpers, zerocast CGEvent post
with `@cImport`. **What is shared is the input-event-codes table**:
zerocast/packages/client/src/linux/keymap.zig hand-writes KEY_* (pure Zig,
no `@cImport`), while zerocast wayland/input.zig:16 and capsper
input.zig:13 `@cImport` linux/input-event-codes.h. macOS kVK codes overlap
too (zerocast macos/keymap.zig vs capsper macos/input.zig:49, 163-165).

### 3.13 Look-alikes, do not extract

Hann window (capsper nemo_mel.zig:159 symmetric N-1 vs ten-vad
ten_vad_ggml.zig:75 periodic N); prop_tests mains (shared boilerplate only,
including a GPA that fdlint's alloc rule would flag); `parseArgs` (flong
check.zig:99, capsper config.zig:649, fulcio main.zig:173,
browser-agent-os shell main.zig:571); `uptimeSeconds` (capsper
status.zig:178 vs zerocast ×3); capsper/ten-vad mel features.

### 3.14 Same-project duplicates (cheap cleanups)

- flong: launch/groups.zig:81, :99 vs launch/subid.zig:210, :228 (`fields`,
  `dropNul`, J 0.80-0.83); test helpers `begin`/`end`, `drain`,
  `removeTree`, `awaitBlocked` (prologue_test.zig:437 = wrapper_test.zig:884),
  `expectKernelMatches`, `closeAt` (fd_props.zig = proc_props.zig).
- quoin: `readFull` ×3 (agent.zig:212, darwin_test.zig:222,
  remote_test.zig:116); four test unix-socket servers (agent.zig:180-225,
  plugin_test.zig:76-135, remote_test.zig:107-182, darwin_test.zig:151-237);
  `openFds` (plugin_test.zig:139 = remote_test.zig:265); lookup/has/number/
  keepAgentPath/groupList (quoin_sudo.zig:46-90, 261-272 vs
  quoin_sudo_darwin.zig:82-121, 251-262); fd.zig/fd_darwin.zig.
- capsper: `computeHannWindow` nemo_mel.zig:159 = nemo_mel_state.zig:169;
  `onProcess` linux/audio.zig:346 ≈ linux/detect.zig:385 (J 0.93).
- zerocast: `jsonExtract` (control.zig:232 = session.zig:978),
  `writeJsonEscaped`, `toSockaddr` (cli.zig:270 = daemon.zig:656),
  `buildExtraArtifacts` (build_linux.zig:157 = build_macos.zig:87),
  `elapsed` (linux app_share.zig:194-200, macos :75-80, :357-362) and
  `viewerCount`; `launchApp` (app_share.zig:540-570 ≈
  compositor_test.zig:95-120); std_options log level (main.zig:5-7,
  daemon.zig:12-14).
- glim: sockaddr_un from a path ×3 (notify.zig:19-25, events.zig:109-114,
  dbus.zig:189-208).
- browser-agent-os: WASI extern blocks in 8 files; `Dirent`/readdir ×3;
  `joinPath` (cp:234 = rm:168); `errName` ×4 (cat:58, head:97, tail:117,
  wc:140); head/tail arg parsing (:5-38). Deliberate ("hand-roll per
  binary", docs/ROADMAP.md).
- wasiglk: libsetjmp lookup ×3; `glk_exit` ×2.

---

## 4. Extraction list

Only things used by two or more projects, plus fd/sys/errno/lint which
PLAN.md mandates.

| Module | Source file(s) | Consumers | Notes for the design phase |
|---|---|---|---|
| `sys` | flong/src/sys.zig (1181) | flong; candidates quoin, glim, capsper, zerocast | `Result(T)` errno-as-value. Port for 0.16: `linux.errno` for `E.init` (sys.zig:119), `Stat`/`Statx` asserts (:50, :592), `waitid` 5th arg (:741), `SIG` enum (:760, :1102), and **argv/environ (:80-116) must come from `std.process.Init.Minimal`** — decide how the program hands them over (and whether `argvSlots` stays writable). Add `ioctl.encode(dir, type, nr, len)` (3.5) and a `sockaddr_un` builder (glim ×3, flong :903-907, quoin fd.zig:106-114, zerocast daemon.zig:656/cli.zig:270). Needs a libc/darwin backend if quoin's darwin build adopts `fd`. |
| `errno` | flong/src/errno.zig (223) | flong (+ anyone using `sys`) | glibc text checked by tests/zig/libc_errno.zig; 14 enum builtins in errno.zig (0.17 `@backingInt`, which `zig fmt` upgrades). |
| `fd` | flong/src/fd.zig (1293) | flong, quoin (fd.zig 176, fd_darwin.zig 144) | Capacity via `zigkit_options` (flong 1024/u16, quoin 4/u8; BSS only when used). Kinds open to extension (21 flong kinds vs quoin's one). Error model: quoin's `error.Sys` loses errno and `Closed` is dead; quoin's catch sites (quoin_sudo.zig:116, :120, :189, :192; ids.zig:85; quoin-sudo-pam.zig:19) change if `Result(T)` wins. Add `sendAll` with MSG_NOSIGNAL/SO_NOSIGPIPE, `recvFull`, `peerUid`, `readFile`. Darwin backend with **atomic** CLOEXEC where possible. Add `udp` kind (PLAN §3 phase 2). Keep zwanzig model names (`connectUnix`, `close`) stable for quoin's planted-bug line:col checks. |
| `lint` | flong/tools/fdlint.zig (240) | flong, quoin (lint.zig 143) | Rules selectable per consumer; exemptions supplied by the consumer (paths vs basenames); keep `--as` and the planted self-test; skip `zig-pkg/`, `.zig-cache/`, `zig-out/`; quoin's `export` rule. Port argsAlloc/std.fs/stdout to 0.16. |
| `build` helpers | flong/build.zig (lint 379-401, fmt 448-452, analyze 454-511, compile-fail 325-376, test loop 93-97, `Modules.get` 1667-1688, `-Ddev` 58/88-91) | flong, quoin; partly capsper, zerocast (prop tests, zwanzig), capsper/zerocast/glim (`-Dversion`) | One place for the 0.17 `*Arg2`/`addPassthruArgs`/`b.root` rewrite. `analyzeStep` should add an `expect_stderr_match` to silence 0.16's warning block. Add a **translate-c wrapper** (one place to switch translate-c 1.0.0 `linkSystemLibrary` to 2.x `.link_system_libs`) for quoin, capsper, zerocast, ten-vad-ggml and flong's test translations. Use reflection only via the 0.16/0.17 shim (`comptime std.meta.fieldNames`). Gate every dev-only `lazyDependency` behind `-Ddev`. |
| `docharvest` | flong/build/gen_decl_docs.zig + decl_docs.zig (`containerName`, `docFor`, `equal`) | flong, capsper (gen_config_docs.zig, config_docs.zig) | Option for line joining (newline vs space). Heaviest reflection user: build it on the cross-version shim (defaults need the `@hasField(@TypeOf(info), "field_attrs")` shim). |
| `dbus` (optional) | flong/src/dbus.zig (547) + manager.zig:268-286 auth | flong, glim (independent, 442) | Allocating vs fixed buffers; byte order; address discovery from glim; fd ownership. Decide whether glim moving onto zigkit fd is in scope. |
| `input` codes (optional) | zerocast/packages/client/src/linux/keymap.zig KEY_* table | zerocast, capsper | Removes two `@cImport`s of linux/input-event-codes.h (capsper input.zig:13 uses ~222 KEY_* + LED_CAPSL; zerocast wayland/input.zig uses BTN_*). Could be generated from the uapi header at build time. |
| `macos/cf` (optional) | fulcio-codesign/src/security.zig CF helpers (cfStr, cfData, createMutableDict, logCFError, checkOSStatus) | fulcio-codesign; capsper and zerocast only for CF calls | Low value: capsper/zerocast need AudioToolbox/CoreGraphics anyway, so translate-c of the frameworks is the real 0.17 route. |
| `msg`/panic | flong/src/msg.zig (`onPanic`, `FullPanic`) | flong; advice for every static binary | 0.16 default panic pulls in `Io.Threaded` (~170 KB); a direct-write `FullPanic` gives 1,848 bytes. Every root also needs `.signal_stack_size = null` where start-code purity matters. |
| minish wiring | flong tests/zig/*_props.zig pattern | flong, quoin, capsper, zerocast (+ restoring ten-vad's lost properties) | minish v0.4.0 only behind `-Ddev`; the pinned minish must itself build on 0.16 (a cached v0.1.0 breaks every consumer). |

Not extracted (one project or not Zig): proc/sig/tty/cgroup/mount/etc.
(flong only; `sig` is a candidate twin of zerocast daemon.zig:665-678, not
diffed); quoin chain.zig `/proc/<pid>/stat` parser (≈ flong
proc.zig:367-395, small); glim notify.zig (sd_notify, no twin); capsper
keep_awake spawn; browser-agent-os WASI helpers (std already has them);
wasiglk "global Io for C-ABI libraries" (one consumer; quoin darwin plugin
has the same need, so revisit); zerocast clock.zig vtable.

---

## 5. Upstream facts

Sources: release notes dumped to `/tmp/claude-1000/upstream-research/rn-0.16.0.txt`
and `rn-0.17.0.txt`; tarballs `zig016/` (sha256 `70e49664…3d00` verified)
and `zig017/` (0.17.1, sha256 not checked); "probed" means compiled with
the real compiler.

### 5.1 Releases

`https://ziglang.org/download/index.json`: 0.16.0 (2026-04-13), 0.17.0
(2026-10-01), **0.17.1 (2026-11-02, latest)**. **No 0.16.1.** Codeberg
tags 0.16.0 = 24fdd5b7a4c1c8b5deb5b56756b9dbc8e08c86a8, 0.17.0 =
7647adab…, 0.17.1 = be248cc1…. A 0.17.x branch exists, no 0.16 branch.
0.17.1 points at the 0.17.0 notes.

### 5.2 Zig 0.16.0 breaking changes vs 0.15.2

- **Language:** `@Type` removed → `@Int`, `@Struct`, `@Union`, `@Enum`,
  `@Pointer`, `@Fn`, `@Tuple`, `@EnumLiteral` (rn:1495); `std.meta.Int`
  deprecated. Switch on packed types; all-discarded prong captures
  forbidden. Runtime vector indexing forbidden. Vector/array in-memory
  coercion removed. `return &local` is an error (only direct `&runtime_var`;
  `x[0..]` of a local still compiles). Small ints coerce to floats;
  `@intFromFloat` deprecated. Packed unions: equal field bit sizes,
  explicit backing int allowed, **no pointers in packed types**.
  **extern/export enums and packed types need explicit backing types.**
  Lazy field analysis; `*u8` ≠ `*align(1) u8`; new dependency-loop cases;
  zero-bit tuple fields not implicitly comptime.
- **std.Io** (rn:2216–2950): every I/O call takes an `Io`
  (`Io.Threaded` = 0.15 behaviour; `Evented` experimental; Uring/Kqueue
  PoC; `Dispatch`, `failing`; `std.testing.io`; escape hatch `var t:
  Io.Threaded = .init_single_threaded; const io = t.io();`). `pub fn
  main(init: std.process.Init)` gives `io`, `gpa`, `arena`, `environ_map`,
  `preopens`, `minimal.args`, `minimal.environ`; or `Init.Minimal`; a bare
  `main()` cannot read argv/env. `getEnvVarOwned`, `argsAlloc`,
  `posix.getenv` absent (probed).
- **Files:** `fs.*` → `std.Io.Dir`/`File` with `io`; `fs.cwd` →
  `Io.Dir.cwd`; `makeDir` → `createDir`; `makePath` → `createDirPath`;
  `read/write/pread/pwrite` → `readStreaming/writeStreaming/
  readPositional/writePositional`; `writeAll` → `writeStreamingAll`;
  `getEndPos/setEndPos` → `length/setLength`; `Mode` →
  `File.Permissions`; `selfExePath*` → `std.process.executablePath*`;
  `fs.path` deprecated for `std.Io.Dir.path`; `readFileAlloc(io, name,
  gpa, .limited(n))` returns `StreamTooLong`.
- **Net:** `net` → `Io.net`; **no AF_UNIX through `Io.net` yet** (rn:2883)
  — though glim's report found `Socket.Mode.seqpacket` and `UnixAddress`
  in Io/net.zig:838, :1057-1068 (not exercised).
- **Process:** `Child.init`+`spawn` → `std.process.spawn(io, .{…})`;
  `Child.run` → `std.process.run(gpa, io, …)`; `execv` →
  `std.process.replace`; `getCwd` → `currentPath`; `mlock*` →
  `lockMemory*`; typed mmap flags.
- **Sync:** `Thread.Mutex/Condition/RwLock/Semaphore/ResetEvent/WaitGroup/
  Futex` → `Io.*` (`Event`, `Group`); `std.once`, `Mutex.Recursive`,
  **`Thread.Pool` removed**; `Thread.spawn` stays.
- **Time:** `Instant`, `Timer`, `timestamp` → `Io.Timestamp`/`Io.Clock`;
  `milliTimestamp`, `nanoTimestamp`, `std.time.sleep`, `Thread.sleep`
  absent; `{D}` → `Io.Duration` with `{f}`.
- **Entropy:** `std.crypto.random`, `getrandom` → `io.random`/
  `randomSecure`; `std.Random.IoSource`.
- **posix:** most wrappers removed. Absent (probed): write, close, open,
  pipe2, fork, dup2, socket, waitpid, execve, ioctl, epoll_create1,
  timerfd_create, nanosleep, clock_gettime, exit. Present: read, openat,
  poll, kill, mmap, tcgetattr, sigaction, signalfd, errno, E, system, all
  `std.os.linux`. A raw-syscall `sys` layer is the right shape.
- **Readers/writers:** `std.io` → `std.Io`; Generic/Any Reader/Writer,
  `FixedBufferStream`, `null_writer`, `CountingReader` removed;
  `Reader.fixed`/`Writer.fixed`.
- **Containers:** managed ArrayHashMap variants removed; PriorityQueue
  without allocator field; `ThreadSafeAllocator` removed; ArenaAllocator
  thread-safe; **`GeneralPurposeAllocator` absent**, `DebugAllocator`
  present. ArrayList needs `.empty` (no `.{}`).
- `indexOf*` → `find*` (old names still exist in 0.16 and 0.17.1);
  `Formatter` → `Alt`; `bufPrintZ` → `bufPrintSentinel`; error renames
  (`CrossDevice`, `FileBusy`); BitSet `.empty/.full`; `builtin.subsystem`,
  `ucontext_t` removed; `std.meta.intToEnum` gone (`std.enums.fromInt`);
  `trimLeft`/`trimRight` gone (`trimRight` → `trimEnd`);
  `testing.refAllDeclsRecursive` gone.
- **Build system** (rn:3877–4046): packages fetched into **`zig-pkg/`
  next to build.zig**; global cache keeps `$GLOBAL/p/$HASH.tar.gz`;
  dependencies without `.fingerprint` or with string `.name` fail; legacy
  hash format removed; new `--fork`, `--test-timeout`, `--error-style`,
  `--multiline-errors`; `--prominent-compile-errors`, `RemoveDir`,
  `b.makeTempPath` removed; `ZIG_BTRFS_WORKAROUND` ignored; `zig fetch
  <url>` without `--save` refuses outside a project (regression fixed in
  0.17). Compile methods `linkLibC`, `linkLibCpp`, `linkFramework`,
  `linkSystemLibrary(2)`, `addObjectFile`, `addCSourceFile(s)`,
  `addIncludePath`, `addObject`, `linkLibrary` move to `root_module`.
  `NativePaths.detect(arena, io, *Target, *Environ.Map)`;
  `Child.Term` fields lower-case; `captureStdOut(.{})`;
  `b.graph.environ_map`.
- **OS minimums:** Linux 5.10 (rn16:1235; also already 5.10 in 0.15.2
  Target.zig:437), macOS 13.0 (0.16 Target.zig:561-565).

**`@cImport` in 0.16:** still compiles with no warning (probed), now on
Aro + translate-c. The 0.16 release notes deprecate it in favour of
`b.addTranslateC` (rn:1434, rn:4050), but the bundled langref
(doc/langref.html, `@cImport` section :10334 on, ":14337" "@cImport vs
translate-c") does not mark it deprecated. **`b.addTranslateC` /
`Step.TranslateC` is the recommended, non-deprecated route in 0.16**
(Build.zig:1115, no deprecation comment); deprecated only in 0.17.

**Probed existence in 0.16.0:** `std.lang` no (std.zig:65 is
`pub const builtin`); `@backingInt`/`@fromBackingInt` no;
`SafeAllocator` no; `BufferFirstAllocator` no (heap.zig:370-418 still has
`stackFallback`); `@splat` for arrays yes; `Run.addPassthruArgs` no (`b.args`
still a field); `containsAtLeastScalar2` yes (mem.zig:1734; old one
deprecated at :1724); `@import("builtin").target.os` yes;
`std.meta.fieldNames` yes. `a ** n`, `void{}`, `i0`, `errdefer |err|`,
`@bitCast(extern struct)` all still compile on 0.16.

### 5.3 Corrections to PLAN.md §1

| PLAN.md §1 says | Correction |
|---|---|
| Table: flong `fd.zig` 1272 lines (:38) | **1293** (wc on disk). |
| Table row quoin: minish `-`, zwanzig `-` (:25) | **minish v0.1.0, zwanzig v0.15.1** (sudo/build.zig.zon:7-16), same hashes as flong. |
| quoin `@cImport` files: 3 (`build.zig`, `abi.zig`, `lint.zig`) | **1** (abi.zig:8). build.zig:186 is a comment, lint.zig:125 a string. |
| capsper `@cImport`: 6 (… input ×2, build.zig) | **5** (pipewire_c.zig:1, opus.zig:16, macos/audio.zig:17, ort_c.zig:5, linux/input.zig:13). build.zig:154 and input.zig:17 are comments. |
| capsper provisioned by `.mise.toml`, mise-action CI | Also Nix `zig_0_15` (nix/package.nix:11, :50), consumed by nix-config. |
| glim provisioned by `.mise.toml`, mise-action CI | Also Nix `zig_0_15` (nix/package.nix:8, :29) and a NixOS VM check. |
| browser-agent-os row `?` | mise.toml:2 0.16.0, mise-action@v2; minish `-`, zwanzig `-`, `@cImport` 0, `@bitCast` 0. |
| wasiglk row "unknown, check" | 0.15.2 (mise.toml:6, "0.16 breaks it"); no zon so no minish/zwanzig; `@cImport` 0; `@bitCast` 5; **blocked on 0.16** for 7 setjmp interpreters. |
| (missing) ten-vad-ggml | No pin, no build; `@cImport` 3; `@bitCast` 0; 74 `a ** n`. |
| "All repos … with an `origin` under github.com/danielbodart" (:33-34) | wasiglk's origin is **github.com/bodar** (master). No `.gitignore` lists `zig-pkg/`; quoin has no `.gitignore`. |
| "`add*Arg` replaced by `*Arg2`" (:70) | **Deprecated, not removed** in 0.17.1 (Run.zig:270 "Deprecated, use addArtifactArg2", Run.zig:354 keeps `addFileArg`); prefixed/decorated variants fold into the `*2` forms (rn17:2297-2309). |
| `if (b.args)` → `addPassthruArgs` | Confirmed (rn17:2429; Run.zig:593). `args` is **no longer a field** of `std.Build`, so reading `b.args` breaks. |
| `b.build_root` → `b.root` | Confirmed (rn17:2281; Build.zig:37 `root: Cache.Path`). **About 30 other `std.Build` fields are also gone**: `args`, `build_root`, `cache_root`, `install_prefix`, `install_path`, `exe_dir`, `lib_dir`, `h_dir`, `dest_dir`, `sysroot`, `search_prefixes`, `release_mode`, `libc_file`, `verbose*`, `enable_qemu`, `enable_wine`, `reference_trace`, `pkg_config_pkg_list`, `build_id` (`release_mode`, `search_prefixes` moved to `b.graph`, Build.zig:93, 101). |
| ZLS doesn't work with 0.17.0 | Confirmed for 0.17.0 (rn17:2554ff); 0.17.1 unknown. |
| "`@cImport` removed (already deprecated in 0.16)" (:73) | Removed in 0.17 (probed). In 0.16 it compiles silently; deprecated in the release notes, not in the langref (5.2). |
| "`std.Build.Step.TranslateC` is deprecated" (:75) | **Only in 0.17**; recommended in 0.16. `addTranslateC` still exists in 0.17.1 (Build.zig:949). |
| `@bitCast` logical bits | Confirmed (rn17:1519, 1660). Probe: `[4]u8`→`u32` and `@Vector(4,u8)`→`u32` unchanged (0x04030201); `[2]u24`→`u48` 0x2222aa111111 → 0x222222111111; `[8]bool`→`u8` 0xab → 0x81. So the silent changes hit arrays/vectors whose element bit size is not a multiple of 8 or has padding, and everything on big-endian. extern struct casts are compile errors (probed). |
| `containsAtLeastScalar` swapped its parameters | Confirmed, plus: **`containsAtLeastScalar2` is removed in 0.17** (probed). On 0.16 use `…2` (fails loudly on 0.17) or neither (e.g. `std.mem.count`). |
| `StackFallbackAllocator` → `BufferFirstAllocator` | Confirmed; `stackFallback` removed (probed); heap.zig:16 in 0.17.1. |
| `DebugAllocator` deprecated for `SafeAllocator` | Confirmed (alias at heap.zig:23-24); `std.heap.Check` also deprecated. **`GeneralPurposeAllocator` is already gone in 0.16** (capsper, zerocast, fulcio use it). |
| `std.builtin` → `std.lang` | Confirmed (std.zig:71 `pub const builtin = lang;`). |
| `@intFromEnum`/`@enumFromInt` → `@backingInt`/`@fromBackingInt` | Confirmed; still compile on 0.17.1; **`zig fmt` upgrades them automatically** (rn17:1660); `@backingInt` only on enums and bitpacks with explicit backing ints. |
| `@import("builtin").cpu/os/abi/object_format` removed in 0.18 | Confirmed (rn17:2223). |
| "Minimum Linux is 5.10 and minimum macOS is 15.0" (:89) | **Linux 5.10 is not new** (0.16 already, and 0.15.2). **macOS rises 13.0 → 15.0** in 0.17 (0.17.1 Target.zig:589; notes say "Darwin 15.0+"). |
| "A grep found no … in the six main projects" (:91-92) | There are nine projects. The extra greps found 0 `errdefer |`, `containsAtLeast`, `stackFallback` in all nine. |
| §2.8 "`@cImport` … deprecated in 0.16, so this is legal now" (:123-124) | Legal in 0.16 via the translate-c package or `b.addTranslateC`; `@cImport` is not flagged by the compiler. |
| (missing) | **New 0.17 hazards fixable on 0.16 now:** `== .Debug/.ReleaseX` comparisons (0.17 `lang.Optimize` tags `debug/safe/fast/small`; error "no field named 'Debug' in enum 'lang.Optimize'") → `switch` or `std.builtin.OptimizeMode.Debug` (works on both, stops after 0.18); `@typeInfo(T).@"struct"/@"enum"/@"union".fields` and `std.meta.fields` (0.17 `field_names`/`field_types`/`field_attrs`/`field_values`, `decls` → `decl_names`, `meta.zig:248` compileError) → `inline for (comptime std.meta.fieldNames(T))` + `@FieldType`/`@field` (`comptime` required; `fieldNames` deprecated in 0.17, return type changes); defaults need a `@hasField(@TypeOf(info), "field_attrs")` shim; `std.meta.declarations` return type differs; `@Enum(u8, .exhaustive, names, values)` works on both; **`@hasDecl` sees only `pub` decls in 0.17** (so `zigkit_options` must be `pub`); `readPackedIntNative/Foreign` → `readPackedInt`; `ascii.indexOfIgnoreCase` → `findIgnoreCase`; `addOptionPath` splits into three; `findProgram` → `findProgramLazy`; `LazyPath.getDisplayName/basename`; Fmt step `paths` → `b.pathList`; no filesystem I/O in build.zig at configure time. Other 0.17 library changes: `allocPrint` → `a.print` (old kept), `getLast` → `last()`, `bit_set` renames, `zon.parse` struct args, `Uri.getHost`, `std.gpu` → `std.spirv`, memory_pool, `DoublyLinkedList.pop` → `popLast`, `parseCpuModel` optional, `mem.eql` on floats honours NaN, empty enums need `noreturn`, `internal`/`link_once` linkages removed, `@divCeil`, LLVM 22, `--pkg-path`/`ZIG_LOCAL_PKG_DIR`, known regression #36444 (response files in `Step.Run`, rn17:2810). |

### 5.4 translate-c package

| Ref | Commit | Date | min Zig | aro dependency |
|---|---|---|---|---|
| tag **1.0.0** | 42f630117deffd5318b4a6ceffbd366dd0af8cee | 2026-05-02 | 0.16.0 | `git+https://github.com/Vexu/arocc#5f5a050569a95ecc40a426f0c3666ae7ef987ede`, `aro-0.0.0-JSD1Qi7QNgDnfcrdEJf82v3o6MhZySjYVrtdfEf3E4Se` |
| branch zig-0.16.x | 6fe0ffc | | 0.16.0 | README-only commit on 1.0.0 |
| tags 2.0.0 / 2.0.1 / 2.0.2 | 0da7a16 / 4f5ad37 / c110300 | | 0.17.0 | branch zig-0.17.x |

**For 0.16 use 1.0.0:**

```
zig fetch --save git+https://codeberg.org/ziglang/translate-c#1.0.0
→ .translate_c = .{ .url = "git+https://codeberg.org/ziglang/translate-c?ref=1.0.0#42f630117deffd5318b4a6ceffbd366dd0af8cee",
                    .hash = "translate_c-1.0.0-Q_BUWo75BgBmrtWsRz9QVUZlzvsSUUe_IIXD9vmWguhX" },
```

The bundled 0.16 translate-c is the same code (MacroTranslator.zig,
ast.zig byte-identical; Translator.zig ~34 lines of wiring).

API at 1.0.0 (`build/Translator.zig`; copy at
`/tmp/claude-1000/upstream-research/Translator-1.0.0.zig`):

```zig
const Translator = @import("translate_c").Translator;
const tc_dep = b.dependency("translate_c", .{});
const t: Translator = .init(tc_dep, .{
    .c_source_file = b.path("src/c.h"),   // or b.addWriteFiles().add("w.h", "#define _GNU_SOURCE\n#include <x.h>\n")
    .target = target, .optimize = optimize,
    .name = null, .link_libc = true, .warnings = .ignore,
    .module_libs = true, .pub_static = null, .func_bodies = null,
    .keep_macro_literals = null, .default_init = null, .strict_flex_arrays = null,
});
t.addIncludePath(p); t.addSystemIncludePath(p); t.addAfterIncludePath(p);
t.addFrameworkPath(p); t.addSystemFrameworkPath(p); t.addEmbedPath(p); t.addConfigHeader(ch);
t.defineCMacro("NAME", "value");
t.linkSystemLibrary("pipewire-0.3", .{}); // pkg-config at configure time
t.linkLibrary(lib); t.addObject(obj);
exe_mod.addImport("c", t.mod);
```

- **0.17 hazard inside translate-c:** in 2.x `linkSystemLibrary` is no
  longer `pub`; replaced by `.link_system_libs = &.{ .{ .name = "glfw" } }`,
  `.libc_file`, `.extra_args`. Wrap construction in one helper.
- Tool defaults at 1.0.0: `pub_static=true`, `func_bodies=true`,
  `keep_macro_literals=true`, `default_init=false`, `strict_flex_arrays=2`;
  wrapper passes `-fmodule-libs`.
- Smoke test (`/tmp/claude-1000/upstream-research/tcdemo`): object-like
  macros, function-like `SQUARE(x)`, `static inline` bodies, libc `abs` all
  work. As with `@cImport`, bitfield structs and forward-declared structs
  become `opaque {}` (src/Translator.zig:634-638) and unsupported macros
  become `@compileError`. **`#pragma pack` (174a76a) is only in 2.0.0+**:
  check packed headers (SVT, capture) on 1.0.0.
- Nix: no LLVM/libclang (Aro is pure Zig). Network only to fetch **two**
  packages (codeberg translate-c and github Vexu/arocc). **NixOS gotcha
  (reproduced):** a native Linux target does not add Zig's bundled libc
  headers (Translator.zig:115), relying on `NativePaths.detect`
  (`NIX_CFLAGS_COMPILE`, NativePaths.zig:26) → `'stdlib.h' not found`
  outside a stdenv shell; an explicit target (`-Dtarget=x86_64-linux-gnu`)
  works. `linkSystemLibrary` needs pkg-config in `nativeBuildInputs`.

### 5.5 minish v0.1 → 0.16-tag API diff

| Tag | Commit | Date | min Zig |
|---|---|---|---|
| v0.1.0 (= v0.1.0-alpha.1) | 8e44fc95d9b896fcfd6fa6d7f6582526b5b5dd55 | 2025-12-19 | 0.15.2 |
| v0.2.0 | 90d19a86071356e2ce58686e25029afed451409f | 2026-04-14 | 0.16.0 |
| v0.3.0 | 8246bf71595a07baa61a5d8bd64f69a89d14639e | 2026-04-15 | 0.16.0 |
| **v0.4.0** | dc49fbe5de746c4562379e31d752c31cb804f1d1 | 2026-09-27 | 0.16.0 |
| v0.5.0 | 61ac1da11cdc6e30ee3bb5ad630af459113de88b | 2026-10-03 | 0.17.0 |

```
.minish = .{ .url = "https://github.com/CogitatorTech/minish/archive/refs/tags/v0.4.0.tar.gz",
             .hash = "minish-0.4.0-SQtSTf4pAwD_cAeVB-_rEHWmgovGY-yaCGjX9hpVKxKH" },
```

(also `…/archive/v0.4.0.tar.gz` and `git+…?ref=v0.4.0#dc49fbe5…`, same
hash; module `minish`, build.zig:8). v0.4.0 CI green on Linux, macOS,
Windows with 0.16.0. **v0.4.0 has no dependencies**, so chilli drops out.

- v0.1.0 cannot build on 0.16: `@Type` (src/minish/gen.zig:38),
  `std.time.milliTimestamp` (runner.zig:49), `std.fs.cwd` (build.zig:29).
- Unchanged: `minish.check(allocator, generator, test_fn, options) !void`
  (runner.zig:70, was :41); `run` alias; `Options` `num_runs=100`, `seed:
  ?u64 = null`, `max_shrink_attempts=1000`, `verbose=false`; most `gen.*`
  (int, intRange, float*, boolean, char*, enumValue, uuid, timestamp*,
  string(StringConfig), list, nonEmpty*, hashMap, array, optional,
  constant, tuple2, tuple3, structure); flatMap, filter, sized, frequency;
  existing `shrink.*`; `TestCase`, `GenError`.
- Added: `Statistics`, `Coverage(T)`, `checkWithCoverage`
  (runner.zig:18-90); `Options.statistics`; `Generator.cloneFn` (optional,
  null default, gen.zig:32-41); shrinkers listAtLeast, listWithOwnership,
  cloneList, arrayWithOwnership, cloneArray, structure, cloneStruct.
- **Breaking:** `gen.oneOf`/`gen.dependent` → `combinators.oneOf/dependent`
  (v0.3.0); **`gen.tuple()` removed**; **`combinators.map` gains a required
  5th parameter** `free_fn` (`map(T, U, base, f, null)`, v0.3.0).
- Behaviour: seed from Wyhash of a stack address (was milliTimestamp);
  failures print "To reproduce: .{ .seed = N }"; shrinking stops exactly at
  `max_shrink_attempts`; iterator torn down before freeing (UAF fix);
  shrink candidates must be independently owned.

### 5.6 Lazy dependencies on 0.16 (who fetches and compiles what)

- A lazy dependency that is not requested **and not on disk** is neither
  fetched nor compiled (online and under `unshare -rn`; Build.zig:2043-2048,
  build_runner.zig:467-485; `dependencies.zig` shows `available = false`).
- If it is already on disk (project `zig-pkg/` or a tarball in
  `$ZIG_GLOBAL_CACHE_DIR/p/`), it is `available = true` and its build.zig
  (and its dependencies', e.g. chilli) is compiled **whether requested or
  not**; 0.16 re-extracts the tarball into `zig-pkg/` offline. That is why
  flong saw minish break "even without -Ddev" (the real `~/.cache/zig/p`
  holds minish-0.1.0, chilli-0.2.2, zwanzig-0.15.1 tarballs).
- **zigkit consumers:** a consumer does not fetch zigkit's dev-only lazy
  deps; but if its global cache holds the minish version zigkit pins, it
  compiles it. **zigkit's pinned minish and zwanzig must therefore build on
  0.16** (v0.4.0 and v0.15.1 do).
- capsper's Nix build (explicit fileset, fresh cache, `-Dprop-tests=false`)
  needs no minish bump on 0.16. Logs under the session scratchpad `lazy/`.

### 5.7 Nix fixed-output deps under zig_0_16

- Both nixpkgs revs (c56a87 store `/nix/store/qyccs9pc…-source`, b1b87
  `/nix/store/bgxq0a75…-source`) have `zig_0_16` 0.16.0 and default `zig` =
  `zig_0_16` (all-packages.nix c56a87 :7237/:7247, b1b87 :6740/:6750);
  `fetchDeps = callPackage ./fetcher.nix` (passthru.nix:30).
- fetcher.nix c56a87 (:14-33): `zig build --fetch=all`, `mv
  $ZIG_GLOBAL_CACHE_DIR/p $out`, recursive hash. b1b87 (:14-41) adds `mkdir
  -p $ZIG_GLOBAL_CACHE_DIR/tmp` for 0.16 (:28-32, ziglang/zig 31964 and
  35264). Neither handles zig-pkg; setup-hook only sets the cache dir.
  nixpkgs 0.16 packages (river, ncdu package.nix:44-51) use the same
  symlink; zmx uses `cp -rLT` + `chmod -R u+w`.
- **Experiment (quoin copy, zig_0_16, minish v0.4.0, zwanzig v0.15.1):** FOD
  hash **`sha256-Gzt+mB0nGWjlgIdaP20G0gL336igSyE/xNEQVOkJFqo=`** on both revs
  (c56a87 without the tmp workaround also worked). The 0.16 FOD holds only
  tarballs (`minish-0.4.0-SQtSTf4p….tar.gz` 42,975; `zwanzig-0.15.1-oiXZliOB….tar.gz`
  257,781); the 0.15 one held unpacked directories. Offline check build
  found deps through the existing symlink, unpacked into
  `/build/source/zig-pkg/`; zwanzig ran ("No issues found"). The source tree
  must be writable. flong's dependency set is identical, so the same new
  hash is expected (not built).

### 5.8 zwanzig v0.15.1

- Tag 23bb911f11f342c93f6b1907cdb6324258a9e967, 2026-08-27, min 0.15.2;
  latest release (main bdc10e5, 2026-10-30, dependabot only). No 0.17
  support (build.zig only distinguishes `minor == 16`); README: both
  frontends maintained through v0.17.x, v0.18.0 drops 0.15.2.
- As a zon dependency (`b.dependency("zwanzig", .{ .target =
  b.graph.host, .optimize = .ReleaseFast })`, `dep.artifact("zwanzig")`):
  **the frontend is chosen automatically** by the building Zig
  (build.zig:9 picks `src/main_0_16.zig`); no option. Verified building on
  0.16 in flong, quoin and Nix trials. Prebuilt assets
  `zwanzig-v0.15.1-zig-{0.15.2|0.16.0}-{linux-x86_64.tar.gz|macos-aarch64.tar.gz|windows-x86_64.zip}`;
  `ZWANZIG_ZIG_FRONTEND=0.16.0` in CI.
- Its build.zig iterates fixture directories at configure time
  (`build.zig:~196-215`), a 0.17 smell in its test steps only.
- Rules: dupe-import, todo, file-as-struct, unused-decl, unused-parameter,
  unreachable-code, empty-defer, empty-errdefer, shadowed-variable,
  sentinel-alloc, identifier-style, deinit-lifecycle; engines
  unreachable-code-engine, optional-unwrap, empty-catch-engine,
  swallowed-error, store-violations-engine, stack-escape-engine,
  divide-by-zero-engine, slice-bounds-engine; CHANGELOG adds
  defer-frees-escapee (0.12.0).
- `.zwanzig.json` keys: `enabled_rules`/`disabled_rules`,
  `max_worklist_steps`, `max_states_per_point`, `use_widening`,
  `resource_models[{kind, method_name, receiver_type, return_type, fqn}]`,
  `escape_models[…]`, `escape_max_depth`. CLI `--do`/`--skip`; default
  runs everything except sentinel-alloc; **exit 1 on any diagnostic**
  (src/cli/run.zig:332); no quiet flag.
- zerocast v0.8.1 → v0.15.1: both `--do` names still exist
  (store_violations_engine.zig:21, unreachable_code_checker.zig:24);
  diagnostics may shift (0.9.0 dedupe and `.call_comma`, 0.12.0 switch
  arms, 0.12.1 appendSlice, 0.12.2 RSS, 0.14.0 project-aware resolution,
  0.15.0 typed ZIR gating); 64 files changed (+7515/−931).

### 5.9 Provisioning availability

| Lock | Rev | Date | zig_0_16 | zig_0_15 | zig_0_17 | zig |
|---|---|---|---|---|---|---|
| flong nixpkgs (nixos-26.05) | c56a87d8e41ce69bd2af57eb35dcc117a9dfca33 | 2026-11-05 | 0.16.0 | 0.15.2 | absent | 0.16.0 |
| flong nixpkgs-unstable | f4a03c008967841f2a0b3cb7c436fc6fc31671f6 | 2026-11-07 | 0.16.0 | 0.15.2 | 0.17.1 | **0.17.1** |
| quoin nixpkgs (nixos-unstable) | b1b875982b17dabde9b4a37f3e229e74913e6db3 | 2026-09-16 | 0.16.0 | 0.15.2 | absent | 0.16.0 |
| capsper/glim (nixos-26.05) | c25784012c99… | | 0.16.0 | 0.15.2 | | |
| frisket | fa1373130d | | 0.16.0 | | | |

No lock needs to move for 0.16. Use `zig_0_16`, never bare `zig`.

- mise: `mise ls-remote zig` lists 0.16.0, 0.17.0, 0.17.1. zig@0.16.0 is
  installed (`~/.local/share/mise/installs/zig/0.16.0`), it appeared at
  16:55 during the inventory (not installed by any reader).
  `installs/zig/latest` → 0.15.2, `installs/zig/0` → 0.16.0. Local mise
  2026.10.17 (2026.11.6 available); `zig.use_community_mirrors=true`.
- mlugg/setup-zig@v2 = v2.2.1 = d1434d08867e3ee9daa34448df10607b98908d29
  (2026-01-19); supports `version: 0.16.0` (modern tarball names,
  community mirrors, minisign; 0.16.0 `.minisig` files return 200); empty
  `version` reads `minimum_zig_version`; no 0.16/0.17 issues; uses
  `node20` (deprecation warnings unknown).

---

## 6. gokit release scheme and proposed zigkit files

### 6.1 gokit (read the v0.8.7 module cache, not the local clone)

The local `/home/dan/Projects/gokit` is **stale** (f5215a4, 4 commits,
tags v0.1.0–v0.3.0, dbusx/v0.1.0; `mise.toml` has no tasks; ci.yml tags
every green build). The released gokit is
`/home/dan/go/pkg/mod/github.com/danielbodart/gokit@v0.8.7/` (commit
2a59cc1f…, 2026-11-01T22:11:53Z, not in the local clone). PLAN.md §2.9's
reference to the release-due task only exists there (and in flong). Run
`git -C ~/Projects/gokit pull` before copying anything.

- **Layout:** `MAJOR`, `mise.toml`, `run` (POSIX sh), `go.mod`, `LICENSE`,
  `README.md`, `.gitignore` (`.idea`, `.claude`), `.github/workflows/{ci,update}.yml`;
  one directory per package with `doc.go`; dbusx a second module.
- **README:** `# gokit` + one paragraph (who it serves: "Small helpers
  shared by Dan Bodart's Go daemons - Portical, Tidewaiter and Treesinger -
  pulled out once a second one needed them."); install block; `## Packages`
  with exactly one prose paragraph per package opening with a bold link
  (`**[httpx](httpx)** joins …`), sub-packages right after their parent,
  identifiers in backticks, platform limits last; a dbusx paragraph;
  `## Rules` bullets with bold leads; command block (`mise run check`,
  `mise run build`); `## Channels` (Built, Updated weekly Sunday 21:23 UTC,
  Consumed); `## Versions` (released when a release path changed, MAJOR
  committed, MINOR commit count, PATCH run number); `## License` "Apache
  License 2.0".
- **mise.toml (v0.8.7):** `[tools] go = "1.27.3"`; `[env] RELEASE_PATHS =
  "*.go go.mod dbusx/go.mod dbusx/go.sum MAJOR"`; tasks check (depends lint,
  test), lint, test, build; `release-due`:
  ```
  set -f
  last=$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null) || { echo true; exit 0; }
  if [ -n "$(git diff --name-only "$last" HEAD -- $RELEASE_PATHS)" ]; then echo true; else echo false; fi
  ```
  Always exits 0; read as `due=$(mise run -q release-due)`. `set -f` keeps
  `*.go` a pathspec (matches across `/`).
- **Two conventions exist:** flong's release-due (mise.toml:31-37) prints
  `false` and **exits 1**; portical (run:179-191) uses the exit code only.
  With `bash -e`, `due=$(…)` fails the step if the task exits 1. **Pick
  one.**
- `run version` (run:31-38): refuses shallow clones; `MAJOR.$(git rev-list
  --count HEAD).${GITHUB_RUN_NUMBER:-timestamp}`. `MAJOR` = `0\n`, in
  RELEASE_PATHS (flong uses `VERSION`).
- **ci.yml (v0.8.7):** triggers push main, pull_request,
  workflow_dispatch input `release` (boolean); `contents: read`; `check`
  (checkout@v7, mise-action@v5, `mise run check`, no cache); `due`
  (fetch-depth 0, `FORCE`, `release` output); `release` (needs check+due,
  push/dispatch on main and due; `contents: write`; version; idempotent
  annotated tag push for `v$VERSION` and `dbusx/v$VERSION` as
  github-actions[bot] (80-93); `gh release view || gh release create
  --verify-tag --latest --notes-file` (95-104); proxy warm-up (109-116,
  Go-only)).
- **update.yml:** cron `23 21 * * 0` + dispatch, `contents: write`,
  `actions: write`, `concurrency: update`; tools to latest patch (43-53),
  Go requirements (55-66), `mise run check`, commit and push (80-90), then
  dispatch `ci.yml -f release=true` if due (92-101) since GITHUB_TOKEN pushes
  start no CI.
- **LICENSE:** gokit is Apache-2.0 (202 lines, "Copyright 2026 Daniel
  Bodart" at :189). flong and glim are MIT ("Dan Bodart", identical md5
  21b2f93c…). PLAN §2.1 chose Apache-2.0: copy gokit's byte for byte; note
  the name difference.
- Go consumers pin with `require` + go.sum (portical gokit v0.8.7;
  tidewaiter v0.3.0/dbusx v0.1.0; treesinger v0.3.0). Zig equivalent: URL
  + hash in build.zig.zon, as flong does (build.zig.zon:10-19, `.version =
  "0.0.0"` while tags are v0.164.130).
- Zig CI templates: glim (checkout@v4, mise-action@v4, cache@v4, releases
  every push) and capsper (same + LFS, macos-15); **flong's `check` job is
  the better template** (checkout@v7, mise-action@v5, cache@v6, ci.yml:40-66);
  its release (194-236) uses `gh release create … --target "$GITHUB_SHA"`
  (API-created tag, not an annotated pushed tag).

### 6.2 Proposed zigkit `mise.toml`

```toml
# The toolchain, pinned to the patch, for CI and for anyone working on this.
# A consumer builds zigkit with its own Zig, so this pin is not a release path.
[tools]
zig = "0.16.0"

# What a release carries: what build.zig.zon's .paths puts in the package
# hash (less LICENSE/README), and MAJOR. Keep this equal to .paths: a file
# outside .paths cannot change what a consumer fetches.
[env]
RELEASE_PATHS = "build.zig build.zig.zon src MAJOR"

[tasks.build]
description = "Compile every module"
run = "zig build -Ddev=true"

[tasks.test]
description = "Unit, property (minish) and compile-fail tests"
run = "zig build test compile-fail -Ddev=true --summary all"

[tasks.lint]
description = "fdlint over src/ (and its planted files), and zig fmt --check"
run = "zig build lint fmt -Ddev=true"

[tasks.analyze]
description = "zwanzig over src/, and over its planted bugs"
run = "zig build analyze -Ddev=true"

# One zig build invocation rather than `depends` (mise runs depends in
# parallel; four zig builds racing on one cache and interleaving output).
[tasks.check]
description = "build + test + lint + zwanzig: the dev loop and CI's gate"
run = "zig build install test compile-fail lint fmt analyze -Ddev=true"

[tasks.version]
description = "vMAJOR.N: MAJOR from the file, N the commit count of HEAD"
run = '''
[ "$(git rev-parse --is-shallow-repository)" = false ] || { echo "shallow clone: the count is wrong; in Actions use fetch-depth: 0" >&2; exit 1; }
echo "v$(tr -d '[:space:]' < MAJOR).$(git rev-list --count HEAD)"
'''

# gokit's convention: always exit 0 and print true/false, read as
# due=$(mise run -q release-due).
[tasks.release-due]
description = "Print true if anything in RELEASE_PATHS changed since the last release tag, else false"
run = '''
set -f
last=$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null) || { echo true; exit 0; }
if [ -n "$(git diff --name-only "$last" HEAD -- $RELEASE_PATHS)" ]; then echo true; else echo false; fi
'''
```

Also: `MAJOR` = `0`; `.gitignore` = `zig-out/`, `.zig-cache/`, **`zig-pkg/`**,
`.claude`; build.zig.zon `.version = "0.0.0"`, `.minimum_zig_version =
"0.16.0"`, `.paths = .{ "build.zig", "build.zig.zon", "src", "LICENSE" }`,
minish v0.4.0 and zwanzig v0.15.1 `.lazy = true`, requested only under
`-Ddev`, plus a `.fingerprint` and enum `.name` (0.16 requires them of
dependencies).

### 6.3 Proposed `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [trunk]
  pull_request:
  workflow_dispatch:
    inputs:
      release:
        description: Release this commit even if nothing in RELEASE_PATHS changed since the last release
        type: boolean
        default: false

permissions:
  contents: read

defaults:
  run:
    shell: bash

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: jdx/mise-action@v5
      - uses: actions/cache@v6
        with:
          path: ~/.cache/zig
          key: zig-${{ hashFiles('build.zig.zon', 'mise.toml') }}
          restore-keys: zig-
      - run: mise run check

  due:
    runs-on: ubuntu-latest
    outputs:
      release: ${{ steps.due.outputs.release }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: jdx/mise-action@v5
      - id: due
        env:
          FORCE: ${{ inputs.release }}
        run: |
          if [ "$FORCE" = true ]; then due=true; else due=$(mise run -q release-due); fi
          echo "Release due: $due"
          echo "release=$due" >> "$GITHUB_OUTPUT"

  release:
    runs-on: ubuntu-latest
    needs: [check, due]
    if: (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && github.ref == 'refs/heads/trunk' && needs.due.outputs.release == 'true'
    concurrency:
      group: release
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: jdx/mise-action@v5
      - name: Version
        id: version
        run: echo "tag=$(mise run -q version)" >> "$GITHUB_OUTPUT"
      # vMAJOR.N names one commit, so a re-run finds its tag already there:
      # left alone if it names this commit, a failure if it names another.
      - name: Tag
        env:
          TAG: ${{ steps.version.outputs.tag }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          if existing=$(git rev-parse -q --verify "refs/tags/$TAG^{commit}"); then
            [ "$existing" = "$GITHUB_SHA" ] || { echo "$TAG already names $existing"; exit 1; }
          else
            git tag -a -m "$TAG" "$TAG" "$GITHUB_SHA"
            git push origin "refs/tags/$TAG"
          fi
      # The notes carry the line a consumer pastes, hash included, worked out
      # by zig fetch from the very tarball a consumer will fetch.
      - name: Release
        env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ steps.version.outputs.tag }}
        run: |
          gh release view "$TAG" >/dev/null 2>&1 && exit 0
          url="https://github.com/${GITHUB_REPOSITORY}/archive/refs/tags/$TAG.tar.gz"
          hash=$(zig fetch "$url")
          {
            git log -1 --format=%B "$GITHUB_SHA"
            echo
            echo '```sh'
            echo "zig fetch --save=zigkit $url"
            echo '```'
            echo "hash: \`$hash\`"
          } > notes.md
          gh release create "$TAG" --verify-tag --title "$TAG" --latest --notes-file notes.md
```

Consumers: `.zigkit = .{ .url =
"https://github.com/danielbodart/zigkit/archive/refs/tags/v0.N.tar.gz",
.hash = "zigkit-0.0.0-…" }` via `zig fetch --save=zigkit <url>`.

Points to settle:
1. `vMAJOR.N` (no run number) differs from gokit's three parts; same
   commit always gets the same tag; needs linear trunk (no force-push).
2. One release-due convention (the proposal uses gokit's).
3. `--latest` race between out-of-order release jobs (gokit has it too).
4. Action versions copied from flong/gokit v0.8.7; green status unknown.
5. **`zig fetch "$url"` without `--save` on 0.16.0 refuses to run outside a
   project** (upstream reproduced "no build.zig file found"). The release
   job runs in the zigkit checkout, which has a build.zig, so it probably
   works, but this is unverified; test it before relying on it.
6. Optional weekly update.yml (cron `23 21 * * 0`, `contents: write` +
   `actions: write`, dispatch ci.yml when due), moving zon dev deps like
   flong/scripts/zig-deps.sh.
7. Default branch `trunk`.

---

## 7. Open questions, unknowns, environment problems

### 7.1 Contradictions between reports, and what this document believes

| Topic | Reports say | Believed | Why |
|---|---|---|---|
| flong mise.toml lines | flong report: `check` 45-47, `release-due` 55-61, `RELEASE_PATHS` 42-43 | :21-23, :31-37, :19 | Checked on disk (file is 37 lines). |
| flong native.nix `-Ddev` lines | 267, 301 | 266, 300 | Disk. |
| glim nix/package.nix | glim body: `zig_0_15` at :76, :97; build :101-109; check :114-119; install :131 | :8, :29; :33-41; :46-51; :63 | Disk (74-line file); the glim summary line itself said :8, :29. |
| quoin sudo-zig-darwin.nix | `zig_0_15` at :98; also :134, :151-158, :1-5 | `zig_0_15` at **:9** | Disk (77 lines). The other cited lines cannot exist; their content is unverified. |
| fulcio setup-zig line | :23; :16-19; :19; :20-23 | `uses` :21, `version` :23 | Disk. |
| zerocast `-Dversion` lines | glim report: zerocast/build.zig:7-9 | :16-19 | Disk. |
| capsper `zig_0_15` | :11, :50 / :51 / :11, :50, :69 | :11, :50 (zig build at :69) | Disk. |
| flong analyze step on 0.16 | flong report: Run step fails | Passes; warning block only | Follow-up reproduced 0 failures in 6 runs; same block in quoin. |
| minish breaks "even without -Ddev" (flong) vs "once in zig-pkg/" (capsper) | | Both right | It breaks whenever on disk (section 5.6). |
| zwanzig v0.15.1 on 0.16 | gokit report: unknown | Builds and runs | flong, quoin (and in Nix) trials. |
| zig fetch hash on 0.16 | gokit: unconfirmed | `zig fetch --save` works on 0.16 (upstream computed hashes); bare `zig fetch` outside a project refuses | Upstream research. |
| `@cImport` deprecated in 0.16 | upstream: deprecated in release notes; citation follow-up: langref does not flag, compiler silent | Both true | Different sources; practical effect: silent and legal in 0.16. |
| macOS floor | citation follow-up: 0.17 unknown locally | 13.0 (0.16) → 15.0 (0.17.1, Target.zig:589) | Upstream had the 0.17.1 tarball. |
| `*Arg2`, `containsAtLeastScalar2` in 0.17 | citation follow-up: unknown locally | As upstream probed | Upstream probed 0.17.1. |
| `@hasDecl` counts | spot-check: zerocast 2, glim 1 | 0 in both | Follow-up searched every file at 9b95b86 and 05cbed5; source of the count unknown. |
| flong `**` | 49 lines vs 48 non-comment lines | ~48–49 | Different counting rules. |
| capsper `**` | 21 vs 20 | 20 real + 1 comment | pipewire_c.zig:61 is a comment. |
| ten-vad `**` | 74 occurrences on 71 lines vs 72 lines vs 75 tokens | 74 array repeats | 75th token is a pointer type (:301). |
| wasiglk setjmp interpreters | research doc: 9–10 incl. scott/taylor/plus | 7 | Follow-up: no setjmp in those three, no EH target. |
| capsper shipped Linux size | 9,566,632 (local) vs 9,558,216 (release) | Both, labelled | Different commits (4f4bb26 vs v0.420.300). |
| fulcio shipped size | -1 (unknown) | 585,568 | Downloaded v0.8.9 release. |
| zerocast `[4]u8` bitCast under 0.17 | zerocast: unknown | Unchanged on little-endian | Upstream probe of `[4]u8`↔`u32`. |
| flong `realpathAlloc` replacement | `realPathFileAbsoluteAlloc(b.graph.io, …)` vs `cwd().realPathFileAlloc(io, zig_exe, alloc)` | Either configures | Both copies reached `zig build --help`. |
| PLAN §2.9 "see gokit ci.yml / release-due" | | Only in gokit v0.8.7 module cache and flong | Local gokit clone stale. |

### 7.2 Environment problems

- **SSH push is broken on this host** (ssh agent socket missing). Every
  Phase 3 push and fulcio-codesign's macOS verification depend on it.
  wasiglk's relative submodule URLs (garglk, git, glulxe, hugo) need SSH.
- **Stale local clones** (pinned commits not in the object store):
  quoin local v0.85.17 vs pinned v0.100.28; capsper v0.410.293 vs
  v0.420.300; glim v0.4.3 vs v0.13.10; frisket v0.186.117 vs v0.199.126
  (frisket/flake.nix:11, :22 also stale). flong (v0.164.130 + 2) and chase
  are current; zerocast v0.192.139 has no consumer. gokit is stale too.
  **Fetch each before Phase 3** or the work starts 7–15 releases back.
  No reader fetched.
- mise configs untrusted (flong, browser-agent-os); readers used
  `MISE_TRUSTED_CONFIG_PATHS`, never `mise trust`.
- wasiglk bootstrap.sh (`MISE_GLOBAL_CONFIG_FILE=/dev/null`) fails with
  current mise; mise `node = latest` fails to install on NixOS; wasm-opt
  needs node.
- No Playwright Chromium (`~/.cache/ms-playwright` missing): 2
  browser-agent-os tests fail.
- No macOS host: fulcio-codesign, capsper macOS, zerocast macOS and quoin
  darwin cannot be built or run here. quoin darwin is built by no CI.
- NixOS lacks FHS headers: translate-c 1.0.0 with a native target needs
  `NIX_CFLAGS_COMPILE` or an explicit target; flong `mise run check` needs
  `nix develop`; zerocast needs a nix-shell for its system libs; zerocast's
  `/usr/include/pixman-1` does not exist.
- zig 0.16.0 appeared under mise at 16:55 from an unknown process.
- Leftovers: `/nix/var/nix/builds/nix-806484-115097035` (needs root);
  `/tmp/rt-*.c`, `/tmp/rt-*.sh` from clang crashes (rm was refused).
- The sandbox blocked running ten-vad's binary linked against downloaded C,
  so its runtime on 0.16 is unknown.
- Codeberg full-text search for `__c_longjmp` returned nothing, so a newer
  duplicate of zig#31436 is possible.

### 7.3 Unknowns

- Whether 0.16 `std.process.Init.Minimal` argv/environ slots are writable
  kernel memory (flong `argvSlots`).
- flong's 0.16 shipped size (no full 0.16 build); whether the port adds an
  `Io`/`debug_io` user.
- Whether flong's compile-fail messages survive 0.16 wording.
- 0.16 `std.http.Client` signatures past `.io` (fulcio); whether 0.16's
  args iterator yields `[:0]`.
- Whether `std.process.Init`/`std.Io` stay stable into 0.17
  (browser-agent-os).
- ZLS status on 0.17.1.
- Whether any chase or frisket check imports `quoin.nixosModules.sudo`
  (compiling quoin Zig in their CI); frisket VM test at :474 not checked;
  talebrary/bun.lockb contents; remote tag state.
- Target Mac's macOS version (quoin, zerocast floors).
- Which ggml version ten-vad was validated against.
- quoin's analyze step in the Nix 0.16 check failed with an unknown cause
  (check.log:469); probably the same warning vs a real check failure,
  not determined.
- Whether the Linux 5.10 floor matters for glim on Razer laptops
  (assumed not).
- setup-zig's node20 deprecation status.

### 7.4 Decisions the design phase must make

1. **Error model** for `sys`/`fd`: flong's `Result(T)` (errno as a value)
   vs quoin's `error{TableFull, Closed, Sys}`; quoin's catch sites change
   if `Result(T)` wins; drop the dead `Closed`.
2. **Kind extensibility:** `Fd(Kinds, k)` with a program enum vs base set +
   extras (flong's 21 kinds, quoin's single socket kind, new `udp`).
3. **Options mechanism:** `@import("root").zigkit_options`, which must be
   `pub` (0.17 `@hasDecl`); capacity 1024/u16 vs 4/u8.
4. **OS backends:** does `fd`/`sys` get a darwin/libc backend (quoin
   fd_darwin, with atomic CLOEXEC where possible), or does quoin darwin
   stay outside zigkit?
5. **argv/environ handoff** from `std.process.Init(.Minimal)` into `sys`.
6. **Libraries with no `main`** (quoin darwin plugin, wasiglk): a zigkit
   "global Io" helper, or libc.
7. **Lint packaging:** per-consumer rule sets and exemptions (paths vs
   basenames vs callback); ship as a build helper step or a tool artifact.
8. **Build helpers scope:** lint/fmt/analyze/compile-fail/prop-test steps,
   `-Dversion`, translate-c wrapper; how they absorb the 0.17 build API
   rewrite; zero cost for consumers that do not use them.
9. **Reflection shim** for 0.16/0.17 (`fieldNames`, defaults,
   declarations), used by build helpers and docharvest.
10. **Optional modules:** `dbus` (and whether glim moves onto zigkit fd),
    `docharvest`, `input` codes, `macos/cf`.
11. **Panic/start-code guidance** as a module (`msg`) or documentation.
12. **Release scheme:** release-due convention (print vs exit code), tag
    form `vMAJOR.N`, whether to add update.yml.
13. **Phase 3 order** must respect downstream: chase after quoin and flong;
    wasiglk needs the setjmp workaround (split EH objects on 0.16) or stays
    on 0.15.2.
