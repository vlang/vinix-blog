Vinix adopts many of OpenBSD's security ideas while running Linux user programs on
a kernel written in V. We have implemented pledge and unveil, W^X, immutable
mappings, signal-frame cookies, address and PID randomization, and securelevel.
Much of this is adaptation of OpenBSD's designs to a different kernel and syscall ABI.

The useful differences from Linux are in the interfaces,
defaults, and policies that applications actually use. Modern Linux has close
counterparts to many of these protections, as well as security infrastructure
that Vinix has yet to build. Here is a comparison of the current development
implementation, rather than a security score for either system.

## The mechanisms at a glance

| Security goal | Vinix | Linux |
| --- | --- | --- |
| Reduce an application's available operations | Application opts into `pledge`; Linux-compatible seccomp also available | Application or launcher installs seccomp filters |
| Restrict filesystem access | Application opts into `unveil` and locks its view | Landlock for self-confinement; SELinux/AppArmor for administrator policy |
| Prevent writable executable memory | W^X by default, with administrator-authorized exceptions | Userspace restrictions through MDWE or security policy; separate strict kernel RWX protections |
| Seal mapping permissions and lifetime | Application opts into `mimmutable` | Application or runtime opts into `mseal` |
| Harden signal return | OpenBSD-style frame cookies by default | Architecture-specific state validation; optional x86 CET shadow stacks |
| Make addresses and identifiers less predictable | ASLR and randomized PIDs/TIDs | ASLR; ordinary PID allocation advances through available IDs |
| Avoid copying sensitive state into forked children | `minherit` and Linux-compatible memory advice | `MADV_WIPEONFORK` and `MADV_DONTFORK` |
| Protect the kernel boundary | Checked copies, hardware access restrictions, canaries, read-only kernel mappings | Corresponding protections, plus a broader hardening framework |
| Protect sealed files from privileged changes | Immutable/append flags plus configured securelevel | Immutable/append flags, capabilities and mandatory policy |

The table summarizes the mechanisms discussed below. Application opt-in means
that an ordinary Linux binary does not gain the protection merely by running on
Vinix. Linux's effective coverage likewise depends on its kernel configuration,
hardware, distribution policy, and application choices.

## Application confinement: pledge, unveil, seccomp and Landlock

OpenBSD's pledge gives a program a compact vocabulary for giving up operations.
Vinix's [implementation](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/proc/pledge.v)
adapts those promises to Linux syscalls. A process can promise `stdio rpath`:
basic descriptor I/O and reading paths. Opening an ordinary file for writing
violates the promise. Promises can only narrow; a violation normally terminates the process
with `SIGABRT`, while the `error` promise requests an `ENOSYS` failure instead.
Children inherit promises, and `execpromises` selects restrictions for the next
executable. Without execpromises, exec starts the next program unpledged.

Linux's [seccomp](https://docs.kernel.org/userspace-api/seccomp_filter.html) gives
programs lower-level filters over syscall numbers and raw arguments. A filter
can reject a socket family or an ioctl number, but cannot dereference an
`open()` argument to inspect its pathname. It is a building block for a sandbox,
used together with filesystem and privilege controls. Vinix also implements
classic-BPF seccomp for Linux compatibility; pledge adds a simpler semantic
interface rather than replacing it.

[Unveil](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/proc/unveil.v)
provides the path boundary. A program exposes selected paths with read, write,
execute, or create/remove permissions, then calls `unveil(NULL, NULL)` to lock
the view. For example, an image processor could expose its input directory for
reading and its output directory for writing and creation, then pledge only
the operations its processing stage needs. Vinix checks resolved targets, so a
symlink cannot simply redirect an allowed lookup outside the view. Already-open
descriptors remain authority the program must account for separately.

Linux has a close self-confinement counterpart in
[Landlock](https://docs.kernel.org/userspace-api/landlock.html). Unprivileged
programs can restrict access to filesystem hierarchies, with additional
networking and IPC controls in newer ABIs. Enforced restrictions cannot be
removed and pass to descendants. Applications query the supported ABI and
rights. Linux therefore supports application-controlled path confinement too;
unveil offers a different, compact interface.

Linux additionally has administrator-controlled mandatory access control through
its [LSM framework](https://docs.kernel.org/admin-guide/LSM/index.html), including
SELinux and [AppArmor](https://docs.kernel.org/admin-guide/LSM/apparmor.html).
These can constrain applications according to deployed policy without requiring
them to add unveil calls. Vinix currently has no equivalent general MAC framework.
Its pledge and unveil mechanisms need application adoption: we have not yet made
the ordinary imported application set use them automatically.

## Executable memory: defaults and compatibility

Vinix enforces **W^X** for user mappings by default: a mapping may be writable or
executable, but cannot ordinarily be both simultaneously. A JIT can write code
into RW pages and then switch them to RX. The rule is enforced for both mapping
creation and protection changes.

Exceptions require authorization. An environment request alone does not grant
it: the [exec policy](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/fs/wx_policy.v)
requires either an effective-root launcher with `CAP_SYS_ADMIN` in the initial
user namespace, or an administrator-authorized `wxallowed` executable mount.
This accommodates runtimes that still require writable executable memory while
keeping an unprivileged process from granting itself the exception.

Linux does not impose the same universal userspace W^X rule in its ordinary
mapping API. It offers opt-in
[Memory-Deny-Write-Execute](https://man7.org/linux/man-pages/man2/PR_SET_MDWE.2const.html)
through `PR_SET_MDWE`, and additional restrictions through security policy.
MDWE's `PR_MDWE_REFUSE_EXEC_GAIN` is stricter than Vinix's ordinary W^X: it
rejects new RWX mappings and stops nonexecutable mappings from gaining execute
permission. A normal RW-to-RX JIT transition is therefore incompatible with that
mode. These policies address related risks with different compatibility choices.

Our [Firefox bring-up](https://blog.vinix-os.org/post/33/what-chromium-and-firefox-taught-us-about-our-kernel)
provided a concrete example. SpiderMonkey content processes requested RWX pages,
which Vinix refused. Firefox's existing code-write-protection preference made
them switch permissions instead, allowing the JIT to operate within W^X.

## Immutable mappings: Linux has mseal too

W^X controls which permissions may coexist. Vinix's
[`mimmutable`](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/memory/mmap/immutable.v)
adds a one-way restriction on the mapping itself. Once sealed, its protection
cannot be changed, and the mapping cannot be removed, moved, or replaced through
the ordinary memory-management interfaces. Applications choose the regions and
their final permissions before sealing. The current ELF loader does not
automatically seal every executable's text with this syscall.

Modern Linux provides [`mseal`](https://docs.kernel.org/userspace-api/mseal.html),
whose documentation explicitly acknowledges OpenBSD's `mimmutable`. It likewise
prevents permission changes, removal and replacement, together with selected
destructive memory-advice operations. Their ABIs and edge cases differ, but the
security goal is closely related.

For both, sealing mapping metadata is distinct from making its bytes read-only.
A sealed writable mapping remains writable. Applications must pick appropriate
permissions as well as deciding which mappings should survive unchanged.

## Signal return: cookies and hardware shadow stacks

Signal return restores registers from a frame on the application's stack. A
forged frame can make this restoration useful to an exploit.

Vinix uses an [OpenBSD-style cookie](https://github.com/vlang/vinix/commit/53f6e456b448675625e71960a99a572b1a94ce88):
a process secret combined with the frame's address. The kernel checks it before
restoring state and clears it after successful use. Exec changes the secret;
fork preserves it so inherited handlers can return. This is a check against
forged frames, not a cryptographic signature over all saved registers. Signal
handlers can still edit their context, and disclosure of a valid frame weakens
the cookie's protection.

Linux also validates signal-return state. Its
[x86-64 implementation](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/signal_64.c)
checks frame accessibility and restricts restored privilege state and flags. On
compatible x86 systems, enabled
[CET userspace shadow stacks](https://docs.kernel.org/arch/x86/shstk.html) add a
protected shadow-stack token that signal return verifies. Hardware, kernel,
toolchain and runtime support are required. Vinix's default cookies and Linux's
architecture-specific validation and optional CET are different mechanisms.

## Randomness, ASLR and forked secrets

Vinix randomizes PIE, interpreter, stack, mmap and heap-break placement, and the
ARM64 signal trampoline. It also uses
[random process and thread IDs](https://github.com/vlang/vinix/commit/e9afc902)
after init, avoiding recently released IDs and IDs still naming live groups or
sessions. Allocation has a sequential fallback. Random PIDs reduce predictability;
they do not replace permission checks.

Linux already has [ASLR for those major process regions](https://docs.kernel.org/admin-guide/sysctl/kernel.html#randomize-va-space),
including heap randomization at the full setting. Its
[ordinary PID allocator](https://github.com/torvalds/linux/blob/master/kernel/pid.c)
advances through available IDs cyclically. Neither system can move the code of a
fixed-address executable simply because PIE randomization is enabled.

Vinix's [ChaCha20 generator](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/krandom/random.v)
adopts OpenBSD-style fast key erasure, periodic reseeding and explicit erasure of
temporary secrets. Boot seeding uses hardware or platform sources when available,
with a timing-jitter fallback. Seed quality matters to the resulting
mitigations, and some early-boot consumers permit output before the generator is
marked ready. We have not established equivalent entropy quality across all
supported machines.

Linux also uses a [ChaCha-based generator with fast key erasure](https://github.com/torvalds/linux/blob/master/drivers/char/random.c).
Its default [`getrandom()`](https://man7.org/linux/man-pages/man2/getrandom.2.html)
waits for initialization; a nonblocking request can return `EAGAIN`. Strong
randomness and careful initialization are shared requirements, rather than an
OpenBSD idea that Linux has omitted.

Forking must not make a child's secret state an accidental duplicate of its
parent's. Vinix supports OpenBSD-style `minherit` and Linux-compatible
`MADV_WIPEONFORK` and `MADV_DONTFORK`: selected ranges become zero-filled or
absent in the child. [Linux supplies the same memory advice](https://man7.org/linux/man-pages/man2/madvise.2.html).
Programs must opt in for the state that needs it.

The network stack needs unpredictability too. Vinix now uses kernel randomness
for ephemeral ports, DNS and DHCP, an OpenBSD-inspired shuffled IPv4-ID scheme,
and keyed TCP initial sequence numbers. The
[network change](https://github.com/vlang/vinix/commit/1f8a7c5afd5554c1d82f5d552b03cf7977309c26)
uses SipHash for the TCP function. Linux has its own
[keyed sequence-number and port-selection machinery](https://github.com/torvalds/linux/blob/master/net/core/secure_seq.c).
The algorithms and integration differ; both aim to frustrate off-path prediction.

## The kernel boundary and memory corruption

Vinix checks pointers when copying between userspace and the kernel. Drivers and
resources receive kernel-owned buffers. Invalid user addresses should produce
`EFAULT` instead of becoming kernel reads or writes. This closed actual defects
in earlier pipe, socket and syscall paths, described in the
[checked-copy conversion](https://github.com/vlang/vinix/commit/e7cf86f26262be134cc1c5d32a6e4779e6e51127).
Linux and OpenBSD already make checked user-memory transfers a basic kernel rule.
Linux additionally has
[hardened usercopy checks](https://github.com/torvalds/linux/blob/master/mm/usercopy.c)
on kernel object boundaries and inappropriate kernel-memory exposure.

Vinix enables SMEP/SMAP on supported x86 CPUs and PXN/PAN on ARM64 where the
implementation is enabled, protecting the boundary from unintended kernel
execution of, or access to, user pages. Its kernel code and read-only data are
also read-only through the physical-memory direct-map alias. Linux supports
these hardware boundaries and
[strict kernel and module RWX protections](https://docs.kernel.org/security/self-protection.html).
They are shared hardening techniques.

Vinix's [SMAP/PAN policy](https://github.com/vlang/vinix/commit/324a89d3c84394ff4c34f9a95b8daabef434105b)
has deployment qualifications: production defaults to strict enforcement on
supported x86 and the tested ARM64 EL1 path; debug builds default to an audit
mode that allows and logs accesses. On Apple hardware at EL2, PAN currently
requires explicit opt-in. Those distinctions affect the protection a running
machine actually receives.

The kernel also uses stack canaries and slab allocation diagnostics. Freed
objects are poisoned and checked before reuse, exposing writes after free. Such
a check [found a terminal-node lifetime bug](https://github.com/vlang/vinix/commit/ac7df8fc).
Detection logs and counts the corruption but continues, so it is a useful
diagnostic with limits. Linux offers a broader collection of
[allocator and kernel hardening options](https://github.com/torvalds/linux/blob/master/kernel/configs/hardening.config),
whose deployment depends on configuration.

Writing much of Vinix in V does not make this kernel automatically memory-safe.
It builds with manual memory management and no garbage collector, uses unsafe
code and C components, and still needs correct bounds and object lifetimes.

## Privileges, process inspection and sealed files

Vinix implements Linux-style credentials, capability sets and `no_new_privs`.
Access to another process's sensitive procfs information uses
[inspection checks](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/proc/inspect.v)
covering credentials, dumpability, capabilities and user-namespace identity.
These are important companions to ASLR: address randomization has limited value
if an unrelated process can freely read the target's mappings.

Linux has its own process-inspection permission checks, with additional policy
such as [Yama's configurable ptrace restrictions](https://docs.kernel.org/admin-guide/LSM/Yama.html).
Vinix's compatibility remains incomplete. Its user-namespace ID mapping and
network-namespace isolation do not yet provide Linux-equivalent behavior, and
its seccomp implementation lacks notification listeners. The ordinary Vinix
startup path also begins with root credentials and full capabilities until
software explicitly drops them. Interfaces for reducing privilege are useful
only when the application and launch policy actually use them.

Vinix exposes immutable and append-only inode flags through the same ioctl ABI
used by Linux's `chattr +i` and `chattr +a`. Tmpfs keeps them in memory; ext2
persists them on disk. [Linux supports these flags too](https://man7.org/linux/man-pages/man1/chattr.1.html),
with filesystem-dependent support and suitable privilege required to set or
clear them. `CAP_LINUX_IMMUTABLE` is part of its
[capability model](https://man7.org/linux/man-pages/man7/capabilities.7.html).

Vinix adds OpenBSD-style
[securelevel](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/kernel/security/securelevel.v).
At a positive level, ordinary privileged processes cannot clear existing
immutable or append-only bits. It defaults to 0, so administrators must choose
the restriction. Vinix's level 2 currently adds nothing over level 1: OpenBSD's
additional raw-disk, clock and firewall restrictions are not all implemented.

Linux's [kernel lockdown](https://man7.org/linux/man-pages/man7/kernel_lockdown.7.html)
can restrict privileged access to the running kernel and selected hardware
interfaces. Capabilities and mandatory policy provide other ways to constrain
root. These controls have different scopes from Vinix's file-focused securelevel;
neither a root UID nor the name of a security feature describes every remaining
permission.

## Where Linux has a larger security system

The remaining gap includes system integrity and operational policy. Vinix does
not yet authenticate its bootloader, kernel, command line or root filesystem.
Sealing a file after boot does not protect against replacing the boot artifacts.
Linux offers facilities such as
[enforced module signatures](https://docs.kernel.org/admin-guide/module-signing.html)
and [dm-verity](https://docs.kernel.org/admin-guide/device-mapper/verity.html) for
verified block-device contents. Their protection also depends on how keys,
policies and trust anchors are deployed.

Vinix has initial [structured seccomp auditing](https://github.com/vlang/vinix/blob/e4c5ffc4b05f8fa9cd66b1268e6b9aecb7545ffc/tests/security-audit/README.md):
the newest 128 selected decisions in a bounded buffer, readable only with the
required initial-namespace authority. It is not a complete persistent audit
system. General MAC policy, a comparable broad speculation-mitigation policy,
and the surrounding production security tooling remain work ahead.

The existing Vinix regressions exercise pledge violations, hidden paths, forged
signal frames, memory protections, fork wiping, bad user pointers and file flags
on both architectures. They establish specific behaviors; they are not evidence
that every kernel path or application is secure.

Vinix's distinctive direction is a compact OpenBSD-inspired set of application
interfaces, together with stricter defaults in areas such as writable executable
memory and signal-frame cookies. Linux has counterparts for much of that, and a
much larger set of policies and deployment mechanisms around them. Our next step
is making these boundaries routine in Vinix applications and launchers while
continuing to test their enforcement. Vinix remains pre-alpha; this comparison
does not establish that it is safer than a maintained, well-configured Linux system.
