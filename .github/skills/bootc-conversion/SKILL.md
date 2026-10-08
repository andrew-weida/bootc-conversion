---
name: bootc-conversion
description: >-
  Analyze how an existing application is installed and convert it to run in RHEL image
  mode as a bootc bootable container image. Use when converting, migrating, or porting an
  app, RPM, or package-based install to bootc / image mode / a bootable container; when
  auditing an RPM, install script, or Containerfile for bootc or ostree compatibility;
  when deciding what belongs in /usr, /etc, /var, or /opt on an image mode system; or when
  writing or reviewing a Containerfile built FROM rhel-bootc. Covers RPM scriptlet
  adaptation, systemd-sysusers, systemd-tmpfiles, first-boot services, bound container
  images, and validating the result by booting it.
---

# Converting an existing application to RHEL image mode (bootc)

Procedure for converting an application installed on a package-based RHEL system into a
bootc (image mode) container image.

Reference material, read on demand — do not load it up front:

- [the reference](./references/bootc-reference.md) — full bootc manual. Cite it by
  heading when explaining a decision.
- [validation](./references/validation.md) — how to test the result.

Targets RHEL 10 by default. Ask which RHEL version before recommending soft reboot
(`--soft-reboot`, RHEL 10 only) or anything the reference marks **RHEL 9 vs RHEL 10**.

---

## The two facts that drive every decision

1. **systemd is not running during the image build.** No D-Bus, no services, no TTY, no
   `/run/systemd/system`, no real hardware. Anything an installer does that assumes a live
   system must be moved to boot time.
2. **`/var` is seeded only on first install.** It is never updated by later images. `/usr`
   is read-only at runtime but fully writable at build time. `/etc` is 3-way merged.

Almost every conversion defect traces back to one of these. When something in the source
application looks wrong, ask which one it violates.

---

## Phase 0 — Decide whether it belongs in the image at all

Settle this before inventorying anything; it changes every downstream decision.

| Situation | Where it goes |
|---|---|
| OS-level, must run from early boot, must not drift from the host (agents, log forwarders, monitoring, security tooling) | Bake into the bootc image |
| Already containerized, or updates on a cadence independent of the OS | Bound container image — logically bound by default, physically bound for air-gapped or registry-outage resilience. See the reference → *Bound Container Images*. |
| Updates far more often than the OS does | Bound, or keep it off the image entirely. Baking it in means a full image rebuild and a reboot for every application release. |
| Requires runtime package installs, an interactive installer, or self-updates in place | Poor fit. Say so before starting; the conversion will fight the platform the whole way. |

Deciding question is **cadence coupling**: if the app and OS should version and update as
one unit, bake it in. If they must move independently, do not.

---

## Phase 1 — Inventory the existing install

Do not write a Containerfile until you can answer all of these, from evidence rather than
assumption. Run the commands if the RPMs or a running system are available; otherwise read
the installer scripts directly.

```bash
rpm -qlp <package>.rpm                  # every file the package ships
rpm -q --scripts -p <package>.rpm       # %pre/%post/%preun/%postun — the main hazard
rpm -qp --requires <package>.rpm        # runtime deps that must be in the image
```

For non-RPM installers (tarball, `install.sh`, vendor binary, Ansible role), read the
install script end to end and treat it as the scriptlet equivalent.

Collect:

| Question | How to find it |
|---|---|
| What files land where? | `rpm -qlp`, or the installer's copy/extract targets |
| What does the installer *execute*? | scriptlets, `install.sh`, Ansible tasks |
| What users/groups does it create? | `useradd`/`groupadd`/`getent` calls; desired UID/GID stability |
| What services does it define? | unit files in the payload; `systemctl enable/start` calls |
| What paths does it **write to at runtime**? | config, logs, spool, DB, cache, PID, sockets, uploads |
| What is host-specific? | hostname, IP, FQDN, licenses, certs, cluster IDs, secrets |
| What ports does it need? | `firewall-cmd` calls, docs, unit `ExecStart` flags |
| Does it need SELinux policy/contexts? | `semanage`/`restorecon`/`.pp` modules in the payload |
| Does it ship or expect containers? | podman/docker calls, compose files |

**Ask the user** when the inventory does not answer:

- Which data must survive a reboot/update, and which is rebuildable?
- Which config is per-machine vs. identical on every machine?
- Does the app need a stable UID/GID on disk (NFS, shared storage, existing data)?
- What is the update cadence — does the app ship with the OS image or separately?

---

## Phase 2 — Classify every path

Build this table before writing any code — it is the core deliverable and the thing most
conversions get wrong. For each path the application touches, assign one of:

| Class | Goes to | Mechanism |
|---|---|---|
| Static program content, assets, web root, defaults | `/usr` | `COPY`/`dnf install` at build time |
| Vendor tree that installs as a unit | `/opt/<app>` (read-only at runtime) | build-time install; redirect its writable subdirs |
| Machine-local config a sysadmin edits | `/etc` | ship defaults; 3-way merge handles edits |
| Mutable state that must persist (DB, logs, spool, uploads) | `/var` | `systemd-tmpfiles` or `StateDirectory=`, **never** `mkdir` in the Containerfile |
| Ephemeral (PID files, sockets, scratch) | `/run` | `RuntimeDirectory=`; prefer `Type=notify`/`Type=exec` over PID files |

Three rules that repeatedly catch people:

- **Content that ships with the image must not live in `/var`.** If the app's payload
  installs into `/var` (web roots, default data, templates), relocate it to `/usr` and
  patch the config to match — otherwise it freezes at first install and never updates.
  See the reference → *Relocating Content Out of `/var`*.
- **A writable path under `/usr` or `/opt` is a bug, not a preference.** Fix it by
  symlinking into `/var` at build time (preferred), or `BindPaths=` in the unit. Reach for
  `transient-ro` only when you need a *top-level* path that cannot exist at build time.
  See the reference → *Handling `/opt`*.
- **Ship config defaults in `/usr`, reserve `/etc` for genuine admin edits.** Where the
  app supports it (`/usr/lib/<app>/`, drop-in directories, systemd unit drop-ins), put
  defaults under `/usr` and leave `/etc` as empty as you can. Every file you ship in
  `/etc` joins the 3-way merge surface, and the merge is where surprising update behavior
  comes from — conflicts resolve toward the local copy, and a uid/gid/xattr change alone
  is enough to pin a file at its local version.

---

## Phase 3 — Translate install-time actions

**Embed the RPM, or reimplement the install?** If the app ships an RPM, prefer
`RUN dnf -y install <pkg>` and compensate for what its scriptlets get wrong.
Reimplementing a vendor install by hand drifts from upstream with every release.

Scriptlets **do execute** during the build. The failure mode is not that they are skipped
— they run without systemd and either fail the build or appear to succeed while
accomplishing nothing (enabling a service that never ends up enabled, creating a `/var`
directory that never reaches upgraded systems). Read the build output; do not trust the
exit code.

Then handle each executed action from Phase 1:

| Installer does | Replace with |
|---|---|
| `systemctl enable --now X` | `RUN systemctl enable X` (drop `--now`) |
| `systemctl start/restart/reload X` | nothing — enabled services start at first boot |
| checks `/run/systemd/system` / `systemd-detect-virt` | remove the check; copy unit files explicitly |
| `useradd` / `groupadd` | `/usr/lib/sysusers.d/<app>.conf` with an explicit UID/GID, or `DynamicUser=yes` if no stable on-disk identity is needed |
| `mkdir -p /var/...` | `/usr/lib/tmpfiles.d/<app>.conf`, or `StateDirectory=` in the unit |
| `read`, `dialog`, any prompt | first-boot unit with `ConditionFirstBoot=yes` |
| hostname/IP/cert/license baked in | first-boot unit, cloud-init, or Ignition |
| `firewall-cmd --add-port/--add-service` | `COPY <app>.xml /usr/lib/firewalld/services/` |
| `semanage`/`restorecon` at install | keep `restorecon -R <paths>` as a build step; ship custom policy modules in the image |
| PID file in `/run` | `Type=notify` or `Type=exec` |
| compiles from source | multi-stage build; only the runtime artifact reaches the final image |
| `rpm-ostree install` / runtime `dnf install` | packages belong in the Containerfile; `dnf install --transient` is debugging only, discarded on reboot |

If the app is distributed as a container and should stay one, do **not** flatten it into
the OS — use a bound container image (logically bound by default; physically bound for
air-gapped or registry-outage resilience). See the reference → *Bound Container Images*.

---

## Phase 4 — Write the Containerfile

Follow the reference → *Complete Example: Web Application Image*. Required elements:

- `FROM registry.redhat.io/rhel10/rhel-bootc:<tag>` (or the RHEL 9 equivalent) — pin an
  explicit tag, never `latest`, so rebuilds are reproducible. The registry needs
  authentication; Red Hat refreshes base images at least monthly and **your derived image
  does not rebuild itself** — that is the pipeline's job.
- `LABEL containers.bootc=1` and `LABEL ostree.bootable=1`
- `dnf -y install ... && dnf clean all` in a single layer
- sysusers / tmpfiles drop-ins under `/usr/lib/`
- `RUN systemctl enable <units>`
- `RUN restorecon -R <paths you COPYed>`
- `RUN bootc container lint`
- `CMD ["/sbin/init"]` for container-mode testing

`ENTRYPOINT`, `CMD`, `ENV`, `USER`, and `EXPOSE` are **ignored on an installed system**.
If the source app relied on them, the behavior must be re-expressed in a systemd unit
(`Environment=`, `User=`) or firewalld XML.

Keep unit files, tmpfiles, sysusers, and firewalld XML as separate files in the repo and
`COPY` them rather than generating them with heredocs — they stay reviewable and lintable.

---

## Existing deployments: state migration

An app already running somewhere has state — database, logs, uploads, tuned config.
**The image carries none of it**; a fresh bootc machine boots with empty state directories.

- Back up on the old host, restore into `/var` on the new one. Never ship production data
  in the image.
- Automated restores must be **idempotent** one-shot or first-boot units.
- Check the UID/GID owning the old data matches what `systemd-sysusers` allocates. Most
  common post-migration failure: files restore fine, service cannot read them. Main reason
  to pin explicit UIDs.
- Fleet-wide `/etc` customizations become image defaults; per-machine ones become
  provisioning (cloud-init, Ignition, config management). Hand-editing `/etc` on the new
  system reintroduces the drift image mode exists to remove.

---

## Phase 5 — Validate

Three tiers, each proving strictly more than the last. Full procedure, including how to
detect whether Tier 3 is possible on this host, is in `references/validation.md`.

| Tier | What it is | Proves |
|---|---|---|
| 1 | `podman build` + `bootc container lint` + payload diff | the image builds, lints, and contains what the package shipped |
| 2 | `podman run --privileged --systemd=always` | units, enablement, `systemd-sysusers`, `systemd-tmpfiles` |
| 3 | boot a real VM or disk | boot, `/usr` read-only, SELinux enforcing, `/var` seeding, upgrade/rollback |

Tier 3 requires a hypervisor and may be unavailable where you are running — check first,
and if it is not possible, finish Tiers 1-2 and report explicitly what went unverified.
Never claim success from an earlier tier alone; always state which tier you reached.

---

## Hard rules

Never produce a Containerfile containing any of these:

1. `systemctl enable --now`, `systemctl start`, or `systemctl restart`
2. `mkdir` under `/var` for directories the app needs at runtime
3. `firewall-cmd`
4. anything that can block on input (`read`, `dialog`, `dnf` without `-y`)
5. `rpm-ostree` package installs
6. runtime secrets (API keys, passwords, private keys) baked into the image
7. application state written into `/usr` or `/opt` at runtime
8. build toolchains in the final stage
9. globally enabling `/usr/lib/bootc/storage` in `storage.conf`

Also watch for, and flag to the user:

- changing uid/gid/xattrs on an `/etc` file — counts as a local modification and blocks
  the image's version from applying on update
- shipping SSH keys or home content under `/var/home` in a later image version — it never
  reaches already-deployed systems
- NetworkManager keyfiles not mode `600` — silently ignored
- `dracut` without an explicit kernel version — it grabs the running kernel and errors
- `anaconda-iso` reformats the first disk it finds, without prompting

---

## What to hand back

A conversion analysis should produce:

0. **The Phase 0 verdict** — bake in, bind as a container, or do not convert — and the
   cadence-coupling reason for it. If the answer was "do not convert," stop here.
1. **Path classification table** (Phase 2) — every path, its class, its mechanism.
2. **Action translation table** (Phase 3) — what the installer did, what replaces it.
3. **Containerfile** plus the sidecar files it references (units, tmpfiles, sysusers,
   firewalld XML, first-boot script).
4. **Deferred-to-first-boot list** — everything that cannot happen at build time and how
   it gets provisioned (first-boot unit, cloud-init, Ignition, `bootc install` flags).
5. **Migration plan**, if any deployment already exists — what state moves, how, and
   whether the UID/GID of the old data matches the image.
6. **Open questions / unresolved risks** — anything that needs the app owner's input,
   especially stable UIDs, and licensing or secrets handling.

Distinguish what you verified by running something from what you inferred by reading. If
part of the app cannot be converted cleanly — writes to `/usr` by design, requires an
installer GUI, needs a package manager at runtime — say so rather than papering over it
with `transient-ro` or a transient overlay.
