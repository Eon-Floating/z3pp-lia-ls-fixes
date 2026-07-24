# LIA Local Search 版本记录

本文记录当前 QF_LIA local-search 优化过程中几个主要版本的改动、验证结果和对应日志目录。

## 版本概览

| 版本 | 对应状态 | 主要改动 | 三年 SMT-COMP QF_LIA 结果 |
| --- | --- | --- | --- |
| baseline | `qflia-lssmt-smoke` | 原始 LIA local search 行为 | solved 7217, error 1, PAR2 4.509 |
| score0-occurs | `bd74d90` | `score == 0` 时优先 occurrence 更少的 LIA 变量 | solved 7275, error 3, PAR2 4.271 |
| crash-fixes | `c48b5f8` | 修复 score0-occurs 暴露的数组越界和空实例崩溃 | solved 7271, error 0, PAR2 4.286 |
| low-residual reversal | 当前主版本组成 | 低残差纯 LIA 阶段跳过 tabu 中的 `score == 0` 直接反向 move | solved 7286, error 0, PAR2 4.253 |
| fixed resolution limits | 实验版本 | 固定限制 resolution 笛卡尔积和 clause 增长 | solved 7447, error 0, PAR2 4.069 |
| adaptive resolution limits | 当前主版本 | 只在 resolution 已明显膨胀后限制大消元 | solved 7488, error 0, PAR2 3.942 |

## baseline

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-smoke
```

这是原始 Z3++ LIA local search 基线结果。

三年 SMT-COMP QF_LIA：

```text
2023: solved=2634, error=1, PAR2=4.122
2024: solved=2330, error=0, PAR2=4.353
2025: solved=2253, error=0, PAR2=5.098
ALL : solved=7217, error=1, PAR2=4.509
```

## score0-occurs

提交：

```text
bd74d90 prefer sparse zero-score lia moves
```

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-score0-occurs-full
```

改动内容：

- 在 `pick_critical_move(__int128_t &best_value)` 中调整 LIA candidate tie-break。
- 当 `score == 0` 且候选分数相同时，优先选择 `_vars[var].literals.size()` 更小的变量。
- occurrence 相同后，再按原有 `last_move` 逻辑打破平局。
- 同一规则也应用于 swap-candidate fallback。
- 不改变 `score > 0` 的优先级，不改变 random walk、bool move 或公式语义。

目的：

- 避免 `score == 0` 平台上优先移动高 occurrence 变量，减少无收益移动对大量 literal 的扰动。

三年 SMT-COMP QF_LIA：

```text
2023: solved=2646, error=1, PAR2=4.010
2024: solved=2339, error=1, PAR2=4.249
2025: solved=2290, error=1, PAR2=4.584
ALL : solved=7275, error=3, PAR2=4.271
```

说明：

- 该版本整体 solved 和 PAR2 明显优于 baseline。
- 但它改变搜索轨迹后暴露出原实现中的若干内存安全问题，因此不适合作为最终主版本。

## crash-fixes

提交：

```text
020f087 修复 LIA 局部搜索候选数组越界
c48b5f8 修复 LIA 变量归约空实例崩溃
```

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-after-crash-fixes
```

改动内容：

- `insert_operation()` 中，当 `operation_idx` 超过 `operation_var_idx_vec` / `operation_change_value_vec` 当前容量时动态扩容。
- `critical_score_subscore()` 中，`lit_exist` 在当前实例或实际 `lit_idx` 超过容量时动态扩容。
- `reduce_vars()` 中，先初始化空 `pair_x` / `pair_y`，然后在 `_tmp_vars.size() == 0` 时直接返回，避免 `tmp_vars_size - 1` unsigned 下溢。

定位到的问题：

```text
insert_operation.bad_operation_idx:
  operation_idx == vector.size()
  写 operation_var_idx_vec[operation_idx] 越界

critical_score_subscore.bad_lit_exist:
  lit_idx >= lit_exist.size()
  static lit_exist 复用旧容量导致越界

reduce_vars tmp_vars_size_zero:
  tmp_vars_size == 0
  tmp_vars_size - 1 unsigned 下溢，随后访问空数组
```

三年 SMT-COMP QF_LIA：

```text
2023: solved=2644, error=0, PAR2=4.039
2024: solved=2337, error=0, PAR2=4.276
2025: solved=2290, error=0, PAR2=4.572
ALL : solved=7271, error=0, PAR2=4.286
```

说明：

- 这是当前已经提交、完整验证过的稳定主版本。
- 相比 baseline：三年 net solved +54，error 从 1 降到 0。

## low-residual reversal

状态：

```text
当前主版本组成
```

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-low-residual-reversal
```

改动内容：

- 新增上一条 LIA move 记录：`_last_lia_move_var_idx` 和 `_last_lia_move_change`。
- 在普通 critical LIA candidate 和 swap fallback candidate 中，跳过一种非常窄的无收益振荡 move：

```text
unsat_clauses <= 5
_bool_lit_in_unsat_clause_num == 0
score == 0
当前 move 是上一条 LIA move 的直接反向
当前方向仍在 tabu 中
```

对应过滤条件：

```cpp
if(unsat_clauses->size()<=5&&_bool_lit_in_unsat_clause_num==0&&score==0&&reversal&&tabu_before){continue;}
```

目的：

- 针对低残差平台上的 `+1/-1` 直接反向振荡。
- 特别是 SharedMemory 类实例中观察到的模式：同一个变量在 tabu 状态下反复执行 `score == 0` 的正反 move，既不减少 unsat clause，也不减少 unsat literals。

三年 SMT-COMP QF_LIA：

```text
2023: solved=2652, error=0, PAR2=3.995
2024: solved=2341, error=0, PAR2=4.248
2025: solved=2293, error=0, PAR2=4.549
ALL : solved=7286, error=0, PAR2=4.253
```

相对 crash-fixes：

```text
2023: net +8
2024: net +4
2025: net +3
ALL : gained=31, lost=16, net +15
```

说明：

- 改动很小，作用条件很窄。
- 三年完整结果均为正收益，且没有引入 error。
- 主要负面影响仍集中在少数 `bofill-scheduling/SMT_real_LIA` 实例。

## fixed resolution limits

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-resolution-limits
```

改动内容：

- 当单个 bool variable 的正负 clause 笛卡尔积超过 `100000` 时跳过该次消元。
- 当 resolution 生成的 clause 存储超过初始数量的 2 倍时结束 resolution。
- 目的为避免 `nec-smt` 中 resolution 第一轮将约 2 万个 clause 膨胀到 20 多万个。

三年 SMT-COMP QF_LIA：

```text
2023: solved=2696, error=0, PAR2=3.819
2024: solved=2388, error=0, PAR2=4.120
2025: solved=2363, error=0, PAR2=4.298
ALL : solved=7447, error=0, PAR2=4.069
```

相对 low-residual reversal：

```text
gained=188, lost=27, net +161
```

## adaptive resolution limits

状态：

```text
当前主版本
```

日志目录：

```text
/data/yx/WorkPlace/SMTTestLogs/qflia-lssmt-resolution-adaptive
```

改动内容：

- 保留 clause 存储超过初始数量 2 倍时结束 resolution 的全局限制。
- 只有当前 clause 存储已经超过初始数量的 1.5 倍时，才对大于 `100000` 的正负 clause 笛卡尔积跳过消元。
- 正常公式的早期 resolution 不受单变量限制；出现明显膨胀后才启用保护。

当前条件：

```cpp
if(_num_clauses>2*initial_num_clauses){return;}

uint64_t resolution_pairs=
    static_cast<uint64_t>(pos_clause_size)*neg_clause_size;
if(_num_clauses>initial_num_clauses+initial_num_clauses/2&&
   resolution_pairs>100000){
    continue;
}
```

三年 SMT-COMP QF_LIA：

```text
2023: solved=2712, error=0, PAR2=3.730
2024: solved=2408, error=0, PAR2=3.929
2025: solved=2368, error=0, PAR2=4.192
ALL : solved=7488, error=0, PAR2=3.942
```

相对 fixed resolution limits：

```text
gained=52, lost=11, net +41
PAR2: 4.069 -> 3.942
```

相对 low-residual reversal：

```text
gained=213, lost=11, net +202
```

相对 baseline：

```text
gained=293, lost=22, net +271
error: 1 -> 0
PAR2: 4.509 -> 3.942
```

## 当前建议

- 若要求只使用已提交版本，推荐 `c48b5f8`：稳定、三年 error 为 0。
- 当前主版本为 adaptive resolution limits，并包含 low-residual reversal 和 crash fixes。
