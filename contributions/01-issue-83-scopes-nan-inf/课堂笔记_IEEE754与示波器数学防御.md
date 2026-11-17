# 《Oak 示波器数学算法防御与 IEEE 754 底层实战》—— 课堂精要与核心沉淀笔记

- **所属模块**：Oak Video Editor (openKylin SIG 163) · `crates/oak-app/src/widgets/scopes/math.rs`
- **对应任务**：Issue #83 & Pull Request #89
- **归档定位**：408 计算机考研核心理论（计组浮点数、数据结构与离散数学）与 Rust 开源工业级工程深度结合
- **学习者**：@liangshengsheng12333

---

## 一、 知识全景图与场景直觉（生活隐喻先行）

在专业视频剪辑软件（如 Premiere、达芬奇、剪映以及 Oak）中，剪辑师不能只靠眼睛肉眼看画面（因为显示器偏色、人眼适应会受欺骗），必须借助**客观仪器（示波器 Scopes）**来量化画面中的光影、色彩与声音。

示波器包含 5 种核心仪表，在程序里对应 5 个纯数学计算：

| 仪表类型 | 生活直觉比方 | 核心任务 | 对应的代码函数 |
| :--- | :--- | :--- | :--- |
| **直方图 (Histogram)** | **分苹果分桶**：按大小分到 4 个不同果篮里 | 统计画面中各个亮度区间的像素数量 | `histogram_bins` |
| **波形图 (Waveform)** | **找班级最高和最矮**：每个竖条列一个上下范围 | 算出每一列画面亮度的最高点和最低点（信封边缘） | `waveform_envelope` |
| **矢量示波器 (Vectorscope)** | **雷达圆盘打靶**：看子弹打在哪个方位 | 算出色彩偏向哪个方向、鲜艳度有多高 | `vectorscope_points` |
| **音量柱 (Audio Meter)** | **跳动的电池格/楼层灯**：总共 10 格该亮几格 | 根据当前音量大小比例，计算点亮几格灯 | `meter_lit_segments` |
| **峰值衰减 (Peak Decay)** | **羽毛慢慢飘落**：音量冲顶后留下的小横线缓缓落下 | 计算每一帧画面刷新时，最高峰小横线的新位置 | `decay_peak` |

---

## 二、 408 计组底层理论攻坚：IEEE 754 浮点数的致命陷阱

### 1. 浮点数的非全序性（Partial Order）
普通数学中，任意两个实数 $a$ 和 $b$ 要么 $a < b$、要么 $a = b$、要么 $a > b$（这叫全序关系）。
但在计算机硬件（IEEE 754 标准）中，浮点数存在一个极其特殊的特殊值：**`NaN`（Not a Number，不是数 / 无效数据）**。

> ⚠️ **核心铁律**：
> **在 IEEE 754 标准中，任何数字与 `NaN` 进行大小比较（`<`、`>`、`<=`、`>=`），结果统统都是 `false`（假）！甚至连 `NaN == NaN` 都是 `false`！**

### 2. 为什么原版的 `.clamp()` 拦不住 `NaN`？
原版代码中，学长写了 `sample.clamp(0.0, 1.0)`，以为能把所有数值都限制在 0 到 1 之间。
但 `.clamp(min, max)` 的内部底层实现是这样的判断：
```text
1. 先问：sample < 0.0 吗？（如果是负数，就纠正为 0.0）
   -> 传入 NaN 时：NaN < 0.0 结果是 false！
2. 再问：sample > 1.0 吗？（如果是超大数，就纠正为 1.0）
   -> 传入 NaN 时：NaN > 1.0 结果依然是 false！
3. 既然两个条件全都不满足，程序就误以为它“在 0.0 到 1.0 之间”，直接原封不动把它放行了！
```
**结论**：`NaN` 像一个幽灵隐形人，轻松穿透了 `.clamp()` 的防线，直接杀入后方的绘图渲染引擎！

### 3. 为什么正无穷大（`+Inf`）同样会导致系统卡死？
在音量峰值衰减 `decay_peak` 中：
如果音量输入了正无穷大（`+Inf`），更新后的峰值变成 `peak = +Inf`。
到下一帧即使声音变小了，计算峰值下落：$+\infty - decay = +\infty$！
**峰值小横线永远停留在正无穷大，永久自锁卡死，永远无法衰减！**

### 4. 工业级标准防线：`f32::is_finite()`
Rust 标准库提供的 `.is_finite()` 是最佳守卫：
* 如果是正常有效数字（比如 0.5、1.0、-0.2），返回 `true`；
* 如果是 `NaN`、`+Inf` 或 `-Inf`，返回 `false`。
使用它，可以在入口处将所有非法浮点数一网打尽！

---

## 三、 五大函数深度解剖与【闭环复盘】

---

### 第 1 个函数：`histogram_bins`（直方图分苹果）

#### 1. 标识符三步法
* **拆零件**：`histogram`（直方图、频数柱状图）+ `bins`（箱子、垃圾桶、分拣框）。
* **拼整句人话**：**“直方图的各个分拣桶”**。
* **软件职责**：把输入的所有像素亮度，统计到对应区间的桶里去。

#### 2. 函数声明（外壳）显微镜与英语成分对照
```rust
pub fn histogram_bins(samples: &[f32], bins: usize) -> Vec<u32>
```
* `pub fn`：公开函数。
* `samples: &[f32]`：
  * **主语名词**：`samples`（传入的样本数据列表，切片引用 `&[...]`）。
  * **类型**：`f32`（小数）。
* `bins: usize`：
  * **宾语名词**：`bins`（总共切分成几个桶，比如 4 个桶、64 个桶）。
* `-> Vec<u32>`：
  * **输出成品**：返回一个装满无符号整数的动态数组（每个桶装了多少个像素）。

#### 3. 核心防御代码与语法对照
```rust
for &sample in samples {
    if !sample.is_finite() {
        continue;
    }
    let clamped = sample.clamp(0.0, 1.0);
    let index = ((clamped * bins as f32) as usize).min(bins - 1);
    out[index] += 1;
}
```
* **英语语法对照**：
  * `for &sample in samples`：循环遍历每一个苹果。
  * `sample`（名词主语）+ `.is_finite()`（谓语动词，问它是不是好数）+ `!`（连词逻辑非）。
  * `continue`：如果是坏苹果，立即跳过，不计入任何桶。
* **【合拢复盘】**：
  “遍历每一个像素样本，如果发现不是正常数字（`!is_finite()`）就直接跳过丢弃；正常数字卡在 0 到 1 之间，算出它属于第几个桶，让该桶的计数加 1。”

---

### 第 2 个函数：`waveform_envelope`（波形信封 / 上下边界）

#### 1. 标识符三步法
* **拆零件**：`waveform`（波形）+ `envelope`（信封、外壳、包络线）。
* **拼整句人话**：**“波形图的上下信封轮廓”**。
* **软件职责**：算出画面每一列的最亮像素和最暗像素，画出上下轮廓。

#### 2. 核心防御代码与语法对照
```rust
let mut min = f32::MAX;
let mut max = f32::MIN;
let mut has_finite = false;

for &sample in &samples[start..end] {
    if sample.is_finite() {
        min = min.min(sample);
        max = max.max(sample);
        has_finite = true;
    }
}
if end > start && has_finite {
    out[column] = (min, max);
}
```
* **英语语法对照**：
  * `has_finite`：名词状态标记（旗子），记录是否遇到过有效数字。
  * `min = min.min(sample)`：主语 `min` 调用谓语动词 `.min(...)`，更新最小值。
  * `if end > start && has_finite`：连词 `&&`（并且），只有切片非空且确实遇到了有效数字，才写入结果。
* **【合拢复盘】**：
  “每一列竖切片中，只有有效数字才有资格去刷新最小值和最大值，并立起旗子；如果这一列全是一堆损坏的 NaN，就不会写入反转的极值，而是保留最初安全的 `(0.0, 0.0)` 兜底。”

---

### 第 3 个函数：`vectorscope_points`（雷达打靶点）

#### 1. 标识符三步法
* **拆零件**：`vectorscope`（矢量示波器）+ `points`（点集坐标）。
* **拼整句人话**：**“矢量示波器上的投影打靶点”**。
* **软件职责**：把色度 $(u, v)$ 坐标映射到屏幕圆盘中心 `-0.5..0.5` 的坐标点。

#### 2. 核心防御代码与语法对照
```rust
samples
    .iter()
    .filter(|(u, v)| u.is_finite() && v.is_finite())
    .map(|&(u, v)| ((u - 0.5).clamp(-0.5, 0.5), (v - 0.5).clamp(-0.5, 0.5)))
    .collect()
```
* **英语语法对照（流水线）**：
  * `samples.iter()`：主语名词，拿出点集流。
  * `.filter(...)`：谓语动词，过滤筛子。
    * `|(u, v)|`：提取出横纵坐标两个主语。
    * `u.is_finite()`：第一个判断谓语（u 是不是有效）。
    * `&&`：连词 and（并且）。
    * `v.is_finite()`：第二个判断谓语（v 是不是有效）。
  * `.map(...)`：谓语动词，对留下来的合格点进行中心偏移映射。
  * `.collect()`：谓语动词，打包收集成最终数组。
* **【合拢复盘】**：
  “从输入的所有颜色坐标点里，用过滤筛子筛掉任何横坐标或纵坐标损坏的脏点，只让合格的双有效坐标去计算中心偏移，最后打包收集返回。”

---

### 第 4 个函数：`meter_lit_segments`（音量表点亮几格）

#### 1. 标识符三步法
* **拆零件**：`meter`（仪器表、音量表）+ `lit`（被点亮的）+ `segments`（格子、段）。
* **拼整句人话**：**“音量表里被点亮的格子数”**。
* **软件职责**：输入当前声音大小比例，算出音量柱该亮几格灯。

#### 2. 核心防御代码与语法对照
```rust
pub fn meter_lit_segments(level: f32, segments: usize) -> usize {
    if level.is_nan() {
        return 0;
    }
    let lit = (level.clamp(0.0, 1.0) * segments as f32).round() as usize;
    lit.min(segments)
}
```
* **英语语法对照**：
  * `if level.is_nan()`：主语 `level` + 谓语 `.is_nan()`，如果是坏数据直接提前交卷 `return 0`。
  * `level.clamp(0.0, 1.0)`：把音量强制卡在 0 到 1 之间。
  * `* segments as f32`：乘以总格数（例如 10.0 格）。
  * `.round()`：四舍五入。
  * `lit.min(segments)`：主语 `lit` 调用动词 `.min`，确保最多亮到满格，绝不越界。
* **【合拢复盘】**：
  “如果声音数据坏掉了（`is_nan`），直接亮 0 格静音；否则把音量卡在 0~1 之间，乘总格数并四舍五入，最后取两者较小值保证不超过最大格数。”

---

### 第 5 个函数：`decay_peak`（最高峰平滑衰减）

#### 1. 标识符三步法
* **拆零件**：`decay`（衰减、慢慢下降）+ `peak`（最高峰值）。
* **拼整句人话**：**“衰减最高峰值 / 峰值小横线慢慢飘落”**。
* **软件职责**：算出每一帧最高峰值小横线应该停留在哪个新位置。

#### 2. 核心防御代码与语法对照
```rust
pub fn decay_peak(peak: f32, level: f32, decay_per_frame: f32) -> f32 {
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
}
```
* **英语语法对照**：
  * 前三行：清洗输入，把可能的 NaN 或无穷大重置为安全的 0.0。
  * `if level >= peak`：如果当前声音创了新高，峰值立刻冲上去等于 `level`。
  * `(peak - decay).max(level)`：如果声音变小了，让峰值减去掉落步长，并调用 `.max(level)` 托底，保证峰值绝不低于当前正在发声的大小。
* **【合拢复盘】**：
  “先把三个输入量清洗干净，排除无穷大和 NaN；如果当前声音更大就冲顶刷新；如果当前声音较小就慢慢往下掉，但绝不掉得比当前声音还低。”

---

## 四、 工业级测试驱动开发（TDD）轻量验证法

### 1. 为什么不用跑全仓重度编译？
在 Oak 项目中，直接跑 `cargo test` 会去拉取 GPU 渲染管线、PipeWire 音频驱动和自编译 FFmpeg C 库，需要庞大的系统环境支持。
但 `math.rs` 是**无耦合的纯数学函数（Pure Math Functions）**。
在 408 软件工程规范中，模块的高内聚低耦合使得我们可以直接使用 Rust 官方编译器进行单文件毫秒级单测验证：

```cmd
rustc --test crates\oak-app\src\widgets\scopes\math.rs -o math_test.exe
math_test.exe
del math_test.exe math_test.pdb
```

### 2. 边界值分析（BVA）新增的 4 组极限单测

```rust
// 1. 直方图极端测试：包含 NaN、正负无穷大，验证非法值被彻底跳过
#[test]
fn histogram_filters_nan_and_inf() {
    let samples = [f32::NAN, 0.5, f32::INFINITY, -f32::INFINITY];
    let bins = histogram_bins(&samples, 2);
    assert_eq!(bins, vec![0, 1]);
}

// 2. 波形信封极端测试：整段全是 NaN 与 Inf，验证安全回退 (0.0, 0.0)
#[test]
fn waveform_envelope_handles_nan_and_inf() {
    let samples = [f32::NAN, f32::INFINITY, -f32::INFINITY];
    let env = waveform_envelope(&samples, 1);
    assert_eq!(env, vec![(0.0, 0.0)]);

    let mixed = [f32::NAN, 0.2, 0.8, f32::INFINITY];
    let env_mixed = waveform_envelope(&mixed, 1);
    assert_eq!(env_mixed, vec![(0.2, 0.8)]);
}

// 3. 矢量示波器极端测试：坐标含有 NaN 或 Inf，验证被安全筛除
#[test]
fn vectorscope_filters_nan_and_inf() {
    let samples = [(0.5, 0.5), (f32::NAN, 0.5), (0.5, f32::INFINITY), (1.0, 0.0)];
    let points = vectorscope_points(&samples);
    assert_eq!(points.len(), 2);
}

// 4. 音量表与衰减极端测试：验证静音兜底与正无穷不卡死
#[test]
fn meter_and_decay_nan_and_inf() {
    assert_eq!(meter_lit_segments(f32::NAN, 8), 0);
    assert_eq!(decay_peak(0.8, f32::NAN, 0.1), 0.7);
    assert_eq!(decay_peak(0.8, f32::INFINITY, 0.1), 0.7);
}
```

**运行结果证据**：
```text
test result: ok. 10 passed; 0 failed; finished in 0.00s
```

---

## 五、 开源协作规范与思维模型沉淀

1. **Fork & Pull Request 工作流**：
   * 外部贡献者没有主仓的写入权限；
   * 正规路径：`网页 Fork 到个人仓` $\rightarrow$ `本地分支修改` $\rightarrow$ `推送个人仓` $\rightarrow$ `向官方发起 PR`。
2. **Conventional Commits 提交规范**：
   * `fix(scopes): handle non-finite samples safely in scope math (#83)`
   * 类型明确（`fix`）、作用域清晰（`scopes`）、动词祈使句开头，且末尾直接关联 Issue 编号 `#83`。
3. **开源展示与隐私边界准则**：
   * 技术成果、补丁、PR 和架构分析要**完全公开开源（Public）**，向社区和业界展示自己的工程素养；
   * 现实个人身份（真名、学号、学校、考研隐私）**100% 物理脱敏隔离**，统一使用 GitHub 公开开发者 ID 交流。
