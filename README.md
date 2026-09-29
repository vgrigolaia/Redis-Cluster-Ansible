# Redis Cluster PROD on Proxmox (AlmaLinux 10 + Redis Open Source 8.10.2)

This branch (`prod`) builds the **production** 6-node Redis Cluster: 3 masters + 3 replicas
on AlmaLinux 10 VMs in the Proxmox cluster `DC-A-PVE-11/12/13`, with disks on Ceph RBD.
A master and its own replica never run on the same Proxmox host, so losing any one host
never loses a shard.

The `dev` branch builds the DEV cluster with the same code in the same Proxmox cluster;
only the values differ (see [DEV vs PROD](#dev-vs-prod)).

---

## Contents

1. [Cluster at a glance](#cluster-at-a-glance)
2. [Architecture](#architecture)
3. [Repository layout](#repository-layout)
4. [Prerequisites](#prerequisites)
5. [Secrets (vault)](#secrets-vault)
6. [Build it step by step](#build-it-step-by-step)
7. [Verify and hand over](#verify-and-hand-over)
8. [Day-2 operations](#day-2-operations)
9. [Safety guarantees](#safety-guarantees)
10. [Troubleshooting](#troubleshooting)
11. [Destroy (PROD)](#destroy-prod)
12. [DEV vs PROD](#dev-vs-prod)

---

## Cluster at a glance

| VM ID | Name          | Proxmox host | IP           | Initial role  |
|-------|---------------|--------------|--------------|---------------|
| 291   | redis-prod-01 | DC-A-PVE-11  | 10.50.200.91 | master A      |
| 292   | redis-prod-02 | DC-A-PVE-12  | 10.50.200.92 | master B      |
| 293   | redis-prod-03 | DC-A-PVE-13  | 10.50.200.93 | master C      |
| 294   | redis-prod-04 | DC-A-PVE-11  | 10.50.200.94 | replica of C  |
| 295   | redis-prod-05 | DC-A-PVE-12  | 10.50.200.95 | replica of A  |
| 296   | redis-prod-06 | DC-A-PVE-13  | 10.50.200.96 | replica of B  |

| Item | Value |
|---|---|
| Template | **9020** `alma10-redis-prod-tpl` on DC-A-PVE-11 |
| Proxmox nodes | DC-A-PVE-11 / 12 / 13 = `10.50.200.11` / `.12` / `.13` |
| Network | `10.50.200.0/24`, gateway `10.50.200.1`, bridge `vmbr0` (untagged), DNS `1.1.1.1` |
| IP pool | `10.50.200.91-99`; `.91-.96` used, `.97-.99` spare |
| Storage | Ceph RBD storage `RBD-Disk` (shared by all 3 nodes) |
| VM size | 4 vCPU (`cpu: host`), 4 GB RAM (no ballooning), 10 GB disk |
| OS | AlmaLinux 10 GenericCloud 10.2 (sha256-verified), SELinux **enforcing** |
| Redis | Redis Open Source **8.10.2** from `packages.redis.io`, cluster mode, modules JSON/Bloom/TimeSeries/Search |
| Memory | `maxmemory 2560mb` per node, policy `noeviction` |
| Persistence | RDB snapshots + AOF (`appendfsync everysec`) |
| Auth | **Password required** (`requirepass` + `masterauth`), stored in the Ansible vault |
| Access | firewalld allows 6379 / 16379 only from `10.50.200.0/24` |
| Proxmox tag | `redis-prod` on the template and every VM |

---

## Architecture

```
                         10.50.200.0/24  (vmbr0, gw .1)
   ┌──────────────────────┬──────────────────────┬──────────────────────┐
   │ DC-A-PVE-11 (.11)    │ DC-A-PVE-12 (.12)    │ DC-A-PVE-13 (.13)    │
   │                      │                      │                      │
   │ 291 redis-prod-01    │ 292 redis-prod-02    │ 293 redis-prod-03    │
   │     master  A ───────┼──► 295 redis-prod-05 │     master  C        │
   │                      │     replica of A     │        │             │
   │ 294 redis-prod-04 ◄──┼──────────────────────┼────────┘             │
   │     replica of C     │     master  B ───────┼──► 296 redis-prod-06 │
   │                      │                      │     replica of B     │
   │ 9020 template        │                      │                      │
   └──────────┬───────────┴──────────┬───────────┴──────────┬───────────┘
              └──────── Ceph RBD pool / storage "RBD-Disk" ─┘
```

* **Shards:** 16384 hash slots split over masters A, B, C. Each replica lives on a
  *different* host than its master (checked by 05 and 99).
* **Failover:** if a host dies, the replicas of its masters (on the other two hosts)
  are promoted automatically after `cluster-node-timeout` (5 s).
* **Disks** are on shared Ceph, so `qm clone --target <node>` can place each VM directly
  on its host.

---

## Repository layout

```
ansible.cfg                         # inventory, vault password file (~/.ansible/redis-prod.vault-pass)
requirements.yml                    # ansible.posix (sysctl, firewalld)
site.yml                            # imports 01 → 05, then 99 (verify) and 98 (access page) last
inventory/
  hosts.yml                         # groups: proxmox, pve_template, redis_cluster (+ per-VM ip/vmid/node/role/shard)
  group_vars/all/vars.yml           # EVERY tunable: env_name, VMIDs, sizing, network, Redis
  group_vars/all/vault.yml          # ansible-vault encrypted secrets (see below)
  group_vars/proxmox/vars.yml       # SSH agent for the Proxmox nodes
  group_vars/redis_cluster/vars.yml # SSH user/become + host-key pinning for the VMs
playbooks/
  tasks/preflight.yml               # safety guard shared by 01 and 02
  tasks/live_placement.yml          # real VM -> Proxmox host map, used by 98 and 99
  01_template.yml                   # AlmaLinux cloud-init template on Ceph
  02_vms.yml                        # full clones, cloud-init network, start
  03_os.yml                         # OS tuning, firewalld, SELinux, THP off
  04_redis.yml                      # install + configure Redis (rolling)
  05_cluster.yml                    # create cluster + attach replicas per shard
  98_connection_info.yml            # read-only: developer handoff from the LIVE cluster
  99_verify.yml                     # health checks (+ optional failover test)
  templates/redis.conf.j2
  templates/connection-info.{html,md,env}.j2   # developer access page / handoff (98)
output/                             # generated by 98 (committed on purpose, never holds a password)
```

`env_name: prod` in `vars.yml` drives every environment-specific name: Proxmox tag
(`redis-prod`), template name, image cache dir, output file names, headings.

---

## Prerequisites

### Control node (your laptop)

* Ansible (ansible-core ≥ 2.16) and the collection:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
* Reachability: `10.50.200.11-13` (SSH as root) and `10.50.200.91-96` (SSH as `almalinux`).
* SSH key `~/.ssh/id_ed25519` (its `.pub` is injected into the VMs via cloud-init).
  The key has a passphrase and Ansible cannot prompt for it, so **load it into an agent in
  every new terminal** before running playbooks:
  ```bash
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
  ```
  (`~/.ssh/config` sends `Host *` to the 1Password agent; the inventory overrides that with
  `IdentityAgent=SSH_AUTH_SOCK`, i.e. the agent started above.)
* Vault password file `~/.ansible/redis-prod.vault-pass` (mode 600). See next section.

### Proxmox / Ceph (one-time, already done on this cluster)

* Storage `RBD-Disk` must be usable by **both** the Proxmox CLI and QEMU:
  `pvesm list RBD-Disk` works **and** a VM on it can start. See
  [Troubleshooting → Ceph](#ceph-rbd-disk-storage) for the two keyring problems this
  cluster had.
* Bridge `vmbr0` on all nodes; `iputils-arping` on all nodes (for the IP-conflict probe).
* Time sync (chrony/NTP) on all nodes — Ceph warns about clock skew > 0.05 s.
* The VMs need outbound internet (AlmaLinux mirrors, `packages.redis.io`) via `10.50.200.1`.

---

## Secrets (vault)

`inventory/group_vars/all/vault.yml` is **ansible-vault encrypted** with the PROD vault
password (a different, random password than DEV's). It contains:

| Variable | Used for |
|---|---|
| `vault_almalinux_password_hash` | SHA-512 crypt hash of the `almalinux` **console** password (cloud-init + `user` module). SSH stays key-only. |
| `vault_redis_password` | Redis `requirepass` / `masterauth` (48 hex chars). |

* The vault password lives in `~/.ansible/redis-prod.vault-pass` and is **never committed**
  (`.gitignore`). **Back it up** (e.g. in 1Password). Without it the vault cannot be decrypted.
* Show the secrets:
  ```bash
  ansible-vault view inventory/group_vars/all/vault.yml
  ```
* Edit them:
  ```bash
  ansible-vault edit inventory/group_vars/all/vault.yml
  ```
* New console password hash: `openssl passwd -6`, put it into the vault, run 03.
* Playbooks never print the Redis password: templating and `redis-cli --cluster` calls run
  with `no_log`, and `redis-cli` gets it through the `REDISCLI_AUTH` environment variable,
  never on the command line.

---

## Build it step by step

Always run the **check** (`--check --diff`) first, read it, then apply. Every playbook is
idempotent: re-running an applied playbook changes nothing.

Reading check-mode output: a `qm`/`redis-cli` step that **would run** shows as
`skipping … Command would have run if not in check mode` (`-v` shows it); a step that is
already done shows as `skipping … Conditional result was False`. Read-only steps
(`pvesh`, `qm config`, `cluster info`) really execute in check mode.

### 01 — template (on DC-A-PVE-11)

```bash
ansible-playbook playbooks/01_template.yml --check --diff
ansible-playbook playbooks/01_template.yml
```

Downloads the AlmaLinux image (sha256-verified) to `/var/tmp/redis-prod-image`, creates
VM 9020 (4 vCPU / 4 GB), imports the disk to `RBD-Disk`, adds the cloud-init drive
(user, password hash, SSH key, DNS), resizes to 10G and converts it to a template.
**Expect** at the end: `template: 1`, `scsi0: RBD-Disk:base-9020-disk-0,…size=10G`,
`ide2: RBD-Disk:vm-9020-cloudinit`, `tags: redis-prod`, no `onboot`.

### 02 — VMs

```bash
ansible-playbook playbooks/02_vms.yml --check --diff
ansible-playbook playbooks/02_vms.yml
```

ARP-probes every IP from its target node (aborts if anything answers), full-clones 291-296
directly onto their hosts, sets cloud-init IP/gateway/DNS, verifies the generated
cloud-init network, starts the VMs and waits for SSH. **Expect** the probe to say `FREE`
for all six and a placement table 291-296 on 11/12/13/11/12/13, all `running`.

### 03 — OS tuning

```bash
ansible-playbook playbooks/03_os.yml --check --diff
ansible-playbook playbooks/03_os.yml
```

Checks each VM is AlmaLinux 10 and that its hostname equals its VM name (IP-conflict
guard), installs qemu-guest-agent/firewalld/SELinux tools, sets `vm.overcommit_memory=1`
and `net.core.somaxconn=4096`, `LimitNOFILE=65535`, disables Transparent Huge Pages via
the kernel command line and **reboots each VM once, one at a time**, opens 6379/16379
only to `10.50.200.0/24`, labels the ports for SELinux.
**Expect** `Report` for all six: `THP=always madvise [never] | … | SELinux=enforcing`.

The first SSH connection pins each VM's host key into `~/.ssh/known_hosts_redis_prod`.

### 04 — Redis

```bash
ansible-playbook playbooks/04_redis.yml --check --diff
ansible-playbook playbooks/04_redis.yml
```

Adds the `packages.redis.io` repo (and excludes Valkey), installs `redis-8.10.2`, loads
the package's SELinux module if its post-install skipped it, then configures and starts
Redis **one node at a time** (`serial: 1`) and waits for `PONG` (authenticated).
**Expect** `redis-prod-0X: Redis server v=8.10.2 …` for all six and no SELinux denials.

### 05 — cluster

```bash
ansible-playbook playbooks/05_cluster.yml --check --diff
ansible-playbook playbooks/05_cluster.yml
```

From `redis-prod-01`: if the cluster is already healthy it does nothing; if all 3 masters
are fresh it creates the cluster with the masters only, then attaches each replica to
its designated master (by shard), waits for replication, and runs `--cluster check`.
Any other state **stops without changes**.
**Expect** `[OK] All 16384 slots covered.` and 3 masters + 3 replicas.

Everything at once (after you trust it): `ansible-playbook site.yml` runs 01 → 05, then
99 (verify) and finally 98, which generates the developer access page.

---

## Verify and hand over

```bash
ansible-playbook playbooks/99_verify.yml
ansible-playbook playbooks/98_connection_info.yml
```

* **99** asserts `cluster_state:ok`, 16384 slots, 6 connected nodes, 3+3 roles, every
  replica on a different host than its master (host read live from Proxmox, not from the
  inventory; it also fails if `pve_node` in `hosts.yml` has drifted); writes/reads test keys through different
  nodes and deletes them.
* **98** (also the last step of `site.yml`) reads the live cluster and writes:
  * `output/redis-prod-access.html`: the **developer access page**. Open it in a browser
    or share it: seed nodes with a copy button, live node table (role, slots, Proxmox
    host, link), client snippets with the password read from `REDIS_PASSWORD` (Spring
    Boot, Lettuce, Jedis, Python, ioredis, node-redis, Go, .NET, PHP, redis-cli), coding
    rules and production rules. A red banner appears if the cluster was not healthy, has
    nodes outside the inventory, or a replica shares a Proxmox host with its master.
  * `output/redis-prod-connection.md`: the same handoff as Markdown.
  * `output/redis-prod.env`: env vars for apps.

  All values come from the live cluster (roles after failovers, version, loaded modules,
  auth/TLS, firewall sources). The files **never contain the password**; give it to the
  app team through your secret store. Re-run 98 any time to refresh the page.

Seed nodes for clients (list all; the client discovers the topology):

```
10.50.200.91:6379,10.50.200.92:6379,10.50.200.93:6379,10.50.200.94:6379,10.50.200.95:6379,10.50.200.96:6379
```

Cluster mode, password required (no username = `default` user), no TLS.

Quick manual check from any VM (the password is read from the env, not typed on the
command line):

```bash
export REDISCLI_AUTH='<password from the vault>'
redis-cli -c -h 10.50.200.91 cluster info | head -7
```

### Failover test (before go-live only)

```bash
ansible-playbook playbooks/99_verify.yml -e run_failover_test=true
```

Hard-stops **redis-prod-01** (VM 291), confirms **redis-prod-05** is promoted and the
cluster stays `ok`, starts 291 again, waits until it rejoins as replica, then
`CLUSTER FAILOVER` makes it master again. This causes a few seconds of errors for shard A
clients — **do not run it on a live PROD cluster outside a maintenance window.**

---

## Day-2 operations

| Task | How |
|---|---|
| Change a Redis setting | Edit `vars.yml` (or `redis.conf.j2`), run 04 with `--check --diff`, then 04. Restart is rolling, one node at a time, only where the config changed. |
| Change VM size | `qm set <vmid> --cores N --memory MB` on the owning node, one VM at a time, then reboot that VM; update `vm_cores`/`vm_memory_mb` so new builds match. Adjust `redis_maxmemory` (~60% of RAM). |
| Grow the disk | `qm disk resize <vmid> scsi0 +10G`, then in the VM `growpart`/`xfs_growfs` (cloud-utils-growpart). Update `vm_disk_size`. |
| Rotate the Redis password | `ansible-vault edit …` → new `vault_redis_password`, then run 04. During the rolling restart nodes briefly disagree on the password (replication and clients see auth errors), so do it in a **maintenance window** and update the apps right after. |
| OS updates | Patch one VM at a time (`dnf upgrade`, reboot), wait for `cluster_state:ok` and `master_link_status:up` before the next. Never reboot a master and its replica together. |
| Move a VM to another Proxmox host | Migrate it, update `pve_node` in `hosts.yml`, keep a shard's master and replica on different hosts, then run 99 (it reads the real host from Proxmox and fails on a shared host or a stale inventory). Until the inventory is updated, 01/02 abort on purpose. |
| After a failover | Nothing to do — 05/99 accept a shard in either direction. To restore the original layout run `CLUSTER FAILOVER` on the original master once it is a replica again. |
| Rebuild a single VM | Not automated on purpose. Destroy it manually (see below), run 02-04, then `redis-cli --cluster add-node … --cluster-slave` or `cluster forget` the old ID first. Remove its old host key: `ssh-keygen -R <ip> -f ~/.ssh/known_hosts_redis_prod`. |

Persistence note: data dir `/var/lib/redis` holds `dump.rdb` and `appendonlydir/`.
With 10 GB disks and `maxmemory 2560mb`, an AOF rewrite can temporarily need ~3× the
dataset on disk; watch `df -h /` if the dataset gets close to the limit.

---

## Safety guarantees

* **VMID allow-list:** `pve_managed_vmids: [9020, 291-296]` in `vars.yml` is the complete
  list of VMs these playbooks may touch. It is written out, not derived, so a typo in
  `hosts.yml` cannot redirect them. DEV (9010, 270-275) is outside it.
* **Preflight** (01, 02) reads `/cluster/resources` and **aborts** if any of those IDs
  belongs to a VM with a different name, on a different node, or without the
  `redis-prod` tag. It also checks that `RBD-Disk` exists, can really be listed (Ceph
  auth works) and that `vmbr0` exists on every node. When it aborts, nothing has changed.
* **IP conflicts:** 02 ARP-probes (`arping -D`) each IP before starting a VM; 03 refuses
  to touch a host whose hostname is not the expected VM name.
* **No destructive automation:** nothing stops, migrates or deletes a VM, except the
  opt-in failover test, which re-checks the VM name on Proxmox first.
* **Host keys pinned:** PROD VM host keys are accepted once and then enforced.
* **Cluster creation** only happens when all 3 masters are empty and standalone; any
  partial state stops the run.

---

## Troubleshooting

Problems seen while building this environment, and their fixes.

### SSH: `ssh_askpass … No such file` / `Permission denied (publickey)`
The key's passphrase cannot be typed into Ansible. Start an agent and add the key in the
same terminal (`eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ed25519`); `ssh-add -l` must list it.

### Ceph (`RBD-Disk` storage)
1. `Not a proper rbd authentication file: /etc/pve/priv/ceph/RBD-Disk.keyring` —
   Proxmox's format check expects the key to end in `==`, but this Ceph creates the
   newer, longer keys (60 chars, ending in a single `=`). For this hyper-converged
   (Proxmox-managed) Ceph the storage keyring is not needed; moving it away makes Proxmox
   use the keyring from `/etc/pve/ceph.conf`:
   `mv /etc/pve/priv/ceph/RBD-Disk.keyring /root/RBD-Disk.keyring.bak`
2. `rbd: listing images failed: (13) Permission denied` — Ceph itself refused the login.
   Check `ceph auth get client.<user>` caps and the `username` in `/etc/pve/storage.cfg`.
3. VM start fails: `blockdev … "user":"pve-rbd" … error connecting: No such file or directory` —
   the storage uses Ceph user `pve-rbd`, and QEMU looks for
   `/etc/pve/priv/ceph.client.pve-rbd.keyring` (path from `ceph.conf`). Create it:
   `ceph auth get client.pve-rbd -o /etc/pve/priv/ceph.client.pve-rbd.keyring`
   and test with `rbd -c /etc/pve/ceph.conf --id pve-rbd -p RBD-Disk ls`.

Preflight now runs `pvesm list RBD-Disk`, so auth problems stop 01/02 before any change.

### `qm set … hotplug problem - adding blockdev 'drive-scsi0' failed`
The template VM was running (an old version created it with `onboot=1`). 01 now never sets
`onboot` on the template and refuses to continue while it is running: `qm stop 9020`, re-run.

### 03: `firewall is not currently running`
firewalld had just started and was not ready on D-Bus. 03 now waits for
`firewall-cmd --state` = `running`. If a host failed before its THP reboot, re-running 03
still reboots it (it checks the running kernel's `/proc/cmdline`, not only the boot config).

### 02: IP conflict probe says `IN_USE`
Something already answers on that IP. Do not continue; pick a free IP from `.97-.99`,
change `ansible_host`/`ip` in `hosts.yml`, re-run 02.

### Proxmox nodes: clock skew / `Host key verification failed` between nodes
Not needed by these playbooks, but fix them for Ceph and migrations: configure chrony on
all nodes (`chronyc tracking` must show a reference), and `pvecm updatecerts` for SSH trust.

---

## Destroy (PROD)

> ⚠️ This deletes production data. Make sure nothing uses the cluster any more.
> Run manually on any Proxmox node.

1. Confirm the IDs are still ours. Every line must say `OK`:

```bash
pvesh get /cluster/resources --type vm --output-format json | python3 -c '
import sys, json
want = {9020:"alma10-redis-prod-tpl",291:"redis-prod-01",292:"redis-prod-02",293:"redis-prod-03",294:"redis-prod-04",295:"redis-prod-05",296:"redis-prod-06"}
for v in sorted(json.load(sys.stdin), key=lambda v: v["vmid"]):
    if v["vmid"] in want:
        ok = v["name"] == want[v["vmid"]] and "redis-prod" in v.get("tags", "").split(";")
        print(v["vmid"], v["node"], v["name"], v["status"], "OK" if ok else "MISMATCH - DO NOT DELETE")'
```

2. Stop and delete the VMs (`pvesh` works for any node from any node):

```bash
for p in DC-A-PVE-11:291 DC-A-PVE-12:292 DC-A-PVE-13:293 DC-A-PVE-11:294 DC-A-PVE-12:295 DC-A-PVE-13:296; do
  n=${p%%:*}; id=${p##*:}
  pvesh create /nodes/$n/qemu/$id/status/stop
  pvesh delete /nodes/$n/qemu/$id --purge 1 --destroy-unreferenced-disks 1
done
```

3. Optionally the template and the cached image (on DC-A-PVE-11):

```bash
pvesh delete /nodes/DC-A-PVE-11/qemu/9020 --purge 1 --destroy-unreferenced-disks 1
rm -rf /var/tmp/redis-prod-image
```

4. On the control node, forget the pinned host keys:

```bash
rm -f ~/.ssh/known_hosts_redis_prod
```

---

## DEV vs PROD

Same code, different values. Each branch carries its own `hosts.yml`, `vars.yml`,
`vault.yml` and `ansible.cfg`; switching branches (`git checkout dev` / `git checkout prod`)
switches the whole environment, including which vault password file is used.

| Setting | `dev` branch | `prod` branch |
|---|---|---|
| `env_name` / Proxmox tag | dev / `redis-dev` | prod / `redis-prod` |
| Template | 9010 `alma10-redis-dev-tpl` | 9020 `alma10-redis-prod-tpl` |
| VMs | 270-275 `redis-dev-01..06` | 291-296 `redis-prod-01..06` |
| IPs | 10.50.200.3-8 (pool .3-.10) | 10.50.200.91-96 (pool .91-.99) |
| vCPU / RAM / disk | 1 / 1 GB / 10 GB | 4 / 4 GB / 10 GB |
| `maxmemory` | 256mb | 2560mb |
| Redis password | none | required (vault) |
| Vault password file | `~/.ansible/redis-dev.vault-pass` | `~/.ansible/redis-prod.vault-pass` |
| VM host keys | not stored (rebuilt often) | pinned in `~/.ssh/known_hosts_redis_prod` |

Both clusters share the Proxmox cluster, `RBD-Disk`, `vmbr0` and the `10.50.200.0/24`
network; neither set of playbooks can touch the other's VMIDs.
