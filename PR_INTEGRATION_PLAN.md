# PR Integration Plan: org.osbuild.bfb — Review Comment Fixes

## Overview

The existing PR branch (`bfb-stage`) added `org.osbuild.coreos.bfb` and received review
comments from dustymabe and achilleas-k. All fixes were developed and validated in the
`bfb-test-all-pr-comments` branch. This plan describes how to integrate them cleanly into
the PR as a single commit on top of the existing history.

**Source of fixes**: `bfb-test-all-pr-comments` branch
**Target PR branch**: `bfb-stage`
**Strategy**: One additional commit on top of `bfb-stage` — no rebasing, no force push
**Backup**: `bfb-test-all-pr-comments` branch stays untouched as reference

---

## Review Comments and Exact Fixes

### 1. dustymabe — Stage is CoreOS-specific, should be generic

**Comment**: The stage is named `org.osbuild.coreos.bfb` and hardcodes CoreOS/ignition
boot arguments as defaults. It should be distribution-agnostic.

**Fix**:
- Rename `stages/org.osbuild.coreos.bfb` → `stages/org.osbuild.bfb`
- Rename `stages/org.osbuild.coreos.bfb.meta.json` → `stages/org.osbuild.bfb.meta.json`
- Remove `ignition.firstboot` and `ignition.platform.id=nvidiabluefield` from
  `default_args_v2` in the stage. These are CoreOS-specific and must be supplied by the
  caller via `boot_args_v2` option when needed.

**Before** (`default_args_v2`):
```python
default_args_v2 = [
    "console=hvc0",
    "console=ttyAMA0",
    "earlycon=pl011,0x13010000",
    "initrd=initramfs",
    "modprobe.blacklist=mlxbf_pmc",
    "ignition.firstboot",                        # REMOVED
    "ignition.platform.id=nvidiabluefield"       # REMOVED
]
```

**After** (`default_args_v2`):
```python
default_args_v2 = [
    "console=hvc0",
    "console=ttyAMA0",
    "earlycon=pl011,0x13010000",
    "initrd=initramfs",
    "modprobe.blacklist=mlxbf_pmc"
]
```

---

### 2. dustymabe — `boot_args_v0` defaults are BF1/BF2 specific

**Comment**: The default `boot_args_v0` contained BF1/BF2 hardware-specific arguments.
BF1/BF2 support is not planned.

**Fix**: Set `default_args_v0 = []` (empty list). Anyone with BF1/BF2 hardware can still
supply args explicitly via the `boot_args_v0` option.

**Before**:
```python
default_args_v0 = [
    "console=ttyAMA0",
    "earlycon=pl011,0x01000000",
    "earlycon=pl011,0x01800000",
    "initrd=initramfs"
]
```

**After**:
```python
default_args_v0 = []
```

---

### 3. dustymabe — No RPM dependency documentation

**Comment**: The stage depends on `mlxbf-bfscripts` (for `mlx-mkbfb`) and
`mlxbf-bootimages` (for firmware files) but this is not documented anywhere in the code.

**Fix**: Extract the two hardcoded firmware paths into named constants at module level
with a comment identifying their RPM source:

```python
# BlueField firmware paths - typically from these RPMs:
# - mlxbf-bfscripts: provides /usr/bin/mlx-mkbfb tool
# - mlxbf-bootimages: provides firmware files below
DEFAULT_BFB_PATH = "/lib/firmware/mellanox/boot/default.bfb"
BOOT_CAPSULE_PATH = "/lib/firmware/mellanox/boot/capsule/boot_update2.cap"
```

These constants are then used in the `cmd` list instead of inline strings.

---

### 4. dustymabe — `inputs` schema not properly defined

**Comment**: The original meta.json had `"inputs": {"type": "object", "additionalProperties": true}`
which accepts any inputs without validation. Required inputs (`kernel`, `initramfs`) should
be declared and the optional input (`rootfs`) should be named.

**Before**:
```json
"inputs": {
  "type": "object",
  "additionalProperties": true
}
```

**After**:
```json
"inputs": {
  "type": "object",
  "additionalProperties": false,
  "required": ["kernel", "initramfs"],
  "properties": {
    "kernel": {
      "type": "object",
      "additionalProperties": true
    },
    "initramfs": {
      "type": "object",
      "additionalProperties": true
    },
    "rootfs": {
      "type": "object",
      "additionalProperties": true
    }
  }
}
```

---

### 5. achilleas-k — Schema `default` values missing

**Comment**: The `boot_args_v0` and `boot_args_v2` options have defaults applied in the
stage code but the schema does not document them. Reviewers and users cannot tell from
the schema what values will be used when an option is omitted.

**Fix**: Add `"default"` fields to both options in meta.json:

```json
"boot_args_v0": {
  "type": "array",
  "items": {"type": "string"},
  "description": "Boot arguments for older DPU firmware (BF1/BF2). Empty by default as BF1/BF2 support is not planned.",
  "default": []
},
"boot_args_v2": {
  "type": "array",
  "items": {"type": "string"},
  "description": "Boot arguments for newer DPU firmware (BF3). Defaults include essential hardware-specific arguments (console, earlycon, initrd).",
  "default": ["console=hvc0", "console=ttyAMA0", "earlycon=pl011,0x13010000", "initrd=initramfs", "modprobe.blacklist=mlxbf_pmc"]
}
```

---

### 6. dustymabe — No documentation for kernel arguments

**Comment**: "Is there a document somewhere that documents these as recommended kernel
arguments? It might help any future maintainers if we provide some context here so if
things stop working they have somewhere to look for potential documentation changes."

A contributor clarified that the arguments come from NVIDIA's `bfb-build` repository,
and dustymabe confirmed: "Let's put that context here in a comment."

**Fix**: Add an inline comment above `default_args_v2` explaining the source and purpose
of each argument:

```python
# Hardware-specific defaults required for BlueField DPUs. These arguments
# were taken from NVIDIA's bfb-build repository:
# https://github.com/Mellanox/bfb-build
# - console=hvc0: /dev/rshim0/console (rshim virtual console)
# - console=ttyAMA0: IPMI serial console
# - earlycon=pl011,0x13010000: early serial output on BF3 UART address
# - initrd=initramfs: tell the bootloader which initramfs to use
# - modprobe.blacklist=mlxbf_pmc: avoid PMC driver conflicts on BF3
default_args_v2 = [...]
```

---

### 7. achilleas-k — No unit tests

**Comment**: The stage has no unit tests. Every osbuild stage is expected to have a test
file under `stages/test/`.

**Fix**: Add `stages/test/test_bfb.py` with 4 tests:

| Test | What it verifies |
|------|-----------------|
| `test_bfb_command_generation[basic]` | mlx-mkbfb is called with correct flags: `--image`, `--initramfs`, `--capsule`, `--boot-args-v0`, `--boot-args-v2`, `default.bfb`, output path |
| `test_bfb_command_generation[with-rootfs]` | Same as above but rootfs input present |
| `test_bfb_rootfs_combination` | When rootfs is provided, initramfs and rootfs are concatenated into `combined.img` and that path is passed as `--initramfs` |
| `test_parse_input` | `parse_input()` correctly resolves the single file path from the osbuild input structure |

Key implementation details in the test file:
- Uses `copy.deepcopy(inputs)` before each `main()` call to prevent `parse_input()`'s
  `files.popitem()` from mutating shared test data between parametrized runs
- `STAGE_NAME = "org.osbuild.bfb"` — must match the stage filename exactly for the
  `stage_module` fixture (provided by `conftest.py`) to load the correct stage
- `mocked_temp_dir` fixture patches `tempfile.TemporaryDirectory` so no real temp
  directories are created during testing

---

## Files Changed in the PR Commit

| File | Action | Reason |
|------|--------|--------|
| `stages/org.osbuild.coreos.bfb` | **Remove** | Replaced by renamed generic version |
| `stages/org.osbuild.coreos.bfb.meta.json` | **Remove** | Replaced by renamed fixed version |
| `stages/org.osbuild.bfb` | **Add** | Generic stage (comments 1, 2, 3) |
| `stages/org.osbuild.bfb.meta.json` | **Add** | Fixed schema (comments 4, 5) |
| `stages/test/test_bfb.py` | **Add** | Unit tests (comment 6) |

**Not included in PR** (kept in `bfb-test-all-pr-comments` for future work):
- `test/data/manifests/bluefield-bfb-test.mpp.yaml` — requires BlueField RPMs in CI to
  be runnable; will be added later under `test/data/stages/bfb/` once the package
  repository is configured

---

## Execution Steps

### Step 1 — Verify starting state
```bash
cd ~/Coding/osbuild
git checkout bfb-stage
git status   # must be clean
git log --oneline -3
# Expected: 462317aa stages: add coreos.bfb for NVIDIA BlueField DPUs
```

### Step 2 — Remove old CoreOS-specific files
```bash
git rm stages/org.osbuild.coreos.bfb
git rm stages/org.osbuild.coreos.bfb.meta.json
```

### Step 3 — Pull the final versions of the 3 new files from the fix branch
```bash
git checkout bfb-test-all-pr-comments -- stages/org.osbuild.bfb
git checkout bfb-test-all-pr-comments -- stages/org.osbuild.bfb.meta.json
git checkout bfb-test-all-pr-comments -- stages/test/test_bfb.py
```

### Step 4 — Verify the staged diff
```bash
git diff --staged --stat
# Expected:
#  stages/{org.osbuild.coreos.bfb => org.osbuild.bfb}           | ...
#  ...{org.osbuild.coreos.bfb.meta.json => org.osbuild.bfb.meta.json} | ...
#  stages/test/test_bfb.py                                       | 141 +++
#  3 files changed (2 renames + 1 new)

git diff --staged
# Manually verify:
# - No ignition.* args in default_args_v2
# - default_args_v0 = []
# - DEFAULT_BFB_PATH / BOOT_CAPSULE_PATH constants with RPM comments
# - inputs schema has required: [kernel, initramfs] with named properties
# - boot_args_v0 and boot_args_v2 have "default" fields
# - STAGE_NAME = "org.osbuild.bfb" in test file
```

### Step 5 — Run tests locally on the remote VM
```bash
# Sync the staged files to remote VM for testing before committing
scp stages/org.osbuild.bfb root@10.6.135.182:/root/osbuild/stages/
scp stages/org.osbuild.bfb.meta.json root@10.6.135.182:/root/osbuild/stages/
scp stages/test/test_bfb.py root@10.6.135.182:/root/osbuild/stages/test/

ssh root@10.6.135.182 'cd /root/osbuild && python3 -m pytest stages/test/test_bfb.py -v'
# Expected: 4 passed
```

### Step 6 — Create the single commit
```bash
git commit -m "stages: rename and generalize org.osbuild.bfb

Address review comments on the original org.osbuild.coreos.bfb stage:

- Rename to org.osbuild.bfb - the stage is not CoreOS-specific
- Remove ignition.firstboot and ignition.platform.id from default boot
  args; callers that need these (e.g. RHCOS) supply them via boot_args_v2
- Set default_args_v0 to [] - BF1/BF2 support is not planned
- Add constants for firmware paths with RPM source documentation
- Fix inputs schema: declare required inputs (kernel, initramfs) and
  optional input (rootfs) with additionalProperties: false
- Add default values for boot_args_v0 and boot_args_v2 in the schema
- Add unit tests in stages/test/test_bfb.py"
```

### Step 7 — Verify final state
```bash
git log --oneline -3
# Expected:
# <new sha>  stages: rename and generalize org.osbuild.bfb
# 462317aa   stages: add coreos.bfb for NVIDIA BlueField DPUs
# ...

git show --stat HEAD
# Must show exactly the 5 file operations above (2 removes + 3 adds)
```

### Step 8 — Push to the PR branch
```bash
git push origin bfb-stage
```

---

## Risk Assessment

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| `git diff --staged` shows unexpected files | Low | Review carefully in Step 4 before committing |
| Tests fail after sync to VM | Very low | Already validated on remote VM; 4 tests passing |
| Git rename detection fails | Very low | Git detects renames by similarity; content is >50% identical so it will show as rename in GitHub PR view |
| Commit message rejected by CI hooks | Very low | Follows existing repo style from `git log` |

---

## Rollback

If anything goes wrong before Step 8 (push):
```bash
git checkout bfb-stage          # discard all local changes
git reset HEAD~1                # if commit was made locally, undo it
```

The `bfb-test-all-pr-comments` branch is untouched throughout and serves as the
permanent reference for all work done.

---

## Future Work (not in this PR)

- Add `test/data/stages/bfb/manifest.mpp.yaml` for integration testing once the
  BlueField RPM repository (`mlxbf-bfscripts`, `mlxbf-bootimages`) is configured
  in the osbuild CI environment
