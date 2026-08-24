# 无 Root Xposed 模块运行框架

## 1. 概述

一个**无需 root 权限**即可运行 Xposed 模块的 Android 应用，基于 [Vector](https://github.com/JingMatrix/Vector) 的 LSPlant ART Hook 能力构建。面向 Xposed 模块**开发者**，提供模块开发与调试的可靠环境。

核心价值：将 Vector 原本依赖 Zygisk + root 的 Hook 能力，以应用层方案提供给开发者，免 root 即可完成模块的开发、调试与分发产物生成。

## 2. 核心模式

产品包含两个协同模式：

- **重打包器**：选择目标应用 → 勾选已启用模块 → 注入 Vector 运行时与模块 → 生成可独立安装、可分发的补丁应用。
- **调试器（首次调试补丁模式）**：对目标应用打一次"动态加载器"补丁，之后模块代码改动只需重装模块 APK 并重启目标应用即生效，无需反复打补丁。

## 3. 验收标准（AC）

### 3.1 Happy Path（正常流程）

- **AC-001**：Given 开发者在 App 内选择一个 Xposed 模块 APK 安装，When App 解析该 APK 并识别为有效 Xposed 模块，Then 模块出现在模块列表中且默认未启用。
- **AC-002**：Given 模块列表中存在已安装模块，When 开发者切换某模块的启用开关，Then 该模块状态更新，并在后续打补丁/调试时按状态生效。
- **AC-003**：Given 开发者选择一个已安装的目标应用并勾选若干已启用模块，When 执行重打包，Then 生成注入 Vector 运行时与所选模块的新 APK，安装后启动该应用，所选模块生效。
- **AC-004**：Given 开发者提供一个 APK 文件作为打包目标，When 执行重打包，Then 同 AC-003 产出可安装的补丁应用。
- **AC-005**：Given 开发者首次对目标应用执行调试模式，When App 对其打入动态加载器补丁，Then 生成调试版应用，安装后启动时从本 App 读取已启用模块并加载。
- **AC-006**：Given 调试版目标应用已安装，When 开发者修改模块代码、重新打包模块 APK 并重装模块、重启目标应用，Then 新模块代码生效，且无需对目标应用重新打补丁。
- **AC-007**：Given 目标应用运行中，When 开发者打开模块日志视图，Then 可查看各模块输出的日志，支持按模块过滤并导出。
- **AC-008**：Given 目标应用运行中，When 开发者打开加载状态视图，Then 可查看每个模块的加载结果、Hook 是否成功及失败原因。
- **AC-009**：Given 目标应用因模块发生崩溃，When 崩溃发生，Then App 捕获并显示崩溃堆栈，并标注是否由模块导致。
- **AC-010**：Given 开发者打开环境信息视图，When 查看，Then 显示当前生效的 Hook 列表、作用域、Vector 框架版本与支持的 Xposed API。

### 3.2 Edge & Error Cases（边界与异常）

- **AC-011**：Given 用户安装一个非 Xposed 模块的普通 APK，When App 解析后未识别到模块元信息，Then 拒绝安装并给出"非有效 Xposed 模块"的明确提示。
- **AC-012**：Given 目标应用被加固/混淆/为 split APK/体积过大，When 执行重打包，Then 打包失败并给出具体原因，不产出损坏的半成品 APK。
- **AC-013**：Given 某模块导致目标应用崩溃或无法启动，When 开发者在 App 内禁用该模块，Then 重启目标应用后可正常运行（恢复手段可用）。
- **AC-014**：Given 设备上已安装原版目标应用，When 安装补丁应用时发生签名冲突，Then App 检测冲突并引导用户（卸载原版或以克隆包名安装）。
- **AC-015**：Given 设备存储空间不足，When 执行重打包，Then 在产出失败前/后给出"空间不足"的明确提示。
- **AC-016**：Given 目标应用已被打过补丁，When 再次对其执行重打包，Then App 识别为已有补丁，提示更新或警告覆盖。
- **AC-017**：Given 模块 Hook 了目标应用中不存在的类/方法，When 目标应用加载该模块，Then 优雅降级（记录警告日志）而不导致应用崩溃。

### 3.3 Business Rules（业务规则）

- **AC-018**：Given 设备未 root，When 执行完整流程（安装模块、重打包、调试、诊断），Then 全部功能正常运行，不依赖任何 root 权限。
- **AC-019**：Given 任何情况下，When 重打包/调试，Then 不修改原应用 APK 文件；卸载补丁应用即可完全恢复原状，修改可逆。
- **AC-020**：Given 模块使用传统 Xposed API 或 libxposed API，When 加载该模块，Then 两种 API 均被支持并正常工作。
- **AC-021**：Given 设备 Android 版本在 9 至 17 之间，When 使用本 App，Then 框架在该版本范围内正常运行。（依据：基座 LSPatch 要求 Android 9+/API 28）

## 4. 范围界定

### 本次做
- 重打包器（目标来源：已安装应用 + APK 文件）
- 调试器（首次调试补丁模式）
- 模块管理（安装 / 启停）
- 诊断能力（模块日志 / 加载状态 / 崩溃捕获 / 环境信息）
- 双 Xposed API 支持（传统 Xposed API + libxposed API）
- Android 8.1–17 兼容

### 本次不做
- 虚拟容器运行模式
- 模块在线商店 / 下载
- 自动作用域推断
- 多用户 / 工作配置文件支持
- Magisk / KernelSU 集成

## 5. 默认假设

- 打包目标来源同时支持「已安装应用」与「APK 文件」两种。
- Android 版本兼容范围与 LSPatch 基座一致：9–17（LSPatch 要求 Android 9+/API 28）。
