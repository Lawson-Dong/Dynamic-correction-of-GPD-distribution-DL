# 共用代码与模型：因子、前向传播、配置规则各只有一份实现

| 文件 | 内容 | 使用者 |
|---|---|---|
| `factor_core.py` | 13 个因子的**唯一**实现（价格因子、目标 Y、历史因子、VIX / SEC 对齐、实盘因子面板） | `features.py`（构建 `dataset.csv`）、`quantconnect/main.py` |
| `tail_model.py` | 两段模型的**唯一**推理实现：前向传播、S(y\|x)、VaR、ES、风险评分 R、回测工具、模型版本的读写 | `evaluate_tail.py`、`gpd_predict.py`、`quantconnect/main.py`、`trading_strategy.ipynb` |
| `strategy_core.py` | 配置规则 `allocate_from_risk`：R → v → B → c → w | `quantconnect/main.py`、`trading_strategy.ipynb` |
| `models/` | 模型版本（JSON）；`models/LATEST` 写着当前版本，说明见 `models/README.md` | 以上所有使用者 |

三个代码文件都只依赖 numpy / pandas，整个项目不需要 torch。每个文件都在 3.2 万字符以内，可以原样加入 QuantConnect 项目。

```
features.py ──▶ factor_core.py ◀── quantconnect/main.py
                      │                     ▲
             artifacts/dataset.csv          │ Object Store（models/ 里的 JSON）
                      │                     │
evaluate_tail.py ─┐   ▼                     │
gpd_predict.py ───┼─▶ tail_model.py ──▶ models/LATEST ──▶ models/<版本>.json
notebook ─────────┘
```

## `tail_model.py` 用法

```python
from tail_model import TwoStageTailModel

m = TwoStageTailModel.load()                   # models/LATEST
m = TwoStageTailModel.load("models/<版本>.json")
m.version, m.info                              # 版本、配置、数据区间、验证结果

out = m.predict(df)                    # DataFrame：xi_b, beta_b, xi_h, beta_h, pi, p_u
m.risk_score(df, d=0.07)               # R = E[(L-d)^+] = p_u · E[(L-d)^+ | L>u]
m.risk_score(df, d=0.07, use_pu=False) # 只看条件部分
m.tail_prob(df, loss=0.10)             # P(第二天跌幅 > 10%)
m.var(df, alpha=0.01), m.es(df, alpha=0.01)   # 无条件 VaR99 / ES99（p_u ≤ α 时为 NaN）
m.cond_survival(df, y), m.cond_quantile(df, 0.99)
```

输入：第 t 天的 13 个**原始**因子（未标准化），可以是 DataFrame、dict 或 (n, 13) 数组。NaN 会填成训练期中位数。

函数版本：输入参数数组 `(n, 5)`，供画图和检验使用：`mixture_survival`、`mixture_logpdf`、`mixture_pdf`、`mixture_quantile_excess`、`tail_probability`、`var_es`、`mixture_tail_risk`、`gpd_expected_excess`、`auc_score`、`kupiec_test`、`var_backtest`、`var99_by_decile`。

命令行：

```bash
python tail_model.py --selftest [--model models/<版本>.json]
python tail_model.py --csv factors.csv --out predictions.csv --d 0.07 --alpha 0.01
python tail_model.py --export-artifacts <PyTorch 产物文件夹> --version <版本名> --description "..."
```

## QuantConnect 项目需要的文件

- `main.py`（来自 `quantconnect/`），以及仓库根目录原样复制的 `factor_core.py`、`tail_model.py`、`strategy_core.py`；
- Object Store：把 `models/LATEST` 指向的 JSON 上传到键 `MODEL_KEY`；可选上传 `qc_sec_factors.csv`（键 `SEC_KEY`）。

`main.py` 的配置项 `USE_PU = True`：风险评分乘上 p_u。设为 False 就是原来只用条件部分的做法。

## 测试

```bash
python tests/test_parity.py
```

1. `factor_core` 和构建数据集时的因子公式逐项一致；
2. `models/` 里所有版本都能读取，JSON 读写前后输出完全一致，特征顺序与 `factor_core` 一致，`LATEST` 指向的文件存在；
3. 如果有 `artifacts/dataset.csv`，能复现最新版本 `info` 里记录的验证结果；
4. 配置规则。
