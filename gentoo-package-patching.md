# Creating and Managing Custom Package Patches in Gentoo

This guide covers the complete workflow for creating, extracting, and maintaining custom patches for Gentoo packages using `/etc/portage/patches/` and `git` inside Portage's temporary build directory.

---

## 1. Where to Store User Patches (`/etc/portage/patches/`)

Portage automatically applies user patches placed in `/etc/portage/patches/` during the `src_prepare` phase for virtually all modern ebuilds (EAPI 6, 7, and 8).

Using `/etc/portage/patches/` is preferred over modifying `/var/db/repos/gentoo/<category>/<pkg>/files/` because:
1. **It survives `emerge --sync`** (your changes won't be wiped out when updating the Gentoo tree).
2. **No `Manifest` or `.ebuild` modifications are required.**
3. **It can be version-independent** so upgrades automatically inherit your patch.

### Directory Structure Options:
* **All versions of a package (Recommended):**
  ```text
  /etc/portage/patches/<category>/<package>/my-custom-fix.patch
  ```
  *(Example: `/etc/portage/patches/app-emulation/qemu/npcm750-evb-enable-mmc.patch`)*

* **A specific version or revision only:**
  ```text
  /etc/portage/patches/<category>/<package>-<version>/my-custom-fix.patch
  ```
  *(Example: `/etc/portage/patches/app-emulation/qemu-10.2.3-r2/npcm750-evb-enable-mmc.patch`)*

> **Note:** Patch filenames **must** end with `.patch` or `.diff`, and patches are applied in alphabetical order (commonly prefixed like `01-fix-foo.patch`, `02-enable-bar.patch`). Do not use `-9999` in the directory name unless you are specifically emerging a live Git (`9999`) ebuild.

---

## 2. One-Time Git Setup for `/var/tmp/portage/`

Because `/var/tmp/portage/` is owned by the `portage:portage` user (which has `/sbin/nologin` as its shell), running `git` inside the build directory as `root` or a normal user will trigger Git's `safe.directory` ownership check.

Run this once to allow `root` and your user to run `git` inside any Portage workdir:

```bash
git config --global --add safe.directory '*'
sudo git config --global --add safe.directory '*'
```

*(Ensure `root` also has a basic git identity configured if you plan to run `sudo git commit`)*:
```bash
sudo git config --global user.name "Your Name"
sudo git config --global user.email "you@example.com"
```

---

## 3. Step-by-Step Workflow: Creating a Patch with Git

### Step 1: Temporarily move out any broken user patches
If you are updating an existing patch in `/etc/portage/patches/` that currently fails to apply, move it out first so `prepare` doesn't fail:
```bash
sudo mv /etc/portage/patches/<category>/<pkg>/<name>.patch ~/
```

### Step 2: Unpack and prepare the official source tree
Use the `ebuild` command to unpack the tarball and apply all official Gentoo patches, stopping right before configuration/compilation:
```bash
sudo ebuild /var/db/repos/gentoo/<category>/<pkg>/<pkg>-<version>.ebuild clean prepare
```
*(Example: `sudo ebuild /var/db/repos/gentoo/app-emulation/qemu/qemu-10.2.3-r2.ebuild clean prepare`)*

### Step 3: Initialize a baseline Git repository in the work directory
Navigate to the unpacked source directory (`$S`) and create an initial baseline commit:
```bash
cd /var/tmp/portage/<category>/<pkg>-<version>/work/<source_dir>

sudo git init
sudo git add .
sudo git commit -m "baseline"
```
> **Pro-Tip for Huge Packages (like QEMU, Chromium, or GCC):**
> Instead of `sudo git add .` (which indexes tens of thousands of files), you can index only the subdirectories you plan to touch:
> `sudo git add hw/ include/`

### Step 4: Make your code changes
Edit the source files as needed:
```bash
sudo vim path/to/file.c
```
*(Or, if partially applying an older patch to fix rejects manually: `sudo patch -p1 < ~/old.patch`, clean up any `.orig` / `.rej` files, and edit the remaining files).*

Check which files have been modified:
```bash
sudo git status
```

### Step 5: Generate and save the patch
Create the target patch directory if it doesn't exist yet:
```bash
sudo mkdir -p /etc/portage/patches/<category>/<pkg>/
```

Choose **Option A** (plain diff) or **Option B** (formatted Git commit patch with metadata):

#### Option A: Plain Unified Diff (`git diff`)
```bash
sudo git diff | sudo tee /etc/portage/patches/<category>/<pkg>/my-feature.patch > /dev/null
```

#### Option B: Git Format-Patch with Commit Message (`git format-patch`)
```bash
sudo git add -u
sudo git commit -m "Brief description of what this patch fixes"
sudo git format-patch -1 --stdout | sudo tee /etc/portage/patches/<category>/<pkg>/my-feature.patch > /dev/null
```

### Step 6: Test and Emerge the Package
Run `emerge` normally. Portage will clean the work directory, unpack fresh sources, apply the official patches, and then automatically apply your new patch from `/etc/portage/patches/`:
```bash
sudo emerge -av <category>/<pkg>
```

Verify in the build output that you see:
```text
 * Applying user patches from /etc/portage/patches ...
 *   my-feature.patch ...                                                [ ok ]
```

---

## 4. Troubleshooting & Useful Commands

### Impersonating the `portage` User
If you ever prefer to work inside `/var/tmp/portage/` directly as the `portage` user (so created files naturally have `portage:portage` ownership), bypass `/sbin/nologin` by passing `-s /bin/bash`:
```bash
sudo -u portage -s /bin/bash
```

### Compiling In-Place with `ebuild ... compile`
If you want to test-compile your changes immediately inside the workdir (`sudo ebuild ... compile`) *without* running a full `emerge` from scratch:
* **Always remove the temporary `.git` directory first:**
  ```bash
  sudo rm -rf /var/tmp/portage/<category>/<pkg>-<version>/work/<source_dir>/.git
  ```
  Some build systems (such as QEMU's `meson.build`) check for the presence of a `.git` folder and will attempt to run `git submodule` commands over the network if they find one, which triggers a failure in Gentoo's `network-sandbox`.

### Restoring a Corrupted Ebuild or `Manifest` in `/var/db/repos/gentoo`
If you accidentally edited an official `.ebuild`, added files to `files/`, or broke a `Manifest` in the main Gentoo repository:
* **If `/var/db/repos/gentoo` is synced via `git`:**
  ```bash
  cd /var/db/repos/gentoo/<category>/<pkg>
  sudo git checkout -- .
  sudo git clean -fd
  ```
* **If `/var/db/repos/gentoo` is synced via `rsync`:**
  Remove any untracked files from `files/`, revert your `.ebuild` edits, and regenerate the `Manifest`:
  ```bash
  sudo ebuild /var/db/repos/gentoo/<category>/<pkg>/<pkg>-<version>.ebuild manifest
  ```
