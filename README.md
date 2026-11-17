# Oak Video Editor (openKylin SIG 163) 开源贡献与补丁归档

[中文说明] | [English](README_EN.md)

[![Oak Upstream](https://img.shields.io/badge/Upstream-OakVideoEditorCommunity%2Foak-blue)](https://github.com/OakVideoEditorCommunity/oak)
[![Rust](https://img.shields.io/badge/Rust-1.85%2B-orange?logo=rust)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-GPL--3.0-green.svg)](https://www.gnu.org/licenses/gpl-3.0)

本仓库用于归档与展示参与 **[Oak Video Editor](https://github.com/OakVideoEditorCommunity/oak)**（国产非线性视频剪辑开源项目，openKylin SIG 163）的个人贡献、Bug 修复补丁及单元测试实现。

---

## 📌 贡献追踪表 (Contribution Tracker)

| 序号 | 关联 Issue | Pull Request | 模块 | 核心改动与技术点 | 状态 |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **01** | [#83 Scopes: handle NaN and infinite samples safely](https://github.com/OakVideoEditorCommunity/oak/issues/83) | [PR #89](https://github.com/OakVideoEditorCommunity/oak/pull/89) | `oak-app::scopes::math` | 针对示波器数学计算中的非有限浮点数（`NaN`、`±Inf`）建立五大安全防线，解决 `.clamp()` 无法拦截 `NaN` 的底层缺陷，补充 4 组极端边界单测（10/10 通过） | 🚀 Open |

---

## 📁 目录结构

```text
oak-contributions/
├── README.md                           # 中文主页说明与贡献索引
├── README_EN.md                        # English documentation
└── contributions/                      # 贡献任务详情归档
    └── 01-issue-83-scopes-nan-inf/     # Issue #83 示波器数学算法防御
        ├── README.md                   # 详细技术复盘与根因分析
        ├── math.rs                     # 修复后的完整源码与测试用例
        └── scopes-nan-inf.patch        # 独立的 Git Patch 补丁文件
```

---

## 🔗 相关项目与链接

- **上游主仓库 (GitHub)**：[OakVideoEditorCommunity/oak](https://github.com/OakVideoEditorCommunity/oak)
- **openKylin 官方 SIG 163**：[OakVideoEditor SIG 详情](https://www.openkylin.top/sig/sig_detail.html?sig_id=163)
- **维护者 GitHub**：[@liangshengsheng12333](https://github.com/liangshengsheng12333)
