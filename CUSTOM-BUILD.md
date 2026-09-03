# 自定义构建说明（Fork 增量）

> ⚠️ **本文件为 AI 辅助制作**（Hermes Agent / Nous Research），记录本 fork（`Lokitla/zzh_GKI_KernelSU_SUSFS`）在**上游 zzh20188/GKI_KernelSU_SUSFS 基础上新增的自定义功能**，供自用与后续维护参考。上游原始文档见 [README.md](./README.md) / [README-EN.md](./README-EN.md)。

---

## 🧩 本 Fork 相对上游的改动

### 1. `build.yml` / `kernel-custom.yml`：新增 `sukisu_branch` 可选开关
- **作用**：控制 SukiSU 变体拉取 SukiSU-Ultra 的哪个分支。
- **可选值**：`main`（默认，完整版）/ `dev` / `builtin`。
- **为什么加**：上游 `SukiSU` 变体在 SUSFS 开启时**硬编码 `BRANCH="-s builtin"`**，而 `builtin` 分支的 `kernel/feature/kernel_umount.c` **缺少 `kernel_umount_feature_set(u64 value)` 的函数定义**（只有 `feature_get`），导致编译报 `undeclared identifier 'kernel_umount_feature_set'` + `incompatible pointer types`。`main` 分支（v4.2.0）是完整版，有正确的 `feature_set(u64 value)` 签名，可正常编译。
- **用法**：触发时传 `-f sukisu_branch=main`（默认即 main，可不传）。

### 2. `get-manager.yml` / `kernel-custom.yml`：新增 `get_spoofed` 可选开关
- **作用**：是否额外拉取 **Spoofed（伪装包名）** 管理器 APK。
- **背景**：SukiSU-Ultra 自身 Actions 会用 `matrix.spoofed: ["false","true"]` 构建**普通 + spoofed** 两种管理器（spoofed = 随机化包名/应用名，用于隐藏、骗过反检测）。但上游 zzh 的 `get-manager.yml` **只给 ReSukiSU 变体抓 `Spoofed-Manager-release`**，SukiSU 变体走 else 分支只匹配名字含 `manager` 的产物，**完全忽略 Spoofed**。
- **用法**：触发时传 `-f get_spoofed=true`，SukiSU 变体也会额外拉 spoofed 管理器。

### 3. `Auto_Trigger.yml`：改为每天自动检测 SukiSU 更新并自动构建
- **修改内容**：
  - 频率 `*/3` 天 → **每天** `0 0 * * *`（UTC 00:00）。
  - owner 白名单加入 `Lokitla`（否则该 job 被跳过，不在 fork 上运行）。
  - 增加 `permissions: {contents: write, actions: write}`（否则 push `sha` 分支 + `gh workflow run` 触发构建会因 fork 默认只读 token 失败）。
  - 把原本指向不存在 `test_release.yml` 的注释触发逻辑，改为 `gh workflow run kernel-custom.yml`（传目标配置参数）。
- **流程**：每天 UTC 00:00 查 SukiSU-Ultra `main` 最新 SHA，与 `sha` 分支里存的上次 SHA 对比；有新 commit → 自动触发构建；无 → 跳过。
- **自动构建参数**（硬编码在 workflow 里，按需调整）：
  ```bash
  gh workflow run kernel-custom.yml \
    -f kernel_version=5.10 -f os_patch_level=2025-05 \
    -f kernelsu_variant=SukiSU -f sukisu_branch=main -f get_spoofed=true \
    -f use_zram=true -f use_kpm="enabled (开启)" \
    -f cve_2026_43499_patch=true -f "artifact_upload_mode=仅上传 AnyKernel3.zip"
  ```

---

## 🚀 手动触发目标构建

```bash
gh workflow run kernel-custom.yml --repo Lokitla/zzh_GKI_KernelSU_SUSFS \
  -f kernel_version=5.10 -f os_patch_level=2025-05 \
  -f kernelsu_variant=SukiSU -f sukisu_branch=main -f get_spoofed=true \
  -f use_zram=true -f use_kpm="enabled (开启)" \
  -f cve_2026_43499_patch=true -f "artifact_upload_mode=仅上传 AnyKernel3.zip"
```

- choice 型参数必须传**精确 label**（如 `use_kpm="enabled (开启)"` 带括号与空格）。
- **不是所有 `kernel-custom.yml` 里出现过的变量都能通过 `-f` 传**：只有 workflow 顶层 `workflow_dispatch.inputs` 定义的键（`kernel_version`/`os_patch_level`/`kernelsu_variant`/`version`/`build_time`/`use_kpm`/`droidspaces`/`artifact_upload_mode`/`droidspaces_ntsync`/`use_zram`/`use_bbg`/`use_rekernel`/`cve_2026_43499_patch`/`cancel_susfs`/`supp_op`/`sukisu_branch`/`get_spoofed`）。job 里的 env 不算。例如传 `-f enable_susfs=true` 会报 `HTTP 422: Unexpected inputs: ["enable_susfs"]`（SUSFS 默认开启，用 `cancel_susfs` 关）。

---

## ⚠️ 同步上游须知（重要）

- **上游默认分支是 `dev`**（不是 main），构建相关改动都在 dev。
- **本 fork 有自定义改动**，同步上游时**不要用 `gh repo sync --force`**（会 hard reset，抹掉全部自定义改动）。用 Git fork 页面的 **Sync fork** 按钮或：
  ```bash
  git fetch origin dev && git fetch fork dev
  git checkout dev && git pull fork dev
  git merge origin/dev
  git push fork dev
  ```
- 自定义改动集中在 `build.yml` / `kernel-custom.yml` / `get-manager.yml` / `Auto_Trigger.yml` 的**新增 option 分支**里，上游 `update-pages.yml` 只更新 `data/**`/`web/**`/`scripts/**`，一般不会改这几个 workflow 文件，冲突概率低；万一冲突手动解这几个文件即可。

---

## 📝 环境速记（本 fork 相关）

- **x86 账户**：`Lokitla`（gh CLI 已认证，API 5000 次/小时）。
- **目标配置**：SukiSU 5.10.236 + KPM + LZ4KD(ZRAM) + CVE-2026-43499 修复 + SUSFS。
- **切换 SukiSU 分支**：上游 `main` = 完整版（推荐）；`builtin` = 有 `kernel_umount_feature_set` bug（勿用，除非已修复）。
- **构建内核很慢**（几十分钟），用 GitHub Actions 页面或 cron 监控，别阻塞等待。
