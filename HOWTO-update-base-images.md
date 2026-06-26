# HOWTO: Update the FreeBSD OCaml base images

This is the runbook for refreshing the OCaml base images on the FreeBSD build
farm — e.g. adding a new compiler (`5.5.0`), bumping a point release
(`4.14.3` → `4.14.4`), or migrating to a new FreeBSD release.

The Ansible playbooks here do most of the work; this document captures the
surrounding process (when to pause the worker, how to build only what changed,
and how to verify) so it doesn't have to be rediscovered each time.

## The setup at a glance

- **Host:** `rosemary.caelum.ci.dev` — the live FreeBSD worker (the inventory
  `hosts` file lists it by FQDN).
- **OCluster pool:** `freebsd-x86_64`, scheduled by `scheduler.ci.dev:8103`.
- **Admin capability:** `~/admin.cap` (used by every `ocluster-admin` command).
- **ZFS pool on the host:** `obuilder`, base images under
  `obuilder/base-image/...`, obuilder cache under `obuilder` / state in
  `/obuilder/state`.

Base images are built by the `base-image` role (`roles/base-image/tasks/main.yml`)
and listed in `playbook.yml`.

## ⚠️ The one gotcha: dataset names are major.minor only

The role names each dataset `freebsd-<release>-ocaml-<major.minor>`:

```yaml
base_image: "freebsd-15.0-ocaml-{{ version.split('.')[:2] | join('.') }}"
```

So **the point release is dropped from the dataset name**:

- `5.5.0`  → `freebsd-15.0-ocaml-5.5`  (new dataset)
- `4.14.4` → `freebsd-15.0-ocaml-4.14` (**same** dataset as `4.14.3` — a
  point-release bump just rebuilds it in place)

Only the opam switch *inside* the image carries the full version. Verify the
point release by inspecting the switch directory name (see step 5), not the
dataset name.

## Procedure

### 1. Edit `playbook.yml`

Update the `base-image` role list to the desired version set. Example from the
4.14.4 / 5.5.0 change:

```yaml
- { role: base-image, version: "busybox", user_name: opam, zfs_pool: "obuilder", default: false }
- { role: base-image, version: "4.14.4", user_name: opam, zfs_pool: "obuilder", default: false }
- { role: base-image, version: "5.4.1", user_name: opam, zfs_pool: "obuilder", default: false }
- { role: base-image, version: "5.5.0", user_name: opam, zfs_pool: "obuilder", default: false }
```

- To **replace** a point release, edit the version string in place
  (`4.14.3` → `4.14.4`).
- To **add** a compiler, append a new line.
- `default: true` clones the image to the `freebsd` dataset used as the pool's
  default — set it on exactly one entry if you want a default, otherwise leave
  all `false`.

### 2. Decide: targeted build vs. full playbook

The role **destroys and rebuilds** every image it is given. Running the full
`playbook.yml` therefore wipes and rebuilds *working* images (`busybox`, and any
compiler you didn't touch) — wasteful and risky.

**Prefer a targeted build** that only builds the new/changed images. Create a
throwaway playbook in the repo root (so `roles/` resolves) listing just those:

```yaml
# deploy-targeted.yml  (do not commit; delete after)
---
- hosts: all
  name: Install Roles (targeted)
  pre_tasks:
    - name: Remove stale BSDINSTALL_DISTDIR
      file:
        path: /usr/freebsd-dist
        state: absent
      become: yes
  roles:
    - { role: base-image, version: "4.14.4", user_name: opam, zfs_pool: "obuilder", default: false }
    - { role: base-image, version: "5.5.0", user_name: opam, zfs_pool: "obuilder", default: false }
```

### 3. Pause and drain the worker

The role destroys the dataset being rebuilt, so the worker must not be using it.
Pause it and wait for in-flight jobs to finish (this can take a while — long
builds must drain):

```sh
ocluster-admin -c ~/admin.cap pause --wait freebsd-x86_64 rosemary
```

New jobs queue as a backlog while paused; they run once you unpause. Confirm it's
idle:

```sh
ocluster-admin -c ~/admin.cap show freebsd-x86_64      # rosemary ... (0 running)
```

### 4. Build the images

```sh
ansible-playbook -i hosts deploy-targeted.yml          # targeted
# or, for a full rebuild:
# ansible-playbook -i hosts playbook.yml
```

**Gotcha:** the inventory host is the FQDN `rosemary.caelum.ci.dev`. Don't pass
`--limit rosemary` (the short name won't match — "no hosts to target"). There's
only one host, so just omit `--limit`.

Each image downloads the FreeBSD base, builds opam, clones opam-repository, and
runs `opam init` (which compiles the compiler) — budget tens of minutes per
image. Watch for `PLAY RECAP ... failed=0`.

### 5. Verify

```sh
ssh rosemary.caelum.ci.dev '
  echo "=== datasets ===";  zfs list -H -o name -r obuilder/base-image | grep ocaml
  echo "=== snapshots ==="; zfs list -H -t snapshot -o name | grep ocaml
  echo "=== switch versions (the real point release) ===";
  for v in 4.14 5.5; do
    echo -n "$v: ";
    ls /obuilder/base-image/freebsd-15.0-ocaml-$v/rootfs/home/opam/.opam/ \
      | grep -E "^[0-9]";    # the switch dir is named e.g. 4.14.4 / 5.5.0
  done'
```

Confirm: each expected dataset exists, has an `@snap` snapshot, and the opam
switch directory inside is named with the exact point release you intended.

### 6. Unpause

```sh
ocluster-admin -c ~/admin.cap unpause freebsd-x86_64 rosemary
ocluster-admin -c ~/admin.cap show freebsd-x86_64 rosemary    # -> rosemary: Running
```

The backlog that accumulated while paused now drains.

### 7. Clean up and commit

```sh
rm deploy-targeted.yml          # don't commit the throwaway
git add playbook.yml
git commit -m "Update to OCaml X.Y.Z / ..."
git push -u origin <branch>
gh pr create --base master --title "..." --body "..."
```

## Bumping the FreeBSD release (e.g. 14.3 → 15.0)

When moving to a new FreeBSD release, also edit `roles/base-image/tasks/main.yml`:

- The `base_image` fact's release prefix (`freebsd-15.0-...`).
- The `bsdinstall` step's `BSDINSTALL_DISTSITE` URL (point at the new
  `…/<release>-RELEASE` path) and `DISTRIBUTIONS`.

This changes every dataset name, so all images are rebuilt from scratch (no
in-place reuse). Old `freebsd-<oldrelease>-ocaml-*` datasets are left behind —
remove them once the new generation is confirmed good:

```sh
ssh rosemary.caelum.ci.dev 'zfs destroy -R -r obuilder/base-image/freebsd-14.3-ocaml-4.14'  # repeat per stale dataset
```

## Related knobs and housekeeping

- **opam versions installed in the image** are controlled by the `for version in
  …` loop in the `base-image` role's setup script (`main.yml`), independent of
  the OCaml compiler version.
- **Cache prune threshold:** the worker runs with
  `--obuilder-prune-threshold=N` (set in `/usr/local/etc/rc.d/worker` on the
  host). `N` is a *free-space* percentage — obuilder prunes stored builds when
  **free space falls below N%**. Default is `30` (prune below 30% free ≈ 70%
  used). After editing, restart **only when the worker is idle**:
  `service worker restart`.
- **Stale scheduler registrations:** a decommissioned worker lingers under
  `disconnected:` in `ocluster-admin show <pool>`. Clear it with
  `ocluster-admin -c ~/admin.cap forget <pool> <worker>` (works while
  disconnected; if it's mid-reconnect, stop its service first).

## Quick reference

| Step | Command |
|------|---------|
| Pause + drain | `ocluster-admin -c ~/admin.cap pause --wait freebsd-x86_64 rosemary` |
| Pool status | `ocluster-admin -c ~/admin.cap show freebsd-x86_64` |
| Build (targeted) | `ansible-playbook -i hosts deploy-targeted.yml` |
| Build (full) | `ansible-playbook -i hosts playbook.yml` |
| List datasets | `ssh rosemary.caelum.ci.dev 'zfs list -r obuilder/base-image'` |
| Unpause | `ocluster-admin -c ~/admin.cap unpause freebsd-x86_64 rosemary` |
| Forget dead worker | `ocluster-admin -c ~/admin.cap forget freebsd-x86_64 <worker>` |
