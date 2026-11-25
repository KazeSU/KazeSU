# Old-kernel and manual-hook support

KazeSU's default hooking method registers a `sys_enter` tracepoint and
redirects marked processes through a dispatcher.  That needs GKI 2.0
(5.10+) and `CONFIG_KPROBES`.  For vendor/older trees there is a second,
additive mode: **manual hook**.

This work is adapted from [BakaSU](https://github.com/Baka-SU/BakaSU)
(`kernel/compat/`, `kernel/hook/lsm_hooks.c`, `kernel/tools/*.mk`), which in
turn builds on the KernelSU and [backslashxx](https://github.com/backslashxx/KernelSU)
compatibility work.  KazeSU keeps its existing tracepoint path as the default;
nothing below changes a normal GKI build.

## Status

| Kernel | Tracepoint (default) | Manual hook |
| --- | --- | --- |
| 5.10+ GKI 2.0 | Supported | Supported (patched tree) |
| 5.10+ non-GKI/vendor | Usually supported | Supported (patched tree) |
| 5.4 / 4.19 | — | Supported core policy path (patched tree); `selinux_hide` is disabled |
| 4.14 | — | Experimental; finds the pre-4.17 policy globals through kallsyms (enable `CONFIG_KALLSYMS_ALL` or make the three globals non-static) |
| < 4.14 | — | Not yet |

The compat layer under `kernel/compat/` handles most of the non-SELinux API
drift (uaccess, `kernel_read`/`kernel_write`, `path_mount`, `task_work_add`,
static keys, `struct_size`, and so on).  `kernel/selinux/` now has two
policy paths, adapted from BakaSU:

- **5.10+**: the upstream `struct selinux_policy` with `policy_mutex`.
- **4.19/5.4**: the pre-5.7 `selinux_state.ss->policydb` + `policy_rwlock` +
  `latest_granting` path, with `ksu_dup_policydb()`/`ksu_destroy_policydb()`.
- **4.14**: the same legacy path using the global `policydb`,
  `policy_rwlock` and `latest_granting`, resolved at init through kallsyms.

`feature/selinux_hide.c` still relies on the post-5.7 policy container and
the selinuxfs status page in `selinux_state`; it is compiled out on
pre-5.10 and the feature reports itself as unavailable.

## Selecting the manual hook

```
CONFIG_KSU=y
CONFIG_KSU_MANUAL_HOOK=y
CONFIG_KPROBES is not required
```

The tracepoint method remains the default when `CONFIG_KPROBES` is set.

Manual mode builds `kernel/compat/kernel_compat.o` and
`kernel/hook/lsm_hooks.o` instead of the tracepoint objects, and disables the
`preinit` runner (there is no execve redirect to hand PID 1 the runner).

Three helpers can replace the corresponding kernel patches with LSM hooks:

- `CONFIG_KSU_MANUAL_HOOK_AUTO_SETUID_HOOK` — `task_fix_setuid`
- `CONFIG_KSU_MANUAL_HOOK_AUTO_INITRC_HOOK` — `file_permission`
- `CONFIG_KSU_MANUAL_HOOK_AUTO_INPUT_HOOK` — `input_event` patch

They require Linux 4.2+ (`security_add_hooks`).  On 6.8+ the LSM argument
layout changed, so the setuid and init.rc helpers cannot be used; patch the
call sites below instead.

## Required call sites

`tools/manual_hook_check.mk` greps the kernel tree at build time and stops
with a message for every missing hook.  The entry points are declared in
`kernel/feature/sucompat.h`, `kernel/runtime/ksud.h` and
`kernel/supercall/supercall.h`.

### `fs/exec.c`

In `do_execveat_common()`, before `bprm_execve()` (or where the kernel's
`execve` path keeps `struct filename`):

```c
ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
```

### `fs/open.c`

In `do_faccessat()` / `do_faccessat2()`:

```c
ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
```

### `fs/stat.c`

After the native `stat` is copied out in `vfs_statx()`/`vfs_fstatat()`, and
after `vfs_fstat()` for the fd variant:

```c
ksu_handle_stat(&dfd, &filename, &flags);
ksu_handle_newfstat_ret(&fd, &statbuf);
#if defined(__ARCH_WANT_STAT64) || defined(__ARCH_WANT_COMPAT_STAT64)
ksu_handle_fstat64_ret(&fd, &statbuf);
#endif
```

`ksu_handle_fstat64_ret()` is declared and built only on architectures that
define `__ARCH_WANT_STAT64`/`__ARCH_WANT_COMPAT_STAT64`, so guard the call with
the same condition (not `CONFIG_COMPAT`; arm64 has compat but no `stat64`).

### `kernel/reboot.c` (or `kernel/sys.c` before 3.12)

In `__do_sys_reboot()`, before the switch:

```c
ksu_handle_sys_reboot(magic1, magic2, cmd, &arg);
```

The handler installs the KSU fd for the manager and returns `-EINVAL` when
the magic does not match.

### `fs/read_write.c`

Only when `CONFIG_KSU_MANUAL_HOOK_AUTO_INITRC_HOOK=n`: in `ksys_read()`:

```c
ksu_handle_sys_read(fd, &buf, &count);
```

### `kernel/sys.c`

Only when `CONFIG_KSU_MANUAL_HOOK_AUTO_SETUID_HOOK=n`: in
`__sys_setresuid()`, after the new credentials are prepared:

```c
ksu_handle_setresuid(old->uid.val, new->uid.val);
```

### `drivers/input/input.c`

Only when `CONFIG_KSU_MANUAL_HOOK_AUTO_INPUT_HOOK=n`: in `input_event()`:

```c
ksu_handle_input_handle_event(&type, &code, &value);
```

## Symbol exports

`feature/selinux_hide.c` locates its targets with `kallsyms`.  If
`CONFIG_KALLSYMS_ALL` is disabled, export the symbols listed in
`tools/static_export_check.mk` from the kernel tree; the build check prints
the exact names.

## Limitations

- `preinit`, the `ksud` fd install on `execve` for non-`su` binaries, and
  some module/rc paths are tracepoint-only.  Manual mode still handles
  `su` → `ksud`, `init.rc` reads and the reboot-magic fd install.
- `sulog` execve records are best-effort in manual mode.
- adb-root uses the same call sites as tracepoint mode; verify it on your
  tree.
- On 4.19/5.4 the `selinux_hide` feature is compiled out (the core policy
  editor still works).  On 4.14 the pre-4.17 policy globals are resolved
  through kallsyms, so enable `CONFIG_KALLSYMS_ALL` or export `sel_mutex`,
  `policy_rwlock` and `latest_granting`.
- No pre-5.10 build has been validated in this repository yet.  See the
  status table above.

## Credits

- [BakaSU](https://github.com/Baka-SU/BakaSU): the compat layer, manual-hook
  scaffolding and LSM helpers this port is adapted from.
- [KernelSU](https://github.com/tiann/KernelSU): upstream.
- [backslashxx/KernelSU](https://github.com/backslashxx/KernelSU): kernel
  compatibility reference.
- [susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu): hiding patches and
  related compat work.

Files ported from BakaSU keep a comment pointing at the original path.  They
are GPL-2.0-only, matching the rest of `kernel/`.
