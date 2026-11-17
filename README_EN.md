# Oak Video Editor (openKylin SIG 163) Contributions & Patches

[中文说明](README.md) | [English]

[![Oak Upstream](https://img.shields.io/badge/Upstream-OakVideoEditorCommunity%2Foak-blue)](https://github.com/OakVideoEditorCommunity/oak)
[![Rust](https://img.shields.io/badge/Rust-1.85%2B-orange?logo=rust)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-GPL--3.0-green.svg)](https://www.gnu.org/licenses/gpl-3.0)

A personal archive of contributions, bug fixes, standalone patches, and unit test implementations for [Oak Video Editor](https://github.com/OakVideoEditorCommunity/oak) (openKylin SIG 163).

---

## 📌 Contribution Tracker

| Index | Target Issue | Pull Request | Component | Description | Status |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **01** | [#83 Scopes: handle NaN and infinite samples safely](https://github.com/OakVideoEditorCommunity/oak/issues/83) | [PR #89](https://github.com/OakVideoEditorCommunity/oak/pull/89) | `oak-app::scopes::math` | Added non-finite (`NaN`, `±Inf`) float defenses across 5 scope math functions; addressed `.clamp()` bypass under IEEE 754; added 4 boundary unit tests (10/10 passed). | 🚀 Open |

---

## 📁 Directory Structure

```text
oak-contributions/
├── README.md                           # Main Chinese documentation & tracker
├── README_EN.md                        # English documentation
└── contributions/                      # Contribution details
    └── 01-issue-83-scopes-nan-inf/     # Issue #83: Non-finite sample safety
        ├── README.md                   # Technical breakdown & test notes
        ├── math.rs                     # Updated implementation & unit tests
        └── scopes-nan-inf.patch        # Standalone Git patch
```

---

## 🔗 References

- **Upstream Repository**: [OakVideoEditorCommunity/oak](https://github.com/OakVideoEditorCommunity/oak)
- **openKylin SIG 163**: [OakVideoEditor SIG Detail](https://www.openkylin.top/sig/sig_detail.html?sig_id=163)
- **Maintainer**: [@liangshengsheng12333](https://github.com/liangshengsheng12333)
