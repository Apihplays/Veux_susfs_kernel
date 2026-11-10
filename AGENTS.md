# AGENTS.md — Veux SuSFS Kernel CI/CD

Guidelines for AI agents and maintainers operating on `Apihplays/Veux_susfs_kernel`.

## Repository Context

Automated CI/CD for building non-GKI Android kernels integrated with **SuSFS**, **KernelSU forks** (BakaSU/ReSukiSU, xxKSU), and device patches.
- **Primary Target**: Xiaomi Redmi Note 11 Pro 5G / POCO X4 Pro 5G (`veux`/`peux`, SM6375, Kernel 5.4).
- **Upstream Parent**: `cyberc3dr/nGKI_Kernel_Build` (default branch: `rebase`).
- **Patch Ecosystem**: `JackA1ltman/NonGKI_Kernel_Patches` & local `Patches/`.

---

## Branch Architecture

| Branch | Purpose | Upstream Tracking |
|---|---|---|
| `rebase` | Upstream mirror & default branch. Required for GitHub Actions workflow registration. | Tracks `upstream/rebase`. |
| `backup-rebase` | Active working branch containing custom device workflows, compatibility fixes, and local patches. | Diverged base with preserved historical commits. |
| `ak3` | Orphan branch containing the custom AnyKernel3 zip template ("positron kernel" branding). | Standalone, no upstream. |

---

## Upstream Sync Invariants

1. **Never Hard Reset Without Backup**:
   - Upstream frequently rebases history.
   - Always snapshot before syncing:
     ```bash
     git branch backup-pre-sync origin/rebase
     git push origin backup-pre-sync
     ```
2. **Workflow Registration Constraint (GitHub Actions)**:
   - `workflow_dispatch` **requires** the workflow YAML to exist on the default branch (`rebase`) to be indexed.
   - When introducing or modifying a workflow in `backup-rebase`:
     ```bash
     git checkout rebase -- .github/workflows/<workflow-name>.yml
     git commit -m "workflows: register <workflow-name>" && git push origin rebase
     ```
   - Then trigger against `backup-rebase`:
     ```bash
     gh workflow run <workflow-name>.yml --ref backup-rebase
     ```

---

## Workflow Conventions & Envs

Workflow files reside in `.github/workflows/` and invoke composite actions in subdirectories:
- `build-env`: Sets up runner OS (Ubuntu 24.04/22.04), Coccinelle, libufdt, Python.
- `build-ready`: Clones kernel, toolchains, AnyKernel3, and KernelSU.
- `patch-susfs`: Applies kernel version-specific SuSFS patch (`Patches/Patch/susfs_patch_to_<ver>.patch`).
- `patch-no-kprobe`: Applies inline or syscall hooks based on KSU fork.
- `build-process`: Configures defconfig and executes Clang/GCC compilation.
- `pack-process`: Assembles output into AnyKernel3 zip or boot image.
- `upload-files`: Uploads build artifacts to GitHub Actions.

### Key Environment Variables

- `KERNELSU_AUTO_FORK`: Supports `bakasu` (`Baka-SU/BakaSU`), `resukisu` (`ReSukiSU/ReSukiSU`), `xxksu`, `rsuntk`, `sukisu`, `next`.
- `AK3_SOURCE` / `AK3_BRANCH`:
  - Default: `https://github.com/dereference23/AnyKernel3.git` (`veux` branch).
  - Custom branding: `https://github.com/Apihplays/Veux_susfs_kernel.git` (`ak3` branch).
- `SUSFS_ENABLE`: `true` enables SuSFS patching and `CONFIG_KSU_SUSFS=y`.
- `SUSFS_FOLDER_FIXED` / `SUSFS_PATCH_FIXED`: Points to device-specific fixes in `PATCHES_SOURCE`.

---

## Operational Commands

```bash
# Sync upstream rebase safely
git fetch upstream
git checkout rebase && git reset --hard upstream/rebase
git push --force-with-lease origin rebase

# Compare branches
git log --oneline --left-right origin/rebase...upstream/rebase

# Trigger workflow on backup-rebase
gh workflow run build-redmi-note11pro-5g-aosp-a16.yml --ref backup-rebase

# Monitor run
gh run view <run-id> --log
```
