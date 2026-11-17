# Issue #83: Scopes: handle NaN and infinite samples safely

- **关联 Issue**：[Oak #83](https://github.com/OakVideoEditorCommunity/oak/issues/83)
- **提交 PR**：[PR #89](https://github.com/OakVideoEditorCommunity/oak/pull/89)
- **目标代码**：`crates/oak-app/src/widgets/scopes/math.rs`
- **提交 Commit**：`40bcb24 fix(scopes): handle non-finite samples safely in scope math (#83)`

---

## 一、Bug 背景与理论根因 (IEEE 754)

在示波器计算模块中，原代码广泛使用 `sample.clamp(0.0, 1.0)` 对输入做截断。

在计算机科学与 IEEE 754 浮点数标准中：
> **任何数字与 `NaN`（Not a Number）进行大小比较（`<`、`>`、`<=`、`>=`），结果统统返回 `false`。**

在 Rust 标准库中，`clamp(min, max)` 的逻辑本质是先比较是否小于最小值，再比较是否大于最大值。由于与 `NaN` 的比较均为 `false`，导致 `NaN` 可以直接穿透 `.clamp()`，逃逸到渲染层：

1. **`histogram_bins`**：`NaN as usize` 隐式归零，坏数据单向扎堆在第 0 个桶，导致暗部像素统计失真；
2. **`waveform_envelope`**：如果切片全是非法值，`min` 维持 `f32::MAX`、`max` 维持 `f32::MIN`，两极倒挂导致坐标变换异常；
3. **`vectorscope_points`**：色度损坏坐标直接流入 GPUI 绘制流程；
4. **`decay_peak`**：若音量输入正无穷（`+Inf`），由于浮点数算术性质，`peak - decay` 依然为 `+Inf`，峰值小横线被永久卡死在最大值，无法衰减。

---

## 二、五大防御实现细节

统一采用标准库 `f32::is_finite()` 建立边界守卫，兼顾 `NaN` 与 `±Inf`：

1. **`histogram_bins`**
   ```rust
   for &sample in samples {
       if !sample.is_finite() {
           continue; // 坏数据直接跳过，不计入直方图统计
       }
       let clamped = sample.clamp(0.0, 1.0);
       let index = ((clamped * bins as f32) as usize).min(bins - 1);
       out[index] += 1;
   }
   ```

2. **`waveform_envelope`**
   ```rust
   let mut min = f32::MAX;
   let mut max = f32::MIN;
   let mut has_finite = false; // 标记切片内是否存在有效有限数
   for &sample in &samples[start..end] {
       if sample.is_finite() {
           min = min.min(sample);
           max = max.max(sample);
           has_finite = true;
       }
   }
   if end > start && has_finite {
       out[column] = (min, max);
   } // 全坏数据安全回退到默认的 (0.0, 0.0)
   ```

3. **`vectorscope_points`**
   ```rust
   samples
       .iter()
       .filter(|(u, v)| u.is_finite() && v.is_finite()) // 剔除非法坐标
       .map(|&(u, v)| ((u - 0.5).clamp(-0.5, 0.5), (v - 0.5).clamp(-0.5, 0.5)))
       .collect()
   ```

4. **`meter_lit_segments`**
   ```rust
   if level.is_nan() {
       return 0; // 遇到 NaN 音量表显示静音
   }
   let lit = (level.clamp(0.0, 1.0) * segments as f32).round() as usize;
   lit.min(segments)
   ```

5. **`decay_peak`**
   ```rust
   let peak = if peak.is_finite() { peak } else { 0.0 };
   let level = if level.is_finite() { level } else { 0.0 };
   let decay = if decay_per_frame.is_finite() && decay_per_frame >= 0.0 {
       decay_per_frame
   } else {
       0.0
   };
   if level >= peak {
       level
   } else {
       (peak - decay).max(level)
   }
   ```

---

## 三、单测与验证

保留原有 6 个功能测试，新增 4 个极端边界单测：
- `histogram_filters_nan_and_inf`
- `waveform_envelope_handles_nan_and_inf`
- `vectorscope_filters_nan_and_inf`
- `meter_and_decay_nan_and_inf`

### 本地轻量级单测命令
由于 `math.rs` 为无依赖的纯数学模块，可直接用 `rustc --test` 秒级验证：
```cmd
rustc --test math.rs -o math_test.exe
math_test.exe
del math_test.exe math_test.pdb
```

输出证据：
```text
running 10 tests
test tests::histogram_bins_correctly ... ok
test tests::histogram_clamps_out_of_range ... ok
test tests::histogram_filters_nan_and_inf ... ok
test tests::histogram_handles_empty_and_zero_bins ... ok
test tests::meter_and_decay_nan_and_inf ... ok
test tests::meter_lit_and_peak_decay ... ok
test tests::vectorscope_filters_nan_and_inf ... ok
test tests::vectorscope_projection_centers ... ok
test tests::waveform_envelope_handles_nan_and_inf ... ok
test tests::waveform_envelope_slices ... ok

test result: ok. 10 passed; 0 failed; finished in 0.00s
```
