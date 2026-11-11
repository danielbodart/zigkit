# zigkit: design

This is the binding design for zigkit's Phase 2 (build it on Zig 0.16.0)
and Phase 3 (move each consumer onto it). It answers the thirteen
questions in [docs/inventory.md](docs/inventory.md) §7.4, gives every
module's public API as Zig 0.16 signatures, and ends with the descriptor
layer's own documentation, taken from flong.

It starts from the design judged best on balance (the "0.17 adoption cost
first" design: one file per 0.17 hazard, verified on 0.17.1), and takes
from the others: the empty published `build.zig.zon` and the size gate
(size), the guarantees that hold by construction (safety), and the
unformatted panic and the `sys.zig` facade (simple). Where the designs or
the judges disagreed, the choice and its reason are stated.

Every claim about how this compiles was built on Zig 0.16.0
(`mise exec zig@0.16.0 -- zig`) under the synthesis prototype,
`/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/synth/`
(called `$S` below), or is cited to an earlier Phase 1 prototype under the
same `phase1/` directory (`size/`, `zig017/`, `safety/`, `simple/`). The
fixes made after the first review were built in `phase1/fix1/` (`$F`, a
copy of `$S`'s zigkit with each fix applied), against the review's own
reproductions in `phase1/review1-*/`. The fixes made after the second
review were built in `phase1/fix2/` (`$G`: `$G/zk` is `$F/zk` with the
libc branches, the listener kinds, `accept`'s options and the lint rules
below added), against `phase1/review2-*/`. The fixes made after the
third review were built in `phase1/fix3/` (`$H`: `$H/zk` is `$G/zk` with
`openTree`'s flags struct, `fd.clone3`, `shutdown`, `getppid` and the
Darwin kind gate added), against `phase1/review3-*/`. The fixes made
after the fourth review were built in `phase1/fix4/` (`$I`: `$I/zk` is
`$H/zk` with `fd.clone3`'s `CloneOptions`, the handles' ioctl allow-list,
Darwin `accept`'s `O_NONBLOCK`, `sys.ignoreSigpipe` and the lint's new
tokens added), against `phase1/review4-*/`, with 0.17.1 as `Z17` (§14). §14
lists each with its command and result. Anything not built is marked **to
verify in Phase 2**. Sizes are x86_64-linux, `-Dcpu=baseline`, static,
stripped, single-threaded, `stack_size = 0`, read with
`$S/tools/elfsize.py`.

Repository heads read for this design: flong c72db3b, quoin 42ca783,
capsper efc0411, zerocast 7005f73, glim aa330c3, browser-agent-os 8b37f70,
gokit 21bb03a (re-read for the fourth review's fixes: the same heads). Since the inventory, quoin changed only its zon comment and
dependency pins, capsper and glim gained a `mise.toml` (glim's
`.mise.toml` is gone), and gokit changed its release convention (§5.12).

---

## 1. What the prototypes settled

1. **The descriptor table is BSS only if a free slot is all-zero bits.**
   flong's free marker `raw = -1` puts the whole table in `.data`: 12,288
   bytes of file at 1024 slots (size, zig017 and safety all measured it;
   flong's shipped binary has `.data` = 12,312 bytes holding the pattern,
   safety P7b). zigkit marks a free slot with a `state` byte that is 0, so
   the table is `.bss`, and a program that imports `fd` but opens nothing
   has no table at all (`stdio_only`: `.bss` 0).
2. **zigkit's published `build.zig.zon` declares no dependencies.**
   `zig build --fetch=all`, which nixpkgs' `fetchDeps { fetchAll = true; }`
   runs for flong (native.nix) and quoin (nix/sudo-zig.nix), fetches a
   dependency's *lazy* dependencies too. With one lazy minish in zigkit's
   zon, an offline consumer failed "unable to connect to server"; with an
   empty one it built and fetched nothing (`$S/fetchtest`). zigkit's own
   dev dependencies live in a nested `dev/` package outside `.paths`.
3. **Under `zig test`, `@import("root")` is std's test runner**, so a test
   silently gets the default options (zig017, safety, simple's judge). In
   zigkit, a test whose root has no `zigkit_options` does not compile, and
   `zigkit.addTest` runs tests under a runner that is std's own plus the
   program's options (`$S/consumer`: the test saw capacity 4 and the kind
   `.gate`; a plain `b.addTest` of the same file failed with "build it with
   zigkit.addTest").
4. **A non-`pub` `zigkit_options` is silently ignored on 0.16**, not only
   on 0.17. A lint rule catches it (`options-pub`).
5. **0.16's default panic and `std_options` cost ~190 KB in ReleaseSafe**,
   and `std.debug.FullPanic` still formats its bounds and union panics,
   which links `std.Io.Writer`. zigkit's unformatted panic namespace plus
   two `std_options` fields give 3,224 bytes for an empty program, and a
   quoin-shaped client is 5,240 bytes against 10,032 with `FullPanic`.
6. **A hand-written panic namespace is a 0.17 hazard**: 0.17.1 requires
   `unexpectedErrorCode` and `loadUninstantiableType`. With both added, the
   prototype's `zigkit`, `sys`, `fd` and `panic` sources build and run on
   0.16.0 and 0.17.1 (`$S/consumer/o17`). That is the prototype's subset:
   the full Linux `sys` and the moved `fd_props.zig` need the port §5.8
   lists (getdents64's `c_uint` length, wait4's `*i32` status, the
   `newfstatat` name), because Zig analyses only what something calls.
7. **A `.so` with no `main` must not build an `Io`.** A hand-built
   `Io.Threaded` read costs 276–283 KB ReleaseSafe and 256 KiB of TLS per
   host thread; the same read through `fd` is 9,648 bytes (size).

## 2. Measurements

`$S/consumer`, built on 0.16.0 with zigkit's `panic.handler` and
`panic.std_options` unless the name says otherwise.

| Program | ReleaseSmall | ReleaseSafe | `.bss` | `.data` |
|---|---|---|---|---|
| `none`: no zigkit, 0.16 defaults | 1,736 | 196,608 | 262,184 | 19,056 |
| `none_lean`: zigkit panic + std_options, nothing else | 1,528 | 3,224 | 0 | 0 |
| `stdio_only`: imports `fd`, writes to `Stdio.out` | 1,616 | 3,400 | 0 | 0 |
| `fd_q4`: capacity 4, open, read, close | 2,672 | 4,488 | 48 | 0 |
| `fd_f1024`: capacity 1024, same body | 2,680 | 4,552 | 12,288 | 0 |
| `fd_q4_default_panic` | 113,240 | 197,648 | 262,232 | 19,056 |
| `quoinlike`: capacity 4, kind `gate`, connect, sendAll, recvFull | 3,192 | 5,240 | 48 | 0 |
| `quoinlike_formatted`: the same with `std.debug.FullPanic` | 3,192 | 10,032 | 48 | 0 |
| `argv`: `main(init: Init.Minimal)`, argv and environ via `sys` | 1,984 | 3,864 | 0 | 0 |
| On 0.17.1: `quoinlike` / `fd_q4` / `fd_f1024` | | 4,632 / 4,104 / 4,184 | 48 / 48 / 12,288 | 0 |
| `fd_q4` with §4.4's low-number move and retired slots (`$F`) | 2,888 | 4,744 | 48 | 0 |
| `log_std`: two `std.log` lines, std's `defaultLog` (`$F`) | 112,088 | 180,464 | 16 | 0 |
| `log_lean`: the same two lines through `panic.logFn` (`$F`) | 3,048 | 5,688 | 0 | 0 |

Readings:

- Naming `fd` and opening one file costs about 1.1 KB ReleaseSmall over an
  empty lean program. Capacity 4 → 1024 costs 8 bytes of file and 12,240
  bytes of BSS (12 bytes a slot).
- The two table guards added after review cost `fd_q4` 256 bytes
  ReleaseSafe and 216 ReleaseSmall: moving a number below 3 is 224 / 200
  of them, retiring a spent slot 32 / 16 (`$F`, each built without the
  other).
- One `std.log` (or `std.debug.print`) call costs about 175 KB
  ReleaseSafe on 0.16, because both go through `std.Options.debug_io`, an
  `Io.Threaded`; zigkit's panic and `std_options` do not remove it.
  `panic.logFn` (§4.5) formats into a stack buffer and makes one write(2):
  5,688 bytes for the same program.
- With `panic.std_options` and `exe.stack_size = 0`, strace shows `execve`
  followed directly by the program's first `openat` (`$S`, `fd_q4`).
  Without `stack_size = 0`, 0.16's start code makes two
  `prlimit64(RLIMIT_STACK)` calls (size, safety).
- The `.so` numbers (size): zigkit `fd` read 9,648 B ReleaseSafe; the same
  through `Io.Threaded.init_single_threaded` 282,784 B with the default
  panic, 276,160 B with a lean one, plus 262,168 B of `.tbss`.

---

## 3. Repository layout

```
zigkit/                        (published: build.zig.zon .paths)
  MAJOR  LICENSE  README.md  DESIGN.md  PLAN.md  mise.toml  .gitignore
  build.zig                    modules, host tools, unit tests, and the helpers' re-exports
  build.zig.zon                .dependencies = .{}
  build/helpers.zig            build helpers for consumers (§8)
  build/compat.zig             every std.Build call 0.17 changes (§8.2)
  build/test_runner.zig        std's runner + the program's zigkit_options
  build/default_options.zig    `pub const zigkit_options: Options = .{};`
  src/zigkit.zig               module zigkit: Options, the resolved `options`
  src/sys.zig                  module sys: the facade; Result, unwrap, argv, every call
  src/sys/linux.zig            raw syscalls on std.os.linux
  src/sys/libc.zig             Darwin (and BSD) over std.c
  src/sys/sockaddr.zig         SockAddr: unix, ip4, ip6
  src/sys/ioc.zig              ioc.encode
  src/sys/kernel.zig           Entries (getdents64; Darwin getdirentries), InotifyEvents: parsers, no table
  src/errno.zig                module errno (Linux)
  src/fd.zig                   module fd: the table, handles, kinds, minting
  src/fd/kinds.zig             base kind names, extra-kind merge
  src/fd/paths.zig             SelfPath, PidPath
  src/panic.zig                module panic
  src/compat.zig               module compat: reflection and mode across 0.16/0.17
  src/input.zig                module input: KEY_*, BTN_*, LED_*, EV_* (generated, committed)
  src/docharvest.zig           module docharvest: the runtime walker
  tools/lint.zig               zigkit-lint (host tool)
  tools/docharvest.zig         zigkit-docharvest (host tool: Ast → field docs)
  tools/compat.zig             the two tools' 0.16/0.17 forks: ZON parsing, Ast.parse, the `**` token
  guards/compile_fail/consumer/*.zig  fd and sys guarantees under the consumer's options: zigkit and each consumer
  guards/compile_fail/zigkit/*.zig    option handling, each case with its own options: zigkit's dev only (§7)
  guards/fd_props.zig          the table's model property (minish), the same
  guards/fd_props_darwin.zig   the properties Darwin can run (no /proc, no close_range)
  dev/                         NOT published: zigkit's own checks
    build.zig  build.zig.zon   deps: zigkit (path ".."), minish v0.4.0, zwanzig v0.15.1, translate_c 1.0.0
    .zwanzig.json  zigkit-lint.zon (zigkit's own lint config, §10)
    props/*.zig  lint/{bad,good}.zig  analyze/bugs.zig  size/*.zig  tc/*.zig  consumer/
    canary/                    deps: zigkit (path "../..") only; the 0.17.1 canary (§10)
  .github/workflows/ci.yml  update.yml
```

`build.zig.zon`:

```zig
.{
    .name = .zigkit,
    .version = "0.0.0",
    .fingerprint = 0x…, // made by `zig build` in the real repo
    .minimum_zig_version = "0.16.0",
    // None: a consumer's `zig build --fetch=all` (nixpkgs fetchDeps) would
    // fetch a dependency's lazy dependencies too. zigkit's own are in dev/.
    .dependencies = .{},
    .paths = .{ "build.zig", "build.zig.zon", "build", "src", "tools", "guards", "LICENSE" },
}
```

`.gitignore`: `zig-out/ .zig-cache/ zig-pkg/ .claude` (unanchored, so they
cover `dev/` and its nested packages too).

---

## 4. Modules and their public API

Each module is one `b.addModule` with no target or optimize mode, so it
takes the consumer's. Imports point one way:

```
zigkit (Options)   no imports
sys                std only
errno              sys
fd                 sys, zigkit
panic              sys, zigkit
compat             std only
input              none
docharvest         compat
```

```zig
pub fn build(b: *std.Build) void {
    const zigkit = b.addModule("zigkit", .{ .root_source_file = b.path("src/zigkit.zig") });
    const sys = b.addModule("sys", .{ .root_source_file = b.path("src/sys.zig") });
    _ = b.addModule("errno", .{ .root_source_file = b.path("src/errno.zig"), .imports = &.{.{ .name = "sys", .module = sys }} });
    _ = b.addModule("fd", .{ .root_source_file = b.path("src/fd.zig"), .imports = &.{
        .{ .name = "sys", .module = sys }, .{ .name = "zigkit", .module = zigkit } } });
    _ = b.addModule("panic", .{ .root_source_file = b.path("src/panic.zig"), .imports = &.{
        .{ .name = "sys", .module = sys }, .{ .name = "zigkit", .module = zigkit } } });
    const compat = b.addModule("compat", .{ .root_source_file = b.path("src/compat.zig") });
    _ = b.addModule("input", .{ .root_source_file = b.path("src/input.zig") });
    _ = b.addModule("docharvest", .{ .root_source_file = b.path("src/docharvest.zig"),
        .imports = &.{.{ .name = "compat", .module = compat }} });
    // Host tools: compiled only when a consumer's step runs them.
    b.installArtifact(b.addExecutable(.{ .name = "zigkit-lint", .root_module = b.createModule(.{
        .root_source_file = b.path("tools/lint.zig"), .target = b.graph.host, .optimize = .ReleaseSafe }) }));
    b.installArtifact(b.addExecutable(.{ .name = "zigkit-docharvest", .root_module = b.createModule(.{
        .root_source_file = b.path("tools/docharvest.zig"), .target = b.graph.host, .optimize = .ReleaseSafe }) }));
    // `zig build test`: each module's unit tests, through helpers.addTest.
}
```

A consumer that imports both `sys` and `fd` gets one `sys` module, so one
`Result` type (size's `same_sys.zig` asserts it at comptime). Import names
are zigkit's (`sys`, `fd`, `errno`), so flong's `@import("fd")` lines stay
as they are, and the compiler's type names stay `fd.Handle(.file,.owned)`
and `sys.Result(usize)`, which flong's compile-fail expectations name.

### 4.1 `zigkit`: options

```zig
pub const Options = struct {
    fd: Fd = .{},
    panic: Panic = .{},

    pub const Fd = struct {
        /// Slots in the table: 12 bytes of BSS each, and only in a program that opens.
        capacity: u16 = 1024,
        /// The program's own kinds, each a distinct type with the operations of its `like`.
        extra_kinds: []const ExtraKind = &.{},
        /// The table has no lock. A build that is not single-threaded must say
        /// `.serialized`: one thread at a time mints, uses and closes handles.
        threads: Threads = .single,
        /// ioctl requests the program's handles may make beyond zigkit's own
        /// (the terminal's), each a whole request number (§4.2). None may be
        /// a request that returns a descriptor.
        ioctls: []const u32 = &.{},
    };
    pub const Threads = enum { single, serialized };
    pub const ExtraKind = struct { name: [:0]const u8, like: @EnumLiteral() };
    pub const Panic = struct {
        prefix: []const u8 = "",   // written before "internal error: ", e.g. "glim: "
        status: u8 = 134,
    };
};

/// The program's options, or the defaults. A test whose root has none does not compile.
pub const options: Options = if (@hasDecl(root, "zigkit_options"))
    root.zigkit_options
else if (builtin.is_test)
    @compileError("zigkit: this test's root has no zigkit_options, so it would test the defaults, not the program's options; build it with zigkit.addTest")
else
    .{};
```

### 4.2 `sys`: the syscall layer

`src/sys.zig` is a facade: it defines `Result`, `unwrap` and the types, and
re-exports each call from the backend the target selects. `Result` must be
defined in a file named `sys.zig`, or the compiler names it
`linux.Result(usize)` and flong's compile-fail wording breaks (simple
measured both). Linux uses raw syscalls on `std.os.linux` even when libc
is linked (quoin's `.so`), so there is one errno path and no glibc errno
TLS. Darwin uses `std.c`. A Linux-only declaration is analysed only when
used, so a Darwin build fails only where it calls one (simple measured
"root source file struct 'sys.darwin' has no member named 'ioc'"; Phase 2
gives these a `@compileError` naming the call and the target).

```zig
pub const backend: enum { linux, libc } = …;     // by builtin.target.os.tag
pub const E = …;                                 // linux.E or std.c.E: non-exhaustive
pub fn Result(comptime T: type) type { return union(enum) { ok: T, err: E }; }
/// A Result, or an open's error{TableFull}!Result, as one error union; the errno is dropped here.
pub inline fn unwrap(r: anytype) Unwrapped(@TypeOf(r));
pub fn Unwrapped(comptime R: type) type;         // (R's error set ||) error{Errno} ! payload
pub const fd_t, pid_t, uid_t, mode_t, O, ACCMODE, AT, S, Stat, SYS, pollfd, POLL, timespec, socklen_t;

// ---- everything flong's sys.zig has today, ported to 0.16 ----
// Every call that makes a descriptor adds its own close-on-exec flag
// (O_CLOEXEC, SOCK_CLOEXEC, MFD_CLOEXEC, …) whatever `flags` says; dup2 and
// fcntl(F_DUPFD) are the exceptions, being how a spawn's child places a
// number it inherits. The `sys-mint` lint rule (§5.7) keeps every one of
// these calls inside `fd` and the program's declared syscall-layer files.
// `ioctl` is in `sys-mint` too: some requests return a descriptor, which its
// Result(void) would drop. flong's `ioctlFd` is private to `fd` (`openNs`),
// and a handle's `ioctl` takes only a listed request (below).
pub fn read(fd: fd_t, buf: []u8) Result(usize);            // EINTR retried
pub fn write(fd: fd_t, bytes: []const u8) Result(usize);   // EINTR retried
pub fn writev(fd: fd_t, iov: []const Iovec) Result(usize);
pub fn openat(dir: fd_t, path: [*:0]const u8, flags: O, mode: mode_t) Result(fd_t); // O_CLOEXEC forced
pub fn openat2(dir: fd_t, path: [*:0]const u8, how: *const OpenHow) Result(fd_t);   // O_CLOEXEC forced
pub fn close(fd: fd_t) void;
pub fn closeChecked(fd: fd_t) Result(void);
pub fn closeRange(first: u32, last: u32, flags: u32) Result(void);
// mkdirat unlinkat renameat fstatat fstat (fstat(2)/newfstatat(2), below) fchmod fchmodat pread pwrite flock
// mmapRead getrandom setgroups prctl capset ioctl sigDefault emptyMask chdir execve
// the mount API (openTree moveMount fsopen fsconfig fsmount mountSetattr umount2 mountId statmount)
// setns unshare setresuid setresgid setfsuid setfsgid fchownat umask readlinkat clone3 (fd.clone3's)
// waitidPidfd pidfdOpen pidfdSendSignal poll pipe2 sigprocmask signal signalfd fcntl dup2
// getuid geteuid getgid getpid getppid memfdCreate access inotifyInit1 inotifyAddWatch linkat
// the terminal (tcgetattr tcsetattr cfmakeraw isatty getWinsize setWinsize tcgetpgrp tcsetpgrp
// unlockpt ptyNumber getpgrp) kill sigaction sigmask clockMonotonic
// and their structs, each size and offset asserted (Iovec, OpenHow, MountAttr, StatMount,
// CloneArgs, Termios, Winsize, KSigaction, CapHeader/CapData, Stat for x86_64 and aarch64)

// ---- added: sockets (quoin, glim, zerocast, the Mosh port) ----
pub const SockAddr = union(enum) {
    /// The length travels with a unix address: an abstract name's length is
    /// part of the name (the kernel compares path[0..len - offsetof(path)]).
    un: struct { addr: sockaddr.un, len: socklen_t }, in: sockaddr.in, in6: sockaddr.in6,
    /// "@name": abstract, len = offsetof(path) + 1 + name.len; a pathname:
    /// len = offsetof(path) + path.len + 1. ENAMETOOLONG past 108 bytes.
    pub fn unix(path: []const u8) Result(SockAddr);
    pub fn ip4(addr: [4]u8, port: u16) SockAddr;
    pub fn ip6(addr: [16]u8, port: u16) SockAddr;
    pub fn port(self: SockAddr) u16;
    pub fn family(self: SockAddr) u32;
    pub fn eql(a: SockAddr, b: SockAddr) bool;
};
pub const Datagram = struct { len: usize, truncated: bool, from: SockAddr };
pub const Cred = struct { pid: ?pid_t, uid: uid_t, gid: u32 };
pub fn socket(family: u32, kind: u32, nonblock: bool) Result(fd_t);  // always SOCK_CLOEXEC (Darwin: fcntl, §5.4)
pub fn connect(fd: fd_t, addr: SockAddr) Result(void);               // EINTR retried; addr's own length
pub fn bind(fd: fd_t, addr: SockAddr) Result(void);
pub fn listen(fd: fd_t, backlog: u31) Result(void);
pub fn accept(fd: fd_t, nonblock: bool) Result(fd_t);                // accept4(SOCK_CLOEXEC)
pub fn getsockname(fd: fd_t) Result(SockAddr);
pub fn sendto(fd: fd_t, bytes: []const u8, to: ?*const SockAddr) Result(usize); // MSG_NOSIGNAL
pub fn recvfrom(fd: fd_t, buf: []u8) Result(Datagram);               // MSG_TRUNC tells truncation
/// shutdown(2): `.send` half-closes, so the peer reads EOF while this end
/// still reads (quoin's test servers, zerocast's control client). 0.16 has
/// no std.posix.shutdown.
pub fn shutdown(fd: fd_t, how: enum { recv, send, both }) Result(void);
pub fn setsockopt(fd: fd_t, level: i32, name: u32, value: anytype) Result(void);
pub fn peerCred(fd: fd_t) Result(Cred);                              // SO_PEERCRED; Darwin getpeereid

// ---- added: ioctl numbers sized by a length (capsper _ioc, glim iocWR) ----
pub const ioc = struct {
    pub const Dir = enum(u2) { none = 0, write = 1, read = 2, read_write = 3 };
    pub fn encode(dir: Dir, @"type": u8, nr: u8, len: u14) u32;
};

// ---- added: what a library with no main needs instead of an Io ----
pub fn random(buf: []u8) Result(void);   // getrandom; Darwin arc4random_buf
pub fn clockRealtime() timespec;
/// clock_nanosleep(CLOCK_MONOTONIC), EINTR resumed with the remainder;
/// Darwin nanosleep. 0.16 has no std.Thread.sleep: its only sleep takes an Io.
pub fn sleep(ns: u64) Result(void);

// ---- added: the kernel's directory and inotify records, parsed (src/sys/kernel.zig) ----
// Parsers need no table, so a threaded program (capsper) uses them without
// fd's threads check or its test-root check. fd re-exports them.
pub fn getdents(fd: fd_t, buf: []align(8) u8) Result(usize);   // Linux getdents64 (len @intCast: c_uint on 0.17); Darwin getdirentries
pub const Entries = struct { buf: []align(8) const u8, off: usize = 0, pub fn next(self: *Entries) ?Entry; };
pub const Entry = struct { ino: u64, type: u8, name: []const u8 };   // either backend's record
pub const InotifyEvent = extern struct { wd: i32, mask: u32, cookie: u32, len: u32 };
pub const InotifyEvents = struct { buf: []const u8, off: usize = 0, pub fn next(self: *InotifyEvents) ?Event; };

// ---- argv and environ, from main's Init.Minimal (§5.5) ----
pub fn argv(init: std.process.Init.Minimal) []const [*:0]const u8;
pub fn argvSlots(init: std.process.Init.Minimal) [][*:0]const u8;     // writable; argv[argc] == null
pub fn environ(init: std.process.Init.Minimal) []const [*:0]const u8;
pub fn environSlots(init: std.process.Init.Minimal) [][*:0]const u8;
pub fn getenv(init: std.process.Init.Minimal, name: []const u8) ?[:0]const u8; // a scan, no allocation
pub fn exitGroup(status: u8) noreturn;

// ---- added: SIGPIPE, which 0.16's start code no longer ignores (§5.11) ----
/// SIGPIPE to SIG_IGN (rt_sigaction; Darwin sigaction), so a write to a
/// peer that hung up answers EPIPE instead of killing the process.
pub fn ignoreSigpipe() Result(void);
```

The 0.16 port of flong's file: `linux.errno(rc)` replaces `E.init`;
`fstat` and `fstatat` stay fstat(2) and newfstatat(2), made with
`linux.syscall2`/`syscall4` since 0.16's std.os.linux has neither wrapper
nor `Stat` (only statx), and `sys.Stat` is the kernel's `struct stat`
written out for x86_64 and aarch64, asserted against the uapi headers in
the abi test that moves into zigkit (§10); `waitid` takes its fifth
argument; signals use the `SIG` enum; `bufPrintZ` becomes
`bufPrintSentinel`; `builtin.cpu.arch` becomes `builtin.target.cpu.arch`
(sys.zig:382, :806, :1118; deprecated in 0.17, removed in 0.18), so the
extracted file passes zigkit's own `builtin-target` rule from its first
commit (`$F/z17/abi.zig` builds that spelling on 0.16.0 and 0.17.1), and
the facade's `builtin.os.tag` is spelt `builtin.target.os.tag` for the same
rule; every minting call adds its CLOEXEC flag itself (above). Every
`@bitCast` gets a `// bitcast: <why>` comment, so zigkit's own sources
pass the `bitcast` rule from their first commit (§10): flong's sys.zig has
24 lines with one, none commented (20 pass a descriptor, AT_FDCWD's −1
included, through `isize` into a syscall register; pread/pwrite's `i64`
offset; signalfd's −1; getpgid's `u32` → `pid_t`), fd.zig has two (`O`'s
packed flags into `OpenHow.flags`, fd.zig:663, :686), and
tests/zig/fd_props.zig:187 one (fork's register back to `isize`).
`shutdown` and `getppid` are new on both backends (`$H/zk`: Linux
`linux.shutdown`/`getppid`, Darwin `std.c.shutdown`/`getppid`).

**Each call keeps its syscall.** flong init runs under the session's
seccomp filter, and flong's DESIGN.md and docs/declaration.md name the
calls seeding needs (`openat2`, `mkdirat`, `fchmod`, `fstat`, `mmap`,
`umask`), so a port that issued statx(2) for `sys.fstat` would break a
policy that allows fstat and not statx, and make the documented
requirement false. The port changes no call's syscall; where 0.16 lost a
wrapper, `sys` makes the raw syscall itself. 0.17.1 renames the syscall
enum's `fstatat64` (0.16's name for newfstatat on both arches) to
`newfstatat`, so `fstatat` picks it with `@hasField(linux.SYS,
"newfstatat")`. `$I/fstat`: built on 0.16.0 and 0.17.1, strace shows
`newfstatat(AT_FDCWD, "/etc/passwd", …)` and `fstat(3, …)`, no statx; it
cross-compiles for aarch64-linux on both; and the two `Stat`s, built with
0.15.2 (whose std.os.linux still has `Stat`), match its size and every
field's offset for x86_64 and aarch64. Phase 2 lists here any call whose
syscall it does change, and flong's docs change in the same commit.

**An ioctl's descriptor.** Whether a descriptor an ioctl returns is
close-on-exec depends on the request, not on the wrapper: `NS_GET_*` and
`PIDFD_GET_*_NAMESPACE` return one with `FD_CLOEXEC`, `TIOCGPTPEER`
returns one with exactly the flags its `arg` names
(review2-safety/ioctlfd: "TIOCGPTPEER fd=4 FD_CLOEXEC=0", "NS_GET_USERNS
fd=6 FD_CLOEXEC=1"). flong's public `ioctlFd` (sys.zig:658-661) let code
outside `fd` mint such a descriptor unseen by `sys-mint`. In zigkit it is
a private helper of `fd.openNs` (flong's only caller, fd.zig:730), which
passes only the namespace requests and adopts the result. A deny-list in
`sys.ioctl` cannot be complete: KVM_CREATE_VM/VCPU, SYNC_IOC_MERGE,
NS_MNT_GET_NEXT/PREV, USERFAULTFD_IOC_NEW, UDMABUF_CREATE,
DMA_HEAP_IOCTL_ALLOC and DRM_IOCTL_PRIME_HANDLE_TO_FD return one too, the
last four inheritable unless the caller asks (review4-safety/ioctl: with
the deny-list, "KVM_CREATE_VM through sys.ioctl: SUCCESS; open
descriptors 4 -> 6", and `/dev/kvm` is mode 0666 on that host). So the
check is inverted, in two places:

- `sys.ioctl(fd, request, arg) Result(void)` passes any request and is in
  `sys-mint` (§5.7): every call outside `fd` is in a file the program's
  lint config exempts with a reason (flong's init.zig for `TIOCSCTTY`,
  capsper's input.zig for evdev and uinput).
- A handle's `ioctl(comptime request: u32, arg: usize)` (on `file`,
  `pty_master`, `pty_slave`, `tty_out` and the kinds `like` them) compiles
  only for a request in zigkit's list, the terminal requests that return
  no descriptor (TCGETS, TCSETS, TCSETSW, TCSETSF, TIOCSCTTY, TIOCGPGRP,
  TIOCSPGRP, TIOCGWINSZ, TIOCSWINSZ, TIOCGPTN, TIOCSPTLCK), or in the
  program's `zigkit_options.fd.ioctls` (glim's `HIDIOCSFEATURE(92)` and
  `HIDIOCGFEATURE(92)`, which are comptime constants). Anything else fails
  "ioctl 0xae01 is neither zigkit's (the terminal's) nor in
  zigkit_options.fd.ioctls; a request that returns a descriptor must
  never be listed".

`$I/ioctl`: a glim-shaped program listing the two hidraw requests builds
and runs both through an `Fd(.file)` (each `NOTTY` on /dev/null), and
`h.ioctl(0xAE01, 0)` (KVM_CREATE_VM) and `h.ioctl(0x5441, 0)`
(TIOCGPTPEER) each fail with that message. What holds is then: no handle
makes an ioctl that returns a descriptor unless the program listed it by
number, and no `sys.ioctl` call exists outside `fd` but where the lint
config names the file and why. A pty's peer is `fd.openSlave(path)`.

`SockAddr.unix` replaces seven hand-built `sockaddr_un`s (glim ×3, flong
sys.zig, quoin fd.zig and fd_darwin.zig, zerocast ×2), and carries the
length glim computes today for an abstract name (notify.zig:179 for an
`@` NOTIFY_SOCKET, dbus.zig:203 for `unix:abstract=`). Sent at
`sizeof(sockaddr_un)` instead, an abstract name is a different socket: the
name padded with NULs to 108 bytes (review1-consumers/abstract: exact
length SUCCESS, sizeof CONNREFUSED). `$F`'s sys test pins glim's
dbus.zig:404 expectation (`offsetof(path) + 1 + 11` for
`abstract=/tmp/dbus-X`) and binds an abstract datagram socket and
`sendto`s it by `SockAddr`. `ioc.encode`'s
test pins glim's `HIDIOCGFEATURE(64) = 0xC0404807` and a capsper
`EVIOCGNAME(len)`. `peerCred` on Darwin is **to verify in Phase 2** (no
Mac here; the macos-15 CI job runs it).

### 4.3 `errno`

flong's errno.zig unchanged: glibc 2.42's text and name for each errno.
Linux only (on Darwin `@import("errno")` is a compile error pointing at
`@tagName(e)`, so it never prints wrong text).

```zig
pub const max_len: usize;
pub fn text(e: sys.E) ?[]const u8;
pub fn name(e: sys.E) ?[]const u8;
pub fn describe(e: sys.E, buf: *[max_len]u8) []const u8;
```

### 4.4 `fd`: the descriptor table

flong's fd.zig, generalised. §13 is its documentation. The public API:

```zig
pub const BaseKind: type;      // zigkit's 26 kinds (§6), from a literal name list with @Enum
pub const Kind: type;          // BaseKind's names, then the program's extra kinds
pub fn like(comptime k: Kind) BaseKind;
pub const capacity: u16;
pub const SlotIndex: type;     // std.math.IntFittingRange(0, capacity - 1): u2 at 4, u10 at 1024
pub const Error = error{TableFull};

pub fn Fd(comptime k: Kind) type;     // owned: close once
pub fn Held(comptime k: Kind) type;   // no close
pub const File = Fd(.file);
pub const Dir = Fd(.dir);
pub const AnyFd = struct {
    zk_slot: SlotIndex, zk_gen: u32, zk_kind: Kind,
    pub fn raw(self: AnyFd) sys.fd_t;
    pub fn isLive(self: AnyFd) bool;
    pub fn any(self: AnyFd) AnyFd;
};
// Every handle, Fd(k) or Held(k):
//   raw() sys.fd_t, isLive() bool, any() AnyFd,
//   close() void, closeChecked() sys.Result(void)    (owned only: compile error on Held)
//   holdUntilExit() Held(k)
//   as(comptime to: Kind) Handle(to, own)            (the program's kind `like` this one; §5.2)
//   and the kind's operations (§6), each a compile error on any other kind;
//   ioctl(comptime request: u32, arg) only for a listed request (§4.2).

pub const Cwd = struct { pub const kind: Kind = .dir; };
pub const cwd: Cwd;

/// open(2)'s flags a caller may choose: O_CLOEXEC is always added, and
/// O_PATH, O_DIRECTORY and O_TMPFILE make other kinds, so none of them is here.
pub const OpenFlags = struct {
    ACCMODE: sys.ACCMODE = .RDONLY, CREAT: bool = false, EXCL: bool = false, NOCTTY: bool = false,
    TRUNC: bool = false, APPEND: bool = false, NONBLOCK: bool = false, NOFOLLOW: bool = false,
};

// Minting: each returns Error!sys.Result(…), every descriptor close-on-exec.
pub fn openFile(at: anytype, path: [*:0]const u8, flags: OpenFlags, mode: sys.mode_t) Error!sys.Result(File);
pub fn openDir(at: anytype, path: [*:0]const u8) Error!sys.Result(Dir);
pub fn openDirNoFollow(at: anytype, path: [*:0]const u8) Error!sys.Result(Dir);
pub fn openCgroup(at: anytype, path: [*:0]const u8) Error!sys.Result(Fd(.cgroup));
pub fn openPath(at: anytype, path: [*:0]const u8, flags: PathFlags) Error!sys.Result(Fd(.path));
pub fn walkOpen(at: Fd(.path), name: [*:0]const u8, directory: bool) Error!sys.Result(Fd(.path));
pub fn openExact(comptime k: Kind, path: [*:0]const u8) Error!sys.Result(Fd(k));      // .path or .dir
pub fn openFollowing(comptime k: Kind, path: [*:0]const u8) Error!sys.Result(Fd(k));
/// Always OPEN_TREE_CLONE|OPEN_TREE_CLOEXEC (and AT_EMPTY_PATH for ""), so
/// every Fd(.tree) is a detached copy; `at` is a `path` or a `tree`.
pub fn openTree(at: anytype, path: [*:0]const u8, flags: OpenTreeFlags) Error!sys.Result(Fd(.tree));
pub const OpenTreeFlags = struct { recursive: bool = false };   // AT_RECURSIVE
pub fn fsopen(name: [*:0]const u8) Error!sys.Result(Fd(.fsctx));
pub fn fsmount(ctx: Fd(.fsctx), attrs: u32) Error!sys.Result(Fd(.tree));
pub fn openNs(pidfd: Fd(.pidfd), comptime k: Kind) Error!sys.Result(Fd(k));
pub fn openUserns(pid: sys.pid_t) Error!sys.Result(Fd(.userns));
pub fn openNetns(pid: sys.pid_t) Error!sys.Result(Fd(.netns));
pub fn openTmpfile(dir: anytype) Error!sys.Result(Fd(.tmpfile));        // flong's openRecord
pub fn memfd(name: [*:0]const u8) Error!sys.Result(File);
pub fn pipe() Error!sys.Result(Pipe);   // Pipe = struct { r: Fd(.pipe_r), w: Fd(.pipe_w) }
pub fn pidfdOpen(pid: sys.pid_t) Error!sys.Result(Fd(.pidfd));
pub fn inotifyInit() Error!sys.Result(Fd(.inotify));
pub fn openSignalfd(mask: u64) Error!sys.Result(Fd(.signalfd));          // SFD_CLOEXEC|SFD_NONBLOCK
pub fn openPtmx() Error!sys.Result(Fd(.pty_master));                     // RDWR|NOCTTY|NONBLOCK
pub fn openSlave(path: [*:0]const u8) Error!sys.Result(Fd(.pty_slave));
pub fn reopenOut() Error!sys.Result(Fd(.tty_out));                       // /proc/self/fd/1, NONBLOCK
// The unix ones take a pathname or "@name" (abstract), through sys.SockAddr.unix.
pub fn connectUnix(path: []const u8) Error!sys.Result(Fd(.unix));
pub fn listenUnix(path: []const u8, o: ListenOptions) Error!sys.Result(Fd(.unix_listener));
pub fn connectSeqpacket(path: []const u8) Error!sys.Result(Fd(.seqpacket));
pub fn listenSeqpacket(path: []const u8, o: ListenOptions) Error!sys.Result(Fd(.seqpacket_listener));
pub fn unixDgram() Error!sys.Result(Fd(.unix_dgram));
pub fn udp(o: UdpOptions) Error!sys.Result(Fd(.udp));
/// The caller handed over number `n`. Refused, as errnos: 0-2 (Stdio's,
/// INVAL), a number a live slot holds (BUSY), a number not open (F_GETFD's
/// BADF). Then FD_CLOEXEC is set (F_SETFD's errno if not).
pub fn adoptInherited(n: sys.fd_t) Error!sys.Result(Fd(.inherited));
pub fn readFile(at: anytype, path: [*:0]const u8, buf: []u8) Error!sys.Result([]u8); // opened and closed here
pub const ListenOptions = struct { backlog: u31 = 16, nonblock: bool = false, mode: ?sys.mode_t = null };
/// A listener's `accept(o)`: `.nonblock` makes the accepted socket
/// non-blocking, so a send to a peer that stopped reading answers AGAIN
/// (glim drops such a listener; events.zig:56). ListenOptions.nonblock
/// sets only the listening socket: the accepted one's O_NONBLOCK is
/// exactly `o.nonblock` on both backends (Darwin's accept copies the
/// listener's, so the libc backend sets or clears it, §5.4).
pub const AcceptOptions = struct { nonblock: bool = false };
pub const UdpOptions = struct { family: enum { ip4, ip6 } = .ip4, nonblock: bool = false };
pub const PathFlags = struct { DIRECTORY: bool = false, NOFOLLOW: bool = false };

// fork and spawn
/// What a caller chooses for the child; fd builds the CloneArgs. There is no
/// `flags` field: CLONE_FILES, CLONE_VM, CLONE_VFORK, CLONE_THREAD,
/// CLONE_SIGHAND and CLONE_SETTLS cannot be asked for.
pub const CloneOptions = struct {
    cgroup: ?Fd(.cgroup) = null,      // CLONE_INTO_CGROUP with cgroup.raw(); null: the caller's own
    exit_signal: sys.SIG = .CHLD,
};
/// clone3(2) with CLONE_PIDFD (and CLONE_INTO_CGROUP for `o.cgroup`), the
/// args built here and `.pidfd` pointed at fd's own variable: the pidfd's
/// slot is reserved first (TableFull before any child exists), freed if
/// clone3 fails, and filled from that variable. The reservation is
/// private, so no caller-supplied number enters the table.
pub fn clone3(o: CloneOptions) Error!sys.Result(Forked);
pub const Forked = union(enum) { child, parent: struct { pid: sys.pid_t, pidfd: Fd(.pidfd) } };
pub const CloseRangeFailure = struct { low: u32, err: sys.E };
pub fn retainOnly(keep: []const AnyFd) ?CloseRangeFailure;    // panics on a keep list longer than the table
pub fn closeUntracked() ?CloseRangeFailure;

// Stdio, poll, reading
pub const Stdio = enum(u2) { in = 0, out = 1, err = 2, … };  // read write writeAll isatty tcgetattr
                                                             // tcsetattr getWinsize tcgetpgrp tcsetpgrp
pub fn pollEntry(h: anytype, events: i16) sys.pollfd;        // any handle, Stdio, or an optional of either
pub fn readAll(h: anytype, gpa: std.mem.Allocator) error{OutOfMemory}!sys.Result([]u8);

// Paths, entries, events, tests
pub const SelfPath = struct { pub fn path(self: *const SelfPath) [:0]const u8; };
pub fn selfPath(h: anytype) SelfPath;
pub const PidPath = struct { pub fn path(self: *const PidPath) [:0]const u8; };
pub fn pidPath(pid: sys.pid_t, h: anytype) PidPath;
pub const Entries = sys.Entries;              // re-exported: the parsers live in sys (§4.2)
pub const InotifyEvent = sys.InotifyEvent;
pub const InotifyEvents = sys.InotifyEvents;
pub fn liveCount() usize;
/// Every live slot's number, for tests that compare the table with the
/// kernel. A way a number leaves the table (§13): `raw-number` reports
/// `fd.snapshot` as it reports `.raw`.
pub fn snapshot(buf: *[capacity]sys.fd_t) []sys.fd_t;
```

The changes from flong's fd.zig, and why:

- **Zero-encoded slots** (§1.1):
  `Slot = struct { raw: sys.fd_t = 0, gen: u32 = 0, kind: Kind = @enumFromInt(0), state: State = .free }`
  with `State = enum(u8) { free = 0, live, reserved, retired }`.
- **A generation never wraps.** flong's `gen +%= 1` lets a stale copy
  compare equal again after 2^32 bumps of one slot (review1-safety's
  `wrap.zig` did it in 2.4 s with `as`). A close at `gen == maxInt(u32)`
  retires the slot instead of freeing it: `adopt` and `reserve` skip a
  retired slot forever, so the 8-byte handle and the zero encoding stay,
  for one compare on release. `as` at `maxInt` panics "descriptor
  generation spent" (2^32 retags of one open descriptor). `$F`'s unit test
  pre-sets a slot's gen to `maxInt - 1` and sees the slot retired and the
  next open take another; the same `wrap.zig` against `$F` exits 125 with
  that panic instead of reading through the stale copy.
- **No slot holds 0, 1 or 2.** A program started with one of them closed
  gets that number from its next open. Then `Stdio.out` would write into
  the file the program opened, the panic handler's line would land there,
  and a fork child's `retainOnly`, whose `close_range` starts at 3, would
  leave it open behind the table (review1-safety's `stdio.zig`, run as
  `./stdio >&-`, wrote its status line into its record file). So every
  mint that gets a number below 3 moves it with
  `fcntl(F_DUPFD_CLOEXEC, 3)` and closes the low one (one branch, taken
  only then); the move's errno is the open's. `clone3` moves a
  low pidfd the same way, and panics if that fails, which needs both a
  closed 0–2 and a table of numbers at `RLIMIT_NOFILE` (the move is not
  in `$H`'s `clone3`: **to verify in Phase 2** with property
  9). Stdio then writes
  to a closed descriptor and gets `EBADF`, as the caller arranged.
  `$F`: the same `./stdio >&-` leaves the record file holding only its
  own line. On Darwin the move and `adoptInherited`'s checks go through
  libc: `$F`'s `dupAtLeast`, `getFdFlags`, `setCloexec` and unix-socket
  calls used `linux.fcntl`/`linux.socket` on every backend, so after the
  move no Darwin open compiled ("expected type 'c.darwin.E', found
  'os.linux.E…'", review2-safety/darwinmove). `$G/zk` gives each a libc
  branch (Darwin's `F_DUPFD_CLOEXEC` is 67 in 0.16's `std.c`), and its
  `pipe2` sets `FD_CLOEXEC` on both ends or closes both. `$G/darwin`'s
  open, `adoptInherited`, pipe and every Darwin socket kind (listen,
  connect, `accept(.{ .nonblock = true })`, `unixDgram`, `udp`)
  cross-compile for aarch64-macos and x86_64-macos; running them,
  property 9's Darwin form included (§7), is **to verify in Phase 2** on
  the macos-15 job.
- **The handle's fields are `zk_slot`, `zk_gen`, `zk_kind`.** With `slot`
  and `gen`, the lint's `handle-guts` rule fired on `minish.gen.list(…)`
  (safety found it; flong hides it by exempting all of `tests/`). With the
  prefix the rule is exact (`$S`: `minish.gen` not reported, `h.zk_slot`
  reported).
- **No public `adopt(raw)`.** A descriptor enters the table only through a
  zigkit function that adds CLOEXEC itself. `OpenFlags` has no `CLOEXEC`,
  `PATH`, `DIRECTORY` or `TMPFILE` field, so a `file` cannot be opened
  without close-on-exec or as an O_PATH descriptor (both compile errors in
  `$S`). flong's `Reservation.fill(raw)` was a second adopt of any number
  (`fd.reserve().fill(n)` stored `n` unchecked, fd.zig:762-768, and the
  file that called it was exempt from `sys-mint` and `raw-number`).
  zigkit's `fd.clone3` makes the call itself: it sets `CLONE_PIDFD`, points
  `.pidfd` at its own variable and fills the slot from that, which the
  kernel makes close-on-exec; `reserve`, `fill` and `cancel` are private
  (`$H/clone3`: the child exits 7, `waitid` on the pidfd reads 7, the
  pidfd has `FD_CLOEXEC`; at a full table `clone3` answers `TableFull`
  and strace shows no second clone3; `fd.reserve()` from outside fails
  "'reserve' is not marked 'pub'"). `adoptInherited` is the one adopt of a number, and
  it is checked: flong's (fd.zig:642) is a bare `adopt`, so it aliased a
  number the table already held, took 0–2, and took a closed number with
  no errno path (review1-safety's `alias.zig`: two live handles on one
  descriptor, no stale panic anywhere). zigkit's refuses 0–2, a number in
  a live slot (a capacity-long scan) and a number `F_GETFD` says is not
  open, each as an errno, then sets `FD_CLOEXEC` (flong's `Spawn` clears it
  for each descriptor it keeps, proc.zig's `F_SETFD 0`). flong adopts one,
  in a test: tests/zig/proc_props.zig:344-358 opens a file raw, adopts it
  and hands it to `Spawn.keepInherited`, the only check of keepInherited's
  argv and descriptor numbering; its port unwraps the new `Result` (§9.5).
  `$F`'s unit test gets `INVAL`, `BUSY` and `BADF` for the three.
- **`openTree` takes `OpenTreeFlags`, not a `u32`.** flong's passed the
  caller's flags through and added only `OPEN_TREE_CLOEXEC` (fd.zig:706),
  so a caller that left out `OPEN_TREE_CLONE` got the attached mount
  itself as an `Fd(.tree)`, and `moveTo` then moved the live mount away
  (review3-safety/opentree: "after moveTo: a/marker NOENT, b/marker
  SUCCESS"). zigkit's always adds `OPEN_TREE_CLONE|OPEN_TREE_CLOEXEC`,
  `AT_RECURSIVE` for `.recursive` and `AT_EMPTY_PATH` for an empty path,
  so every `tree` is detached by construction, as `OpenFlags` makes every
  `file` close-on-exec. flong's two callers pass `OPEN_TREE_CLONE` already
  (mount.zig:204, :494). `$H/opentree`, run as `unshare -rm`: the tree's
  mount id differs from its source's ("mountId source 1033, tree 1035"),
  and after `moveTo` both a/marker and b/marker exist; `.{ .clone = false
  }` fails "no field named 'clone' in struct 'fd.OpenTreeFlags'", and a
  raw `0` "expected type 'fd.OpenTreeFlags', found 'comptime_int'".
- **`clone3` takes `CloneOptions`, not `sys.CloneArgs`.** `$H`'s
  `clone3(args: sys.CloneArgs)` ORed in `CLONE_PIDFD` and passed the rest
  through. With `CLONE_FILES` the child shares the parent's kernel
  descriptor table but has its own copy of the slots, so its `retainOnly`
  closed the parent's descriptors while the parent's table still called
  them live, and the next open reused the number under an old handle
  (review4-safety/clonefiles: "secret live=true raw=3; other raw=3", then
  /etc/os-release read through `secret`); with `CLONE_VM` the child's
  `release()` would write the parent's own slots; and `.cgroup` was any
  number, a stale one or a plain `dir`'s. zigkit's builds the args itself:
  `CLONE_PIDFD`, `CLONE_INTO_CGROUP` from a live `Fd(.cgroup)`, the exit
  signal, nothing else, so a child always has its own descriptor table and
  is created only in a `cgroup` (§6). flong needs no other flag (its
  `cloneArgs`, proc.zig:97-108, sets these two only), and no consumer asks
  for a namespace flag through clone3; one that does later gets a
  whitelisted field. `$I/clone3`: the parent opens a file, `clone3(.{})`'s
  child runs `retainOnly(&.{})` and exits, and the parent then opens
  another: "secret live=true raw=3; other raw=4; kernel from 3: { 3, 4 };
  live 2", the read through `secret` gets /etc/hostname; `clone3(.{ .cgroup
  = cg })` into the caller's own cgroup runs its child (strace:
  `CLONE_PIDFD|CLONE_INTO_CGROUP … cgroup=5`); `.{ .flags = 0x400 }` fails
  "no field named 'flags' in struct 'fd.CloneOptions'" and a `dir` as the
  cgroup "expected type '?fd.Handle(.cgroup,.owned)', found
  'fd.Handle(.dir,.owned)'".
- **`as`** mints a program's own kind (§5.2) and moves the generation on.
- `retainOnly` panics on a keep list longer than the table instead of
  silently using its first `capacity` entries.
- `builtin.cpu.arch.endian()` (removed in 0.18) becomes
  `std.mem.readInt(…, .native)`.
- `Entries`, `InotifyEvent` and `InotifyEvents` move to `sys` (§4.2), so a
  program that only parses (capsper) never analyses the table or its
  checks.
- Kinds come from a literal name list through `@Enum`, not from
  `@typeInfo(…).fields`, which 0.17 renames. `$S`'s fd.zig compiles on
  0.17.1 unchanged; the full fd.zig and sys.zig are checked there by the
  canary's `refAllDecls` test (§8.2), not by that subset.

### 4.5 `panic`

```zig
/// A panic namespace whose safety panics say std.debug.simple_panic's words
/// without formatting their operands, so no std.Io.Writer is linked.
pub fn Unformatted(comptime call: fn ([]const u8, ?usize) noreturn) type;
/// Unformatted(f) where f writes "<prefix>internal error: <msg>\n" to fd 2 in
/// one write and exit_group(status), from zigkit_options.panic.
pub const handler: type;
/// std.log's sink with no Io: "<prefix><level>(<scope>): <message>\n" formatted
/// into a 512-byte stack buffer (a longer line ends "...") and one write(2) to fd 2.
pub fn logFn(comptime level: std.log.Level, comptime scope: @EnumLiteral(),
    comptime format: []const u8, args: anytype) void;
/// No sigaltstack before main, no segfault handler, the lean log.
pub const std_options: std.Options = .{ .signal_stack_size = null,
    .enable_segfault_handler = false, .logFn = logFn };
```

0.16's `std.log.defaultLog` and `std.debug.print` both reach
`std.Options.debug_io`, an `Io.Threaded`: one call costs ~175 KB
ReleaseSafe (§2). `logFn` costs the program 5,688 bytes for two log lines
(`$F`, `log_lean`, printing `glim: warning(glim): hidraw /dev/hidraw2:
NODEV`). `std.debug.print` has no such hook, so a program that wants a
small binary writes through `fd.Stdio.err` with `bufPrint` instead. A
program that logs on purpose turns the `std-log` rule off and keeps
`debug-output` (§5.7).

A lean root:

```zig
pub const panic = @import("panic").handler;            // flong: panic.Unformatted(msg.onPanic(status))
pub const std_options = @import("panic").std_options;  // or merge the three fields into your own
pub fn main(init: std.process.Init.Minimal) u8 { … }
// build.zig: exe.stack_size = 0;  (no prlimit64 before main)
```

`Unformatted` carries every function 0.17.1's panic namespace requires;
0.16 ignores the two it does not call (§1.6). It takes the call itself,
so flong passes `msg.onPanic(1)`, not `msg.onPanic`, whose type is
`fn (comptime u8) fn (…) noreturn` ("expected type 'T', found 'fn
(comptime u8) T'", review2-zig017/snippets/onpanic.zig; `$G/snippets/onpanic.zig`
with `Unformatted(onPanic(125))` builds on 0.16.0 and 0.17.1 and exits 125
on an out-of-bounds index).

### 4.6 `compat`: reflection and mode, the same on 0.16 and 0.17

```zig
pub fn fieldNames(comptime T: type) []const [:0]const u8;     // 0.16 .fields[i].name; 0.17 field_names
pub fn declNames(comptime T: type) []const [:0]const u8;      // pub decls on both
pub fn fieldDefault(comptime T: type, comptime name: []const u8) ?@FieldType(T, name);
pub fn enumValues(comptime E: type) []const comptime_int;
pub const is_debug: bool;                                     // replaces `builtin.mode == .Debug`
```

The shape is chosen by `@hasField(@TypeOf(info), "field_names")`. zig017's
`reflect.zig` gave the same answers on 0.16.0 and 0.17.1; re-run in
`$S/reflect` on both. Use it as
`inline for (comptime compat.fieldNames(T)) |n| … @field(x, n) … @FieldType(T, n)`.

Its own tests name no mode tag: `is_debug` is checked against
`std.ascii.eqlIgnoreCase(@tagName(builtin.mode), "debug")`. `$G/zk`'s
compat test compared with `builtin.mode == .Debug` and so did not compile
on 0.17.1 ("no field named 'Debug' in enum 'lang.Optimize'",
review3-zig017/compat), which nothing noticed because no 0.17 check
compiled the module. `$H/canary/compat.zig` (the API above, tested over a
struct, a union and an enum: field names, pub decl names only, defaults,
enum values, `is_debug`) passes 3/3 on 0.16.0 and 0.17.1, Debug and
ReleaseSafe. The canary now runs these tests (§8.2).

### 4.7 `input`

`KEY_*`, `BTN_*`, `LED_*`, `EV_*` from `linux/input-event-codes.h` (the
uapi Zig 0.16 bundles) as plain Zig constants: no `@cImport`, no
translate-c for a consumer. `dev`'s `gen-input` step generates it through
`translateC`, and CI fails when regenerating changes the committed file.
The size, safety and simple prototypes all translated this header
(`KEY_A == 30`, `BTN_LEFT == 0x110`, `LED_CAPSL == 1`; re-run in
`$S/zigkit/dev`, natively and for x86_64-linux-musl).

### 4.8 `docharvest`

```zig
pub fn containerName(comptime T: type) []const u8;
pub fn docFor(comptime T: type, comptime field: []const u8, comptime table: anytype) []const u8;
pub fn equal(a: anytype, b: @TypeOf(a)) bool;   // flong's: recurses into slices and unions
```

The build side is the `zigkit-docharvest` tool and the `docharvest`
helper (§8). `join = .newline` keeps flong's Markdown line breaks;
`.space` gives capsper's. Each project keeps its own output writer (Nix
schema, annotated ZON). Phase 2 builds this last; nobody's migration waits
on it.

---

## 5. The thirteen decisions (inventory §7.4)

### 5.1 Error model: flong's `Result(T)`, with `sys.unwrap` for try chains

The kernel's answer is a value; zigkit's own limit is an error. There are
three shapes:

- `sys.Result(T)` from every call;
- `fd.Error!sys.Result(Fd(k))` from every open, `Error = error{TableFull}`;
- `sys.unwrap(x)`, which turns either into `error{TableFull, Errno}!T` (or
  `error{Errno}!T`) for a caller that only needs "it failed".

Reasons:

- Ignoring a `Result` does not compile ("value of type 'sys.Result(usize)'
  ignored", re-run in `$S`), so an errno is never lost by accident.
- flong's 113 `msg.check(…)` sites take `error{TableFull}!Result` and do
  not change.
- quoin's `error.Sys` drops the errno and its `Closed` is never produced.
  `unwrap` keeps quoin's and glim's `try` chains one line each, and the
  drop is visible at every call site. Where a caller branches on the
  errno or prints it, it switches on the `Result` instead (glim's unlink,
  uleds and hidraw cases, §9.1): `unwrap` is for "it failed", not "why". A free function rather than a method
  because it also takes an open's error union (one concept for both).
- It is inline: the quoin-shaped client is 3,192 bytes ReleaseSmall with
  it, against 3,256 with size's method form.

Rule inside zigkit: switch on a `Result`; never read `r.err` after
`r == .err`. ReleaseSafe keeps a formatted `inactiveUnionField` panic for
the read (simple measured `.text` 7,286 → 3,206 bytes for one function).

### 5.2 Kind extensibility: zigkit's base kinds plus the program's extras, each `like` a base kind

zigkit ships every kind any consumer has today (§6). A program adds kinds
in its options:

```zig
pub const zigkit_options: @import("zigkit").Options = .{ .fd = .{
    .capacity = 4,
    .extra_kinds = &.{ .{ .name = "gate", .like = .unix }, .{ .name = "agent", .like = .unix } },
} };
```

- Each extra kind is a **distinct type** with the operations of the base
  kind it is `like`. Passing the gate where the agent is wanted fails:
  "expected type 'fd.Handle(.unix,.owned)', found
  'fd.Handle(.gate,.owned)'" (`$S`). quoin's single `Sock` type could not
  say that.
- An extra kind is minted with `as`:
  `const gate = (try sys.unwrap(fd.connectUnix(path))).as(.gate);`. `as`
  is a compile error unless the bases match ("as(.gate) of a file
  descriptor: .gate is like .unix", `$S`). It retags the slot and moves the
  generation on, so a copy from before `as` panics "stale descriptor
  handle" (`$S`: `stale as`, exit 125).
- An extra kind named like a base kind, or `like` a kind zigkit lacks, is
  a compile error at the declaration (`$S`).
- An operation only one program needs is a free function in that
  program's own syscall-layer file, on `h.raw()`, which the lint lets that
  file use.

Why not `Fd(Kinds, k)` with a program enum: every `need()` list inside
zigkit names kinds, so the program's enum would have to restate zigkit's;
every type name in every error would carry `Kinds`; and a library could
make its own enum and table, which a fork child's `retainOnly` would not
see. Why not safety's kind specs with an `Op` list: a second vocabulary,
where `like` reuses the base kinds' existing gates.

### 5.3 Options: `pub const zigkit_options: @import("zigkit").Options` in the root

- Typed, so a misspelt field is a compile error ("no field named 'capacty'
  in struct 'zigkit.Options.Fd'", `$S`). simple's anonymous struct checked
  only `fd_`-prefixed keys, so `.capacity = 4` fell back silently.
- `pub`: a non-pub declaration is ignored on 0.16 and 0.17 alike (§1.4);
  the lint rule `options-pub` reports it.
- Capacity is comptime and narrows the slot index (u2 at 4, u10 at 1024);
  a handle is 8 bytes either way. Capacity 0 is a compile error. The
  default is 1024, the default soft `RLIMIT_NOFILE`: since capacity costs
  no file bytes, a smaller default buys nothing.
- **Tests:** a test root without `zigkit_options` is a compile error in
  `fd` and `panic` (§1.3). `zigkit.addTest` builds the test with
  `build/test_runner.zig`, which re-exports std's `main` and `std_options`
  and the program's `zigkit_options`.
- **One options file per program.** A program with several roots keeps
  its options in `src/zigkit_options.zig`, made a module in build.zig;
  each root says
  `pub const zigkit_options = @import("zigkit_options").zigkit_options;`,
  and `addTest` and `fdGuards` take the same module. It has to be a module,
  not a relative file import: the same file in two modules fails ("file
  exists in modules …", zig017).
- **Threads.** The table has no lock. A build that is not single-threaded
  fails to compile unless the options say `.threads = .serialized` (the
  caller's promise that one thread at a time touches handles; `$S`,
  `cf/threads.zig` and its control). quoin's `.so` sets it: sudo calls the
  plugin on one thread. A runtime owner check is not in v0.1.
  The check is top-level in `fd`, so it fires whenever `fd` is analysed:
  every module that reaches `fd` must be built single-threaded or say
  `.serialized`, tests included. `zigkit.addTest` therefore sets the test
  module's `single_threaded = true` unless the caller set it (`$F`:
  zigkit's own fd tests pass with no `.single_threaded` in their module,
  through `addTest`); a test that spawns a thread sets
  `.single_threaded = false` and passes an options module that says
  `.serialized`. What only parses kernel records (`Entries`,
  `InotifyEvents`) is in `sys`, which has neither check: `$F/hotplug.zig`,
  capsper-shaped, builds multi-threaded and passes a plain `zig test`
  using `sys.InotifyEvents`, where naming `fd.InotifyEvents` failed both
  (review1-consumers/capsper).

### 5.4 OS backends: Linux raw syscalls; Darwin through libc, for the portable subset

- `sys` has a `libc` backend for macOS (and the BSDs where std.c has the
  call). `fd` on Darwin mints `file`, `dir`, `pipe_r`/`pipe_w`, `unix`,
  `unix_listener`, `unix_dgram`, `udp` and `inherited`. Every other kind
  is a compile error when minted there, and so are `retainOnly` and
  `closeUntracked`, which need `close_range`.
- Close-on-exec on Darwin is atomic for `open` (`O_CLOEXEC`). Darwin has no
  `SOCK_CLOEXEC`, `pipe2` or `accept4` (0.16's `std.c.SOCK.CLOEXEC` there
  is a Zig shim, "Not actually supported by Darwin"), so a socket or pipe
  gets `fcntl(F_SETFD, FD_CLOEXEC)` at once, and is closed if that fails;
  sockets also get `SO_NOSIGPIPE`; a pipe sets it on both ends, and closes
  both if either fails. BSD and XNU `accept` copy the listener's
  `O_NONBLOCK` to the new socket, where Linux's `accept4` does not, so the
  libc `accept` reads `F_GETFL` and sets or clears `O_NONBLOCK` to exactly
  the caller's choice; without that, `listenUnix(p, .{ .nonblock = true })`
  then `accept(.{})` gave a non-blocking `unix` on macOS only. `$I/accept`
  (that sequence, then `F_GETFL` on the accepted socket): "accepted
  O_NONBLOCK=false" on Linux, and Mach-O built for aarch64-macos and
  x86_64-macos; the macos-15 job runs it as a unit test (§10). A fork on another thread in that window
  would leak the descriptor. quoin's darwin plugin is single-threaded, so
  the window is unreachable there. `sys.atomic_socket_cloexec` is `false`
  on Darwin so code can see it.
- A `dir` iterates on both backends: `sys.getdents` is `getdents64` on
  Linux and `getdirentries` (`__getdirentries64`, what 0.16's own
  `Io.Dir` iterator calls) on Darwin, and `sys.Entries` reads either
  record into the same `Entry`. `$F/darwin/dirents.zig` (a Darwin reader
  that counts `/etc/ssh/ssh_host_*`) cross-compiles for aarch64-macos and
  x86_64-macos; running it is **to verify in Phase 2** on the macos-15 job.
- quoin's darwin plugin then uses the same `fd` as its Linux build, and
  `fd_darwin.zig` goes.
- `$G/darwin` (an open, `adoptInherited`, a pipe, every Darwin socket kind,
  against the sources with §4.4's low-number move) cross-compiles for
  aarch64-macos and x86_64-macos; so does `$G/testserver`, quoin's
  threaded test-server shape (§9.2). The earlier `$S` run predates the
  move and no longer stands for it. zigkit's CI runs `sys` and `fd` tests
  on a real macos-15 runner, and quoin's Phase 3 adds a macos-15 job that
  builds and tests its darwin plugin (§9.2), which no CI does today.

### 5.5 argv and environ: functions of `std.process.Init.Minimal`

`main(init: std.process.Init.Minimal)` hands `init` to `sys.argv(init)`,
`sys.argvSlots(init)`, `sys.environ(init)`, `sys.environSlots(init)` and
`sys.getenv(init, name)`. There is no global, so no BSS and no hidden
state, and code that is not handed `init` cannot read argv. On 0.16
`init.args.vector` and `init.environ.block` are the kernel's own slots
(start.zig takes them from the initial stack): writable, with
`argv[argc] == NULL`. `$S`'s `argv` rewrote slot 0, read it back through
`sys.argv`, and found the NULL; safety exec'd a tail of the slots as an
argv under strace; zig017 checked the same on 0.17.1. This settles
inventory §7.3's first unknown. `Init.Minimal` also keeps the start code
small: no allocator, no environ map, no Io. A program whose `main` takes
`std.process.Init` passes `init.minimal`. flong's `argv` lint rule stays,
for programs that want argv confined to the roots.

### 5.6 Libraries with no `main`: `sys` and `fd`, never an Io

`sys` and `fd` take no `Io`. A `.so` that uses them pays 9.6 KB, against
~276–283 KB and 256 KiB of TLS per host thread for an `Io.Threaded` (§2).
zigkit ships no "global Io" helper, because it would make that cost one
line away. quoin's darwin plugin replaces `std.fs` reads with
`fd.readFile`, its host-key listing (`ownHostKeys`,
quoin_sudo_darwin.zig:267-293, whose `Dir.Iterator.next` takes an `io` on
0.16) with `fd.openDir`, `getdents` and `fd.Entries` (§5.4),
`std.crypto.random` with `sys.random`, and
`std.time.timestamp` with `sys.clockRealtime`; where 0.16's
`KeyPair.generate` wants an `io`, it uses `generateDeterministic` with a
seed from `sys.random`, as sshkey.zig already does. wasiglk (WASI) is
outside zigkit's backends; its own `Io.Threaded` in `wasi_glk` is its
decision.

### 5.7 Lint: one host tool, the consumer's ZON config, exemptions with reasons

- One artifact, `zigkit-lint`, built for the host only when a consumer's
  step runs it.
- Rules and exemptions are the consumer's `zigkit-lint.zon`, read by the
  tool at run time (`std.zon.parse`), so the build does no configure-time
  file I/O, which 0.17 forbids. 0.17 replaces 0.16's
  `fromSliceAlloc(T, gpa, src, diag, opts)` with a struct-args
  `fromSlice`, and `Ast.parse` takes options instead of a mode, so the
  tool reaches both through `tools/compat.zig` (`parseZon`, `parseAst`,
  chosen by `@hasDecl`), shared with `zigkit-docharvest`. `$F/z17`'s test
  parses a `zigkit-lint.zon`-shaped config and finds one `**` on 0.16.0
  and 0.17.1 alike.
- An exemption is a glob (`*` within a path segment, `**` across segments),
  which covers flong's paths (`tests/zig/libc_*.zig`, `src/fixtures/**`)
  and quoin's basenames (`**/*_test.zig`), and it **must carry a `why`**.
  An exemption that spared no finding is itself a finding ("abi.zig:
  unused-exemption: no extern finding there any more; remove it (…)",
  `$S`), so an exemption cannot outlive its reason. Exemption and `.skip`
  globs match each file's name as the walk produced it from the path on
  the command line: a lint run from `dev/` over `../guards` names
  `../guards/compile_fail/…`, so the glob is `../guards/compile_fail/**`
  (`$H/selflint`); a walk of `.` names files without `./`, so quoin's
  lint, run from `sudo/` over `.`, exempts a top-level file by its
  basename, which the same config run from the parent over `sudo` would
  not match (review3-consumers/lint reported it unused that way).
- `zig-pkg/`, `.zig-cache/` and `zig-out/` are never walked (quoin's lint
  reported 10 false findings in `zig-pkg/minish`; `$S` checked a planted
  `zig-pkg/` file is skipped). `--as LABEL FILE` stays.
- Output stays `NAME:LINE:COL: RULE: why`; exit 1 on any finding.

```zig
// zigkit-lint.zon
.{
    .rules = .{
        .{ .rule = .@"raw-number", .except = .{
            .{ .path = "src/proc.zig", .why = "the syscall layer" },
            .{ .path = "tests/**", .why = "tests compare with the kernel" } } },
        .{ .rule = .@"c-import" },
        .{ .rule = .@"options-pub" },
        // …
    },
    .skip = .{ "tests/zig/analyze/**", "tests/zig/compile_fail/**" },
}
```

| Group | Rule | Reports | From |
|---|---|---|---|
| table | `raw-number` | `.raw`, and `fd.snapshot` (every live number at once) | flong, quoin |
| table | `handle-guts` | `.zk_slot`, `.zk_gen`, `.zk_kind` | flong, quoin |
| table | `sys-mint` | `sys.` calls that make a descriptor (`openat`, `openat2`, `socket`, `accept`, `pipe2`, `signalfd`, `memfdCreate`, `inotifyInit1`, `openTree`, `fsopen`, `fsmount`, `pidfdOpen`, `dup2`, `fcntl`, `clone3`, and `ioctl`, since some requests return one, §4.2): outside `fd` such a descriptor is not in the table | new |
| layer | `raw-namespace` | `std.os/fs/c/posix/process`, except `std.process.Init` | flong |
| layer | `posix` | `.posix` | flong |
| layer | `argv` | `sys.argv/argvSlots/environ/environSlots/getenv` | flong |
| C | `extern` | `extern fn/var/const`, `@extern` (extern struct/union/enum pass) | flong, quoin |
| C | `export` | `export` | quoin |
| C | `c-import` | `@cImport`; no exemption is accepted | flong (split out) |
| 0.17 | `array-mult` | `**` after a token that ends an operand (a name, a literal, `)`, `}`, `.*`, `.?`, or a `]` closing an index); a pointer-to-pointer type `**T` passes. Help: one element → `@splat` with an explicit length, and `&` where the `**` result was used as a slice (`&@as([200]u8, @splat('/'))`); more than one → a comptime `repeat(s, n)` | new |
| 0.17 | `bitcast` | `@bitCast` without `// bitcast: <why>` on its line | new |
| 0.17 | `optimize-tag` | `== .Debug`, `!= .ReleaseSafe` … | new |
| 0.17 | `typeinfo-fields` | every `.fields`/`.decls` access in a file that reflects (names `@typeInfo` or `builtin.Type`), however the info was bound: `.@"struct".fields`, a switch capture (`.@"struct" => \|st\| st.fields`), a `const info = @typeInfo(T).@"union"`; and `std.meta.fields`, `std.meta.declarationInfo` (0.17.1 makes both a `@compileError`). A false positive (a field of the program's own named `fields`) takes an exemption with its why | new |
| 0.17 | `typeinfo-renamed` | the other `lang.Type` fields 0.17.1 renames, read in a file that reflects: `.params`, `.calling_convention`, `.is_var_args` (Fn's `param_types`, `param_attrs`, `attrs`), `.is_exhaustive` (Enum's `mode`), `.is_const`, `.is_volatile`, `.is_allowzero`, `.alignment`, `.address_space` (Pointer's `attrs`), `.error_set` (ErrorSet's `error_names`). Help: `compat.paramTypes`, `compat.isExhaustive` | new |
| 0.17 | `enum-int` | `@intFromEnum`, `@enumFromInt`, which 0.17 replaces with `@backingInt`/`@fromBackingInt` and its `zig fmt` rewrites. A 0.16 tree cannot satisfy it; it gives the count before a bump (below) | new |
| 0.17 | `builtin-target` | `.cpu/.os/.abi/.object_format` after `builtin`, after any name bound by `const X = @import("builtin")`, or after `@import("builtin")` itself | new |
| 0.17 | `contains-at-least`, `errdefer-capture`, `void-init` | `containsAtLeastScalar*`, `errdefer \|`, `void{}` | new |
| options | `options-pub` | `const zigkit_options` without `pub` | new |
| hygiene | `debug-output` | `debug.print` | flong |
| hygiene | `std-log` | `std.log` (off where a program logs on purpose, through `panic.logFn`) | flong (split out) |
| hygiene | `catch-unreachable` | without `// proven: ` on its line | flong, quoin |
| hygiene | `alloc` | `GeneralPurposeAllocator`, `DebugAllocator` | flong, quoin |

flong's `adopt-foreign` goes: the function it guarded no longer exists, so
a use is a compile error. The `std.process.Init` exception and the
`minish.gen` fix come from simple's and safety's prototypes. The 0.17
rules match tokens. 0.17's tokenizer has no `asterisk_asterisk` tag
(zig017), so a lint.zig that names it does not compile there:
`compat.isArrayMult` matches the tag on 0.16 and two adjacent `asterisk`
tokens on 0.17 (`$F/z17/tooltest.zig`: one finding on both, none in a
comment, a string or `2 * *q`), and in either form only after a token
that ends an operand: without that check `fn f(p: **u8)` and
`?**const u8` were each reported on both versions (review2-zig017/ptrptr).
`$G/z17/t.zig` finds none in `p: **u8`, `?**const u8`, `= **u8`,
`[]**u8`, `[2]**u8` or `2 * *q`, and all five of `[_]u8{0} ** 4`,
`"ab" ** 2`, `x[0..1] ** 2`, `(y) ** 2`, `z ** 2`, on 0.16.0 and 0.17.1.
`typeinfo-fields` and `builtin-target` as above were built into
`$G/zk/tools/lint.zig`: over `$G/lint/case` it reports all nine planted
forms in `bad.zig` (capture, alias, `std.meta.fields`, `@import("builtin").os`,
an aliased `bi.cpu`) and nothing in `good.zig` (`**u8` types,
`builtin.target.os`, a struct's own `fields` in a file that does not
reflect); over the consumers it reports every capture site the review
named (flong decl_docs.zig:237, :274, :422, :441, :453, :751, check.zig:214,
decl.zig:1121; capsper config_docs.zig:135) and every inline
`@import("builtin").os/.cpu` (capsper keep_awake.zig:145,
nemo_mel_state.zig:195, :244; flong fd.zig:1067). `sys-mint` was checked in `$F`: the
review's `leak.zig` (`sys.openat` with no CLOEXEC outside any
syscall-layer file, unreported before) is reported at 4:17, and the
exemption for `fd.zig` counts as used. `$I/lint` (`$H`'s tool with
`ioctl` in `sys-mint`'s list, `fd.snapshot` under `raw-number`, and the
`typeinfo-renamed` and `enum-int` tokens) reports, over a planted file,
`@typeInfo(F).@"fn".params` and `.@"enum".is_exhaustive`, `sys.ioctl(h,
0xAE01, 0)`, `fd.snapshot(&b)` and `@intFromEnum`, one finding each, and
nothing over a file that does not reflect and has a struct of its own with
`params` and `alignment` fields; over flong's tests/zig/libc_scmp.zig it
reports exactly the four `.params` reads, :71, :79, :80 and :83, which
0.17.1 rejects ("no field named 'params' in struct 'lang.Type.Fn'",
review4-zig017/params). `enum-int` over the consumers finds 25 files
(flong 15, zerocast 4, quoin, capsper and glim 2 each). The planted self-test (one finding per rule at an exact
line and column, and a `good.zig` with none) lives in zigkit's `dev/`;
consumers do not repeat it. Its `bad.zig` plants each 0.17 rule's forms
above (the capture, the alias and the inline `@import("builtin")` among
them, a `.params` read and an `@intFromEnum`) and its `good.zig` holds
`**u8` types. A consumer turns on the 0.17 group in the same commit as its
Zig bump, so the bump gates on those hazards being gone; that holds only
because each consumer's CI runs its `lint` step (§9 names the entry point
per consumer). `enum-int` is the exception: 0.16 has no other spelling, so
before the bump a consumer runs it alone for the count, and in the bump
commit 0.17's `zig fmt` rewrites every site (§5.8), after which it passes.

### 5.8 Build helpers: everything 0.17 changes, in two zigkit files

All in `build/helpers.zig`, reached by `const zigkit = @import("zigkit");`
in a consumer's build.zig, and compiled into the build runner only then.
Every `add*Arg`, `b.args`, `b.build_root` and optimize-mode comparison a
consumer would write is here; every 0.16/0.17 fork among them is in
`build/compat.zig`. §8 has the signatures. A consumer's own Run steps
keep their own `expect_term`, which needs no fork: 0.16 renamed
`Child.Term`'s fields to lower case (`.exited`, `.signal`, …) and 0.17.1
kept them, so `.{ .exited = n }` configures and runs on both and
`.Exited` on neither ("no field named 'Exited' in union
'process.Child.Term'", review3-zig017/opt; `$H/exited`: `zig build t`
exits 0 on 0.16.0 and 0.17.1).

- **compile-fail on `Step.Compile.expect_errors`**, not hand-built
  `zig build-obj -M…` lines: it uses the consumer's real module graph and
  needs no `addPrefixedFileArg` (which 0.17 folds into `*Arg2`). `.contains`
  matches the end of an error line, so each expectation is the message's
  full tail (safety). `$S` ran 18 cases plus a guard case under the
  consumer's options; a control that compiles fails the step.
- **The consumer owns its dev dependencies.** Because zigkit's zon is empty
  (§1.2), `analyze`, `fdGuards` and `translateC` take the consumer's
  `zwanzig`, `minish` and `translate_c` dependency or module as an
  argument. The cost: each consumer moves those pins itself, and the 0.17
  move bumps them in each consumer's zon. Its weekly `zig-deps.sh` moves
  minish and zwanzig (GitHub `archive/refs/tags/vX.Y.Z` URLs); translate_c
  is pinned as `git+https://codeberg.org/ziglang/translate-c?ref=1.0.0`,
  which no `zig-deps.sh` matches, and its majors follow Zig's (1.x for
  0.16, 2.x for 0.17), so it is moved by hand with the Zig bump. That was
  chosen over growing every Nix fixed-output hash by translate-c, aro,
  minish and zwanzig (zig017's one-pin design would have).
- **0.17:** zig017 built the same consumer `build.zig` and Kit on 0.16.0
  and 0.17.1 (test, compile-fail with identical messages, lint, run with
  pass-through args, fmt); without `compat.zig` the only 0.17 errors were
  two lines of zigkit's own Kit, none in the consumer. zigkit's 0.17 port
  is then: change `build/compat.zig`, `tools/compat.zig` (the lint's and
  docharvest's ZON, Ast and `**` forks, §5.7), the translate-c branch,
  **and `sys`, `fd` and `guards/`**: 0.17.1 retypes std.os.linux calls
  that only the full sources reach (getdents64's length is `c_uint`;
  `waitpid`/`wait4` take `*i32`; lseek, llseek and the `sched_*` calls
  change too; `SYS.fstatat64` becomes `newfstatat`), and the prototype's
  subset never had them (review4-zig017/getdents: flong's
  `linux.getdents64(fd, buf.ptr, buf.len)` "expected type 'c_uint', found
  'usize'", fd_props' `var status: u32` "expected type '*i32', found
  '*u32'"). Phase 2 writes these once for both versions, so they are not
  left to the bump: `@intCast(@min(buf.len, maxInt(c_uint)))`, and
  fd_props keeps `var status: u32`, passes `@ptrCast(&status)` and gives
  `W.*` a `@bitCast` with its `// bitcast:` reason (`$I/z17`: `g.zig` and
  `w.zig` build and run on 0.16.0 and 0.17.1, the child's exit 3 read back
  on both); `fstatat` picks the syscall's name by `@hasField` (§4.2). The
  bump commit then runs 0.17's `zig fmt` over every source (below) and
  tags; each consumer bumps its pins and does the same. The canary (§10) compiles both tools on 0.17.1, runs
  the `compat` and `docharvest` modules' tests there, analyses every
  non-generic pub function of `sys` and `fd` (Linux backend) through a
  `refAllDecls` test, and runs a lean exe on zigkit's panic namespace and
  `logFn` (§8.2); `translateC`, `fdGuards`' properties and `analyze` are
  not checked on 0.17 until 0.17-capable pins of their dependencies exist
  (§8.2).
- **`zig fmt` across the bump.** 0.17's `zig fmt` rewrites `@intFromEnum`
  and `@enumFromInt` to `@backingInt` and `@fromBackingInt` (rn-0.17.0:
  "zig fmt automatically performs this upgrade"), so `zig fmt --check`
  fails on 0.17.1 over any file with either builtin, and the rewritten
  file does not compile on 0.16 (review4-zig017/fmt). No tree passes
  `fmt` on both. zigkit's own sources have them (the `Slot` default
  `@enumFromInt(0)`, the libc backend's errno, `like`), as do 25 consumer
  files (`enum-int`, §5.7). So the bump commit, in zigkit and in each
  consumer, runs 0.17's `zig fmt` over the package's own dirs and drops
  the 0.16 branches with it, since the result is 0.17-only; until then
  `fmt` runs on 0.16.0 only. `$I/fmt`: a copy of `$H`'s fd.zig passes
  0.16.0's `zig fmt --check` and fails 0.17.1's; the canary's fixture
  passes both.

### 5.9 Reflection shim: the `compat` module (§4.6)

Users: docharvest; flong's 22 field reads, 4 default reads, abi.zig's
`.decls` and its three `std.meta.fields` loops (abi.zig:85, :95, :247,
which 0.17.1 rejects: "deprecated in favor of @typeInfo"), ported to
`inline for (comptime compat.fieldNames(T))` with `@FieldType` and
`@offsetOf` when the test moves into zigkit (`$F/z17/abi.zig`: the
abi.zig:85 loop so written passes on 0.16.0 and 0.17.1); capsper config.zig, config_docs.zig and settings_form.zig;
zerocast session.zig; flong's and quoin's build.zig `Modules.get` (through
`zigkit.compat`). The lint's `typeinfo-fields` rule sends every other
reader here, a switch capture's or an alias's included (§5.7). `@Enum` behaves the same on both versions, so `fd` needs no
shim at all.

### 5.10 Optional modules

| Module | Decision | Reason |
|---|---|---|
| `input` | ship in v0.1 | Pure constants, zero cost; removes capsper input.zig:13's and zerocast wayland/input.zig's `@cImport` and zerocast's hand-written keymap copy. |
| `docharvest` | ship, last in Phase 2 | Two real users (flong, capsper), and the heaviest 0.17 reflection user, so one copy means one shim. |
| `dbus` | not in v0.1 | flong's writer allocates and reads both byte orders; glim's is a fixed little-endian buffer that owns its socket. Moving glim onto flong's is a behaviour change to a 95,952-byte binary. After glim is on `fd`, extract a writer over a caller's buffer plus `sasl.external(uid, *std.Io.Writer)` (the 19 identical lines), and ship it only if glim's size grows by no more than its dbus.zig shrinks. |
| `macos/cf` | rejected | One real user (fulcio); capsper and zerocast need AudioToolbox and CoreGraphics through translate-c anyway. |
| `cast` | rejected | Named `@bitCast` wrappers move the hazard without removing it; the `bitcast` lint rule makes every cast state why, and Phase 3 pins each with a test. |

### 5.11 Panic and start code: the `panic` module plus a README section

The module is §4.5. The README's "Small binaries" section shows the four
lines (panic, std_options, `main(init: Init.Minimal)`, `exe.stack_size = 0`)
and the measured effect (196,608 → 3,224 bytes ReleaseSafe; no syscall
before main). flong keeps its own `msg.onPanic` line, wrapped as
`panic.Unformatted(msg.onPanic(status))` with each root's own status
(§4.5), and takes `panic.std_options`.

**SIGPIPE.** 0.15's start code set SIGPIPE to SIG_IGN unless
`std_options.keep_sigpipe`; 0.16's does not touch it (0.15.2 start.zig's
`maybeIgnoreSigpipe`, :687-712, has no 0.16 counterpart), so flong's
`keep_sigpipe` goes, and a program that wrote to sockets relying on EPIPE
now dies by the signal. zerocast's daemon and capsper's server do
(zerocast daemon.zig:174, :180 `posix.write` of the reply to a CLI
connection; capsper server.zig:615 turns EPIPE into `error.BrokenPipe` on
purpose), and their ports move those writes onto `sys.write`, plain
write(2). Each long-running program that writes where a peer can hang up
calls `sys.ignoreSigpipe()` first in `main` (§9.3, §9.4); fd's own
`sendAll`, `sendto` and `sys.sendto` already send `MSG_NOSIGNAL` (Darwin:
`SO_NOSIGPIPE`). The disposition is inherited across execve, as 0.15's
was, so a child such a program spawns starts as it did on 0.15.
review4-consumers/sigpipe: write(2) to a closed socketpair peer exits 141
built with 0.16.0 and prints "write returned: PIPE" built with 0.15.2.
`$I/sigpipe`: the same with `sys.write`, exit 141 without
`sys.ignoreSigpipe()` and "write: PIPE", exit 0, with it; Mach-O built for
aarch64-macos and x86_64-macos.

### 5.12 Release scheme: gokit's, as gokit is now

gokit changed after the inventory (21bb03a): `release-due` prints `true`
and succeeds, or prints `false` and **exits 1**, read as
`if mise run -q release-due`; every step runs under bash with
`-eo pipefail`; release notes are the commits since the last release. Its
version is `MAJOR.<commit count>.<run number>`. zigkit follows that:

- `MAJOR` = `0`. Tag `v$MAJOR.$(git rev-list --count HEAD).$GITHUB_RUN_NUMBER`;
  a re-run reuses its tag; an annotated tag is pushed by
  github-actions[bot] only if absent, and the job fails if the tag names
  another commit. Shallow clones are refused.
- `RELEASE_PATHS = "build.zig build.zig.zon build src tools guards MAJOR"`:
  `.paths` less LICENSE, plus MAJOR. A change only to `dev/`, docs, CI or
  the toolchain pin releases nothing.
- Notes: `git log "$prev..$GITHUB_SHA" --pretty='- %s (%h)'`, then the
  line a consumer pastes and its hash:
  `zig fetch --save=zigkit https://github.com/danielbodart/zigkit/archive/refs/tags/$TAG.tar.gz`,
  hash from `zig fetch "$url" | tail -1` run **inside the checkout** (0.16's
  bare `zig fetch` refuses outside a project and prints the project path
  first inside one; simple and zig017 measured both).
- `update.yml` weekly (cron `23 21 * * 0`): mise's zig to the latest 0.16
  patch; the minish and zwanzig pins in every nested zon
  (`dev/build.zig.zon`, `dev/consumer/build.zig.zon`) to the newest tag
  whose `minimum_zig_version` is ≤ 0.16.0, as flong's scripts/zig-deps.sh
  does (translate_c's codeberg `?ref=` pin moves by hand, §5.8;
  `dev/canary` has no pin to move); `mise run check`; commit; then
  `if mise run release-due; then gh workflow run ci.yml -f release=true; fi`.
  Since `dev/` is not a release path, a pin bump alone releases nothing.

### 5.13 Phase 3 order and its constraints

Order (PLAN's, risk ascending): **glim → fulcio-codesign →
browser-agent-os → wasiglk → quoin → capsper → zerocast → flong**.

1. **zigkit is tagged and green before a consumer pins it**, with every
   kind flong needs, the Darwin backend and the helpers. A change a
   consumer needs mid-Phase-3 is a new zigkit tag first; earlier consumers
   bump only if they want it. No path dependencies on trunk. **zigkit is
   never moved unattended:** each consumer's `scripts/zig-deps.sh` moves
   every `archive/refs/tags/vX.Y.Z` URL to the newest tag, which zigkit's
   install line is, and its weekly update is all-or-nothing and releases
   the consumer (build.zig.zon is a `RELEASE_PATH` in glim, capsper, flong
   and quoin's `sudo`). So each script skips the dependency named
   `zigkit`, and a consumer bumps zigkit by hand in a commit that says so.
   That commit also runs `scripts/zig-deps.sh --hashes` (§9), which
   recomputes every Nix fixed-output hash from the current build.zig.zon
   whether or not the script moved a pin: zigkit is in the eager FOD of
   glim, capsper, flong and quoin and in flong's and quoin's dev FOD, and
   without it the consumer's Nix CI fails on a hash mismatch. The same
   holds for translate_c's hand move at the 0.17 bump (§11.2).
2. **zigkit's `fd`, `sys` and `errno` are taken from flong trunk at a
   recorded commit** (c72db3b unless flong moves first). Until flong
   migrates, any change to flong's src/fd.zig, sys.zig or errno.zig is
   ported to zigkit first; flong's Phase 3 worktree diffs them.
3. **glim is the first `sys`/`fd` consumer**: small, single-threaded, about
   30 removed posix calls, and a 95,952-byte gate that proves the size claims
   on a real binary.
4. fulcio-codesign, browser-agent-os and wasiglk take **no zigkit
   dependency** (§9.6); they only move to 0.16. wasiglk lands its 0.16 bump
   only with the split-EH-object setjmp workaround, else stays on 0.15.2;
   it blocks nothing.
5. **quoin before flong**, so the `fd` flong finally switches to has
   already carried capacity 4, extra kinds, Darwin and `unwrap`.
6. **flong last**: most `@bitCast`, Nix-built, and the source.
7. **Downstream:** chase after both quoin and flong; nix-config after
   capsper and glim. quoin's server takes the sudo plugin on autoUpgrade
   with no human gate, so quoin's VM tests rerun before its trunk push.
8. Each consumer drops minish v0.1.0 **in the same commit** as its Zig
   bump (a v0.1.0 on disk breaks every 0.16 build step), and refreshes its
   Nix fixed-output hashes (both, §9, with `zig-deps.sh --hashes`) in that
   commit. At the 0.17 bump the same commit runs 0.17's `zig fmt` over the
   consumer's own dirs (§5.8).
9. Each consumer's gate: tests, lint, compile-fail and analyze green on the
   same zigkit tag; shipped size ≤ its baseline, or the growth explained.

---

## 6. Kinds

| Kind | Minted by | Operations | Linux | Darwin |
|---|---|---|---|---|
| `file` | `openFile`, `memfd` | read, write, writeAll, pread, pwrite, fstat, flock, ioctl (a listed request, §4.2) | ✓ | ✓ |
| `dir` | `openDir`, `openDirNoFollow`, `openExact(.dir)`, `openFollowing(.dir)` | mkdirat, unlinkat, renameat, fstatat, getdents (read with `Entries`), fchownat, fstat, flock; the `at` of an open | ✓ | ✓ (getdents is getdirentries) |
| `path` | `openPath`, `walkOpen`, `openExact(.path)`, `openFollowing(.path)` | mkdirat, unlinkat, fchownat, fstat, mountId, mountSetattr; the `at` of an open | ✓ | |
| `tree` | `openTree` (always a clone), `fsmount` | mkdirat, fchownat, fstat, mountId, mountSetattr, moveTo | ✓ | |
| `fsctx` | `fsopen` | setString, setFlag, setFd (a `dir` only), create | ✓ | |
| `userns`, `mntns`, `netns`, `cgroupns` | `openUserns`, `openNetns`, `openNs` | setns(comptime ns), which must match the kind | ✓ | |
| `cgroup` | `openCgroup` | mkdirat, unlinkat, fstat; the only kind a child is created in (`CloneOptions.cgroup`) | ✓ | |
| `pidfd` | `pidfdOpen`, `clone3` | waitid, sendSignal | ✓ | |
| `inotify` | `inotifyInit` | addWatch, readEvents | ✓ | |
| `signalfd` | `openSignalfd(mask)` | readSiginfo | ✓ | |
| `inherited` | `adoptInherited(n)`: refuses 0–2, a held number, a closed one | kept by flong's Spawn | ✓ | ✓ |
| `tmpfile` (flong's `record`) | `openTmpfile(dir)` | write, pread, pwrite, fstat, flock, linkInto | ✓ | |
| `pipe_r`, `pipe_w` | `pipe` | read / write, writeAll | ✓ | ✓ |
| `pty_master` | `openPtmx` | read, write, unlock, ptyNumber, setWinsize, ioctl | ✓ | |
| `pty_slave` | `openSlave(path)` | setWinsize, tcsetattr, ioctl | ✓ | |
| `tty_out` | `reopenOut` | write, ioctl | ✓ | |
| `unix` (flong's `bus`, quoin's `Sock`) | `connectUnix`, `unix_listener.accept(o)` | read, writeAll, sendAll, recvFull, peerCred, shutdown | ✓ | ✓ |
| `unix_listener` | `listenUnix` | `accept(o: AcceptOptions)` → `Fd(.unix)` | ✓ | ✓ |
| `seqpacket` | `connectSeqpacket`, `seqpacket_listener.accept(o)` | send, recv, peerCred, shutdown | ✓ | |
| `seqpacket_listener` | `listenSeqpacket` (glim's events) | `accept(o: AcceptOptions)` → `Fd(.seqpacket)`; glim passes `.nonblock = true` | ✓ | |
| `unix_dgram` | `unixDgram` (glim's sd_notify) | sendTo(bytes, SockAddr), an abstract `@` address at its own length | ✓ | ✓ |
| `udp` | `udp(.{ .family, .nonblock })` | bind, localAddr, connect, sendto, send, recvfrom, setOption | ✓ | ✓ |

`sendAll` uses `MSG_NOSIGNAL` (a peer that went is an errno, never
SIGPIPE); `recvFull` reads until the buffer is full or the peer closes.
`shutdown(how)` half-closes a `unix` or `seqpacket` (`$H/shutdown`: after
the server's `shutdown(.send)` the client's `recvFull` returns "reply" at
EOF and the server still reads the client's next send; Mach-O built for
aarch64-macos and x86_64-macos; on a `file`, "shutdown on a file
descriptor"). A listener's `accept` takes `AcceptOptions` (§4.4), passed to
`sys.accept(fd, o.nonblock)`, which on both backends leaves the accepted
socket's `O_NONBLOCK` exactly `o.nonblock`, whatever the listener's
(§5.4). Without it an accepted socket blocks: a
seqpacket peer that never reads holds a blocking send forever, where a
non-blocking one answers AGAIN (review2-consumers/seqblock; `$G/accept`'s
`seqnb.zig` over `fd.listenSeqpacket` and `accept(.{ .nonblock = true })`:
"send 278 -> AGAIN", and with `accept(.{})` still blocked after 3 s).
`accept` on any other kind is a compile error ("accept on a file
descriptor", `$G/accept/wrongkind.zig`).
Tag names cost nothing: an operation is analysed only when called.

**udp**, for the Mosh port:

```zig
pub fn bind(self: Self, addr: sys.SockAddr) sys.Result(void);
pub fn localAddr(self: Self) sys.Result(sys.SockAddr);                         // the port bind(…:0) chose
pub fn connect(self: Self, to: sys.SockAddr) sys.Result(void);                 // a fixed peer (the client)
pub fn sendto(self: Self, bytes: []const u8, to: sys.SockAddr) sys.Result(usize); // MSG_NOSIGNAL
pub fn send(self: Self, bytes: []const u8) sys.Result(usize);                  // after connect
pub fn recvfrom(self: Self, buf: []u8) sys.Result(sys.Datagram);              // .from: the sender (roaming)
pub fn setOption(self: Self, level: i32, name: u32, value: anytype) sys.Result(void); // IP_TOS, IP_MTU_DISCOVER, IPV6_V6ONLY
```

size's test bound two sockets to 127.0.0.1:0, sent "hello" between them,
and `recvfrom` reported the sender's port and address; safety's also
checked truncation (`"hel"`, `truncated = true`) and an IPv6 round trip.
In a network namespace with `lo` down, `sendto` answers an errno; zigkit's
udp tests skip on `ENETUNREACH`/`EADDRNOTAVAIL` (safety). A kernel address
of an unknown family is `.err = .AFNOSUPPORT`, never a panic. ECN through
`recvmsg` (IP_RECVTOS) is a later addition, one wrapper with its own gate.

**pty, signalfd and pollEntry** come over from flong unchanged.
`pollEntry(h, events)` takes any handle, `Stdio`, or an optional of
either (null polls nothing, fd -1), and is the only way a number reaches a
pollfd array. A Mosh server polls a udp socket, a pty master, a tty_out
and a signalfd with it.

---

## 7. Guards: how they ship and how each consumer runs them

| Guard | Ships as | zigkit runs it | A consumer runs it |
|---|---|---|---|
| lint | `zigkit-lint` artifact, `zigkit.lint`, the consumer's `zigkit-lint.zon` | `dev`: every rule over `../src ../tools ../build ../guards` with `dev/zigkit-lint.zon` (§10), plus the planted self-test | `zig build lint` (needs no dev dependency), run by the entry point its CI already runs (§9 per consumer) |
| `@cImport` ban | rule `c-import`, no exemption accepted | yes | on in every consumer |
| compile-fail | `zigkit.compileFail`; zigkit's cases in `guards/compile_fail/{consumer,zigkit}/` | `dev`: both sets, each case for its OS (below); the consumer set again under default, glim-shaped and quoin-shaped options (`dev/consumer`) | `zigkit.fdGuards` runs the **consumer set under the consumer's own options**, the cases for its target's OS; `zigkit.compileFail` runs the consumer's own (flong's msg_arguments, passfd_stdio, fork_body_returns, dir_as_cgroup, decl_undocumented) |
| minish properties | `guards/fd_props.zig` (Linux) and `guards/fd_props_darwin.zig`, run by `fdGuards` with the consumer's minish | `dev`: under three option sets (default, capacity 4 with extras, `.serialized`); the Darwin set on macos-15 | `zigkit.fdGuards` with the consumer's minish v0.4.0, behind `-Ddev`, the set for its target; and its own properties |
| zwanzig | `zigkit.analyze` with the consumer's zwanzig v0.15.1 | `dev`: over `../src`, plus planted bugs | `zig build analyze -Ddev=true`, its own `.zwanzig.json` |
| size | `dev/size` | `.data == 0`, `.bss == 12 × capacity` (0 for `stdio_only`), and byte budgets for `fd_q4`, `fd_f1024`, `quoinlike`, `log_lean` (beside `none_lean`) and a libc-linked `.so` with `panic.handler`, beside the same `.so` with the default panic as the control (4,912 against 201,720 bytes ReleaseSafe, review2-consumers/sopanic) | glim, quoin and flong keep "shipped size ≤ baseline" from Phase 3 |

**zigkit's compile-fail cases**, each with its message's full tail, in
two sets. A case of the **consumer set** re-exports the consumer's options
(`pub const zigkit_options = @import("zigkit_options").zigkit_options;`)
and names only base kinds, so its error is the same under any options:
that is the set `fdGuards` runs, and it shows the consumer's own capacity
and extras compile with zigkit's guarantees intact. A case of the
**zigkit set** is about option handling: it declares its own options or
uses the defaults, so it cannot also take the consumer's, and several
fail only under particular options (`threads` compiles under quoin's
`.serialized`; `extra_for_base` under glim's options fails "enum
'fd.enumOf(…)' has no member named 'gate'" instead of its expected
message, review3-consumers/glimopts and review3-build/guardopts). Nothing
about a consumer changes their outcome, so they run only in zigkit's
`dev`. Each case also names its OS set, and `fdGuards` and dev's
`compile-fail` choose by `target.result.os.tag`: on Linux the
`linux_kind_on_darwin` case compiles, and for a Darwin target the Linux
cases fail on the Linux-only call first (review3-build/guardos).

| Set | OS | Cases |
|---|---|---|
| consumer | any | `result_ignored`, `result_unwrapped`, `held_close`, `stdio_close`, `read_on_dir`, `read_on_pipe_w`, `dir_for_file`, `file_as_at` (from flong, wording kept); `open_cloexec`, `open_path`, `sendall_on_udp`, `read_on_udp`, `poll_entry_of_int`, `accept_on_file`, `shutdown_on_file`, `ioctl_unlisted` (KVM_CREATE_VM on a `file`: "ioctl 0xae01 is neither zigkit's (the terminal's) nor in zigkit_options.fd.ioctls; …") |
| consumer | linux | `setns_wrong_ns`, `setfd_path` (from flong); `open_tree_noclone` ("no field named 'clone' in struct 'fd.OpenTreeFlags'"), `clone3_reserve` ("'reserve' is not marked 'pub'"), `clone3_flags` ("no field named 'flags' in struct 'fd.CloneOptions'"), `clone3_dir_cgroup` ("expected type '?fd.Handle(.cgroup,.owned)', found 'fd.Handle(.dir,.owned)'") |
| consumer | darwin | `linux_kind_on_darwin` (a `path` minted for macOS: "fd: path descriptors are Linux-only; this target is macos") |
| zigkit | any | `extra_for_base` (`'fd.Handle(.gate,.owned)'` where `.unix` is wanted), `as_unlike`, `bad_like`, `extra_shadows`, `threads` and its `threads_ok` control, `capacity_zero`, `options_typo`, `options_not_pub`, `test_without_runner` |

`$H/guards` builds this split with an `fdGuards`-shaped selection over
`$H/zk`, twelve consumer cases (nine any, two Linux, one Darwin) and nine
zigkit cases: the consumer set under default, glim-shaped (capacity 16,
`hidraw` and `uled`) and quoin-shaped (capacity 4, `.serialized`, `gate`
and `agent`) options passes 12/12 steps for x86_64-linux and 11/11 for
aarch64-macos and x86_64-macos under each, and the zigkit set 10/10 for
all three targets. The three cases added for the fourth review
(`ioctl_unlisted`, `clone3_flags`, `clone3_dir_cgroup`) were built on
their own against `$I/zk` with their messages (§4.2, §4.4), not yet in
that harness: running them under the three option sets is **to verify in
Phase 2**.

**zigkit's properties** (minish v0.4.0, fixed seeds; a failure prints
`.{ .seed = N }`, which becomes a fixed regression case; properties must
not rely on shrinking, which size saw stop short):

1. The model: after any sequence of opens (file, dir, pipe, udp, under a
   held dir) and closes, the table, a model and `/proc/self/fd` agree;
   every closed handle is stale in every copy; every open one is live.
2. Fork and `retainOnly`: a random table, a random keep subset; the child
   judges itself by exit status (kept not live, dropped still live, the
   kernel's open numbers from 3 up ≠ kept's, or a slot holding a number
   below 3). safety checked it catches a child that does not release
   dropped slots and a skipped `close_range`. The parent, after waiting
   on the pidfd, checks its own side: every handle it held is still live,
   and its table still equals `/proc/self/fd` from 3 up, so a child that
   shared the parent's descriptor table would be caught (`$I/clone3` is
   that check over one file).
3. Close-on-exec: after any sequence of mints of every kind, extra kinds
   included, every live descriptor has `FD_CLOEXEC`.
4. A stale `read` and a wrong-kind `AnyFd` panic in a child with their
   messages.
5. `TableFull` at capacity, the kernel's descriptor closed, the table
   unchanged.
6. udp: random payload and buffer sizes give `len = min`, `truncated`
   exactly when the payload is larger, `from` the sender, v4 and v6.
7. `SockAddr` round trips; `ioc.encode` against std's fixed-size
   `IOCTL` for the sizes both can express.
8. errno: `describe` for every u16 equals glibc's (`test-libc`, linked
   against glibc, Linux CI).
9. Started with 0–2 closed: properties 1–3 run again in a child that
   closes a random subset of 0, 1 and 2 before its first mint; every live
   slot's number is ≥ 3, `Stdio` writes answer `EBADF` for a closed one,
   and the `retainOnly` child's kernel set from 3 up equals its keep list.

Unit tests beside the properties: a non-blocking listener's `accept(.{})`
gives a socket with `O_NONBLOCK` clear, and `accept(.{ .nonblock = true
})` from a blocking one a socket with it set, on Linux and on the macos-15
job (`$I/accept`); `clone3(.{ .cgroup = own })` runs its child in the
caller's own cgroup, skipped where that cgroup refuses it (`$I/clone3`);
`sys.ignoreSigpipe()` then a write to a hung-up socketpair answers `PIPE`
(`$I/sigpipe`); `adoptInherited` refuses 1 (`INVAL`), a
number the table holds (`BUSY`) and a closed one (`BADF`); a slot whose
gen is pre-set to `maxInt - 1` is retired after two closes and the next
open takes another slot (both pass in `$F`, Debug and ReleaseSafe); an
accepted non-blocking seqpacket answers AGAIN once its peer's queue is
full (`$G/accept/seqnb.zig`, as a unit test); a `tree` from `openTree`
has a mount id other than its source's, and after `moveTo` the source
mount still holds its files (in a user and mount namespace, skipped where
`unshare` is refused; `$H/opentree`); `clone3`'s child is waited through
the pidfd it filled, the pidfd has `FD_CLOEXEC`, and at a full table no
child is made (`$H/clone3`); a half-closed `unix` gives its peer EOF and
still reads (`$H/shutdown`).

**On Darwin** the set is `fd_props_darwin.zig`: properties 1 and 3–7 on
the Darwin kinds, with the model compared through `fcntl(F_GETFD)` over
0..capacity+3 instead of `/proc/self/fd`; property 9 without its
`retainOnly` part, in-process since `sys` has no Darwin fork (the test
saves 0–2 above the table with `F_DUPFD_CLOEXEC`, closes a random subset,
mints, checks that every live slot's number is ≥ 3 and that `Stdio`
answers `EBADF` for a closed one, then restores 0–2 with `dup2`);
and `adoptInherited`'s three refusals, and the accept unit test above
(the case Darwin's inherited `O_NONBLOCK` breaks). There is no `retainOnly` property:
Darwin has no `close_range`, `retainOnly` is Linux-only, a compile error
on Darwin like the Linux kinds, and no Darwin code forks with a keep list
(quoin's darwin test servers are threads, §9.2).

**zwanzig models** match by method name, so the names are API: an open
model for every minting function above and for `as`, a close model for
`close` and `closeChecked`. quoin's planted checks keep `connectUnix` and
`close`. A consumer's `.zwanzig.json` is its own, so a rename (flong's
`openRecord` → `openTmpfile`) silently drops every handle of that kind
from the analysis unless the config and a planted bug move with it.
zigkit therefore ships its model list as `guards/zwanzig-models.json`
(the open and close models for every function above; dev's own
`.zwanzig.json` holds the same entries, and dev's planted bugs a double
close per kind), its release notes name any minting function it renames, and each consumer's
Phase 3 commit copies the models for the functions it calls and plants
one double close per kind it mints. How zwanzig treats `as` (the receiver ends, a new handle starts)
is **to verify in Phase 2**; size found a retag modelled as an open caught
its planted double close. `clone3` returns its pidfd inside `Forked`'s
`.parent`, so its open model reads that field; how zwanzig follows a
handle out of a union payload is **to verify in Phase 2** too. `analyze` adds
`expect_stderr_match = "analysis complete"`-style checks so 0.16's warning
block stays quiet, and expects exactly `"Found N issue(s):\n"` over the
planted file. zigkit's `.zwanzig.json` disables `identifier-style`, which
flags every type-returning function (simple).

---

## 8. Build helpers

### 8.1 Signatures (`build/helpers.zig`, re-exported from build.zig)

```zig
/// A test of `root_module` under zigkit's runner and the program's options:
/// the module whose root declares `pub const zigkit_options`, or null for
/// the defaults, which have no extra kinds and capacity 1024. A program with
/// either passes its options module (§5.3); a single-root program may pass
/// its root module itself. A plain b.addTest of an fd user does not compile.
/// Sets root_module.single_threaded = true unless the caller set it (§5.3).
pub fn addTest(b: *std.Build, zigkit: *std.Build.Dependency, root_module: *std.Build.Module,
    options: ?*std.Build.Module) *std.Build.Step.Compile;

pub const CompileFailCase = struct {
    file: std.Build.LazyPath,
    expect: []const u8,            // the full tail of one error line
    single_threaded: bool = true,
    os: enum { any, linux, darwin } = .any,   // skipped unless target.result.os.tag is in it
};
pub fn compileFail(b: *std.Build, step: *std.Build.Step, target: std.Build.ResolvedTarget,
    imports: []const std.Build.Module.Import, cases: []const CompileFailCase) void;

pub const GuardOptions = struct {
    target: std.Build.ResolvedTarget,
    optimize: std.builtin.OptimizeMode,
    options: *std.Build.Module,              // the program's zigkit_options module
    minish: ?*std.Build.Module = null,       // the consumer's; null skips the properties
};
/// zigkit's consumer set (guards/compile_fail/consumer/, §7) under the
/// consumer's options, the cases for o.target's OS; and, with minish, the
/// properties for that OS. The zigkit set is dev's alone.
pub fn fdGuards(b: *std.Build, step: *std.Build.Step, zigkit: *std.Build.Dependency, o: GuardOptions) void;

pub const LintOptions = struct { config: std.Build.LazyPath, paths: []const []const u8 };
/// zigkit-lint from the package root; always runs; findings to the build's output.
pub fn lint(b: *std.Build, zigkit: *std.Build.Dependency, o: LintOptions) *std.Build.Step.Run;

pub const AnalyzeOptions = struct {
    config: std.Build.LazyPath,                                  // .zwanzig.json
    paths: []const std.Build.LazyPath,
    planted: ?struct { file: std.Build.LazyPath, expect: []const []const u8 } = null,
};
pub fn analyze(b: *std.Build, step: *std.Build.Step, zwanzig: *std.Build.Dependency, o: AnalyzeOptions) void;

pub const TranslateOptions = struct {
    name: []const u8,
    header: []const u8,                       // "#define _GNU_SOURCE\n#include <unistd.h>\n"
    target: std.Build.ResolvedTarget,
    optimize: std.builtin.OptimizeMode,
    link_libc: bool = true,
    include_paths: []const std.Build.LazyPath = &.{},
    system_include_paths: []const std.Build.LazyPath = &.{},
    defines: []const [2][]const u8 = &.{},
    system_libs: []const []const u8 = &.{},  // pkg-config names: run while configuring, for every step (§8.3)
    libc_headers: enum { bundled, system } = .bundled,
    framework_paths: []const std.Build.LazyPath = &.{},          // Translator.addFrameworkPath
    system_framework_paths: []const std.Build.LazyPath = &.{},   // Translator.addSystemFrameworkPath
};
/// @cImport's replacement over the consumer's translate_c dependency and
/// its Translator type (`@import("translate_c").Translator`).
pub fn translateC(b: *std.Build, comptime Translator: type, translate_c: *std.Build.Dependency,
    o: TranslateOptions) *std.Build.Module;

/// The -Ddev gate: null, with `step` failing "needs -Ddev=true", unless dev;
/// otherwise b.lazyDependency(name, args).
pub fn lazyDev(b: *std.Build, step: *std.Build.Step, dev: bool, name: []const u8, args: anytype) ?*std.Build.Dependency;
pub fn version(b: *std.Build) *std.Build.Module;            // -Dversion → build_info.version
pub fn passthroughArgs(run: *std.Build.Step.Run) void;      // 0.16 b.args; 0.17 addPassthruArgs
pub fn rootPath(b: *std.Build, sub_path: []const u8) []const u8; // 0.16 b.build_root; 0.17 b.root
pub fn isDebug(mode: std.builtin.OptimizeMode) bool;        // 0.17 renames the tags
/// A path option (`b.addOptions`) naming a file, or a directory: 0.17's
/// addOptionPath must be a file and a directory needs addOptionPathDirectory.
pub fn addOptionFile(o: *std.Build.Step.Options, name: []const u8, p: std.Build.LazyPath) void;
pub fn addOptionDir(o: *std.Build.Step.Options, name: []const u8, p: std.Build.LazyPath) void;
/// `zig fmt --check` over the package's own dirs: "." is refused, since on 0.16
/// it walks zig-pkg/ and checks every dependency's formatting.
pub fn fmt(b: *std.Build, paths: []const []const u8) *std.Build.Step; // 0.17 takes LazyPaths
pub const Join = enum { newline, space };
pub fn docharvest(b: *std.Build, zigkit: *std.Build.Dependency, source: std.Build.LazyPath, join: Join) *std.Build.Module;
pub const compat = @import("../src/compat.zig");            // compat.fieldNames for a build.zig
```

An `@import` path is relative to the importing file, so from
`build/helpers.zig` the module is `../src/compat.zig`; `"src/compat.zig"`
named `build/src/compat.zig` and failed "FileNotFound"
(review2-zig017/compatpath). `$G/optpath` (a zigkit-shaped package whose
build.zig re-exports `helpers.compat`, and a consumer calling
`zigkit.compat.fieldNames` in its build.zig) configures and runs on 0.16.0
and 0.17.1.

A consumer declares its own `dev` option (zig017 found a helper that
declares `-Ddev` panics when called twice). It calls
`b.dependency("zigkit", .{})` once: two instances with different arguments
are two `fd` modules and two tables.

### 8.2 `build/compat.zig`: the 0.17 build-API differences, once each

```zig
pub const is_017 = builtin.zig_version.minor >= 17;
pub fn zigLibPath(b: *Build, sub_path: []const u8) Build.LazyPath;   // test_runner.zig's path
pub fn passthroughArgs(run: *Build.Step.Run) void;
pub fn rootPath(b: *Build, sub_path: []const u8) []const u8;
pub fn isDebug(mode: std.builtin.OptimizeMode) bool;                // case-insensitive "debug"
pub fn fmtCheck(b: *Build, paths: []const []const u8) *Build.Step;
pub fn addOptionFile(o: *Build.Step.Options, name: []const u8, p: Build.LazyPath) void; // addOptionPath on both
pub fn addOptionDir(o: *Build.Step.Options, name: []const u8, p: Build.LazyPath) void;  // 0.17 addOptionPathDirectory
```

zig017 wrote and ran the first five on 0.16.0 and 0.17.1. 0.17 splits
`Step.Options.addOptionPath` three ways: `addOptionPath` (a file),
`addOptionPathDirectory` and `addOptionPathUntracked` (0.17.1
Options.zig:431, :449, :467); one `addOptionPath(o, name, p)` cannot tell
which, and a directory through 0.17's `addOptionPath` fails when the step
runs ("checking cache failed: file_read IsDir", review2-zig017/optpath).
`addOptionDir` calls `addOptionPathDirectory` on 0.17 and `addOptionPath`
on 0.16; `addOptionFile` calls `addOptionPath` on both. `$G/optpath`'s
consumer passes a directory through `addOptionDir` and a built binary
through `addOptionFile`, and `zig build run` prints both on 0.16.0 and
0.17.1 (fresh cache each). The directory's value differs: absolute on
0.16, relative to the build root on 0.17.1 (`corpus`), so a consumer that
opens it must run with the build root as its working directory; flong's
tests are checked for that at the 0.17 move. zigkit targets 0.16.0 only
(PLAN §2.2).

**What the canary can and cannot check.** Once a build.zig names a
dependency's type (`@import("translate_c")`, or `b.lazyImport`, even in a
branch that never runs), 0.17.1 compiles the build.zig of every
dependency on disk, and translate_c 1.0.0 (with its aro), minish 0.4.0
and zwanzig 0.15.1 do not compile there ("no field named 'Debug' in enum
'lang.Optimize'", "no field named 'build_root' in struct 'Build'";
review1-zig017/canary and review1-build/canary). A canary that declares
them is red for reasons that are not zigkit's. So the canary is its own
package, `dev/canary`, whose zon holds zigkit by path and nothing else,
run with a cache dir that holds no dev-dependency tarballs. It exercises
`addTest`, `compileFail`, `fdGuards` with `minish = null` (zigkit's
compile-fail guards, no properties), `lint` (so `zigkit-lint` and its
`tools/compat.zig` are compiled on 0.17), `docharvest` (its tool),
`passthroughArgs`, `rootPath`, `version`, `fmt` (over `dev/canary/fmt/`, a
fixture with no enum casts, since 0.17.1's `zig fmt --check` fails every
`@intFromEnum`/`@enumFromInt` that 0.16 needs, §5.8: it checks the
helper's 0.17 branch, not zigkit's sources), `isDebug`, and
`addOptionFile` and `addOptionDir` with one file and one directory option
read by a program it runs, so 0.17.1 runs the directory branch. It also
compiles the two modules that exist because of 0.17, which nothing above
reaches: Zig compiles a function only when something calls it, the unit
test it runs through `addTest` only reads `fd.capacity` (so its root is
std's test runner, never zigkit's panic), and it runs docharvest's tool,
not the `docharvest` module. So the canary also (1) runs `addTest` over
`src/compat.zig`'s and `src/docharvest.zig`'s own unit tests, which call
`fieldNames`, `declNames`, `fieldDefault` and `enumValues` over a struct,
a union and an enum, and `is_debug` (`$H/canary/compat.zig`: 3/3 on
0.16.0 and 0.17.1; `$G/zk`'s version failed on 0.17.1, §4.6); and (2)
builds and runs a lean exe whose root takes `panic.handler` and
`panic.std_options`, logs one `std.log.warn` line through `logFn` and
then indexes out of bounds, expected to print both lines on stderr and
exit with its configured status (`$H/canary/lean.zig`, prefix "canary: ",
status 99: "canary: warning: lean log 42", "canary: internal error: index
out of bounds", exit 99, on 0.16.0 and 0.17.1, 5,408 and 5,424 bytes
ReleaseSafe stripped); and (3) runs `addTest` over a test whose root is
`test { std.testing.refAllDecls(sys); std.testing.refAllDecls(fd); }`
with the Linux backend, so 0.17.1 analyses every non-generic pub function
of the full facade and table, which no lean exe or `fd.capacity` read
reaches. `$I/z17/r.zig`: an uncalled pub function with flong's
`getdents64(fd, buf.ptr, buf.len)` passes `zig test` on 0.16.0 and fails
on 0.17.1 under `refAllDecls` ("expected type 'c_uint', found 'usize'");
an `anytype` function with the same body is not reported, so a generic
function (`openFile`'s `at`, every handle's operations) is reached only
by sys's and fd's own unit tests, which the canary also runs through
`addTest` (on 0.17.1 these are the Linux tests that need no
dependency). `$F/canary`
(zon: zigkit only; `addTest` through zigkit's runner) passed on 0.16.0 and
on 0.17.1 under `unshare -rn` with an empty cache, once `addTest` reached
the runner through `compat.zigLibPath`; before that, 0.17.1's one error
was zigkit's own `b.graph.zig_lib_directory`, which is the canary doing
its job. **`translateC`, `fdGuards`' properties and `analyze` are not
checked on 0.17** until 0.17-capable pins exist (translate-c 2.x, minish v0.5.0, a
zwanzig with a 0.17 frontend); then a second canary package pins those
and exercises them, including translateC's `@hasField(…,
"link_system_libs")` branch, which no 0.16 pin can reach.

The 0.17 move in zigkit is then: translate-c 2.x's `.link_system_libs`
(chosen by `@hasField` in `translateC` already, unchecked until that
second canary), `tools/compat.zig`'s forks, the
`addFileArg`/`addOutputFileArg` uses in `analyze` and `docharvest` once
0.17 removes the deprecated forms (still present in 0.17.1), the `sys`,
`fd` and `guards/` port of §5.8 (written for both versions beforehand, so
the canary's `refAllDecls` test is green first), 0.17's `zig fmt` over
every source, and dropping the 0.16 branches.

### 8.3 The translate-c wrapper

- It takes the **consumer's** `translate_c` dependency (1.0.0 for 0.16,
  `translate_c-1.0.0-Q_BUWo75BgBmrtWsRz9QVUZlzvsSUUe_IIXD9vmWguhX`) and
  its `Translator` type, so a consumer that translates nothing fetches
  nothing. A consumer whose build needs it at configure time declares it
  plainly. One that only translates in tests (flong) can keep it lazy
  only if **every** `b.lazyImport`/`b.lazyDependency("translate_c")` sits
  behind its `-Ddev` gate: a lazy dependency requested on an ordinary
  configure is marked needed (std/Build.zig:2093-2096) and fetched, which
  fails offline ("unable to connect to server", review2-build/lazyconf),
  and `zig build --fetch` (nixpkgs' `fetchAll = false`) never fetches it,
  so the eager FOD cannot hold it either. `$G/lazydev`, the same package
  with the request inside `if (dev)` and `addFail("needs -Ddev=true")`
  otherwise: `unshare -rn zig build install` with an empty cache exits 0,
  and the test step fails offline with "needs -Ddev=true".
- The header is text, written by `addWriteFiles` (`#define`s first).
- With `.libc_headers = .bundled` and a native target, it translates for
  the same target spelled out, so Zig's bundled libc headers are used.
  That cannot see an SDK framework: translate-c then asks
  `LibCDirs.detect` with a non-native ABI and gets only Zig's
  `any-darwin-any` headers, and Zig bundles no AudioToolbox or
  CoreGraphics. **A Darwin framework header needs `.libc_headers =
  .system`** (the native target, so the SDK's framework dirs are found),
  plus `framework_paths`/`system_framework_paths` for anything outside
  them, passed to translate-c 1.0.0's `addFrameworkPath` and
  `addSystemFrameworkPath` (Translator.zig:218-225). dev's `translate`
  step translates `CoreFoundation/CoreFoundation.h` this way on the
  macos-15 job; until that runs, it is **to verify in Phase 2**.
  translate-c with a native target trusts the host's headers, which NixOS
  has none of outside a stdenv shell ("'sys/un.h' not found", reproduced
  by size and zig017). `.system` keeps the native target for headers that
  only the host has (sudo_plugin.h, pam, pipewire), found through
  `include_paths`, `system_include_paths` or pkg-config.
- `system_libs` uses 1.0.0's `linkSystemLibrary` or 2.x's
  `.link_system_libs`, chosen by `@hasField(Translator.Options, …)`.
  **1.0.0's runs pkg-config while the build is configured**
  (Translator.zig:239-251: `Step.Compile.runPkgConfig`, and
  `std.debug.panic` on any error), where `exe.linkSystemLibrary` asks only
  when its compile step runs. So with a `system_libs` translation every
  `zig build` invocation, `lint`, `fmt` and `--help` included, needs
  pkg-config and the library's `.pc` file, or the build runner aborts
  (review4-build/pkgcfg: `zig build fmt` "panic: pkg-config failed for
  library libpipewire-0.3: PkgConfigNotInstalled", "terminated with
  signal ABRT", and the fmt step never ran). zigkit keeps the call, since
  the translation needs the library's include paths before it runs, and
  makes it the consumer's rule: whatever runs any step of a package with
  a `system_libs` translation installs the C dev packages first (capsper
  §9.3, zerocast §9.4). A consumer that would rather not can pass the
  include paths itself through `include_paths`/`system_include_paths` and
  link on its own module, which needs no pkg-config to configure.
- Every struct a consumer relies on gets a comptime `@sizeOf`/`@offsetOf`
  assert next to its translation (`$S/zigkit/dev/tc/abi.zig` checks
  `sockaddr_un` against `std.os.linux.sockaddr.un`, natively and for
  x86_64-linux-musl: 26/26 steps).
- 1.0.0 has no `#pragma pack` (2.0.0+): zerocast's SVT and capture headers
  need their asserts first. Bitfield and forward-declared structs become
  `opaque {}`, as with `@cImport`.

---

## 9. Adoption per consumer

Every Zig consumer that takes zigkit adds one zon entry, pasted from the
release notes, `zig-pkg/` to its `.gitignore` (quoin gains a
`.gitignore`), and `minimum_zig_version = "0.16.0"`:

```zig
.zigkit = .{ .url = "https://github.com/danielbodart/zigkit/archive/refs/tags/v0.N.R.tar.gz",
            .hash = "zigkit-0.0.0-…" },
```

zigkit has no dependencies of its own, so it adds one tarball to a
consumer's fetch. But the Nix package builds of flong (launcher, seccomp,
fixtures, native-lint, cross-aarch64) and quoin (`quoin_sudo.so`, the
darwin build) link **no** deps today (flong native.nix:72 `deps ? null`;
quoin nix/sudo-zig.nix:77 `plugin null`; sudo-zig-darwin.nix has no
fetch at all), and once zigkit is an eager dependency each of them needs
one: with nothing on disk an offline build fails "unable to connect to
server", and with only zigkit's tarball it builds (review1-build/offline).
Linking the existing `fetchAll = true` FOD instead would put minish,
zwanzig and translate_c on disk, and 0.16 compiles a dependency's
build.zig whenever it is on disk, `-Ddev=false` or not; it would also move
the shipped store paths (the launcher, seccomp, the `.so` quoin's server
autoUpgrades) every week a dev-only pin moved, which flong's native.nix:3
and :42 design against. So **each Nix consumer has two FODs**:

- **eager** (`zig.fetchDeps { fetchAll = false; }`: zigkit, plus
  translate_c and aro where the build translates C at configure time),
  linked into every package, lint, cross and darwin derivation;
- **dev** (the existing `fetchAll = true` one), linked only into the
  `-Ddev` checks (tests, analyze, the minish properties).

Each consumer's `scripts/zig-deps.sh` recomputes both hashes (flong's and
quoin's refuse today unless there is exactly one `zigDeps` hash; glim's
and capsper's recompute none) and skips `zigkit` when moving pins
(§5.13.1). The recompute is split from the pin move: today flong's script
exits at `git diff --quiet build.zig.zon` and quoin's at `$moved || exit
0`, both before any hash, and glim's only pin will be the skipped
zigkit, so a hand bump of zigkit or translate_c would leave every FOD
hash stale with nothing to make the new one. `zig-deps.sh --hashes`
recomputes every FOD hash from the current build.zig.zon (the fake-hash
build and write-back flong's script has) whether or not anything moved;
the weekly run calls the same code after its moves, and the hand-bump
commit runs it (§5.13.1). Phase 3's gate for each Nix consumer includes an offline
configure: `unshare -rn zig build install` with only the eager FOD
linked. The Nix filesets are explicit lists, so each derivation that
compiles a root gains the program's options file (`zigkit_options.zig`,
where it has one), and each that runs `lint` gains `zigkit-lint.zon`.

Each consumer names where its CI runs the guards (§9.1–§9.5): a `lint`
or `fmt` step nothing runs gates nothing, and the 0.17 group's promise
(§5.7) depends on it.

### 9.1 glim (first)

- **zon:** zigkit (its first dependency). **Nix:** a new FOD,
  `zig.fetchDeps { fetchAll = false; }` (eager only; glim has no dev
  dependency, so it needs no second one), linked into the package;
  `zig_0_16`. scripts/zig-deps.sh learns to recompute that hash and to
  skip `zigkit` (§9, §5.13.1).
- **build.zig:** `root.addImport` sys, fd, panic, zigkit; the root module
  and every test module that reaches `fd` get `.single_threaded = true`
  (glim's build.zig:12-19 sets none today, and `fd`'s threads check fails
  such a build: review1-consumers/threads); tests through
  `zigkit.addTest`, which also defaults it (§5.3); `-Dversion`
  becomes `zigkit.version(b)`; `.strip = optimize != .Debug` becomes
  `!zigkit.isDebug(optimize)` (it breaks on 0.17); `zigkit.lint` (with
  `std-log` off: glim logs to the journal on purpose) as a `lint` step,
  `zigkit.fmt` as `fmt`, and `zigkit.fdGuards` hung on `test`. Both
  `addTest` and `fdGuards` take glim's options module
  (`src/zigkit_options.zig`, below): `addTest`'s `null` is the defaults,
  which have no `hidraw` kind, so a test naming `.as(.hidraw)` fails
  "enum 'fd.enumOf(…)' has no member named 'hidraw'"
  (review4-consumers/glimopts; `$I/glimopts`: with `null` that error,
  with the root module passed as its own options 3/3 steps, 1/1 test).
- **CI:** glim's CI runs `mise run check` = `./run lint` (shellcheck only)
  plus `./run test` (`zig build test`). `run.ts`'s `lint()` gains
  `zig build lint fmt`, so the lint, fmt and the 0.17 group gate trunk;
  the guards' compile-fail cases run with `test`. The Nix package
  (nix/package.nix) runs neither, so its fileset needs no `zigkit-lint.zon`,
  and it already takes all of `src/`, the options file included.
- **Options:** capacity 16; extra kinds `hidraw` and `uled` like `.file`;
  `.ioctls = &.{ hidiocsfeature, hidiocgfeature }` (§4.2). They live in
  `src/zigkit_options.zig`, made a module in build.zig and re-exported by
  main.zig, the shape §5.3 gives and `$H/guards` ran with glim-shaped
  options; passing the root module itself works for `addTest` but has not
  been run through `fdGuards`.
- **Sources:**
  - events.zig → `fd.listenSeqpacket`, `accept(.{ .nonblock = true })`
    (events.zig:56 accepts with `SOCK_NONBLOCK` today, so a follower that
    stops reading answers AGAIN and is dropped instead of blocking
    `glim run`'s one poll loop, main.zig:263-276; §6), `Fd(.seqpacket)`;
    notify.zig → `fd.unixDgram().sendTo(…, sys.SockAddr.unix("@…"))`,
    the abstract length carried by the address (§4.2); dbus.zig →
    `fd.connectUnix(path)` with `"@" ++ name` for `unix:abstract=` (its
    writer stays glim's, §5.10), and its parse test (dbus.zig:404) keeps
    its length expectation against `SockAddr.unix`; hidraw.zig and
    uleds.zig → `fd.openFile(…).as(.hidraw)` and
    the handle's `ioctl(hidiocsfeature, …)` with `hidiocsfeature =
    sys.ioc.encode(.read_write, 'H', 0x06, feature_len)`, a comptime
    constant listed in glim's options (§4.2);
    main.zig → `fd.pollEntry`, `main(init: Init.Minimal)` with
    `sys.environ(init)`.
  - **Errnos glim branches on stay values.** `try sys.unwrap(…)` keeps
    its error-union style only where glim does not look at the reason.
    Where it does, it switches on the `Result`: a stale socket's unlink
    ignores `.NOENT` (events.zig:27-30; with `unwrap` glim would refuse to
    start on a clean boot), a missing `/dev/uleds` (`.NOENT`) keeps its
    "load the uleds module" hint (main.zig:243-245), and hidraw's
    `.NODEV`/`.IO`/`.NXIO` become `error.Gone` (hidraw.zig:76-80). The
    log lines that print a reason with `@errorName` (main.zig:138, 159,
    208, 214, 221; hidraw.zig:119; notify.zig:13) take the `Result` and
    print `{t}` of its errno, so they say `NODEV`, not `Errno`.
  - std.log stays, through `panic.logFn` (in `panic.std_options`):
    `panic.handler` with prefix `"glim: "`. `usageError` (main.zig:94)
    and `probe` (main.zig:379) write with `bufPrint` through
    `fd.Stdio.err` and `fd.Stdio.out` instead of `std.debug.print`.
  - `std.Thread.sleep` (gone in 0.16) → `sys.sleep`: hidraw.zig:49's
    800 µs between feature reports and main.zig:373's 250 ms retry while
    waiting for the EC; dbus.zig:121 and :135 `std.time.milliTimestamp` →
    `sys.clockMonotonic`.
  - the 9 `[_]T{x} ** N` become `@splat`.
  - events.zig:77 (`std.fmt.bufPrint(&buf, "{d}\n", .{level}) catch
    unreachable`, which the port keeps) gains `// proven: "255\n" fits in
    4`: with `catch-unreachable` on, the first `zig build lint` after the
    port would otherwise fail (review4-consumers/glimlint ran glim's config
    over aa330c3 and reported it, beside main.zig:363 and build.zig:18,
    which the port removes).
- **lint:** every rule but `std-log` (and `enum-int`, §5.7). As for quoin
  and flong,
  the whole `zigkit-lint.zon` is run through `zigkit-lint` over the ported
  tree before the Phase 3 commit.
- **Deleted:** three sockaddr builders, `iocWR` (its test value stays as a
  check of `ioc.encode`), the `posix.close` bookkeeping.
- **Gate:** ≤ 95,952 bytes, with logging through `panic.logFn`: std's
  `defaultLog` alone would cost ~175 KB ReleaseSafe (§2), nearly twice
  glim's whole binary. glim is the first real check of §2's numbers.

### 9.2 quoin (`sudo/`)

- **zon:** zigkit; minish v0.1.0 → v0.4.0
  (`minish-0.4.0-SQtSTf4pAwD_cAeVB-_rEHWmgovGY-yaCGjX9hpVKxKH`, lazy);
  zwanzig v0.15.1 unchanged (lazy); translate_c 1.0.0 (the build needs it).
  **Nix:** two FODs (§9): an eager one (zigkit, translate_c, aro) linked
  into `package = plugin …` (sudo-zig.nix:77, which links none today) and
  into sudo-zig-darwin.nix, which gains a fetch; the existing one
  (sudo-zig.nix:87) stays for the check. scripts/zig-deps.sh (whose
  line 72 says "the plugin itself fetches nothing") recomputes both;
  `zig_0_16`. Every fileset that compiles a root gains
  `zigkit_options.zig`, and the check's explicit `files` list
  (sudo-zig.nix:85, which runs `test lint fmt analyze`, quoin's CI entry
  point for the guards) gains `zigkit-lint.zon` and loses `lint.zig`.
- **CI: a macos-15 job.** quoin's CI is ubuntu-only today (every job in
  .github/workflows/ci.yml `runs-on: ubuntu-latest`, and the nix matrix
  builds `.#checks.x86_64-linux.*` only), so the darwin check
  (`checks.aarch64-darwin.sudo-zig`, `test lint fmt` from
  sudo-zig-darwin.nix) and the darwin package are built nowhere, and the
  darwin half cannot be cross-built from Linux once it translates with
  `.libc_headers = .system`. quoin's Phase 3 adds a `darwin` job on
  macos-15 (arm64) that installs Nix and runs `nix build --no-link
  --print-build-logs .#checks.aarch64-darwin.sudo-zig
  .#checks.aarch64-darwin.quoin-sudo-plugin` (flake.nix:179-183 defines
  both), and `release` needs it. It is the gate
  for every darwin item below: fd_darwin.zig's deletion, `ownHostKeys`'
  directory listing, `Record.write`'s `O_APPEND`, the darwin roots'
  panic and `fdGuards` for macOS under quoin's options. Until the job is
  green the darwin half of quoin's Phase 3 is not done.
- **build.zig:**
  ```zig
  const zigkit = @import("zigkit");
  const zk = b.dependency("zigkit", .{});
  const abi_c = zigkit.translateC(b, @import("translate_c").Translator, b.dependency("translate_c", .{}), .{
      .name = "abi_c", .target = target, .optimize = optimize, .libc_headers = .system,
      .header = "#define _GNU_SOURCE\n#undef _FORTIFY_SOURCE\n#include <unistd.h>\n#include <grp.h>\n#include <sudo_plugin.h>\n#include <security/pam_appl.h>\n",
      .include_paths = &.{ sudo_include, pam_include },   // from -D options set by Nix
  });
  ```
  (the darwin half adds pam_modules, sysctl and dlfcn). Modules `fd`,
  `sys`, `panic`, `zigkit` from `zk` for both the Linux and the darwin
  build; the options module `zigkit_options.zig`; `zigkit.lint`,
  `zigkit.analyze`, `zigkit.fdGuards`, `zigkit.addTest`; both
  `b.addFmt(.{ .paths = &.{"."} })` (build.zig:120, :380) become
  `zigkit.fmt` over quoin's own files, since on 0.16 `.` walks `zig-pkg/`
  and fails on any unformatted upstream file (review1-build/fmt);
  `Modules.get`
  over `zigkit.compat.fieldNames`. The check-plugin cases' own Run steps
  stay in quoin's build.zig, with `.{ .Exited = c.code }` (build.zig:307)
  spelt `.{ .exited = c.code }`, the same on 0.16 and 0.17 (§5.8); the
  analyze step's `.Exited = 1` (:146) goes with `zigkit.analyze`.
  The test modules that start a server thread (`t.agent`, plugin_test,
  remote_test, darwin_test) go through `zigkit.addTest` with
  `.single_threaded = false` and the options module, whose `.serialized`
  covers them (below).
  `std.os.linux.getuid()` in build.zig stays (a host call) and is noted for
  0.17.
- **Options** (`zigkit_options.zig`, re-exported by every root):
  `.fd = .{ .capacity = 4, .threads = .serialized, .extra_kinds = &.{ .{ .name = "gate", .like = .unix }, .{ .name = "agent", .like = .unix } } }`.
- **Sources:**
  - `fd.Sock` → `fd.Fd(.gate)` or `fd.Fd(.agent)`; `agent.sign` can no
    longer be handed the gate.
  - `try fd.connectUnix(p)` → `(try sys.unwrap(fd.connectUnix(p))).as(.gate)`.
  - `s.sendAll(b) catch …` → `sys.unwrap(s.sendAll(b)) catch …`, the same
    for `recvFull`; `try s.peerUid()` → `(try sys.unwrap(s.peerCred())).uid`.
  - `ids.zig:85` → `sys.unwrap(fd.connectUnix(path)) catch null`.
  - `fd.readFile(path, &buf)` (`?[]u8`) → `sys.unwrap(fd.readFile(fd.cwd, path, &buf)) catch null`.
  - **Every root gets zigkit's panic.** None of `quoin_sudo.zig`,
    `quoin-sudo-pam.zig`, `quoin-sudo-check-plugin.zig` or the darwin
    roots declares one today, so on 0.16 each gets std's default panic:
    in a libc-linked `.so` that alone is 201,720 bytes ReleaseSafe against
    4,912 with zigkit's (review2-consumers/sopanic), four times the
    56,328-byte gate. Each root, the `.so` and both darwin libraries
    included, says `pub const panic = @import("panic").handler;` and
    `pub const std_options = @import("panic").std_options;`.
  - The pam and Linux check-plugin mains take `Init.Minimal`; `std.os.argv` →
    `sys.argv(init)`; `getenv("PAM_TTY")` → `sys.getenv(init, "PAM_TTY")`.
    Both check-plugins' `fail` (quoin-sudo-check-plugin.zig:24-29,
    -darwin.zig:57-62) prints through `std.fs.File.stderr().writer`,
    which 0.16 does not have (and `Io.File.writer` takes an `io`): it
    becomes `bufPrint` into a stack buffer and `fd.Stdio.err.writeAll`.
  - **The darwin check-plugin takes the full `std.process.Init`.** It
    runs `/usr/bin/codesign` and `config.loader`
    (quoin-sudo-check-plugin-darwin.zig:47-55), and 0.16 spawns only
    through an `Io` (`std.process.spawn(io, …)`, process.zig:442); zigkit
    has no Darwin spawn. So its `main(init: std.process.Init)` spawns with
    `std.process.spawn(init.io, …)`; it links libc, so start.zig gives it
    `c_allocator`, and it is an activation-time tool with no size gate.
    Its `std.posix.fstatat` (:40, gone from 0.16's std.posix) becomes
    `sys.fstatat`, and its `std.fs.cwd().readFile` (:30) `fd.readFile`.
    Its lint takes a `raw-namespace` exemption with that reason, since the
    rule allows only `std.process.Init`.
  - The darwin plugin: `std.fs` → `fd.readFile`; `ownHostKeys`
    (quoin_sudo_darwin.zig:267-293: list `host_keys_dir`, keep
    `ssh_host_*_key.pub`, sort, read each) → `fd.openDir`, `getdents`,
    `fd.Entries`, the names sorted in a fixed buffer
    (`[std.Io.Dir.max_name_bytes]u8`, :297, for `std.fs.max_name_bytes`),
    `fd.readFile` under the dir handle, with a darwin_test.zig case run by
    the macos-15 job above;
    `std.crypto.random` → `sys.random`, `std.time.timestamp` →
    `sys.clockRealtime`, `std.c.getpid` (:332) → `sys.getpid` (§5.6).
    `Record.write`, the audit trail of every allow and refuse decision
    (:325-328: `std.fs.cwd().createFile(log_path, .{ .truncate = false,
    .mode = 0o600 })`, `seekFromEnd(0)`, `writeAll`), becomes
    `fd.openFile(fd.cwd, log_path, .{ .ACCMODE = .WRONLY, .CREAT = true,
    .APPEND = true }, 0o600)` and `writeAll` on the `file`: `O_APPEND` in
    place of the seek, which on purpose makes two sudo processes' appends
    atomic where seek-then-write could interleave. darwin_test.zig, which
    reads the log back (:274), asserts the appended line, in the macos-15
    job above.
  - **The test servers stay threads, on raw `sys`.** agent.zig:180-225
    (`TestAgent`), plugin_test.zig:76-135, remote_test.zig:107-182 and
    darwin_test.zig:151-237 serve a unix socket from a `std.Thread` while
    the test thread connects through `fd`. Their `std.net.Address.initUnix`
    and `std.posix` socket/bind/listen/accept are gone in 0.16. Moving the
    server onto `fd` would mint on a second thread, which `.serialized`
    promises away with nothing to catch it; moving it into a forked child
    would break the tests, which assert on what the server recorded in its
    own memory (agent.zig:206-207 → :261-262, plugin_test.zig:93-102 →
    :175-177, remote_test.zig:165-167 → :320-336, :360-364, :378,
    darwin_test.zig:195-196 → :304), and would not build for macOS, where
    `retainOnly` is a compile error and `sys` has no fork. So each server
    keeps its thread and makes its descriptors with raw `sys`:
    `sys.socket`/`bind`/`listen`/`accept`/`read`/`write`/`shutdown`/`close`.
    plugin_test.zig:108 and remote_test.zig:154 half-close after replying
    (`std.posix.shutdown(conn, .send)`, gone in 0.16) and drain until the
    plugin hangs up, so the plugin's `recvFull` sees EOF; a bare close with
    unread data queued would reset the connection and the plugin could see
    `ECONNRESET` instead. They call `sys.shutdown(conn, .send)` (§4.2). `sys`
    has no table, so the server thread never touches `fd`, and the
    `.serialized` promise holds for the test thread's handles. The test
    modules are multi-threaded with quoin's options (above), and
    `sys-mint` is exempted for `**/*_test.zig` and `agent.zig` (whose
    `TestAgent` lives beside the code it tests), each with that reason.
    No server forks, on either OS. `$G/testserver/server.zig` is this
    shape: a listener made with `sys` on the test thread, a `std.Thread`
    accepting and recording with `sys`, the test thread connecting through
    `fd.connectUnix(…).as(.agent)`; built multi-threaded with
    `.serialized`, it runs and the test thread reads what the server
    recorded ("sign this", reply "signed", `liveCount` 0), it
    cross-compiles for aarch64-macos, and without `.serialized` it fails
    with fd's threads message. The pam test's listener
    (quoin-sudo-pam.zig:42-48) has no thread: it binds and closes on the
    test thread, so it uses `fd.listenUnix` and `close`; so does ids.zig's
    own-ids test (:217-223, `std.posix.socket`/`bind`/`listen` and
    `std.net.Address.initUnix`), and its `std.posix.unlink`s become
    `sys.unlinkat`.
  - **Process ids through `sys`.** chain.zig:150 (`parentPid`, `std.c.getppid()`
    or `std.os.linux.getppid()`) and quoin_sudo.zig:79 (the chain walk's
    start) → `sys.getppid()`, added to `sys` on both backends (§4.2);
    the tests' `std.os.linux.getuid/getgid/geteuid` → `sys.getuid`,
    `sys.getgid`, `sys.geteuid`; args.zig:94's `std.fs.path.basename` →
    `std.Io.Dir.path.basename` (the same function, outside the
    `raw-namespace` rule; `$H/iodir` passes on 0.16.0 and 0.17.1 and the
    rule reports nothing); the tests' `std.fs.max_path_bytes` →
    `std.Io.Dir.max_path_bytes`; their `std.fs.cwd()` fixtures (gone in
    0.16) → `std.Io.Dir.cwd()` with `std.testing.io`.
  - The two `expectError(error.Sys, …)` tests expect `error.Errno`.
  - `"/" ** 200` → `&@as([200]u8, @splat('/'))`: the value goes to
    `long.cwd`, a `[]const u8` (sudomsg.zig:19, :111), and an array value
    without `&` does not coerce to a slice ("array literal requires
    address-of operator (&) to coerce to slice type", review2-zig017;
    `$G/snippets/cwd.zig` with `&` passes on 0.16.0 and 0.17.1, as does
    `const long_cwd: [200]u8 = @splat('/'); long.cwd = &long_cwd;`).
    chain.zig's 3 `@bitCast` get `// bitcast:` reasons and tests.
- **lint:** quoin's six rules as globs, each with its why, plus
  `c-import`, `options-pub`, `sys-mint` and the 0.17 group; skip
  `analyze/**`. The exemptions for fd.zig and fd_darwin.zig
  (`raw-number`, `handle-guts`) and abi.zig (`extern`, sudo/lint.zig:30-35)
  go: the first two files are deleted, and abi.zig loses its `@cImport`,
  so each would be an unused-exemption finding. `sys-mint` is exempted for
  `**/*_test.zig` and `agent.zig` (the test servers, above).
  `raw-namespace`, which quoin's lint.zig does not have today, is turned
  on, so the port is gated on its `std.posix`, `std.net`, `std.fs` and
  `std.os` uses being gone, with these exemptions (quoin's `lint` runs from
  `sudo/` over `.`, so a top-level file's name is its basename):
  quoin-sudo-check-plugin.zig ("loads the built plugin through libc's
  `dlopen`/`dlsym`/`dlerror`, which is what it checks"),
  quoin-sudo-check-plugin-darwin.zig ("spawns codesign and the loader
  through `std.process` with an Io, and dlopens the plugin"),
  quoin_sudo.zig ("PAM's conversation replies are `std.c.calloc`/`malloc`'d
  here and freed by PAM with free(3)", :290-300), plugin_test.zig ("the
  oracle runs the real sudo through `std.process.spawn` with
  `std.testing.io`", :424) and build.zig ("a build script: the host's uid,
  `std.os.linux.getuid`", which stays, below). `$H/quoinlint` runs exactly
  this over a copy of sudo/ at 42ca783 (the files Phase 3 deletes
  skipped): no unused exemption, and each of the 109 findings left is a
  use the port replaces (by count: `std.posix` socket, unlink, close,
  read, write, poll, getenv and shutdown → `sys`/`fd`; `std.fs.cwd` and
  `openDirAbsolute` → `fd` or `std.Io.Dir` with `std.testing.io`;
  `std.fs.max_path_bytes`/`max_name_bytes`/`path` → `std.Io.Dir`'s;
  `std.os.linux` ids, `std.c.getppid`/`getpid` and `std.os.argv` → `sys`).
  The config is run again through `zigkit-lint` in quoin's Phase 3
  worktree, after the port, before it is committed.
- **zwanzig:** keep the model names `connectUnix` and `close`; add `as`;
  re-pin `analyze/bugs.zig`'s line:col expectations once.
- **Deleted:** sudo/fd.zig, sudo/fd_darwin.zig, sudo/lint.zig, abi.zig's
  `@cImport` (abi.zig keeps the `SUDO_API_VERSION` arithmetic over the
  translated module).
- **Gate:** ≤ 56,328 / 30,616 / 31,656 bytes (`quoin_sudo.so`, pam,
  check-plugin), reachable only with zigkit's panic in every root (above).
  The `_FORTIFY_SOURCE` undefine for the ReleaseSafe
  getgroups failure is **to verify in Phase 2**; the fallback is
  translating with `.optimize = .Debug`.

### 9.3 capsper

- **zon:** zigkit; minish v0.1.0 → v0.4.0; translate_c 1.0.0. **Nix:** a
  new FOD, the eager one (`fetchAll = false`: zigkit, translate_c, aro):
  `-Dprop-tests=false` keeps minish out of the build steps, but only an
  FOD without minish keeps its build.zig from being compiled (§9);
  `zig_0_16`. scripts/zig-deps.sh recomputes its hash and skips `zigkit`.
- **build.zig:** 5 `@cImport`s → `zigkit.translateC` (opus, ORT, pipewire
  with `.system_libs = &.{"libpipewire-0.3"}`, macOS AudioToolbox with
  `.libc_headers = .system`, §8.3) and `@import("input")` for
  input-event-codes;
  `-Dversion` → `zigkit.version`; `if (b.args)` → `zigkit.passthroughArgs`;
  gen_config_docs → `zigkit.docharvest(b, zk, b.path("src/shared/config.zig"), .space)`;
  `zigkit.lint` with the scoped config below, `zigkit.fmt`.
- **lint config.** capsper has no lint today and is not a `sys`/`fd`
  program: 20 files name `std.os/fs/c/posix/process` (it keeps
  std.http, std.process.Child in keep_awake.zig, std.fs), 16 `.posix`
  tokens, and 62 `extern fn` in its own C shims. So its
  `zigkit-lint.zon` turns on `c-import`, `options-pub`, the 0.17 group,
  `catch-unreachable`, `extern` and `sys-mint`, and leaves
  `raw-namespace`, `posix`, the table rules, `argv`, `debug-output`,
  `std-log` and `alloc` off. `extern` is exempted for
  src/platform/linux/pipewire_c.zig, src/backend/coreml/pipeline.zig,
  src/backend/coreml/init.zig (its two `@extern`s, :31 and :49, resolve
  `capsper_coreml_load` and `capsper_coreml_release` from capsper's CoreML
  shim) and src/platform/macos/{input,audio}.zig, each with its reason
  (the C library it binds). `$H/capsperlint` runs these `extern`
  exemptions (with linux/input.zig standing in for its externs' move to
  `sys`), `c-import` and `options-pub` over capsper's `src build
  build.zig` at efc0411: no `extern` finding and no unused exemption; the
  only findings are the five `@cImport`s the port removes. 0.16 removes
  `std.posix.pipe`, `pipe2`, `open`, `socket` and `accept`, and capsper
  makes descriptors in seven files; each becomes the `sys` call, and each
  file takes a `sys-mint` exemption with its why: src/platform/linux/
  input.zig (evdev and uinput opens and their ioctls, inotify, its two `pipe2`s, :1666 and
  :1930, and the `dup` at :1934 as `fcntl(F_DUPFD_CLOEXEC)`), lid.zig (:62,
  the lid switch), audio.zig (:93) and detect.zig (:277, :416) (the
  PipeWire pipes), src/platform/macos/audio.zig (:83, CoreAudio's pipe),
  src/shared/server.zig (the PTT pipe, :422, and the HTTP listener and
  its accept, :295, :312, unless those move to `std.Io.net` with the
  server's own Io, which is not `sys-mint`'s concern) and src/shared/session.zig (a test's two pipes,
  :617, :619). Phase 3 runs the whole config through `zigkit-lint` over
  the ported tree before committing it.
- **CI:** capsper's CI runs `mise run check` = `./run lint` (shellcheck)
  plus `./run test` (`zig build test`, run.ts:183, :195). `run.ts`'s
  `lint()` gains `zig build lint fmt`, after `ensureDeps()`: the pipewire
  translation's `system_libs` runs pkg-config while configuring (§8.3), so
  every step needs pkg-config and libpipewire-0.3-dev, and mise runs
  `check`'s `lint` and `test` in parallel, where today only `test()` and
  `build()` call `ensureDeps()` (run.ts:126, :192-194; `lint()` at
  :345-347 does not). Phase 3's gate includes `./run lint` on a fresh
  ubuntu-latest runner.
- **SIGPIPE.** server.zig:615 writes each delta to the TCP connection
  with `posix.write(…) catch return error.BrokenPipe`, relying on 0.15's
  start code ignoring SIGPIPE; ported to `sys.write`, a client that
  disconnects would kill capsper. Its `main` calls `sys.ignoreSigpipe()`
  first (§5.11), and Phase 3's gate adds a test: a client that hangs up
  before the reply does not kill the server.
- **Sources:** `_ioc` → `sys.ioc.encode`; the extern `ioctl`/`inotify_*`
  and the `InotifyEvent` copy → `sys` and `sys.InotifyEvents` (in `sys`,
  not `fd`, so a threaded program needs no options and no special test
  runner, §5.3); config.zig, config_docs.zig and settings_form reflection
  → `compat`; 20 `**` → `@splat`, `"    " ** depth` (config_docs.zig:150,
  settings_form.zig:278) as `const pad_buf: [4 * depth]u8 = @splat(' ');
  const pad: []const u8 = &pad_buf;`, the length explicit and the `&`
  kept: config_docs.zig:158 passes `pad` to `writeComment(…, comptime pad:
  []const u8, …)`, and the array itself does not coerce to a slice ("array
  literal requires address-of operator (&) to coerce to slice type",
  review3-zig017/pad, both versions). `$H/pad`, that shape with
  writeComment's `pad ++ "// "` and a `{s}` print of `pad`, passes on
  0.16.0 and 0.17.1; 3 `@bitCast` (AtomicF32) get reasons and tests.
- **Not adopted:** `fd`. capsper is threaded (std.http.Server). Moving its
  input thread's descriptors onto `fd` with `.serialized` is possible later.
  Its build therefore imports `sys` alone, and its lint exempts its
  descriptor-making files from `sys-mint` (above).
- **Deleted:** build/gen_config_docs.zig, the walker half of
  src/shared/config_docs.zig.

### 9.4 zerocast

- **zon:** zigkit; minish v0.1.0 → v0.4.0; zwanzig v0.8.1 → v0.15.1;
  translate_c 1.0.0. No Nix.
- **build.zig:** 11 `@cImport` files → `zigkit.translateC` (wayland,
  wlroots and xkbcommon with `.libc_headers = .system` and pkg-config
  names; SVT, rtc, GLES, macOS capture with `.libc_headers = .system` for
  CoreGraphics, §8.3), each struct asserted; zwanzig via
  `zigkit.analyze` (its two `--do` engines go in a new `.zwanzig.json`);
  `-Dversion`, `passthroughArgs`; `build_shared.zig`'s `b.build_root.path`
  ×3 → `zigkit.rootPath`; `builtin.os.tag` → `builtin.target.os.tag`;
  `zigkit.lint` with a `zigkit-lint.zon` holding at least `c-import`,
  `options-pub` and the 0.17 group (the 11 `@cImport` files and 57 `**`
  are what it gates), and `zigkit.fmt`.
- **CI:** zerocast's `ci` task depends on `lint` and `test`
  (.mise.toml:185-191); `lint` runs `zig build analyze` and shellcheck
  (:102-108) and gains `zig build lint fmt`, and its `sources` add
  `zigkit-lint.zon`. Its wayland, wlroots and xkbcommon `system_libs` run
  pkg-config while configuring (§8.3), so `lint` (which depends only on
  `libs` today) also depends on `deps` (scripts/deps.ts, the apt/brew
  packages), as every task that runs `zig build` must.
- **SIGPIPE.** The daemon writes its reply to the CLI's connection
  (packages/client/src/shared/daemon.zig:174, :180, `posix.write(fd,
  resp) catch {}`); a CLI killed after its half-close (cli.zig:215) and
  before it reads would, on 0.16 with the write on `sys.write`, kill the
  daemon and every live session. Its `main` calls `sys.ignoreSigpipe()`
  first (§5.11); Phase 3's gate adds "a client that hangs up before the
  reply does not kill the daemon".
- **Sources:** keymap.zig's KEY_* and wayland/input.zig's BTN_* →
  `@import("input")`; `toSockaddr` ×2 → `sys.SockAddr.unix`; the control
  client's half-close before it reads the daemon's reply
  (packages/client/src/shared/cli.zig:215, `std.posix.shutdown(sock,
  .send)`, gone in 0.16) → `sys.shutdown(sock, .send)` (§4.2); 57 `**` →
  `@splat`, with `@sizeOf` asserts on each nvenc extern struct before and
  after; `builtin.mode == .Debug or .ReleaseSafe` → `std.debug.runtime_safety`.
  `sys` only, no `fd` (threaded).

### 9.5 flong (last)

- **zon:** zigkit; minish v0.1.0 → v0.4.0; zwanzig unchanged; translate_c
  1.0.0 (lazy: its three test translations). **translate_c is requested
  only under `-Ddev`.** Today the `test-libc` step's three
  `addTranslateC` blocks (scmp.h, build.zig:263-270; fcntl.h, :899-905;
  termios.h, :997-1005) are configured in every build, the last two in
  `{}` blocks beside the `if (dev)` ones; as `zigkit.translateC` over a
  lazy translate_c they would request it on every configure and an
  offline package build would fail (§8.3). So each moves inside
  `if (dev)`, and `test-libc` without `-Ddev` is
  `addFail("needs -Ddev=true")`, as `test` and `analyze` are; native.nix
  already runs `test test-libc` with `-Ddev=true` and the dev FOD (:265).
  **Nix:** two FODs (§9): a new eager one (zigkit alone, which holds
  because nothing outside `-Ddev` requests translate_c) linked into
  seccomp (native.nix:145), fixtures (:161), the launcher (:211),
  native-lint (:280) and cross-aarch64, which link none today; the
  existing `flong-0-zig-deps` (`fetchAll = true`) stays for the test sets
  (:267) and analyze (:301). Phase 3 checks `unshare -rn zig build
  install` with only the eager FOD. scripts/zig-deps.sh recomputes both
  and skips `zigkit`; `zig_0_16` at native.nix:35 and flake.nix:255.
  native-lint's fileset (:282-287, which runs `lint compile-fail fmt`,
  flong's CI entry point for the guards) gains `zigkit-lint.zon`.
  **cross-aarch64** (native.nix:304-365) drops tests/zig/abi.zig and
  abi.h from its fileset and the `-Dabi-plant` loop from its `extra`
  (:339-364); its `cross` step keeps compiling flong-seccomp for
  aarch64-linux and loses "abi's aarch64 half" (build.zig:546, :606).
  The abi check and its plants move to zigkit (§10).
- **build.zig:** `modules()` takes sys, errno and fd from `zk`; the test
  loop uses `zigkit.addTest`; the compile-fail block's 54 lines become
  `zigkit.compileFail` with flong's own five cases plus `zigkit.fdGuards`;
  the fdlint block becomes `zigkit.lint` with `zigkit-lint.zon`:
  `raw-number` exempts `src/proc.zig` and `src/sig.zig` (the syscall
  layer); `sys-mint` exempts `src/proc.zig`, `src/init.zig` ("pid 1
  before the payload: no table by design", init.zig:48-50; its
  `sys.openat2`/`sys.openat` at :287, :362, and `sys.ioctl(0,
  TIOCSCTTY)` at :479, which `sys-mint` now reports, §4.2) and `src/launch/assemble.zig`
  (its test captures stderr through a memfd at fd 2, :1065-1076), but not
  sig.zig, which makes no descriptor; `tests/**` for all but `extern`;
  `tests/zig/libc_*.zig` and `src/seccomp/scmp.zig` for `extern` (not
  tests/zig/abi.zig, which has only `extern struct`s and is deleted);
  the roots for `argv`, and no longer proc.zig, below. As specified
  before, the config failed on today's tree (review2-consumers/flonglint:
  init.zig and assemble.zig findings, two unused exemptions); this one,
  with src/fd.zig and tests/zig/lint/ skipped as their deletion will,
  exits 0 over `src tests/zig` with no finding and no unused exemption
  (`$G/flonglint/lint_migrated.zon`). The analyze
  block becomes `zigkit.analyze` with its 27 planted bugs; the three
  test-libc `addTranslateC` + `NativePaths.detect` become
  `zigkit.translateC` behind `-Ddev` (above), and the `abi` step and its
  `-Dabi-plant` option (build.zig:67, :568-604) are deleted;
  `opts.addOptionPath` becomes `zigkit.addOptionDir` for the directories
  (`corpus`, build.zig:183, :840; `records`, :933) and
  `zigkit.addOptionFile` for the binaries (`driver`, :157, :708;
  `fake_cmd`, :709, :766; `fake_bwrap`, :809), since 0.17's
  `addOptionPath` refuses a directory (§8.2);
  `declFieldDocs` becomes `zigkit.docharvest(…, .newline)`;
  `exe.stack_size = 0` stays.
- **Options:** the shipped roots need none (1024 is the default, and
  their executables are single-threaded already). Tests: none of flong's
  test modules sets `.single_threaded` today (build.zig:93-97, `modules()`
  at :1690ff and the other addTest sites), and `fd`'s threads check fails
  such a build; `zigkit.addTest` makes them single-threaded (§5.3).
  `tests/zig/launch_test.zig:416` spawns a `std.Thread` (`Swap.run`)
  while the test thread mints through `record.create`, so that one module
  sets `.single_threaded = false` and gets
  `tests/zig/launch_options.zig` with `.threads = .serialized`, the
  promise being that `Swap.run` touches no `fd` handle (it works through
  the test's own directory handle today; **to verify in Phase 2** when
  that test is ported).
- **Sources:**
  - `.bus` → `.unix` (manager.zig, 2 sites); `.record` → `.tmpfile` and
    `openRecord` → `openTmpfile` (record.zig, 3 sites).
  - `fork` and `Spawn.start` (proc.zig:127-141, :231-242) call
    `fd.clone3(.{ .cgroup = cgroup })` instead of `reserve`, `sys.clone3`,
    `cancel` and `fill`: `cloneArgs` (:97-108) is deleted, since fd builds
    the args (`CLONE_PIDFD`, `CLONE_INTO_CGROUP` from the typed cgroup,
    SIGCHLD), `error.TableFull` is the "too many open descriptors"
    refusal, `.err` the clone3 failure, and `.child`/`.parent` replace
    `pid == 0`.
  - tests/zig/bwrap_test.zig:452-458 fills the table with the public
    `fd.reserve()` and cancels six, the only check of `Spawn.start`'s
    "clone3 <argv0>: too many open descriptors" refusal and the cleanup
    after it; `reserve` is private in zigkit. That test module gets its
    own options module, `tests/zig/bwrap_options.zig`, with a small
    capacity (64), through `zigkit.addTest`, and fills the table with
    `fd.openFile(fd.cwd, "/dev/null", .{}, 0)` until `TableFull`, closing
    the last six it opened, so `bwrap.spawn`'s pipes take the six and its
    clone3 finds the table full; at the default 1024 the fill would need
    about 1018 kernel descriptors, near `RLIMIT_NOFILE`. `$I/fill`
    (capacity 16): "filled 10 of 16; after six more clone3: TableFull",
    exit 0, and strace shows no clone3 call. Whether 64 holds the rest of
    bwrap_test's tests is **to verify in Phase 2** when it is ported.
  - tests/zig/libc_scmp.zig:70-83 reads `info.params.len` and
    `r.params[0].type.?`, which 0.17.1 rejects; they become
    `compat.paramTypes(@TypeOf(f)).len` and `paramTypes(…)[0].?`
    (`typeinfo-renamed`, §5.7). guards/fd_props.zig's wait (flong
    fd_props.zig:202-203) takes §5.8's `@ptrCast(&status)` form.
  - `.zwanzig.json` replaces its `openRecord` open model with
    `openTmpfile` and adds `as` and `clone3` (from zigkit's
    `guards/zwanzig-models.json`, §7), and analyze/bugs.zig gains a planted
    double close of a `tmpfile`, so the rename cannot drop record handles
    from the analysis while the 27 planted checks stay green. proc.zig keeps its `sys-mint`
    exemption for the child's `fcntl(F_DUPFD_CLOEXEC)`, `dup2` and
    `F_SETFD` (:285-305).
  - `fd.openTree`'s callers pass `OpenTreeFlags`: mount.zig:204
    (`OPEN_TREE_CLONE | AT.EMPTY_PATH | AT_RECURSIVE` on `""`) →
    `.{ .recursive = true }`, :494 (`OPEN_TREE_CLONE`) → `.{}`.
  - tests/zig/proc_props.zig:346 adopts an inherited descriptor and uses
    it as a handle (`keepInherited(inherited)`, `inherited.raw()`,
    `inherited.close()`), so `try fd.adoptInherited(@intCast(rc))` becomes
    `try sys.unwrap(fd.adoptInherited(@intCast(rc)))` for zigkit's
    `Error!sys.Result(Fd(.inherited))` (§4.4); the test keeps its
    keepInherited argv and numbering checks.
  - `openFile`'s flags are `fd.OpenFlags` (every literal flong passes
    today fits: ACCMODE, CREAT, EXCL, NOFOLLOW, NONBLOCK, TRUNC); a flag
    held in a `sys.O` variable, if any, becomes `OpenFlags` (**to verify in
    Phase 2** by compiling).
  - Every root takes `main(init: std.process.Init.Minimal)` and passes
    `init` down; `sys.argv()` → `sys.argv(init)`, the same for
    `argvSlots`, `environ`, `environSlots` (main.zig, seccomp/main.zig,
    four fixtures, six test drivers).
  - **Spawn's environment is always named.** `Spawn.start` defaults a
    null `envp` to the process's own (`self.envp orelse
    @ptrCast(sys.environ().ptr)`, proc.zig:224), which is why fdlint
    exempts proc.zig from `argv` (tools/fdlint.zig:64); with no global
    environ that default has nothing to read. So `Spawn.init` takes the
    envp as an argument and `Spawn.envp` has no null: each `Spawn.init`
    site (15 at c72db3b) says which environment the child gets, the ones
    that set it today (pasta.zig:98, record.zig:622, cmd.zig:111,
    hook.zig:129, proc_props.zig:406) passing what they set, the rest
    `sys.environ(init)` handed down from their root, or in a test the
    runner's (`std.testing.environ`, 0.16). proc.zig no longer reads
    environ and leaves the `argv` exemptions.
  - `sys.fstat` and `sys.fstatat` stay fstat(2) and newfstatat(2) (§4.2),
    so init.zig:355 and :420 make the calls flong's DESIGN.md:287-291 and
    docs/declaration.md:159-163 name for seeding, and those lists stay
    true. A seccomp test (in the test sets, `-Ddev`) launches a seeding
    declaration whose policy denies `fstat` and expects the documented
    failure, so a later change of syscall is seen.
  - Every root: `pub const std_options = …` gains `signal_stack_size =
    null, enable_segfault_handler = false` and loses `keep_sigpipe`;
    `panic = @import("panic").Unformatted(msg.onPanic(<status>))`, each
    root keeping its own status (`failed` in main.zig:74, 1 in
    seccomp/main.zig:36, 125 in the fixtures and test drivers).
  - `@import("builtin").mode == .Debug` → `compat.is_debug`; the 22
    reflection sites → `compat`; 42 `@bitCast` get `// bitcast:` reasons
    and tests; the 49 `**` lines → `@splat` where one element repeats,
    and a comptime `repeat(s, n)` (zig017's p9-repeat helper, returning
    `*const [s.len * n:0]u8`) where more than one does, which no `@splat`
    can express: init.zig:557 `"0" ++ ",0" ** (sys.ngroups_max - 1)`
    becomes `"0" ++ repeat(",0", sys.ngroups_max - 1)` (`$F/z17/mult.zig`
    passes on 0.16.0 and 0.17.1; the `**` form fails on 0.17.1).
- **Deleted:** src/sys.zig, src/errno.zig, src/fd.zig, proc.zig's `cloneArgs`, tools/fdlint.zig,
  tests/zig/lint/, tests/zig/fd_props.zig, the ten fd/sys compile-fail
  cases, build/gen_decl_docs.zig, decl_docs.zig's `containerName`/`docFor`,
  build.zig's `Modules` struct. Kept: msg, num, proc, sig, tty and the
  rest, importing `sys`, `fd` and `errno` under the same names. flong's
  `abi` test of sys's structs moves into zigkit (x86_64 and aarch64), its
  three `std.meta.fields` loops ported to `compat` (§5.9), with its two
  planted controls (§10); so flong also deletes tests/zig/abi.zig,
  tests/zig/abi.h, the `abi` step, `-Dabi-plant`, native.nix's plant block
  in cross-aarch64 and the abi.zig `extern` exemption (all above).
- **Gate:** `.#default` ≤ 1,349,296 bytes and `.#seccomp` ≤ 154,280.
  Expect about 12 KB smaller where a root touches fd (the table leaves
  `.data`); Phase 3 measures it.

### 9.6 No zigkit dependency

- **fulcio-codesign:** one `if (b.args)` line is not worth a dependency;
  it fixes that line when it moves to 0.17. It moves to 0.16 (setup-zig
  `version: 0.16.0`).
- **browser-agent-os:** WASI, hand-rolled per binary on purpose.
- **wasiglk:** WASI; blocked on setjmp (§5.13).
- **ten-vad-ggml:** none until it has a build.zig; then `translateC` for
  ggml and fftw, minish for capsper's lost properties, and the lint.

---

## 10. zigkit's CI and test plan

`mise.toml`:

```toml
[tools]
zig = "0.16.0"

[env]
RELEASE_PATHS = "build.zig build.zig.zon build src tools guards MAJOR"

[tasks.test]
run = "zig build test --summary all && cd dev && zig build test compile-fail props --summary all"

[tasks.check]
description = "Every gate: one zig build per package"
run = "zig build test fmt && cd dev && zig build test compile-fail props lint lint-self fmt analyze translate size cross consumer gen-input-check && cd canary && zig build check"

[tasks.canary]
description = "dev/canary on Zig 0.17.1, with a cache holding no dev-dependency tarballs (informational)"
run = "cd dev/canary && ZIG_GLOBAL_CACHE_DIR=$(mktemp -d) mise x zig@0.17.1 -- zig build check"

[tasks.version]
run = '''
[ "$(git rev-parse --is-shallow-repository)" = false ] || { echo "shallow clone: the count is wrong; in Actions use fetch-depth: 0" >&2; exit 1; }
echo "$(tr -d '[:space:]' < MAJOR).$(git rev-list --count HEAD).${GITHUB_RUN_NUMBER:-$(date -u +%Y%m%d%H%M%S)}"
'''

[tasks.release-due]
description = "Print true and succeed if anything in RELEASE_PATHS changed since the last release tag (or there is none), else print false and fail"
run = '''
set -f
last=$(git describe --tags --abbrev=0 --match 'v[0-9]*' 2>/dev/null) || { echo true; exit 0; }
if [ -n "$(git diff --name-only "$last" HEAD -- $RELEASE_PATHS)" ]; then echo true; else echo false; exit 1; fi
'''
```

**Per-OS test sets.** zigkit's build.zig picks the modules a `test` step
covers by the target's OS. Linux: every module. Darwin: `sys` (the libc
backend), `fd` (its Darwin kinds), `panic`, `compat` and `input`; not
`errno`, which is a compile error there (§4.3), and no test that names a
Linux-only call or kind. `props` likewise runs `fd_props.zig` on Linux
and `fd_props_darwin.zig` on Darwin (§7). The root `zig build test` and
dev's steps use the same choice, so the darwin job runs the steps
everyone runs.

What `dev/` runs:

- `test`: the target's set of unit tests through `addTest` under three
  option sets; the lint's glob tests; the `tools/compat.zig` tests.
- `compile-fail`, `props`: §7; `compile-fail` runs both case sets, each
  case only for a target in its OS set.
- `lint`, `lint-self`, `fmt`: zigkit linted with every rule over `../src
  ../tools ../build ../guards`; the planted files. Every rule is on, so
  the 0.17 group gates zigkit's own sources from the first commit, and
  `dev/zigkit-lint.zon` names what each file is for:

  ```zig
  .{
      .rules = .{
          .{ .rule = .@"raw-namespace", .except = .{
              .{ .path = "../src/sys.zig", .why = "the syscall layer: the facade picks std.os.linux or std.c" },
              .{ .path = "../src/sys/**", .why = "the syscall layer's backends" },
              .{ .path = "../tools/**", .why = "host tools: built for the host with an Io, not sys/fd programs" },
              .{ .path = "../guards/fd_props*.zig", .why = "the properties compare the table with the kernel's own view" } } },
          .{ .rule = .posix, .except = .{ .{ .path = "../tools/lint.zig", .why = "the lint names its rule `.posix`" } } },
          .{ .rule = .@"raw-number", .except = .{
              .{ .path = "../src/fd.zig", .why = "the table: the one place a slot's number is read" },
              .{ .path = "../guards/fd_props*.zig", .why = "the properties compare the table's numbers with /proc/self/fd and F_GETFD" } } },
          .{ .rule = .@"handle-guts", .except = .{ .{ .path = "../src/fd.zig", .why = "the handle's fields are fd's own" } } },
          .{ .rule = .@"sys-mint", .except = .{ .{ .path = "../src/fd.zig", .why = "fd is where sys's descriptors enter the table" } } },
          .{ .rule = .@"std-log", .except = .{ .{ .path = "../src/panic.zig", .why = "logFn is std.log's sink: it names std.log.Level" } } },
          .{ .rule = .@"debug-output", .except = .{ .{ .path = "../tools/**", .why = "host tools print their findings" } } },
          .{ .rule = .@"typeinfo-fields", .except = .{
              .{ .path = "../src/compat.zig", .why = "the shim: .fields/.decls on 0.16, field_names/decl_names on 0.17" } } },
          .{ .rule = .@"typeinfo-renamed", .except = .{
              .{ .path = "../src/compat.zig", .why = "the shim: .params/.is_exhaustive on 0.16, param_types/mode on 0.17" } } },
          // every other rule, with no exemption: extern, export, c-import, argv,
          // catch-unreachable, alloc, options-pub, array-mult, bitcast, optimize-tag,
          // builtin-target, contains-at-least, errdefer-capture, void-init.
          // Not enum-int: 0.16 has no other spelling; 0.17's zig fmt clears it (§5.8).
      },
      // deliberate misuse; globs match the path as given on the command line (§5.7)
      .skip = .{ "../guards/compile_fail/**" },
  }
  ```

  Without the skip, every compile-fail case's `export fn` is an `export`
  finding and options_not_pub's declaration an `options-pub` one
  (review3-build/devlint: 20 findings). `bitcast` needs no exemption
  because each `@bitCast` in sys, fd and fd_props.zig carries its reason
  (§4.2). `$H/selflint` lays out zigkit's tree (`$H/zk`'s fd, sys, panic
  and zigkit, `$H/canary`'s compat, flong's sys.zig as `src/sys/linux.zig`
  with §4.2's `builtin.target` port, flong's errno.zig, `$G`'s lint as
  `tools/lint.zig`, `$S`'s `build/`, `$H/guards`' cases and flong's
  fd_props.zig) and runs this config, in `$G`-lint's rule names (its
  `debug-output` also covers `std.log`; it has no `bitcast` rule), from
  `dev/` over those four paths: exit 0, no finding and no unused
  exemption; with every rule and no exemption it reported 119 findings
  (fd.zig 72, the compile-fail cases 22, compat.zig 7, sys.zig and its
  backend 5 each, the tool 5, fd_props.zig 2, panic.zig 1).
- `analyze`: zwanzig v0.15.1 over `../src` and the planted bugs.
- `translate`: the wrapper over `sys/un.h` and
  `linux/input-event-codes.h` with ABI asserts, natively and for musl; and
  sys's hand-written structs asserted against translations of the uapi
  headers for x86_64 and aarch64 (flong's abi test, moved, its loops on
  `compat`). Its two planted controls move with it (flong abi.zig:14-15,
  :65-69, :88, :115; native.nix:339-364): `-Dabi-plant=arch` and
  `-Dabi-plant=offset` on dev's `translate` step, each run for x86_64 and
  aarch64 and each expected to fail with its message ("abi: x86_64:
  __NR_openat is 257, expected 56", "open_how.flags: offset 0, header 1"
  and their aarch64 counterparts), so a check that checks nothing is
  seen. On a macOS host: `CoreFoundation/CoreFoundation.h` with
  `.libc_headers = .system` (§8.3) instead.
- `size`: §7's gate, the `.so` with zigkit's panic and its default-panic
  control included. Bytes are counted; nothing is timed.
- `cross`: the Linux set compiled for aarch64-linux, and the Darwin set
  (no `errno`, no Linux-only kind or call) for aarch64-macos and
  x86_64-macos, the Darwin compile-fail cases included (a compile-fail
  case only compiles, so a Linux host checks the Darwin set's messages).
- `consumer`: `dev/consumer/`, a package depending on `../..` by path,
  uses every helper (addTest, compileFail, fdGuards, lint, analyze,
  translateC, passthroughArgs, rootPath, version, addOptionFile,
  addOptionDir), so a helper change
  breaks zigkit's CI before any consumer's. It runs `fdGuards` three times,
  under the default options, glim-shaped options (capacity 16, extra
  kinds `hidraw` and `uled`, no `gate`) and quoin-shaped options (capacity
  4, `.serialized`, `gate` and `agent`), so a guard case that only fails
  under some options is found here, not in a consumer's Phase 3 gate. Its
  zon declares minish, zwanzig and translate_c, so it is 0.16 only.
- `dev/canary` (its own package; `check` above runs it on 0.16.0 too, so
  it does not rot): zigkit by path and no other dependency, and the
  helpers that need none (§8.2), so 0.17.1 can configure it; plus the
  `compat` and `docharvest` modules' unit tests, the lean exe on
  `panic.handler` and `panic.std_options` (§8.2), the two modules that
  exist because of 0.17, and the `refAllDecls` test over `sys` and `fd`
  with their own Linux unit tests, so the full facade is analysed on
  0.17.1 (§8.2).
- `gen-input-check`: regenerate `src/input.zig`; fail on a difference.

`.github/workflows/ci.yml` is gokit's current one with `branches: [trunk]`
and `defaults.run.shell: bash`:

- `check`: ubuntu-latest, `actions/checkout@v7`, `jdx/mise-action@v5`,
  `actions/cache@v6` on `~/.cache/zig` keyed on
  `hashFiles('**/build.zig.zon', 'mise.toml')` (every nested zon), then
  `mise run check`.
- `darwin`: macos-15, `zig build test` and
  `cd dev && zig build test compile-fail props translate`: the Darwin sets
  above (the compile-fail cases for Darwin, natively, as quoin's own
  macos-15 job, added in its Phase 3, runs `fdGuards`, §9.2), the
  libc backend actually run, `getdents` over a real directory, `peerCred`,
  `accept`'s `O_NONBLOCK` from a non-blocking listener (§5.4),
  `ignoreSigpipe`, and a framework translation.
- `canary`: `continue-on-error: true`, no `actions/cache` (its cache is
  a fresh `mktemp -d`), `mise run canary`.
- `due`: fetch-depth 0; `if [ "$FORCE" = true ] || mise run -q release-due; then echo release=true >> "$GITHUB_OUTPUT"; fi`.
- `release`: needs check, darwin and due; push or dispatch on trunk;
  `contents: write`; tag and notes as §5.12;
  `gh release create "$TAG" --verify-tag --title "$TAG" --latest --notes-file notes.md`.

The README follows gokit's: one paragraph naming who it serves, the
install line, `## Modules` with one paragraph per module opening with a
bold link, `## Build helpers`, `## Small binaries`, `## Rules`,
`## Channels`, `## Versions`, `## License` (Apache License 2.0, gokit's
LICENSE byte for byte).

---

## 11. Risks

1. **Test-runner wrapper.** It relies on std's `lib/compiler/test_runner.zig`
   exporting `main` and `std_options` (true on 0.16.0 and 0.17.1). If
   that changes, `build/test_runner.zig` changes and no consumer does.
2. **The consumer owns minish, zwanzig and translate-c pins**, so the 0.17
   move touches each consumer's zon, not only zigkit. Each consumer's
   weekly update moves minish and zwanzig, picking tags whose
   `minimum_zig_version` matches its Zig; translate_c (a codeberg `?ref=`
   pin) and zigkit are moved by hand (§5.8, §5.13.1).
3. **translate-c with an explicit triple** uses Zig's bundled glibc
   headers, which may differ from the linked glibc's version; use
   `.system` where the C library itself is linked. quoin's sudo and pam
   headers then come through include paths from Nix, prototyped only for
   bundled headers so far.
4. **0.17 build branches are not gated** (canary only), and the canary
   cannot reach `translateC`, `fdGuards`' properties or `analyze` until
   their dependencies have 0.17 tags (§8.2). zigkit is 0.16-only by PLAN
   §2.2.
5. **zwanzig has no 0.17 frontend**; the 0.17 move waits for it (PLAN §5).
6. **Compile-fail wording** belongs to the compiler for type mismatches.
   zig017 found four messages identical on 0.17.1; the rest are zigkit's
   own `@compileError` text.
7. **Darwin's non-atomic socket and pipe CLOEXEC**, documented; unreachable
   in quoin's single-threaded plugin.
8. **`.serialized` is a promise, not a check.** A program that breaks it
   can race a close against a `raw()`; a lock would not fix that either
   (a `raw()` then `read()` is not atomic against another thread's close).
9. **Extra kinds belong to the program.** Two libraries in one program that
   both want a kind named `gate` collide; the root owns the names.
10. **Capacity-specific tests must go through `addTest`**; a plain
    `b.addTest` of an `fd` user fails to compile, which is the point, but
    each consumer's build.zig must be switched in its Phase 3 commit.
11. **Every Nix consumer gains an FOD**: glim and capsper one each, flong
    and quoin a second, eager-only one (§9). Each hash moves whenever a
    pin in it moves, a hand-moved zigkit or translate_c included; each
    `zig-deps.sh` must learn to recompute all of them on demand
    (`--hashes`, §9; glim's and capsper's recompute none today), or the
    first pin move breaks the package build.
12. **minish 0.4 shrinking** did not shrink a failing input in size's
    negative control. Properties must not depend on shrinking; seeds are
    fixed and printed.
13. **`sys` forces close-on-exec.** Code that called flong's `sys.openat`
    without `O_CLOEXEC` and relied on an exec'd child inheriting the
    result changes behaviour; flong's own Spawn passes descriptors by
    clearing `FD_CLOEXEC` in the child (proc.zig's `F_SETFD 0`) and
    placing them with `dup2`, which stay as they are. Phase 3 greps each
    consumer's `sys.` mints before the switch.
14. **A pidfd below 3 that cannot be moved panics** in `clone3`
    (§4.4), with the child already running. It needs a process started
    with 0–2 closed and at `RLIMIT_NOFILE` at once.
15. **A listed ioctl is the program's word.** `zigkit_options.fd.ioctls`
    takes request numbers on trust; a program that lists one returning a
    descriptor (a DRM or KVM request) leaks it. The list is short and in
    the options, where review sees it, and `sys.ioctl` is in `sys-mint`.
16. **`system_libs` needs the C dev packages to configure** (§8.3), for
    every step of capsper and zerocast; a task runner that forgets
    `ensureDeps` aborts the build runner, not the step.

## 12. Rejected

- **`Fd(Kinds, k)` with a program enum**, and **kind specs with an `Op`
  list**: §5.2.
- **An untyped `zigkit_options`** (simple): a misspelt key without the
  `fd_` prefix fell back to the default silently.
- **Options through `b.addOptions` or `b.dependency` arguments**: puts
  program policy in build.zig, cannot carry kinds, and contradicts PLAN
  §2.4.
- **Declaring minish, zwanzig or translate-c in zigkit's published zon,
  even lazily** (zig017, safety, simple): they land in every consumer's
  Nix FOD (§1.2).
- **A public `adopt(raw)`** (size): the one way a descriptor without
  CLOEXEC would enter the table. `adoptInherited` is the only adopt of a
  number, and it checks the number (§4.4).
- **A public `Reservation.fill(raw)`** (flong): a second unchecked adopt
  of any number. `fd.clone3` makes the call and fills the slot from its
  own variable (§4.4).
- **`fd.clone3(args: sys.CloneArgs)`** (`$H`): the caller's flags and
  cgroup number passed through, so `CLONE_FILES` let a child's
  `retainOnly` close the parent's descriptors behind its table (§4.4).
- **A deny-list in `sys.ioctl`** (`$H`): it refused five requests, and
  KVM_CREATE_VM, SYNC_IOC_MERGE and others still returned a descriptor
  through it (§4.2).
- **`sys.fstat` through statx** (the first port): a different syscall under
  flong init's seccomp filter, against what flong documents (§4.2).
- **Passing glim's root module as its options to every helper**: it works
  for `addTest` (`$I/glimopts`), but `fdGuards` was not run that way, and
  §5.3's options module was (§9.1).
- **`openTree(…, flags: u32)`** (flong): a caller's flags without
  `OPEN_TREE_CLONE` made the attached mount a `tree`, which `moveTo`
  moved (§4.4).
- **A required prologue that opens /dev/null over a closed 0–2**: a
  program that forgets it gets Stdio writing into its own files; moving
  a low number at the mint costs 224 bytes and cannot be forgotten.
- **A u64 generation**: never wraps, but doubles the handle to 16 bytes
  and the slot to 16; retiring a spent slot keeps both.
- **`sys.Raw` as an enum** (safety): it would make `raw()` unusable as a
  pid by type, at the cost of churn through flong's whole syscall layer;
  the `raw-number` rule already confines the number.
- **No `unwrap` at all** (safety): quoin's and glim's try chains would
  become switches.
- **A global Io for libraries** (zig017): 276–283 KB and 256 KiB of TLS
  per thread (§2).
- **`std.debug.FullPanic`** for zigkit's handler: twice the ReleaseSafe
  size of the quoin-shaped client.
- **`b.addTranslateC`** (simple): PLAN §2.8 names the translate-c package,
  and 0.17 deprecates the step.
- **Hand-built `zig build-obj -M…` compile-fail lines** (flong today).
- **A lint exemption callback compiled into the tool**, and
  **basename-only exemptions** (quoin): any `fd.zig` anywhere was exempt.
- **`dbus` in v0.1**, **`macos/cf`**, **`cast`**: §5.10.
- **Default capacity 64** (safety): no size benefit, since capacity is BSS.
- **`vMAJOR.N` tags and an always-exit-0 `release-due`** (size, safety,
  the inventory's proposal): gokit no longer does either.
- **A lock around the table**: §11.8.

---

## 13. Descriptor layer

This section is the descriptor layer's own documentation: what `fd`
promises and why, as flong's docs and fd.zig's comments say it.

### Handles, not numbers

Every descriptor a program opens is in one table. A handle is a slot and
a generation, never a descriptor number. Closing a handle bumps its
slot's generation, so every copy of it (in a struct, a keep list, a forked
child's globals) is stale from then on, and `raw()` on a stale copy panics
instead of reaching whatever file reused the number. A generation never
wraps back to an old one: a slot whose generation is spent is retired,
never used again. The kind is part of
the handle's type: an operation a kind does not have is a compile error,
and a handle of one kind is not another's. A read on a directory, a `dir`
where a cgroup is wanted, or a network namespace handed to a
mount-namespace `setns` does not compile.

The table is a process global with no lock, and a fixed array, so a
forked child can use it without allocating. It is zero-initialised, so it
costs BSS, not file bytes, and only in a program that opens through it.
Every descriptor `fd` mints is close-on-exec, and so is every one `sys`
makes, but for `dup2` and `fcntl(F_DUPFD)`, which a spawn's child uses to
place what it inherits. An ioctl's descriptor is close-on-exec only when
the kernel makes it so for that request: `openNs` makes its own from the
namespace requests, which the kernel makes close-on-exec; a handle's
`ioctl` takes only the terminal's requests and the ones the program lists
in its options, none of which may return a descriptor; and `sys.ioctl`,
which passes any request, is a minting call to the lint. The lint keeps
`sys`'s minting calls inside `fd` and the program's syscall layer
(`sys-mint`), so a descriptor outside the table is one that layer chose
to make, or one a request the program listed made. No slot ever holds 0, 1 or 2:
a program started with one of them closed gets its number moved above 2
at the open. Nothing outside the syscall layer touches a
descriptor's number but through `raw()`, which the lint confines
(`raw-number`).

An open returns `Error!sys.Result(Fd(k))`: the kernel's errno as a value,
or `error.TableFull` when every slot is taken, with the descriptor the
kernel gave already closed, so no caller holds a descriptor the table does
not know. A full table at the default capacity of 1024 is where the
kernel's `EMFILE` would be at the default soft limit. An open that fails
leaves the table as it was.

### The handle types

- `Fd(k)` is an owned handle: `close` once, then every copy is stale.
  `closeChecked` is `close` that answers close(2)'s errno; the handle is
  stale whatever it answers.
- `Held(k)`, from `h.holdUntilExit()`, is the same descriptor with no
  `close`, for what lives until the process exits. A fork child that does
  not keep it drops it, as any other.
- `AnyFd` is a handle of any kind, as a keep list holds it. It carries the
  kind, so a forged or mismatched one panics "descriptor handle of the
  wrong kind" rather than reaching a descriptor.
- `Stdio` is descriptors 0–2, outside the table: read and write, the
  terminal calls, never closed, never made non-blocking. It has no
  `close` (a compile error). The table never holds those numbers, so a
  `Stdio` write reaches what the caller gave the process at 0–2, or
  answers `EBADF` where the caller closed one, never a file the program
  opened.
- `cwd` is the working directory (AT_FDCWD), where a directory handle is
  taken. The `at` of an open is a directory handle (owned or held), `cwd`,
  or an O_PATH handle, a detached tree or a cgroup, which name a directory
  as well. A file there is a compile error.
- `h.as(.k)` turns a handle into the program's own kind `k`, which is
  `like` its kind: the slot's kind changes and its generation moves on.

### How a number leaves the table

Only in these ways, each named in code: a child's argument (`passFd` in a
program's spawn), `/proc/self/fd/N` (`selfPath`), `/proc/<pid>/fd/N`
(`pidPath`, for naming a descriptor to another process by this one's
pid), a filesystem context's `setFd`, a pollfd array (`pollEntry`), a
test's `snapshot` of every live number (which `raw-number` reports as it
reports `.raw`), and a program's own syscall-layer file that the lint
exempts from `raw-number`.

### Kinds worth explaining

- `path` is O_PATH: a place, not its contents. `walkOpen` opens one
  component with `openat2` and `RESOLVE_NO_SYMLINKS | RESOLVE_NO_MAGICLINKS
  | RESOLVE_BENEATH`, so a symlink, the last component's included, is
  `ELOOP` and nothing leaves `at`. `openExact` refuses a symlink anywhere
  on the path; `openFollowing` follows symlinks as its author intended.
- `tree` is a detached mount, from `open_tree`'s clone or `fsmount`.
  `openTree` always asks for the clone (`OPEN_TREE_CLONE`), and its flags
  can say only whether the clone is recursive, so no `tree` is ever the
  attached mount itself. `moveTo` attaches it onto a `path` with both
  `MOVE_MOUNT_*_EMPTY_PATH`, so nothing is resolved by name. `fsctx` is `fsopen`'s filesystem
  context; its `setFd` takes a directory only, since `FSCONFIG_SET_FD`
  refuses an O_PATH descriptor.
- `cgroup` is `O_PATH|O_DIRECTORY|O_NOFOLLOW` of a cgroup directory, so a
  symlink as the last component is refused; it is the only kind a child is
  created in, never whatever directory is at hand.
- The namespace kinds (`userns`, `mntns`, `netns`, `cgroupns`) each name
  one namespace, and `setns(.ns)` must match: entering a network
  namespace's descriptor as a mount namespace does not compile. `openNs`
  reaches a process's namespaces through its pidfd's ioctls, so the
  process is never looked up by number.
- `tmpfile` is an unnamed file made with `O_TMPFILE|O_RDWR` in a
  directory: nobody else can see it until `linkInto` names it, by
  `linkat` of its `/proc/self/fd` path with `AT_SYMLINK_FOLLOW`, which
  needs no capability and refuses a name that exists.
- `inherited` is a descriptor the caller handed over at a number its
  input named, once checked: open (`F_GETFD`), not 0–2, and not a number
  the table already holds, each refusal an errno; a spawn passes it on
  and `close` closes it.
- `signalfd` is non-blocking: `readSiginfo` reads one queued signal, 128
  bytes, or `EAGAIN` when none is.
- `pty_master` is a new pty's master, opened non-blocking; `pty_slave` is
  its slave by path; `tty_out` is the caller's terminal opened again
  through `/proc/self/fd/1` with `O_NONBLOCK`, a new open file description,
  so fd 1's own flags do not change. A terminal the caller may not open
  (after `su`) refuses it.
- `unix` is a connected stream socket. `sendAll` sends with
  `MSG_NOSIGNAL`, so a peer that has gone is an errno, never a SIGPIPE;
  `recvFull` reads until the buffer is full or the peer closes.

### Fork, spawn and the keep list

A fork keeps descriptors whatever their flags, so a fork child keeps only
its keep list. `retainOnly(keep)` closes every descriptor from 3 up that
`keep` does not hold, in the kernel and in the table: every other handle,
in any variable, is stale from then on, the parent's signalfd among them
unless kept, so a wait in the child polls nothing that may be reused. It
makes one `close_range` per gap between kept numbers, the last up to
`~0U`, allocates nothing, and panics on a stale keep handle.
`closeUntracked()` is the other way round: it closes every descriptor from
3 up that the table does not hold, and every handle stays live; it is for
a program's prologue, closing whatever its caller left open.

A child is made with `fd.clone3(.{ .cgroup = cg })`, which takes the
pidfd's slot first, so a full table is found before the child exists,
never after, which would leak the child. It builds clone3's arguments
itself: `CLONE_PIDFD`, `CLONE_INTO_CGROUP` from a `cgroup` handle when one
is given, and the exit signal. No flag that shares the descriptor table
or the memory (`CLONE_FILES`, `CLONE_VM`, `CLONE_VFORK`, `CLONE_THREAD`,
`CLONE_SIGHAND`, `CLONE_SETTLS`) can be asked for, so a child's
`retainOnly` closes only its own descriptors and releases only its own
copy of the slots. The kernel writes the pidfd into fd's own variable,
which goes in the reserved slot, or the slot is freed when clone3 failed;
no caller hands it a number. A reserved slot is not live and holds no
number; the child drops it with every other slot it does not keep.

Because no slot holds 0–2, `close_range` from 3 and the table agree: after
`retainOnly`, the kernel's descriptors from 3 up are exactly the keep
list, whatever the caller left at 0–2. `retainOnly` and `closeUntracked`
are Linux-only; nothing on Darwin forks with a keep list.

### Reading what the kernel wrote

These parsers live in `sys` and need no table; `fd` re-exports them.

- `Entries` iterates one `getdents` buffer (`getdents64` on Linux,
  `getdirentries` on Darwin): each name, inode and type.
- `InotifyEvents` iterates one inotify read, each mask and name (up to its
  first NUL; empty when `len` is 0). The kernel writes whole events, but
  this reads any bytes without a panic: a fixed part or a name that would
  run past the end ends the iteration.
- `readAll(h, gpa)` reads a file or `Stdio.in` to its end; `readFile(at,
  path, buf)` reads the start of a small file into `buf`, opening and
  closing it through the table.

### Lint and analysis

The lint checks over Zig's tokens what the compiler cannot, so comments
and strings never match and an alias is caught where it is made: a
descriptor's `.raw` number outside the syscall layer; a handle's fields
outside `fd`; `std.os`, `std.fs`, `std.c`, `std.posix` and `std.process`
outside the syscall layer; `.posix` anywhere, since std.posix turns errnos
a caller must see into `unreachable`; a C symbol outside the file that
declares the program's C; `@cImport` anywhere; argv and environ outside
the roots; `debug.print` and `std.log`; `catch unreachable` without
`// proven: <why>` on its line; a general-purpose allocator; a `sys` call
that makes a descriptor outside `fd` and the syscall layer.

zwanzig reports a double close or a use after close through one open
model per function that makes a handle and a close model for `close`;
each minting function gets its model when it is written, and planted bugs
keep the models honest. Leaks are the table's `liveCount` and a property
that the table always equals `/proc/self/fd`.

### Tests that hold it

The table's model property (after any sequence of opens and closes, the
table, a model and `/proc/self/fd` agree, every closed handle stale and
every open one live); a stale or wrong-kind handle panicking in a child
rather than reaching a descriptor; `TableFull` at capacity leaving nothing
open behind it; every open close-on-exec; and mutations, each caught when
planted: a child ignoring its keep list, `retainOnly` without its final
`close_range`, `close` without the generation bump; the parent's table
still equal to `/proc/self/fd` after its child's `retainOnly`; the same
model run with 0–2 closed; `adoptInherited` refusing what it must; a
spent slot retired; a `tree` never the mount it was opened from; a
`clone3` pidfd waited on and close-on-exec; an accepted socket blocking
or not exactly as asked.

---

## 14. Prototype index

`$S` = `/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/synth`,
`$F` = `/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/fix1`
(`$F/zk` is `$S/zigkit` with the review fixes applied),
`$G` = `/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/fix2`
(`$G/zk` is `$F/zk` with the second review's fixes),
`$H` = `/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/fix3`
(`$H/zk` is `$G/zk` with the third review's fixes; `$H/*/bx.sh m root` is
`BX m root ../zk/src` without the target, `-mcpu` and `--stack` flags),
`$I` = `/tmp/claude-1000/-home-dan-Projects-zigkit/fa1888ab-6378-4000-bd3f-93a1d45cabf4/scratchpad/phase1/fix4`
(`$I/zk` is `$H/zk` with the fourth review's fixes; `$I/*/bx.sh` is
`$H`'s against `../zk/src`),
`Z16` = `mise exec zig@0.16.0 -- zig`, `Z17` = `/tmp/claude-1000/upstream-research/zig017/zig` (0.17.1).
`BX m root dir` = `$Z16 build-exe -O m -fstrip -fsingle-threaded --stack 0 -target x86_64-linux -mcpu=baseline --dep fd --dep sys --dep zigkit --dep panic -Mroot=root --dep sys --dep zigkit -Mfd=dir/fd.zig -Msys=dir/sys.zig --dep sys --dep zigkit -Mpanic=dir/panic.zig -Mzigkit=dir/zigkit.zig`.

| Claim | Command | Result |
|---|---|---|
| Modules through `b.addModule` with an empty zon; typed options; base + extra kinds via `@Enum` from a literal list; `as`; `OpenFlags`; threads gate; test-root gate; `zk_` fields; unwrap; guard case under consumer options; wrapper runner | `cd $S/consumer && $Z16 build compile-fail test plain-test install --summary all` | 49/49 steps; 18 compile-fail cases with their messages, incl. "as(.gate) of a file descriptor: .gate is like .unix", "no field named 'CLOEXEC' in struct 'fd.OpenFlags'", "no field named 'PATH' in struct 'fd.OpenFlags'", "'file' is zigkit's own kind; an extra kind needs a name of its own", the threads message and its `.serialized` control, "expected type 'fd.Handle(.unix,.owned)', found 'fd.Handle(.gate,.owned)'" (also as a guard case re-exporting the consumer's options module), "value of type 'sys.Result(usize)' ignored"; the test saw capacity 4, kind `.gate`, TableFull after 4; plain `b.addTest` failed "build it with zigkit.addTest" |
| zigkit's own unit tests through `addTest` with default options | `cd $S/zigkit && $Z16 build test --summary all` | 7/7 steps, 5/5 tests (incl. udp bind/sendto/recvfrom with the source address) |
| Sizes, BSS only when used, unformatted vs FullPanic | `cd $S/consumer && for m in ReleaseSmall ReleaseSafe; do $Z16 build install -Doptimize=$m -Dtarget=x86_64-linux -Dcpu=baseline -Dstack0=true --prefix out-$m; done; python3 -I ../tools/elfsize.py out-*/bin/*` | §2's table; `quoinlike` 5,240 vs `quoinlike_formatted` 10,032 ReleaseSafe; `stdio_only` .bss 0 |
| No syscall before main | `strace ./out-ReleaseSafe/bin/fd_q4` | execve, then openat("/etc/passwd", O_RDONLY\|O_CLOEXEC) |
| Stale handle, stale after `as`, TableFull | `./out-ReleaseSafe/bin/stale {stale,as,full}` | "stale: internal error: stale descriptor handle", exit 125 (both); "TableFull after 4" |
| argv slots writable, NULL-terminated | `./out-ReleaseSafe/bin/argv first second` | prints "first", exit 0 |
| Same sources on 0.17.1, panic namespace hazard | `cd $S/consumer && $Z17 build-exe -O ReleaseSafe -fstrip -fsingle-threaded --stack 0 … -Mroot=src/{quoinlike,stale,fd_q4,fd_f1024}.zig -Mzigkit=… -Msys=… -Mfd=… -Mpanic=…` | first: "'std.lang.panic' missing 'unexpectedErrorCode'"; after adding it and `loadUninstantiableType`: all build; `stale as` exit 125; sizes in §2 |
| Reflection shim on both versions | `cd $S/reflect && $Z16 test reflect.zig; $Z17 test reflect.zig` | All 1 tests passed on both |
| `--fetch=all` fetches a dependency's lazy deps | `cd $S/fetchtest/consumer && ZIG_GLOBAL_CACHE_DIR=… unshare -rn $Z16 build --fetch=all` (empty zigkit zon, then one lazy minish) | empty: succeeds, nothing fetched; lazy: "unable to connect to server" |
| Lint: ZON config, exemptions with `why`, unused exemption reported, `zig-pkg/` skipped, `zk_` exact | `cd $S/consumer && ../zigkit/zig-out/bin/zigkit-lint --config lint.zon lintcase src` | c-import 1:11, options-pub 2:7, handle-guts 3:33, raw-number 3:45; `minish.gen` and `zig-pkg/ignored.zig` not reported; three "unused-exemption" findings; exit 1 |
| translate-c wrapper with the consumer's dependency, bundled headers | `cd $S/zigkit/dev && $Z16 build translate --summary all` (and `-Dtarget=x86_64-linux-musl`) | 26/26 steps, 1/1 tests, both |
| macOS cross-compile (before the low-number move; superseded below) | `cd $S/consumer && $Z16 build install -Dtarget=aarch64-macos -Doptimize=ReleaseSmall --prefix out-macos` | builds; `$F`'s sources with the move do not (review2-safety/darwinmove) |
| Retired slots, `adoptInherited`'s refusals, `SockAddr.unix` lengths and an abstract round trip, `sys.sleep`; `addTest` makes fd's test module single-threaded | `cd $F/zk && $Z16 build test -Doptimize={Debug,ReleaseSafe} --summary all` (build.zig's fd test module sets no `.single_threaded`) | 7/7 steps, 10/10 tests, both modes |
| No slot below 3 | `cd $F && BX ReleaseSafe stdio.zig zk/src && ./stdio >&-` (review1-safety's program) | exit 0; record.txt holds only "record" (review1-safety, same program on `$S`: the status line inside it) |
| No wrap | `cd $F && BX ReleaseSafe wrap.zig zk/src && ./wrap` (review1-safety's program) | "wrap: internal error: descriptor generation spent: as() on a slot retagged 2^32 times", exit 125, 2.5 s |
| Cost of the two guards | `BX {ReleaseSmall,ReleaseSafe} fd_q4.zig` over `$S/zigkit/src`, `$F/zk/src`, and `$F/v_nolow`, `$F/v_noretire` (each guard removed) | 2,672 / 4,488 → 2,888 / 4,744; without the move 2,688 / 4,520; without retiring 2,872 / 4,712 |
| Lean log | `cd $F && $Z16 build-exe -O {ReleaseSmall,ReleaseSafe} -fstrip -fsingle-threaded --stack 0 -target x86_64-linux -mcpu=baseline --dep panic --dep zigkit -Mroot=glimlog{,_default}.zig --dep sys --dep zigkit -Mpanic=zk/src/panic.zig -Msys=zk/src/sys.zig -Mzigkit=zk/src/zigkit.zig`; also `glimlog2.zig` taking `panic.std_options` whole | 3,048 / 5,688 (logFn) vs 112,088 / 180,464 (defaultLog); prints "glim: warning(glim): hidraw /dev/hidraw2: NODEV" |
| Parsers in `sys` for a threaded program | `cd $F && $Z16 build-exe -O ReleaseSafe --dep sys -Mroot=hotplug.zig -Msys=zk/src/sys.zig && ./hotplug; $Z16 test --dep sys -Mroot=hotplug.zig -Msys=zk/src/sys.zig` | multi-threaded build, exit 0; plain `zig test` 1/1 (review1-consumers/capsper: both failed through `fd`) |
| `sys-mint` | `cd $F && $Z16 build --build-file zk/build.zig --prefix zk/zig-out && zk/zig-out/bin/zigkit-lint --config lint.zon lintcase zk/src` | "lintcase/leak.zig:4:17: sys-mint: …", exit 1; `fd.zig`'s exemption used |
| Tool forks, abi loop on compat, `repeat`, `builtin.target` | `cd $F/z17 && for f in tooltest abi mult repeat; do $Z16 test $f.zig; $Z17 test $f.zig; done` | all pass on both (tooltest: ZON config parsed, one `**` found; first try on 0.17.1 failed in `Ast.parse(…, .zig)`, hence `parseAst`) |
| Canary with zigkit alone | `cd $F/canary && ZIG_GLOBAL_CACHE_DIR=<fresh> unshare -rn $Z17 build test --summary all` (and `$Z16`) | both 3/3 steps, 1/1 test, offline, once `addTest` used `compat.zigLibPath`; before, 0.17.1's only error: "helpers.zig:139:71: no field named 'zig_lib_directory' in struct 'Build.Graph'" |
| Darwin directory reader | `cd $F && $Z16 build-exe -O ReleaseSafe -target {aarch64,x86_64}-macos -lc darwin/dirents.zig` | Mach-O built for both; not run here |
| Darwin mints after the move, with libc branches | `cd $G/darwin && for f in open_mac adopt_mac kinds_mac; do for t in aarch64-macos x86_64-macos; do $Z16 build-exe -O ReleaseSafe -fsingle-threaded -target $t -lc --dep fd --dep zigkit --dep panic --dep sys -Mroot=$f.zig --dep sys --dep zigkit -Mfd=../zk/src/fd.zig -Msys=../zk/src/sys.zig --dep sys --dep zigkit -Mpanic=../zk/src/panic.zig -Mzigkit=../zk/src/zigkit.zig; done; done` | all six exit 0 (Mach-O); not run here. The same open_mac.zig against `$F/zk/src`: "expected type 'c.darwin.E', found 'os.linux.E…'" |
| quoin's test servers as threads on raw `sys` | `cd $G/testserver && $Z16 build-exe -O ReleaseSafe -lc --dep fd --dep sys --dep zigkit --dep panic -Mroot=server.zig --dep sys --dep zigkit -Mfd=../zk/src/fd.zig -Msys=../zk/src/sys.zig --dep sys --dep zigkit -Mpanic=../zk/src/panic.zig -Mzigkit=../zk/src/zigkit.zig && ./root`; again with `-target aarch64-macos`; again from `server_single.zig` (no `.serialized`) | "server recorded "sign this"; reply "signed"; live 0", exit 0; macOS: Mach-O built; control: "fd.zig:52:9: error: zigkit fd: this build is not single-threaded …" |
| `accept(.{ .nonblock })` | `cd $G/accept && BX ReleaseSafe seqnb.zig ../zk/src && ./seqnb nonblock; timeout 3 ./seqnb block`; `BX ReleaseSafe wrongkind.zig ../zk/src` | "send 278 -> AGAIN", exit 0; exit 124 (still blocked); "fd.zig:262:21: error: accept on a file descriptor" |
| `addOptionFile`/`addOptionDir`; `../src/compat.zig` from build/helpers.zig | `cd $G/optpath/consumer && ZIG_GLOBAL_CACHE_DIR=<fresh> $Z16 build run`; the same with `$Z17` | both exit 0, printing the corpus directory, the driver binary and `zigkit.compat.fieldNames`' "sys,fd" (0.16 corpus absolute, 0.17.1 `corpus`) |
| translate_c requested only under `-Ddev` | `cd $G/lazydev && unshare -rn env ZIG_GLOBAL_CACHE_DIR=<empty> $Z16 build install`; then `… build abi` | install exit 0; abi: "error: needs -Ddev=true" (review2-build/lazyconf, requested unguarded: "unable to connect to server") |
| `Unformatted(msg.onPanic(status))`; `&@as([200]u8, @splat('/'))` | `cd $G/snippets && $Z16 build-exe -O ReleaseSafe -fsingle-threaded --name onpanic --dep panic --dep zigkit -Mroot=onpanic.zig --dep sys --dep zigkit -Mpanic=../zk/src/panic.zig -Msys=../zk/src/sys.zig -Mzigkit=../zk/src/zigkit.zig && ./onpanic`; `$Z16 test cwd.zig`; both again with `$Z17` | "index out of bounds", exit 125, on both; 2/2 tests on both |
| Lint: `typeinfo-fields` through captures and aliases, `builtin-target` through `@import("builtin")`, `array-mult` not on `**T` | `cd $G/lint && $Z16 build-exe -O ReleaseSafe ../zk/tools/lint.zig --name zigkit-lint && ./zigkit-lint --config lint.zon case`; the same binary over flong's decl_docs/check/decl/fd.zig and capsper's config_docs/keep_awake/nemo_mel_state.zig with `--as` | bad.zig: 9 findings (5:43, 6:31, 10:14 typeinfo-fields; 12:20, 13:16, 14:40 array-mult; 15:30, 16:14 builtin-target; 17:20 std.meta.fields), good.zig: none, exit 1; every consumer site §5.7 names is reported |
| `isArrayMult` with the operand check, both versions | `cd $G/z17 && $Z16 test t.zig; $Z17 test t.zig` | 2/2 on both: no finding in five `**T` types or `2 * *q`, five in the five `**` products |
| flong's lint config | `cd ~/Projects/flong && $G/lint/zigkit-lint --config $G/flonglint/lint_migrated.zon src tests/zig` | exit 0, no finding, no unused exemption (src/fd.zig and tests/zig/lint/ skipped, as their deletion will) |
| `openTree` always clones; its flags cannot ask for the attached mount | `cd $H/opentree && ./bx.sh ReleaseSafe ot.zig && unshare -rm ./root`; `$Z16 build-obj -fsingle-threaded --dep fd --dep sys --dep zigkit -Mroot=cf_clone.zig --dep sys --dep zigkit -Mfd=../zk/src/fd.zig -Msys=../zk/src/sys.zig -Mzigkit=../zk/src/zigkit.zig` (and `cf_raw.zig`) | "mountId source 1033, tree 1035, differ true", "after moveTo: a/marker SUCCESS, b/marker SUCCESS", exit 0; "no field named 'clone' in struct 'fd.OpenTreeFlags'"; "expected type 'fd.OpenTreeFlags', found 'comptime_int'" (review3-safety/opentree, the u32 form: a/marker NOENT) |
| `fd.clone3`: the pidfd from fd's own variable; reservation private | `cd $H/clone3 && ./bx.sh ReleaseSafe c3.zig && strace -f -e trace=clone3,waitid ./root`; `strace -e trace=clone3 ./root \| grep -c 'clone3('`; `$Z16 build-obj … -Mroot=cf_fill.zig …` | "child … exited 7; pidfd FD_CLOEXEC=1; live 1", "full table: TableFull, no child", exit 0; one clone3 call; "'reserve' is not marked 'pub'" |
| `sys.shutdown` and the unix kind's `shutdown` | `cd $H/shutdown && ./bx.sh ReleaseSafe sd.zig && ./root`; again with `-target {aarch64,x86_64}-macos -lc`; `$Z16 build-obj … -Mroot=cf_sd.zig …` | "server read "req"; client recvFull "reply" then EOF; server read after shutdown "bye"", exit 0; two Mach-O files (`cf fa ed fe`); "shutdown on a file descriptor" |
| `compat`'s API and tests on both versions | `cd $H/canary && for z in $Z16 $Z17; do for m in Debug ReleaseSafe; do $z test -O $m compat.zig; done; done` | 3/3 in all four (struct, union, enum; pub decls only; `is_debug` without a tag) |
| Lean exe: `panic.handler`, `panic.std_options`, `logFn` on both versions | `cd $H/canary && $z build-exe -O ReleaseSafe -fstrip -fsingle-threaded --dep panic --dep zigkit -Mroot=lean.zig --dep sys --dep zigkit -Mpanic=../zk/src/panic.zig -Msys=../zk/src/sys.zig -Mzigkit=../zk/src/zigkit.zig && ./root` for `$Z16` and `$Z17` | both print "canary: warning: lean log 42" and "canary: internal error: index out of bounds", exit 99; 5,408 / 5,424 bytes |
| capsper's `pad` with `&` | `cd $H/pad && $Z16 test pad.zig; $Z17 test pad.zig` | 1/1 on both |
| `expect_term = .{ .exited = n }` | `cd $H/exited && ZIG_GLOBAL_CACHE_DIR=<fresh> $Z16 build t` (and `$Z17`) | exit 0 on both |
| Guard sets and OS selection under three option sets | `cd $H/guards && for o in default glim quoin; do for t in x86_64-linux aarch64-macos x86_64-macos; do $Z16 build guards -Dopts=$o -Dtarget=$t --summary all; done; done`; `$Z16 build dev -Dtarget=$t` for each `$t` | consumer set: 12/12 steps (Linux), 11/11 (each macOS), under each option set; zigkit set: 10/10 for each target |
| dev's lint config over a zigkit-shaped tree | `cd $H/selflint/dev && $G/lint/zigkit-lint --config zigkit-lint.zon ../src ../tools ../build ../guards`; the same with `all.zon` (every rule, no exemption) | exit 0, no finding, no unused exemption; all.zon: 119 findings, exit 1 |
| quoin's `raw-namespace` exemptions | `cd $H/../review3-consumers/lint/quoin/sudo && $G/lint/zigkit-lint --config $H/quoinlint/quoin.zon .` | no unused exemption; 109 findings, every one a `std.posix`, `std.fs`, `std.os` or `std.c` use §9.2 ports |
| capsper's `extern` exemptions | `cd ~/Projects/capsper && $G/lint/zigkit-lint --config $H/capsperlint/extern.zon src build build.zig` | no `extern` finding, no unused exemption; the five `@cImport`s only |
| `std.Io.Dir.path`/`max_name_bytes` outside `raw-namespace` | `cd $H/iodir && $Z16 test p.zig; $Z17 test p.zig; $G/lint/zigkit-lint --config lint.zon p.zig` | 1/1 on both; lint exit 0 |
| `fd.clone3(CloneOptions)`: the parent's table survives the child's `retainOnly`; typed cgroup; no `flags` | `cd $I/clone3 && ./bx.sh ReleaseSafe c3.zig && ./root && strace -f -e trace=clone3 ./root`; `$Z16 build-obj -fsingle-threaded --dep fd --dep sys --dep zigkit -Mroot=cf_flags.zig --dep sys --dep zigkit -Mfd=../zk/src/fd.zig -Msys=../zk/src/sys.zig -Mzigkit=../zk/src/zigkit.zig` (and `cf_dir.zig`) | "child exited 0; secret live=true raw=3; other raw=4; kernel from 3: { 3, 4 }; live 2", "read through 'secret': dan-desk", "into own cgroup …: child exited 4", exit 0; strace `CLONE_PIDFD` then `CLONE_PIDFD\|CLONE_INTO_CGROUP … cgroup=5`; "no field named 'flags' in struct 'fd.CloneOptions'"; "expected type '?fd.Handle(.cgroup,.owned)', found 'fd.Handle(.dir,.owned)'" (review4-safety/clonefiles, `$H`'s form: "other raw=3") |
| ioctl allow-list on handles | `cd $I/ioctl && ./bx.sh ReleaseSafe ok.zig && ./root`; `$Z16 build-obj … -Mroot=cf_kvm.zig …` (and `cf_peer.zig`) | "HIDIOCSFEATURE on /dev/null: NOTTY; TIOCGWINSZ: NOTTY", exit 0; both cases "ioctl 0xae01 (0x5441) is neither zigkit's (the terminal's) nor in zigkit_options.fd.ioctls; …" (review4-safety/ioctl, the deny-list: KVM_CREATE_VM SUCCESS, descriptors 4 -> 6) |
| Darwin `accept` sets `O_NONBLOCK` exactly | `cd $I/accept && ./bx.sh ReleaseSafe acc.zig && ./root`; again with `-target {aarch64,x86_64}-macos -lc` | "accepted O_NONBLOCK=false", exit 0; two Mach-O files (`cf fa ed fe`); not run on a Mac |
| `sys.ignoreSigpipe` | `cd $I/sigpipe && ./bx.sh ReleaseSafe sp.zig && ./root; ./root ignore`; again with `-target {aarch64,x86_64}-macos -lc` | exit 141 without; "write: PIPE", exit 0 with; two Mach-O files (review4-consumers/sigpipe: 0.15.2 PIPE, 0.16.0 141) |
| `getdents`/`waitpid` written for both; `refAllDecls` reaches an uncalled pub fn on 0.17.1 | `cd $I/z17 && for z in $Z16 $Z17; do $z build-exe g.zig && ./g; $z build-exe w.zig && ./w; $z test r.zig; done` | g exit 0 and w exit 3 on both; r.zig passes on 0.16.0, and on 0.17.1 "r.zig:5:45: error: expected type 'c_uint', found 'usize'" for the non-generic function only (review4-zig017/getdents: the uncast forms fail on 0.17.1) |
| `compat.paramTypes`, `compat.isExhaustive` | `cd $I/compat && for z in $Z16 $Z17; do for m in Debug ReleaseSafe; do $z test -O $m compat.zig; done; done` | 4/4 in all four (review4-zig017/params: `.params` "no field named 'params' in struct 'lang.Type.Fn'" on 0.17.1) |
| `zig fmt` across versions | `cd $I/fmt && for z in $Z16 $Z17; do $z fmt --check fixture.zig; $z fmt --check fd_copy.zig; done` | fixture exit 0 on both; `$H`'s fd.zig (4 enum casts) exit 0 on 0.16.0, 1 on 0.17.1 |
| Lint: `ioctl` in `sys-mint`, `fd.snapshot` in `raw-number`, `typeinfo-renamed`, `enum-int` | `cd $I/lint && $Z16 build-exe -O ReleaseSafe ../zk/tools/lint.zig --name zigkit-lint && ./zigkit-lint --config lint.zon case`; `./zigkit-lint --config lint.zon --as libc_scmp.zig ~/Projects/flong/tests/zig/libc_scmp.zig`; the same config over each consumer | bad.zig 5:31 and 8:33 typeinfo-renamed, 11:13 sys-mint, 13:13 raw-number, 14:19 enum-int; good.zig none; exit 1; libc_scmp :71, :79, :80, :83; enum-int files: flong 15, zerocast 4, quoin, capsper, glim 2 each |
| fstat(2)/newfstatat(2) kept, `Stat` per arch | `cd $I/fstat && for z in $Z16 $Z17; do $z build-exe -O ReleaseSafe run.zig && strace -e trace=fstat,newfstatat,statx ./run; $z build-exe -O ReleaseSafe -target aarch64-linux run.zig; done; for t in x86_64-linux aarch64-linux; do mise exec zig@0.15.2 -- zig build-obj -target $t abi15.zig; done` | `newfstatat(AT_FDCWD, "/etc/passwd", …)` and `fstat(3, …)`, no statx, exit 0, on both; aarch64 builds on both; abi15 builds for both arches (size and offsets equal 0.15.2's `linux.Stat`); first try on 0.17.1: "enum 'os.linux.syscalls.X64' has no member named 'fstatat64'", hence the `@hasField` |
| bwrap_test's full table without `reserve` | `cd $I/fill && ./bx.sh ReleaseSafe fill.zig && ./root; strace -f -e trace=clone3 ./root \| grep -c 'clone3('` | "filled 10 of 16; after six more clone3: TableFull", exit 0; 0 clone3 calls |
| glim's options through `addTest` | `cd $I/glimopts/app && $Z16 build test -Dmode=null --summary all`; `-Dmode=self` | null: "enum 'fd.enumOf(…)' has no member named 'hidraw'"; self: 3/3 steps, 1/1 tests |
| Earlier prototypes cited | review1-* (`phase1/review1-{safety,zig017,consumers,build}`): the reproductions each §4.4, §5.7, §8.2, §9 fix answers; review2-* (`phase1/review2-{safety,zig017,consumers,build}`): ioctlfd, darwinmove, optpath, compatpath, ptrptr, snippets, seqblock, sopanic, flonglint, lazyconf, the reproductions the `$G` rows answer; review3-* (`phase1/review3-{safety,zig017,consumers,build}`): opentree, compat, pad, opt, selflint, glimopts, shutdown, lint, guardopts, guardos, devlint, the reproductions the `$H` rows answer; review4-* (`phase1/review4-{safety,zig017,consumers,build}`): clonefiles, ioctl, getdents, params, fmt, sigpipe, glimopts, glimlint, pkgcfg, the reproductions the `$I` rows answer; size (`phase1/size`): `.so` sizes, raw=-1 `.data`, zwanzig on planted udp bugs; zig017 (`phase1/zig017`): `compat.zig` and Kit on 0.17.1, test runner on both, `zig fetch` inside/outside a project, gokit convention; safety (`phase1/safety`): fork/retainOnly property and mutations, udp truncation and v6, options-missing-in-test gate, Darwin `SOCK.CLOEXEC` shim; simple (`phase1/simple`): `Result` in sys.zig for the wording, `r.err` size trap, `std.process.Init` lint exception | as recorded in each design's prototype list |
