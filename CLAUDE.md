# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AE86Wrt is an OpenWrt firmware distribution that builds custom firmware for multiple router platforms via GitHub Actions CI/CD. There is no local build system — all compilation happens in GitHub Actions workflows.

- **Base source:** coolsnowwolf/lede (Lean's LEDE), with some builds using official OpenWrt or hanwckf/immortalwrt-mt798x
- **Branding:** AE86Wrt, Argon theme, Chinese locale

## Architecture

The build pipeline follows: **Workflow → DIY Scripts → Config → Compile**

```
.github/workflows/   →  Orchestration (clone source, run scripts, apply config, compile)
diy/                 →  Customization scripts run inside the workflow
configs/             →  OpenWrt .config files (package selection, device targets)
```

### Workflow execution order (example: Build_r4s.yml)
1. Clone AE86Wrt repo + coolsnowwolf/lede source into `openwrt/`
2. Run `diy/lean/lean1.sh` — adds custom feeds (2305ipk), removes unwanted packages
3. `feeds update -a && feeds install -a`
4. Run `diy/lean/lean2.sh` — Chinese translations, theme replacement, IP/hostname/branding changes
5. Workflow-specific customizations (e.g., R4S pulls latest argon theme, syncdial, changes IP)
6. Copy device `.config` to `openwrt/.config`, run `make defconfig`
7. `make download` then `make -j$(nproc) V=s`
8. Collect firmware artifacts and create GitHub release

### Script pairing by source base
| Source | Stage 1 (pre-feeds) | Stage 2 (post-feeds) |
|--------|---------------------|----------------------|
| Lean/LEDE (rk, ipq, x86, k3, ax6s) | `diy/lean/lean1.sh` or `lean01.sh` | `diy/lean/lean2.sh` or `lean02.sh` |
| MT798x immortalwrt | `diy/mt798x/op1.sh` or `imm1.sh` | `diy/mt798x/op2.sh` or `imm2.sh` |
| Official OpenWrt / iStoreOS | `diy/op/op1.sh` or `isos1.sh` | `diy/op/op2.sh` or `isos2.sh` |

### Config file organization
- `configs/ARM/ipq/` — Qualcomm IPQ807x (AX3600, AX9000, AX6, 301W)
- `configs/ARM/mt798x/` — MediaTek MT798x
- `configs/ARM/other/` — Rockchip (rk.config, r4s.config), K3, AX6S
- `configs/X86/` — x86-64 variants

## Key conventions

- **Config files** follow OpenWrt Kconfig format. Enable: `CONFIG_PACKAGE_xxx=y`, disable: `# CONFIG_PACKAGE_xxx is not set`. Never just delete a line.
- **R4S has its own workflow** (`Build_r4s.yml`) and config (`r4s.config`), separate from the multi-device `Build_rk35xx.yml`.
- **R4S-specific customizations** (IP, extra packages, theme overrides) go in the workflow file after `lean2.sh` runs, not in the shared scripts.
- **Package feeds** are added in stage-1 scripts via `sed -i "1isrc-git ..." feeds.conf.default`.
- The `2305ipk` feed (`xiangfeidexiaohuo/2305-ipk`) provides many additional packages shared across builds.

## Supported platforms and workflows

| Workflow | Devices | Source |
|----------|---------|--------|
| `Build_r4s.yml` | NanoPi R4S only | Lean/LEDE |
| `Build_rk35xx.yml` | 30+ Rockchip devices | Lean/LEDE |
| `Build_ipq807x.yml` | AX6, AX3600, AX9000, 301W | Lean/LEDE |
| `Build_x86_combined.yml` | x86-64 (4 variants) | Lean/LEDE + OpenWrt |
| `Build_mt798x_combined.yml` | MT7981/MT7986/Filogic | immortalwrt-mt798x |
| `Build_phicomm_k3.yml` | Phicomm K3 | Lean/LEDE (k3 branch) |
| `Build_redmi_ax6s.yml` | Redmi AX6S | Lean/LEDE |
| `Build_isos_3rd_rk.yml` | iStoreOS Rockchip/ARMSR | iStoreOS |
| `Build_isos_3rd_seed.yml` | Seed AC1/AC3 | iStoreOS |
