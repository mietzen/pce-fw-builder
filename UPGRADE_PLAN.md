# Fresh Build: Coreboot 25.12 + EDK2 UEFI for PC Engines APU2

## Summary

**Goal**: Create a clean, modern CI workflow for building APU2 firmware with mainline coreboot 25.12 and EDK2 UEFI payload.

**Approach**: Create an **orphan branch** with minimal files. The old pce-fw-builder infrastructure (build.sh, Dockerfile, scripts/) was designed for PC Engines fork versions. For mainline coreboot, we can use the official `coreboot/coreboot-sdk` Docker image directly.

---

## What to Keep vs. Discard

### Discard (not needed for mainline)

- `build.sh` - Complex version detection for PC Engines fork; not needed
- `Dockerfile` - Just wraps coreboot-sdk; use image directly
- `scripts/` - PC Engines specific build scripts
- `build_apus.sh` - Dasharo-specific
- `.config` - Uses deprecated TIANOCORE options; regenerate fresh
- `version_verifier.sh` - PC Engines version testing

### Keep/Create

| File | Purpose |
|------|---------|
| `.github/workflows/build.yml` | Modern GitHub Actions workflow |
| `README.md` | Updated documentation |
| `LICENSE` | Keep existing (if applicable) |

---

## Step 1: Create Orphan Branch

```bash
git checkout --orphan coreboot-25.12-edk2
git rm -rf .
```

This creates a fresh branch with no history, keeping only what we explicitly add.

---

## Step 2: Create GitHub Actions Workflow

File: `.github/workflows/build.yml`

```yaml
name: Build APU2 Coreboot + EDK2 UEFI

on:
  workflow_dispatch:
    inputs:
      coreboot_version:
        description: 'Coreboot version/tag'
        required: true
        default: '25.12'
      edk2_revision:
        description: 'EDK2 revision'
        required: true
        default: 'edk2-stable202502'
      platform:
        description: 'Target platform'
        required: true
        default: 'apu2'
        type: choice
        options:
          - apu2
          - apu3
          - apu4
          - apu5
  push:
    branches: [coreboot-25.12-edk2]

env:
  COREBOOT_SDK: coreboot/coreboot-sdk:2025-10-19_4a3cc37cbd

jobs:
  build:
    runs-on: ubuntu-24.04

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Clone coreboot
      run: |
        git clone --depth=1 -b ${{ inputs.coreboot_version || '25.12' }} \
          https://review.coreboot.org/coreboot.git
        cd coreboot
        git submodule update --init --checkout

    - name: Configure for ${{ inputs.platform || 'apu2' }} with EDK2
      run: |
        cd coreboot
        # Start from upstream APU2 defconfig
        cp configs/config.pcengines_${{ inputs.platform || 'apu2' }} .config

        # Switch payload from SeaBIOS to EDK2
        scripts/config --disable CONFIG_PAYLOAD_SEABIOS
        scripts/config --enable CONFIG_PAYLOAD_EDK2
        scripts/config --set-str CONFIG_EDK2_REVISION_ID "${{ inputs.edk2_revision || 'edk2-stable202502' }}"

        # Expand config
        make olddefconfig

    - name: Build coreboot
      run: |
        docker run --rm \
          -v $PWD/coreboot:/home/coreboot/coreboot \
          -w /home/coreboot/coreboot \
          ${{ env.COREBOOT_SDK }} \
          make -j$(nproc)

    - name: Verify build
      run: |
        ls -la coreboot/build/coreboot.rom
        sha256sum coreboot/build/coreboot.rom

    - name: Upload ROM
      uses: actions/upload-artifact@v4
      with:
        name: coreboot-${{ inputs.platform || 'apu2' }}-${{ inputs.coreboot_version || '25.12' }}-edk2
        path: coreboot/build/coreboot.rom
        retention-days: 90
```

---

## Step 3: Create README

File: `README.md`

```markdown
# APU2 Coreboot + EDK2 UEFI Builder

Builds PC Engines APU2/3/4/5 firmware using mainline coreboot with EDK2 UEFI payload.

## Quick Start

1. Go to Actions tab
2. Select "Build APU2 Coreboot + EDK2 UEFI"
3. Click "Run workflow"
4. Download the ROM from artifacts

## Defaults

- **Coreboot**: 25.12 (mainline)
- **EDK2**: edk2-stable202502
- **SDK**: coreboot/coreboot-sdk:2025-10-19_4a3cc37cbd

## Flashing

```bash
flashrom -p internal -w coreboot-apu2-25.12-edk2.rom
```

⚠️ Have SPI programmer ready for recovery if needed.
```

---

## Step 4: Commit and Push

```bash
git add .github/workflows/build.yml README.md
git commit -m "Initial: Coreboot 25.12 + EDK2 UEFI for APU2"
git push -u origin coreboot-25.12-edk2
```

---

## Implementation Checklist

1. [ ] Create orphan branch `coreboot-25.12-edk2`
2. [ ] Add `.github/workflows/build.yml`
3. [ ] Add `README.md`
4. [ ] Push branch
5. [ ] Trigger workflow
6. [ ] Verify ROM builds successfully
7. [ ] (Optional) Flash and test on real APU2

---

## Verification

1. **GitHub Actions**: Workflow completes with green checkmark
2. **Artifact**: `coreboot-apu2-25.12-edk2` artifact contains `coreboot.rom` (~8MB)
3. **Hardware test**: Flash to APU2, connect serial (115200), see UEFI shell

---

## Future Enhancements (Optional)

- Add matrix build for all APU platforms
- Add release workflow for tagged versions
- Add firmware signing
- Add automated testing with QEMU (limited for APU2)
