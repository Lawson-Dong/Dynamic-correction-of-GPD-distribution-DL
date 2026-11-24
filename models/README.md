# models/：模型版本

每个模型是一个 JSON 文件，`LATEST` 里写着当前使用的版本。整个项目（`evaluate_tail.py`、`gpd_predict.py`、notebook、QuantConnect）默认读取 `LATEST` 指向的模型，推理代码统一用仓库根目录的 `tail_model.py`。

```python
from tail_model import TwoStageTailModel

m = TwoStageTailModel.load()                       # models/LATEST
m = TwoStageTailModel.load("models/2026-11-21_two-stage_wide-heavy.json")   # 指定版本
m.version, m.info["description"], m.info["results"]["validation"]
```

## 文件内容（格式 `gpd_mixture_numpy_v1`）

| 字段 | 内容 |
|---|---|
| `meta` | 推理必需的信息：特征顺序、训练期中位数/均值/标准差、截断值、阈值 u、π_min、BN ε |
| `weights` | 第二段（门控 GPD 混合模型）的权重和 BN 滑动统计量 |
| `pu` | 第一段（跌破概率网络 p_u）的权重 |
| `info` | 版本号、说明、创建时间、架构和训练配置、数据区间与切分日期、验证结果（`results`、`p_u.results`） |

## 版本

| 版本 | 说明 | 验证集（2023-03-30 起） |
|---|---|---|
| `2026-11-21_two-stage_wide-heavy` | 两段模型：p_u 网络 + 门控 GPD 混合模型（gate_bn32，π_min 0.01，θ_β 初值 1.85，lr 3e-4），13 个因子；训练数据 2016-03 至 2023-03 | GPD NLL −3.1203；S<1% 0.89%，S<2.5% 2.12%；p_u AUC 0.711；无条件 VaR99 超出率 0.83% |

## 新增一个版本

1. 生成 JSON：
   - 用 `TwoStageTailModel(...).save_version(版本名, 说明, **其他信息)` 写成 `models/<版本名>.json`，同时更新 `LATEST`；
   - 如果是 PyTorch 训练的结果：`python tail_model.py --export-artifacts <文件夹> --version <版本名> --description "..."`。
2. 如果架构变了，在同一次提交里修改 `tail_model.py` 的前向传播。
3. 运行 `python tests/test_parity.py`：会检查所有版本都能正常读取，`LATEST` 指向的文件存在，特征顺序与 `factor_core.py` 一致。
4. 在上面的表格里补一行。
5. QuantConnect：把新的 JSON 上传到 Object Store（键 `MODEL_KEY`）。

旧版本保留不删，方便对比和回退：把 `LATEST` 改回旧文件名即可。
