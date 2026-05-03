---
title: "SELinux MCS challenges with GitLab Runners"
date: 2026-05-01T21:00:00-04:00
type: post
url: /2026/05/01/selinux-mcs-challenges-gitlab-runners/
categories:
  - Planet GNOME
  - Syseng
tags:
  - planet-gnome
  - syseng
  - gitlab
  - selinux
  - podman
  - microvms
  - security

---
## Table of Contents
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [The MCS problem](#the-mcs-problem)
- [The test script](#the-test-script)
- [GitLab's official suggestion and why it falls short](#gitlabs-official-suggestion-and-why-it-falls-short)
- [How GNOME currently handles this](#how-gnome-currently-handles-this)
- [Exploring libkrun](#exploring-libkrun)
- [Firecracker and the custom executor path](#firecracker-and-the-custom-executor-path)
- [What comes next](#what-comes-next)

## Introduction

GNOME's GitLab runners use Podman as the container runtime with SELinux in Enforcing mode on Fedora. The GitLab Runner Docker/Podman executor spawns multiple containers per job: a **helper** container that clones the repository and handles artifacts, and a **build** container that runs the actual CI script. Both containers need to share a `/builds` volume — and this is where SELinux's **Multi-Category Security (MCS)** becomes a problem.

## The MCS problem

An SELinux label has four fields: `user:role:type:level`. For containers the interesting part is the **level**, also called the MCS field. A level looks like `s0:c123,c456` — `s0` is the sensitivity (always `s0` in targeted policy), and `c123,c456` are the **categories**. A process or file can carry up to two categories.

MCS access is based on **dominance**. A subject's label dominates an object's label if the subject's categories are a superset of (or equal to) the object's categories:

| Subject | Object | Access? | Why |
|---|---|---|---|
| `s0:c100,c200` | `s0:c100,c200` | Yes | Exact match |
| `s0:c100,c200` | `s0:c100` | Yes | Subject's categories are a superset |
| `s0:c100,c200` | `s0:c100,c300` | No | Subject lacks `c300` |
| `s0:c0.c1023` | `s0:c100,c200` | Yes | Full range dominates everything |
| `s0` | `s0:c100,c200` | No | No categories can't dominate any |
| `s0` | `s0` | Yes | Both have no categories |

How this applies to the runners:

- Container A runs as `container_t:s0:c100,c100` — it can only access objects labeled `s0:c100,c100` (or `s0:c100`, or `s0`)
- Container B runs as `container_t:s0:c200,c200` — it can only access objects labeled `s0:c200,c200` (or `s0:c200`, or `s0`)
- Container A cannot access Container B's files — `c100,c100` doesn't dominate `c200,c200`
- Overlay layers labeled `s0` (no categories) — accessible by all containers since every category set dominates the empty set
- Podman at `container_runtime_t:s0-s0:c0.c1023` — the full range means it dominates every possible category combination, so it can manage all containers

The **range syntax** (`s0-s0:c0.c1023`) is used for processes that need to operate across multiple levels. It means "my low clearance is `s0` and my high clearance is `s0:c0.c1023`." The process can read objects at any level within that range and create objects at any level within it. This is why Podman needs the full range — it creates containers with different MCS labels and needs to access all of them.

When Podman starts a container, it picks a random pair of categories (e.g., `s0:c512,c768`) from within its allowed range and assigns that as the container's process label. Files created by the container inherit that label. Another container gets a different random pair (e.g., `s0:c33,c901`). Since `c512,c768` and `c33,c901` do not match — neither is a superset of the other — SELinux denies cross-container file access. This is the isolation mechanism, and the root cause of the problem with GitLab Runner's multi-container-per-job architecture.

The helper container gets one random MCS pair, writes the cloned repo to `/builds` labeled with that pair, and the build container gets a different pair. The build container cannot read or write those files. The `:Z` volume flag (exclusive relabel) relabels the volume to the mounting container's category, but that only helps the first container — the second one still has a different label.

## The test script

I wrote a script that demonstrates the problem with both standard containers (crun) and microVMs (libkrun). The script creates two containers per test — a helper that writes a file to a shared `/builds` volume, and a build container that tries to read it — simulating the GitLab Runner workflow:

```bash
#!/bin/bash
# Description: SELinux MCS Diagnostic (crun vs krun)

if [ "$(getenforce)" != "Enforcing" ]; then
    echo "WARNING: SELinux is not in Enforcing mode. This test requires Enforcing mode."
    exit 1
fi

TEST_BASE="/tmp/gitlab-runner-mcs-test"
CRUN_DIR="$TEST_BASE/crun-builds"
KRUN_DIR="$TEST_BASE/krun-builds"

# Cleanup from previous runs
rm -rf "$TEST_BASE"
mkdir -p "$CRUN_DIR" "$KRUN_DIR"

echo "======================================================="
echo " TEST 1: Standard Container Isolation (crun)"
echo "======================================================="

# 1. CREATE Helper
podman create --name crun-helper -v "$CRUN_DIR:/builds:Z" fedora bash -c "
    echo '[crun] -> Helper Process Context (Inside):'
    cat /proc/self/attr/current
    echo 'crun-data' > /builds/artifact.txt
    echo '[crun] -> File Label INSIDE Helper:'
    ls -Z /builds/artifact.txt
" > /dev/null

echo "[crun] Starting Helper Container (applying :Z relabel)..."
HELPER_HOST_LABEL_CRUN=$(podman inspect -f '{{.ProcessLabel}}' crun-helper)
echo "[crun] -> HOST METADATA: Podman assigned process label: $HELPER_HOST_LABEL_CRUN"
podman start -a crun-helper

echo ""
echo "[crun] -> File Label ON HOST (Notice the specific MCS category):"
ls -Z "$CRUN_DIR/artifact.txt"

# 2. CREATE Build Container (The Victim)
podman create --name crun-build -v "$CRUN_DIR:/builds" fedora bash -c "
    echo '  [Build-Internal] Process Context:'
    cat /proc/self/attr/current 2>/dev/null
    echo '  [Build-Internal] Executing ls -laZ /builds :'
    ls -laZ /builds 2>&1 | sed 's/^/    /'
    echo '  [Build-Internal] Executing cat /builds/artifact.txt :'
    cat /builds/artifact.txt 2>&1 | sed 's/^/    /'
" > /dev/null

echo ""
echo "[crun] Starting Build Container to inspect shared volume..."
BUILD_HOST_LABEL_CRUN=$(podman inspect -f '{{.ProcessLabel}}' crun-build)
echo "[crun] -> HOST METADATA: Podman assigned process label: $BUILD_HOST_LABEL_CRUN"
podman start -a crun-build

podman rm -f crun-helper crun-build > /dev/null


echo ""
echo "======================================================="
echo " TEST 2: MicroVM Isolation (libkrun / virtio-fs) FIXED"
echo "======================================================="

# --- Write the execution scripts to the host to avoid parsing errors ---
cat << 'EOF' > "$TEST_BASE/krun_helper.sh"
#!/bin/bash
echo '[krun] -> Helper Process Context (Inside VM):'
cat /proc/self/attr/current 2>/dev/null || echo '    (SELinux disabled/unavailable in guest kernel)'
echo 'krun-data' > /builds/artifact.txt
echo '[krun] -> File Label INSIDE Helper VM (Blindspot):'
ls -laZ /builds/artifact.txt 2>&1 | sed 's/^/    /'
EOF

cat << 'EOF' > "$TEST_BASE/krun_build.sh"
#!/bin/bash
echo '  [Build-Internal] Process Context (Inside VM):'
cat /proc/self/attr/current 2>/dev/null || echo '    (SELinux disabled/unavailable in guest kernel)'
echo '  [Build-Internal] Executing ls -laZ /builds :'
ls -laZ /builds 2>&1 | sed 's/^/    /'
echo '  [Build-Internal] Executing cat /builds/artifact.txt :'
cat /builds/artifact.txt 2>&1 | sed 's/^/    /'
EOF

chmod +x "$TEST_BASE/krun_helper.sh" "$TEST_BASE/krun_build.sh"
# ---------------------------------------------------------------------

# 1. CREATE Helper MicroVM
podman create --name krun-helper --runtime krun --memory=1024m \
    -v "$KRUN_DIR:/builds:Z" \
    -v "$TEST_BASE/krun_helper.sh:/script.sh:ro,Z" \
    fedora /script.sh > /dev/null

echo "[krun] Starting Helper MicroVM (applying :Z relabel)..."
HELPER_HOST_LABEL_KRUN=$(podman inspect -f '{{.ProcessLabel}}' krun-helper)
echo "[krun] -> HOST METADATA: Podman assigned process label: $HELPER_HOST_LABEL_KRUN"
podman start -a krun-helper

echo ""
echo "[krun] -> File Label ON HOST (Podman applied the helper's MCS category via :Z):"
ls -Z "$KRUN_DIR/artifact.txt"

# 2. CREATE Build MicroVM (The Victim)
podman create --name krun-build --runtime krun --memory=1024m \
    -v "$KRUN_DIR:/builds" \
    -v "$TEST_BASE/krun_build.sh:/script.sh:ro,Z" \
    fedora /script.sh > /dev/null

echo ""
echo "[krun] Starting Build MicroVM to inspect shared volume..."
BUILD_HOST_LABEL_KRUN=$(podman inspect -f '{{.ProcessLabel}}' krun-build)
echo "[krun] -> HOST METADATA: Podman assigned process label: $BUILD_HOST_LABEL_KRUN"
echo "        *** THE virtiofsd DAEMON ON THE HOST IS TRAPPED IN THIS CONTEXT ***"
podman start -a krun-build

# Cleanup
podman rm -f krun-helper krun-build > /dev/null

echo ""
echo "======================================================="
echo " Test Complete."
```

**Test 1 (crun)** creates a helper container that mounts the builds directory with `:Z` (exclusive relabel) and writes `artifact.txt`. Podman assigns it a random MCS label — in this run it was `s0:c20,c540`. The file on disk inherits that label. Then a second container (the build container) mounts the same path without `:Z` and gets a different random label (`s0:c46,c331`). Since `c46,c331` does not dominate `c20,c540`, the build container is denied access to the file.

**Test 2 (krun)** runs the same scenario but with `--runtime krun`, which boots each container inside a lightweight microVM via [libkrun](https://github.com/containers/libkrun). The helper VM gets `container_kvm_t:s0:c823,c999` and the build VM gets `container_kvm_t:s0:c309,c405` — same MCS mismatch, same denial. The type changes from `container_t` to `container_kvm_t`, but the MCS mechanism is identical. On the host side, `virtiofsd` — the daemon that serves the volume into the VM via virtio-fs — runs under the MCS label Podman assigned to the VM. The build VM's `virtiofsd` is trapped in `s0:c309,c405` and cannot access files labeled `s0:c823,c999`.

An interesting detail: inside the libkrun VMs, `cat /proc/self/attr/current` returns just `kernel` — SELinux is not available in the guest. The VM thinks it has no mandatory access control, but the host-side `virtiofsd` is still fully subject to MCS enforcement. This is a blindspot worth being aware of.

The output from a run on Fedora with SELinux Enforcing and Podman 5.8.2:

```
=======================================================
 TEST 1: Standard Container Isolation (crun)
=======================================================
[crun] Starting Helper Container (applying :Z relabel)...
[crun] -> HOST METADATA: Podman assigned process label: system_u:system_r:container_t:s0:c20,c540
[crun] -> Helper Process Context (Inside):
system_u:system_r:container_t:s0:c20,c540 [crun] -> File Label INSIDE Helper:
system_u:object_r:container_file_t:s0:c20,c540 /builds/artifact.txt

[crun] -> File Label ON HOST (Notice the specific MCS category):
system_u:object_r:container_file_t:s0:c20,c540 /tmp/gitlab-runner-mcs-test/crun-builds/artifact.txt

[crun] Starting Build Container to inspect shared volume...
[crun] -> HOST METADATA: Podman assigned process label: system_u:system_r:container_t:s0:c46,c331
        *** COMPARE THE cXXX,cYYY ABOVE TO THE FILE LABEL. THIS MISMATCH CAUSES THE DENIAL ***
  [Build-Internal] Process Context:
system_u:system_r:container_t:s0:c46,c331   [Build-Internal] Executing ls -laZ /builds :
    ls: cannot open directory '/builds': Permission denied
  [Build-Internal] Executing cat /builds/artifact.txt :
    cat: /builds/artifact.txt: Permission denied

=======================================================
 TEST 2: MicroVM Isolation (libkrun / virtio-fs) FIXED
=======================================================
[krun] Starting Helper MicroVM (applying :Z relabel)...
[krun] -> HOST METADATA: Podman assigned process label: system_u:system_r:container_kvm_t:s0:c823,c999
[krun] -> Helper Process Context (Inside VM):
kernel [krun] -> File Label INSIDE Helper VM (Blindspot):
    -rw-r--r--. 1 root root system_u:object_r:container_file_t:s0:c823,c999 10 May  2  2026 /builds/artifact.txt

[krun] -> File Label ON HOST (Podman applied the helper's MCS category via :Z):
system_u:object_r:container_file_t:s0:c823,c999 /tmp/gitlab-runner-mcs-test/krun-builds/artifact.txt

[krun] Starting Build MicroVM to inspect shared volume...
[krun] -> HOST METADATA: Podman assigned process label: system_u:system_r:container_kvm_t:s0:c309,c405
        *** THE virtiofsd DAEMON ON THE HOST IS TRAPPED IN THIS CONTEXT ***
  [Build-Internal] Process Context (Inside VM):
kernel   [Build-Internal] Executing ls -laZ /builds :
    ls: /builds: Permission denied
    ls: cannot open directory '/builds': Permission denied
  [Build-Internal] Executing cat /builds/artifact.txt :
    cat: /builds/artifact.txt: Permission denied

=======================================================
 Test Complete.
```

## GitLab's official suggestion and why it falls short

GitLab's documentation on [configuring SELinux MCS](https://docs.gitlab.com/runner/executors/docker/#configure-selinux-mcs) suggests applying the same MCS label to all containers launched by a runner:

```toml
[[runners]]
  [runners.docker]
    security_opt = ["label=level:s0:c1000,c1000"]
```

This works — all containers get the same category pair, so the helper and build containers can share files. But it collapses MCS isolation between all concurrent jobs on that runner. With `concurrent = 4`, four simultaneous jobs all run as `s0:c1000,c1000` and can read each other's `/builds` content — cloned source code, build artifacts, cached dependencies. On a shared or multi-tenant runner, this is a security regression: it trades MCS isolation for functionality.

For runners with `concurrent = 1` or dedicated single-tenant runners this is an acceptable tradeoff, but it does not generalize to shared infrastructure where multiple untrusted projects run side by side.

## How GNOME currently handles this

GNOME's runners are managed via an [Ansible role](https://gitlab.gnome.org/GNOME/ansible/-/tree/master/roles/gitlab-runner) that enforces SELinux in Enforcing mode, installs rootless Podman running as a dedicated `podman` system user with linger enabled, and deploys custom SELinux policy modules. The Podman service runs under `SELinuxContext=system_u:system_r:container_runtime_t:s0-s0:c0.c1023` via a systemd override — the full MCS range (`s0-s0:c0.c1023`) gives the container runtime the ability to spawn containers at any MCS level and relabel volumes accordingly, as explained in the dominance rules above.

Four custom SELinux `.te` modules are compiled and loaded on every runner host: `pydocuum` (allows the image cleanup daemon to talk to the Podman socket), `podman` (grants `user_namespace create` and `/dev/null` mapping), `flatpak` (permits the filesystem mounts flatpak builds need), and `gnome_runner` (covers `binfmt_misc` access, device nodes, and other permissions GNOME OS builds require).

For the MCS problem specifically, the runner `config.toml` — rendered from a Jinja2 template via per-host Ansible variables — sets a fixed MCS label per runner type. Here's a representative snippet from one of the runner hosts:

```toml
[[runners]]
  name = "a15948139c78"
  executor = "docker"
  [runners.docker]
    image = "quay.io/fedora/fedora:latest"
    privileged = false
    security_opt = ["label=level:s0:c100,c100"]
    devices = ["/dev/kvm", "/dev/udmabuf"]
    cap_add = ["SYS_PTRACE", "SYS_CHROOT"]

[[runners]]
  name = "a15948139c78-flatpak"
  executor = "docker"
  [runners.docker]
    image = "quay.io/gnome_infrastructure/gnome-runtime-images:gnome-master"
    privileged = false
    security_opt = ["seccomp:/home/podman/gitlab-runner/flatpak.seccomp.json", "label=level:s0:c200,c200"]
    cap_drop = ["all"]
```

This is the same approach GitLab's documentation suggests, with one refinement: we use **different fixed categories per runner type** — `c100,c100` for untagged runners and `c200,c200` for flatpak runners — so that flatpak builds and regular builds remain MCS-isolated from each other, even though builds of the same type share a category.

This is a pragmatic compromise, not an ideal solution. All concurrent jobs on the same runner type share the same MCS category. With `concurrent: 4` on our Hetzner runners, four simultaneous untagged jobs can read each other's `/builds` content. For GNOME's use case — a community CI infrastructure where the runners are shared by GNOME project maintainers — this is an acceptable tradeoff. The alternative, leaving MCS labels random, would break every single job. But it is precisely this tradeoff that motivates exploring per-job VM isolation via microVMs.

## Exploring libkrun

[libkrun](https://github.com/containers/libkrun) is a lightweight Virtual Machine Monitor (VMM) that integrates with Podman via `--runtime krun`, running each container inside a microVM with its own lightweight kernel. The appeal is strong: per-container VM isolation would give each job its own kernel and address space, making the MCS cross-container problem irrelevant inside the VM.

I tested libkrun on a Fedora system and hit an immediate blocker: `Fatal glibc error: rseq registration failed`. The **rseq** (Restartable Sequences) syscall was introduced in Linux kernel 5.3 and is required by glibc >= 2.35. libkrun uses a custom minimal kernel that does not expose `rseq` support. Since the guest images — Fedora in our case — ship modern glibc that expects `rseq` to be available, the process aborts at startup before any user code runs.

The libkrun kernel is compiled into the library itself and cannot be modified or replaced by the user. This is not a configuration issue but a fundamental limitation of the current libkrun release.

Even if the `rseq` issue were resolved, the MCS challenge would still be there — as the test script demonstrates in Test 2. On the host side, Podman assigns MCS labels to the `virtiofsd` process that serves the volume into the VM via virtio-fs. Different VMs get different host-side MCS labels, meaning the same `:Z` relabel / cross-container access denial applies. The mechanism changes from overlay mounts to virtio-fs, but the SELinux enforcement is identical: `virtiofsd` for the build VM runs at `container_kvm_t:s0:c309,c405` and cannot access files labeled `s0:c823,c999` by the helper VM's `virtiofsd`.

## Firecracker and the custom executor path

[Firecracker](https://firecracker-microvm.github.io/) is another microVM technology, the one behind AWS Lambda and Fly.io, that could provide strong per-job isolation. However, there is no native GitLab Runner executor for Firecracker. The only integration path is the [Custom Executor](https://docs.gitlab.com/runner/executors/custom.html), which requires implementing `prepare`, `run`, and `cleanup` scripts from scratch.

The job image is exposed via `CUSTOM_ENV_CI_JOB_IMAGE`, but everything else is on the operator: pulling the OCI image, extracting a rootfs, booting a Firecracker VM with the right kernel and network configuration, injecting the build script, mounting or copying the cloned repository into the VM, collecting artifacts and cache after the job finishes, and tearing the VM down. GitLab provides an [LXD-based example](https://docs.gitlab.com/runner/executors/custom_examples/lxd/) that shows the pattern — `prepare` creates a container and installs dependencies, `run` pipes the job script into it, `cleanup` destroys it — but adapting that to microVMs adds the complexity of VM lifecycle management, kernel and rootfs preparation, networking, and storage. This is a significant engineering effort, essentially rebuilding the entire Docker executor workflow from scratch.

## What comes next

MCS is a core SELinux feature. Type enforcement (TE) already confines processes by type — `container_t` can only access `container_file_t`, not `user_home_t` or `httpd_sys_content_t` — but TE alone cannot distinguish one `container_t` process from another. MCS adds that layer: by assigning each container a unique category pair, the kernel enforces isolation *between* processes that share the same type. Container A at `s0:c100,c100` and Container B at `s0:c200,c200` are both `container_t`, but MCS ensures they cannot touch each other's files. The conflict with GitLab Runner's multi-container-per-job architecture is that two containers that *need* to share a volume are given different categories by default. The workarounds we deploy today, including the fixed MCS labels on GNOME's runners, trade that inter-container isolation for functionality.

The most promising direction I've found so far is the combination of [Cloud Hypervisor](https://www.cloudhypervisor.org/) and the [fleeting-plugin-fleetingd](https://github.com/helmholtzcloud/fleeting-plugin-fleetingd) plugin. Cloud Hypervisor is built on Intel's Rust-VMM crate and is essentially a more capable sibling of Firecracker — it supports CPU and memory hotplugging, VFIO device passthrough, and virtio-fs, features that are often necessary for complex CI tasks like building large binaries or running UI tests and that Firecracker's minimalist design deliberately omits. The fleeting-plugin-fleetingd is a community plugin for GitLab's **Instance Executor** (the modern evolution of the Custom Executor) that automates the full VM lifecycle: downloading cloud images, creating Copy-on-Write disks, launching Cloud Hypervisor VMs with direct kernel boot, provisioning them via cloud-init, and tearing them down after each build. Each job gets a fresh disposable VM, which is exactly the per-job isolation model we need. The plugin already handles networking via TAP interfaces and nftables SNAT, and supports customization of the VM image through cloud-init commands — so preinstalling Podman or other build tools is straightforward.

Beyond that, I'll also keep evaluating libkrun (promising Red Hat technology), Firecracker with a hand-rolled custom executor, and QEMU's microvm machine type. The common denominator across all of these — except for the fleeting-plugin-fleetingd path — is that none of them have an existing GitLab Runner integration. Regardless of which microVM technology we settle on, the path forward involves either building a workflow from scratch using the [Custom Executor](https://docs.gitlab.com/runner/executors/custom.html) and its `prepare`, `run`, `cleanup` hooks, or leveraging the fleeting plugin ecosystem that GitLab has been building around the Instance and Docker Autoscaler executors.

### CVE-2026-31431

The urgency of per-job VM isolation was underscored by [CVE-2026-31431](https://www.helpnetsecurity.com/2026/04/30/copyfail-linux-lpe-vulnerability-cve-2026-31431/) ("Copy Fail"), a nine-year-old logic bug in the kernel's `algif_aead` cryptographic module disclosed at the end of April. The flaw lets an unprivileged local user write four controlled bytes into the page cache of any readable file — enough to patch a setuid binary like `/usr/bin/su` and escalate to root. Unlike Dirty Cow or Dirty Pipe, Copy Fail requires no race condition: the exploit is deterministic, leaves no trace on disk, and — critically — can break out of container isolation. In a shared-runner CI environment, any project that can execute arbitrary code in a job already has exactly the access the exploit needs. Separately, [Claude Mythos](https://www.computing.co.uk/analysis/2026/claude-mythos-how-ai-broke-out-of-its-sandbox) — an Anthropic model trained for cybersecurity research that escaped its own sandbox during a red-team exercise in April — demonstrated that AI-assisted vulnerability discovery and exploitation is no longer theoretical; models can now autonomously find and chain bugs that would take human researchers weeks to exploit. The combination of a reliable, public kernel LPE and AI-augmented offensive tooling makes the case for ephemeral microVMs compelling: when every CI job boots a fresh, disposable VM with its own kernel, a vulnerability like Copy Fail becomes a local-root inside a throwaway guest that is destroyed seconds later, not a stepping stone to the host or adjacent jobs.

That should be all for today, stay tuned!
