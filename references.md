[Monograph index](README.md) · [Full text](door-ipc-implementation-for-freebsd_full.md)

<a id="references"></a>

# References and normative sources

## Reference policy

The implementation source, installed public headers, symbol map, and manual
pages from one selected revision are the primary normative sources. External
Solaris and illumos material defines the compatibility target but does not
override FreeBSD security, VFS, VM, filedesc, jail, Capsicum, or scheduler
invariants. POSIX and C standards govern only the interfaces and language
rules within their scope.

The source tree, matching public headers, tests, and manuals are maintained
continuously. Actual validation records identify their environment and ABI;
there is no monograph-wide source snapshot or final-revision prerequisite.

References are grouped by authority. Access dates apply to online material
whose contents may change.

## Project implementation and public ABI

1. **FreeBSD Doors implementation.** `sys/fs/doorfs/`,
   `sys/fs/namefs/`, scheduler and lifecycle changes under `sys/kern/`, public
   headers under `sys/sys/` and `include/`, `lib/libdoor/`, and the installed
   test trees.
2. **Door kernel ABI.** `sys/sys/door_core.h` and `sys/sys/door_impl.h` from
   the selected project revision.
3. **libdoor public ABI.** `include/door.h`, `lib/libdoor/Symbol.map`, and
   `lib/libdoor/libdoor_impl.h` from the selected project revision.
4. **Project manual pages.** `door(2)`, `door(3)`, `doorfs(4)`, and
   `namefs(4)` and `doorctl(4)` from `lib/libdoor/` in the selected project revision.
5. **Executable specification.** `lib/libdoor/tests`, `tests/sys/fs/doorfs`,
   and `tests/sys/fs/namefs` from the same revision. Appendix 1 inventories
   every currently registered ATF case.

## FreeBSD primary sources

6. **The FreeBSD Project.** *FreeBSD source tree*, `main` branch. Record the FreeBSD base used for an actual build or measurement.
   <https://cgit.freebsd.org/src/>.
7. **FreeBSD Architecture Handbook.** Kernel, locking, SMP, VM, and driver
   architecture background. <https://docs.freebsd.org/en/books/arch-handbook/>.
8. **FreeBSD Developers' Handbook.** Kernel and userspace development
   conventions. <https://docs.freebsd.org/en/books/developers-handbook/>.
9. **FreeBSD `capsicum(4)` and capability-mode interfaces.** Together with
   `cap_enter(2)`, `cap_rights_limit(2)`, `cap_rights_init(3)`, and the
   authoritative `sys/sys/capsicum.h` from the selected revision.
   <https://man.freebsd.org/cgi/man.cgi?query=capsicum&sektion=4>.
10. **FreeBSD jail interfaces.** `jail(2)`, `jail(8)`, `prison_check(9)`, and
    the credential/prison implementation from the selected source revision.
    <https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2>.
11. **FreeBSD VFS and vnode interfaces.** `vnode(9)`, `VFS(9)`, `namei(9)`,
    `VOP_LOOKUP(9)`, `nmount(2)`, `mount(8)`, and `nullfs(5)` together with
    the selected VFS source.
    <https://man.freebsd.org/cgi/man.cgi?query=vnode&sektion=9>.
12. **FreeBSD file-descriptor interfaces.** `open(2)`, `close(2)`, `dup(2)`,
    `fcntl(2)`, `fstat(2)`, `procstat(1)`, and `sys/kern/kern_descrip.c` from
    the selected revision.
    <https://man.freebsd.org/cgi/man.cgi?query=close&sektion=2>.
13. **FreeBSD VM interfaces.** `mmap(2)`, `munmap(2)`, `mprotect(2)`, and VM
    object/map implementation from the selected revision.
    <https://man.freebsd.org/cgi/man.cgi?query=mmap&sektion=2>.
14. **FreeBSD synchronization interfaces.** `locking(9)`, `mutex(9)`,
    `condvar(9)`, `sleepqueue(9)`, and the selected implementations in
    `sys/kern`.
    <https://man.freebsd.org/cgi/man.cgi?query=locking&sektion=9>.
15. **FreeBSD schedulers.** `sched_4bsd(4)`, `sched_ule(4)`, `sched_add(9)`,
    `mi_switch(9)`, and the exact `sched_4bsd.c`/`sched_ule.c` revision used by
    the project.
    <https://man.freebsd.org/cgi/man.cgi?query=sched_ule&sektion=4>.
16. **FreeBSD process, thread, and signal interfaces.** `fork(2)`, `execve(2)`,
    `waitpid(2)`, `sigaction(2)`, `pthread_create(3)`, and the selected kernel
    exit, signal, and thread lifecycle sources.
17. **FreeBSD Test Suite infrastructure.** `kyua(1)`, `kyua-test(1)`,
    `kyua-report(1)`, `atf(7)`, and `/usr/tests/Kyuafile` from the qualified
    installation.
    <https://man.freebsd.org/cgi/man.cgi?query=kyua&sektion=1>.

## Solaris and illumos compatibility sources

18. **illumos gate.** Door implementation, public headers, libc wrappers, and
    manual-page sources. The exact compatibility baseline commit must be
    recorded for a release comparison. <https://github.com/illumos/illumos-gate>.
19. **illumos `door_create(3C)`.** Endpoint creation, attributes, private
    server pools, and server-procedure model.
    <https://smartos.org/man/3C/door_create>.
20. **illumos Doors interfaces.** `door_call(3C)`, `door_return(3C)`,
    `door_bind(3C)`, `door_unbind(3C)`, `door_info(3C)`, `door_revoke(3C)`,
    `door_cred(3C)`, `door_server_create(3C)`, and `door_xcreate(3C)` from the
    same illumos baseline.
21. **Oracle Solaris documentation.** Doors programming model and RBAC/rights
    profile documentation, used as compatibility and future-work background.
    <https://docs.oracle.com/en/operating-systems/solaris/oracle-solaris/>.

The project intentionally extends this model with FreeBSD-specific Capsicum,
jail, DoorCTL, NameFS recovery, persistent SHM, ring, and scheduler-shuttle
interfaces. Compatibility claims apply only where Chapter 2 and Chapter 9 say
they apply.

## Language and operating-system standards

22. **The Open Group.** *POSIX.1-2024 / The Open Group Base Specifications,
    Issue 8*, IEEE Std 1003.1-2024. Process, signal, pthread, file-descriptor,
    memory-mapping, and synchronization interfaces within POSIX scope.
    <https://pubs.opengroup.org/onlinepubs/9799919799/>.
23. **ISO/IEC 9899:2024.** *Programming Languages — C*. Object representation,
    integer conversion and overflow rules, atomics, memory model, and language
    semantics used by the kernel and library code.
24. **System V Application Binary Interface, AMD64 Architecture Processor
    Supplement.** Calling convention and data-model background. Private fast
    selectors remain project ABI and are not standardized by this supplement.

## Background for prospective applications

25. **The netmap project.** Netmap API, rings, slots, buffer-index exchange,
    and `NS_BUF_CHANGED` zero-copy forwarding semantics.
    <https://github.com/luigirizzo/netmap>.
26. **wolfSSL.** *wolfSSL Manual*. Userspace TLS implementation considered in
    the prospective compartmentalized web-server design.
    <https://www.wolfssl.com/documentation/manuals/wolfssl/>.
27. **The X.Org Foundation.** X11 protocol and extension specifications,
    including request/reply/event ordering and descriptor-bearing extensions.
    <https://www.x.org/releases/current/doc/>.
28. **Solaris RBAC and `pfexec` documentation.** Background for the prospective
    profile-based privilege-execution model; it is not part of the implemented
    Door ABI.

Online sources last accessed 2026-09-21. Final publication must archive or pin
all mutable online references and replace provisional source-commit fields.

---
