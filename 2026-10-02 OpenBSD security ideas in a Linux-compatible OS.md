Vinix runs Linux user programs on a kernel written in V. As that compatibility
has grown, we have also been adopting security designs from OpenBSD: pledge and
unveil, immutable mappings, signal-frame cookies, stronger randomness, and checks
at the boundary between a process and the kernel.

We implemented these ideas and interfaces in Vinix, adapting them to Linux
programs. Fitting them requires work in the syscall tables, ELF loader,
filesystem, memory manager and device paths. Here are some of the changes already
in place, and what they protect.

## Let a program give up what it no longer needs

OpenBSD's [pledge](https://man.openbsd.org/pledge.2) gives a program a small
vocabulary for describing the operations it will keep using. Vinix implements
that interface over its Linux syscalls. A process that pledges `stdio rpath` can
continue basic descriptor I/O and open files for reading. Opening an ordinary
file for writing violates that promise.

Promises can only be narrowed. By default, a violation terminates the process
with an uncatchable `SIGABRT`, and the kernel reports the missing promise and
syscall number. With the `error` promise, the refusal returns `ENOSYS`
instead. Children inherit the restrictions; `execpromises` can specify the
restrictions for the next executable.

[Unveil](https://man.openbsd.org/unveil.2) supplies the path boundary. A program
selects the files or directories it needs, with read, write, execute and
create/remove permissions, then locks its view with `unveil(NULL, NULL)`.
Uncovered paths fail with `ENOENT`; an operation beyond a visible path's
permissions fails with `EACCES`. A more specific rule can hide a subtree.

For example, an image processor could unveil an input directory for reading and
an output directory for writing and creation, lock the view, then pledge
`stdio rpath wpath cpath`. Its decoding code can keep using normal file calls
while the kernel limits which paths and operations those calls can reach.
Already-open descriptors remain usable, so setup also has to account for them.

**Applications must opt into pledge and unveil.** An existing Linux binary does
not acquire this sandbox merely by running on Vinix. The regression programs
provide small syscall wrappers; the
[ABI guide](https://github.com/vlang/vinix/blob/master/docs/openbsd-security.md#calling-them)
lists the Vinix extension numbers for ARM64 and x86-64.

## Enforce the policy where Linux actually does the work

The [pledge and unveil implementation](https://github.com/vlang/vinix/commit/28ca193bda867f79b1debc40f3bae75914a990e0)
has two checking stages. At syscall entry, the kernel classifies arguments
already in registers: an ioctl request, a socket family, executable-memory
permissions, clone flags or a signal target. Filesystem operations check the
resolved target they will use. Creation checks the destination directory and
name. A symlink must not redirect an approved operation to a hidden target.

Linux libc also needs allowances that differ from OpenBSD's libc. Vinix's
`stdio` promise permits the terminal queries musl and glibc use to choose stdout
buffering. Unveil permits inspection of ancestor directories for libc's
`realpath()` and `getcwd()` fallbacks. Resolver and timezone files have specific
compatibility exceptions.

Another example is `clone3`: its flags are in user memory, where another thread
could change them after a policy check. Pledged callers receive `ENOSYS`, allowing
libc to fall back to `clone`, whose flags arrive in a register. The implementation
therefore follows OpenBSD's design while defining behavior for a different ABI.
Vinix's unveil rules currently follow absolute pathnames, and an existing rule
can be widened until the view is locked. Applications should finish configuration
and lock the view before handling untrusted input.

## Make executable memory harder to change

Vinix enforces **W^X** for user mappings by default: memory may be writable or
executable, but ordinarily cannot be both at once. Programs that generate code
can write it first and then change the mapping to executable. Compatibility
exceptions require administrator authorization; an unprivileged program cannot
grant itself an exception just by setting `VINIX_ALLOW_WX=1`.

OpenBSD's [mimmutable](https://man.openbsd.org/mimmutable.2) adds a separate
protection. Once Vinix marks a mapping immutable, later attempts to change its
protection, remove it or replace it are refused. Applications can explicitly
seal mappings after choosing their final permissions.
This preserves the mapping and its permissions; it does not make the contents
of a mapping that was sealed writable become read-only.

The kernel also had a writable alias of its own executable pages in the direct
map of physical memory. We made its text and read-only data
[read-only through that alias](https://github.com/vlang/vinix/commit/2d8e97274cbf018eb6a11357d6e22fa81c009b7f),
too. Kernel stack canaries check for overwritten stack frames, and the slab
allocator checks the poison left in freed objects before reusing them.

The latter caught a real lifetime bug during a desktop session: a shell still
held a terminal node that devtmpfs had freed. On exit, it modified the old
object's open-file count. The poison check reported a changed word at offset
136 of a 192-byte object. The [lifetime fix](https://github.com/vlang/vinix/commit/9905f7c83b5f8a95de45af927ba2ee426b2bf34c)
keeps the node until its last open description closes and a grace period passes.
The check reports corruption and then continues; it does not prevent every
use-after-free.

## Protect signal return and the kernel boundary

A signal handler returns through `rt_sigreturn`, which restores registers from
a frame on the program's stack. A forged frame can turn that restoration into
an exploit primitive. Vinix now puts a per-process cookie, bound to the frame's
address, in every frame it builds. The return path checks and clears it before
trusting the frame; a failed check terminates the process with `SIGSEGV`.

The [signal-frame change](https://github.com/vlang/vinix/commit/53f6e456b448675625e71960a99a572b1a94ce88)
fits this into the Linux layouts: a reserved field on x86-64 and a private
header on ARM64. Exec chooses a new secret; fork preserves it so the child can
return through inherited signal frames.

Checked user-memory transfers address a more basic failure. A bad pointer to
`fstat` previously could fault inside the kernel. Some device, pipe and socket
paths also received user pointers directly. The [checked-copy conversion](https://github.com/vlang/vinix/commit/e7cf86f26262be134cc1c5d32a6e4779e6e51127)
resolves user pages and copies through kernel mappings, giving resource
implementations kernel buffers.
Invalid addresses produce `EFAULT`, and failed reads are checked before consuming
a pipe's bytes or a socket's message.

That conversion enabled [SMAP on x86-64 and PAN on ARM64](https://github.com/vlang/vinix/commit/324a89d3c84394ff4c34f9a95b8daabef434105b):
hardware can fault when the kernel accidentally accesses memory through a user
address. Production builds default to strict enforcement on supported x86 CPUs
and the tested ARM64 EL1 path. Debug builds default to auditing; on physical
Apple hardware, where Vinix runs at EL2, PAN remains opt-in pending validation.

## Improve the randomness behind the mitigations

Randomized PIE, interpreter, stack and mmap placement now has company: randomized
heap-break placement and process/thread IDs. The [kernel's ChaCha20 generator](https://github.com/vlang/vinix/commit/74c1667853acd254e619ae549f6e81eb380c1f08) also
adopts the key-erasure and reseeding ideas used by OpenBSD's
[arc4random](https://man.openbsd.org/arc4random.3).

After supplying output, it replaces the key from a block callers never receive.
Periodic reseeding mixes event timings and available hardware randomness into
the state. Keys, seeds and working buffers are explicitly erased. Seed quality
still matters: these mechanisms do not manufacture trustworthy entropy from a
predictable boot environment.

Fork needs care too. A child that inherits a user-space random generator's state
can repeat its parent's output. Vinix implements OpenBSD-style [minherit](https://man.openbsd.org/minherit.2) and
Linux's `MADV_WIPEONFORK`/`MADV_DONTFORK`, letting a program zero or omit selected
ranges in the child. The application chooses which state needs that treatment.

The [network stack](https://github.com/vlang/vinix/commit/1f8a7c5afd5554c1d82f5d552b03cf7977309c26)
now randomizes ephemeral ports and adopts OpenBSD's shuffled IPv4-ID algorithm.
TCP initial sequence numbers combine a clock with a secret keyed function of
the connection's addresses and ports. Our implementation uses SipHash; OpenBSD
uses SHA-512. DNS and DHCP randomness also comes from the kernel generator.

## Seal files through familiar Linux interfaces

Immutable and append-only files use the flags behind Linux's `chattr +i` and
`chattr +a`. Tmpfs keeps them in memory; ext2 stores them in the inode's on-disk
flags. This brings OpenBSD's file-sealing idea to tools Linux applications
already understand.

Vinix exposes securelevel through `/proc/sys/kernel/securelevel`. By default,
it starts at 0. Once raised above 0, existing immutable and append-only flags
cannot be cleared, even by root. Only init can lower a positive runtime level,
subject to any configured boot floor. In the current development kernel, the
optional `vinix.securelevel` boot setting establishes a minimum that even init
cannot lower. Authentication of the bootloader, kernel and root filesystem
remains future work.

This is a subset of [OpenBSD's securelevel](https://man.openbsd.org/securelevel.7).
Vinix's level 2 currently has the same effect as level 1; OpenBSD's additional
raw-disk, clock and firewall restrictions are not yet implemented. File flags
and securelevel are administrator choices, rather than automatic sealing of
every file.

## Check behavior, and keep the compatibility limits explicit

The [OpenBSD-security regression suite](https://github.com/vlang/vinix/tree/master/tests/openbsd-security)
boots a test program as PID 1 and runs cases in children, including expected
pledge deaths, hidden paths, forged signal frames, fork-time wiping and bad user
pointers. The SMAP/PAN commit records successful runs on both architectures,
alongside ARM64 desktop, Docker and network checks.

For this post we also reran the existing host tests for the kernel RNG and
network-randomness code; both passed. They exercise production algorithms with
test clocks and hardware hooks. They do not measure physical entropy quality or
establish that every kernel path is safe.

Some OpenBSD protections need more cooperation from Linux runtimes. Syscall
instruction pinning and mandatory stack-mapping validation remain unimplemented
in the current mainline: static Linux programs and runtimes can issue their own
syscalls and use stacks allocated without `MAP_STACK`.

The useful direction is already visible: applications can give up operations
and paths, executable mappings can be sealed, and kernel boundaries can reject
bad inputs before they reach drivers. OpenBSD supplies many of the designs;
Vinix implements them while keeping the Linux programs we want to run working.
The [security overview](https://github.com/vlang/vinix/blob/master/docs/openbsd-security.md)
documents the interfaces and further details.
