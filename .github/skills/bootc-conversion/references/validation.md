# Validating a bootc image

Three tiers. Each proves strictly more than the last. Never claim success from an earlier
tier alone; always state which tier you reached.

## Tier 1 — build and lint

Works anywhere podman runs.

```bash
podman build -f Containerfile -t <image> .
podman run --rm <image> bootc container lint

# Did every file in the original package land? Scriptlet failures show up here.
rpm -qlp <original>.rpm | sort > /tmp/expected
podman run --rm <image> rpm -ql <pkg> | sort > /tmp/actual
diff /tmp/expected /tmp/actual
```

## Tier 2 — systemd as PID 1 in a container

No VM required. With `CMD ["/sbin/init"]`, podman's systemd mode runs systemd as PID 1,
so services actually start.

```bash
podman run -d --name test --privileged --systemd=always <image>
podman exec test systemctl --failed
podman exec test journalctl -b --no-pager -p warning
podman exec test ls -ld /var/lib/<app> /var/log/<app>   # did tmpfiles/sysusers run?
```

Validates: unit syntax, enablement and ordering, `systemd-sysusers`, `systemd-tmpfiles`.

Does **not** validate: `/usr` read-only (writable here), ostree `/var` seeding, SELinux
enforcing, kernel/initrd/bootloader, `bootc upgrade`. A compliance scan at this tier is
misleading — see [the reference](./bootc-reference.md) → *Security Hardening and Compliance*.

## Tier 3 — boot it

Only this tier proves the image boots, and it may be unavailable where you are running.
Check before you plan, not after you have built something.

### Step 1 — probe the host

Probe the machine you are actually running on. Do not infer it from the user's OS, the
repo contents, or a previous session.

```bash
uname -s; uname -m                                  # kernel + host arch
ls /dev/kvm 2>/dev/null        || echo "no /dev/kvm"
test -e /run/.containerenv -o -e /.dockerenv && echo "running inside a container"
command -v podman vfkit qemu-system-$(uname -m) 2>/dev/null
podman machine list 2>/dev/null                     # macOS/Windows: is there a Linux VM?
```

Windows, from PowerShell: `wsl.exe -l -v` and `Get-ComputerInfo -Property 'HyperV*'`.

### Step 2 — map the result to a path

Compare host arch to the *production* target arch. If they differ, you are validating
conversion logic, not arch-specific behavior — say so.

| Probe says | Path |
|---|---|
| Linux, `/dev/kvm` present | `bootc-image-builder --type qcow2` → libvirt/QEMU, or `bootc install to-disk` against a spare device. |
| macOS, `arm64`, podman machine running | The Mac is the hypervisor via Virtualization.framework. Build rootful inside the podman machine (`podman machine set --rootful`), `--type raw` — Virtualization.framework takes raw, not qcow2 — boot with `vfkit` using the EFI bootloader. Output is **aarch64**. |
| Windows, WSL2 or Hyper-V present | Hyper-V is the hypervisor; WSL2 runs on it. Build rootful inside the WSL2 podman machine, `--type vhd`, boot as a **Generation 2** VM. Normally **x86_64**, so arch matches production. Gen 2 defaults to the Windows Secure Boot template and will refuse to boot — use the Microsoft UEFI Certificate Authority template or disable Secure Boot. The builder needs privileged loop-device access; check that first if it fails under WSL2. Alternative: `nestedVirtualization=true` in `.wslconfig`, then qcow2 + qemu-kvm inside WSL2. |
| Inside a container, or no `/dev/kvm` and no hypervisor | **Not possible here.** Go to Step 4. |

### Step 3 — verify on the booted system

- services started (`systemctl --failed`, `journalctl -b`)
- `/var` directories exist with the right owner
- the app can write everywhere it needs to — where `/usr`-write bugs surface
- SELinux enforcing and the app still works (`ausearch -m avc -ts boot`)
- an update lands: build v2, `bootc upgrade`, reboot, re-verify

### Step 4 — when Tier 3 is not possible locally

Do not fake it, do not substitute Tier 2 and call the image validated, and do not use
cross-arch emulation — a cloud instance of the right arch is faster and cheaper.

1. Finish Tiers 1 and 2; report what they covered.
2. Produce an artifact someone else can boot: push to a registry, or build `--type ami` /
   `--type qcow2` as a CI artifact.
3. Hand over the Step 3 checklist so a human or a KVM-capable CI job can finish.
4. State in your summary: *"Not booted — Tier 3 unavailable on this host (reason).
   Unverified: boot, SELinux enforcing, `/var` seeding, upgrade/rollback."*

The upgrade/rollback check has no Tier 2 approximation.
