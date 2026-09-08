# protein-structure-ai  — 蛋白质结构 × AI

蛋白质结构相关的研究，含结构预测、构象采样、结构设计、序列设计、动力学分析等；方法限定为深度学习、分子动力学与对接。

每周从 PubMed / arXiv / bioRxiv / medRxiv / chemRxiv 抓取最新文献元数据，提交并推送回本仓库，同时创建一条 Issue 汇总。本地用 Zotero 按 `_ids.txt` 批量导入筛选。

反漏设计：推送周期 P=7（每周一跑），查询窗口 W=14（`window_days: 14`）> P，重叠半窗由 `.monitor_state/` 状态文件去重（`dedup: true`）。保证 W ≥ P + L（L = PubMed `[dp]` 标引时滞上限 ~7 天），不漏迟标引的新条目。Issue 收紧为「标题 | 日期」两列（`compact_issue: true`），因为峰值周全格式会超 GitHub Issue 65KB 上限。

## 仓库结构

- `monitor.py` — 读取 `config.yaml`，逐平台检索，规范化后写入 `Discovery/`（合并 CSV + `_ids.txt`）与 `Archive/`（逐篇元数据 JSON），并生成 Issue 正文与标题；内置 `.monitor_state/` 重叠窗口去重。
- `config.yaml` — 检索配置：课题短名、时间窗口、`dedup`/`compact_issue` 开关、每平台一条布尔检索式。PubMed 邮箱与 API key 通过环境变量注入，不写入文件。
- `.github/workflows/monitor.yml` — 每周一 09:23 UTC 自动运行，支持 `workflow_dispatch` 手动触发。

## 产出

```
Archive/                                # 逐篇完整元数据 JSON，只增不删，按发表年月归档
  {source}/{year}/{month}/{id}/{id}.json
Discovery/                              # 每次运行一份合并 CSV 与 _ids.txt，按抓取日归档
  {year}/{month}/protein-structure-ai_{date}.csv
  {year}/{month}/protein-structure-ai_{date}_ids.txt
.monitor_state/                         # dedup 状态文件（已 emit 的 zotero id 集合），随 commit 持久化
  protein-structure-ai.json
```

`source` 取值为 `pubmed`、`arxiv`、`biorxiv`、`medrxiv`、`chemrxiv`。

CSV 共 9 列：`source, id, doi, title, authors, journal, published_date, url, abstract`。`id` 为各平台主键（PubMed 为 PMID，预印本为 DOI），`doi` 为跨平台规范标识，`published_date` 统一为 ISO 日期（PubMed 的 DP 字段归一为 `YYYY-MM-DD`）。

`_ids.txt` 每行一个标识符，带类型前缀（`pmid:xxx`、`arXiv:xxx`，DOI 裸写），供 Zotero「按标识符添加」批量导入。

**重叠窗口去重**：开启 `dedup: true` 后，`Discovery/` 与 Issue 只落「新条目」（宽窗重叠部分被状态文件过滤），`Archive/` 不受影响、仍幂等归档每一次抓取。**不做跨平台去重**，同一篇文献若同时出现在两个平台会各保留一条，由 Zotero 最终去重。当周无命中时，CSV 仅含表头。

## 密钥（PubMed）

PubMed 检索需要邮箱（必填）与 NCBI API key（可选），通过环境变量注入，不写入仓库：

- 本地：`export ENTREZ_EMAIL=you@example.com`，可选 `export NCBI_API_KEY=...`
- GitHub Actions：仓库 Settings → Secrets and variables → Actions → New repository secret，添加 `ENTREZ_EMAIL` 与 `NCBI_API_KEY` 两个 secret。

## 本地运行

```bash
pip install pyPaperFlow
export ENTREZ_EMAIL=you@example.com
python monitor.py --config config.yaml --out-dir . --issue-body /tmp/issue.md
```

调整时间窗口：`--window-days 1`，或改 `config.yaml` 的 `window_days`（默认 14，配合 `dedup` 反漏；手动收窄窗口会削弱反漏保护）。

平台 query 语法与调优记录见母仓 `docs/topics-catalog.md` 与本课题 `topics/protein-structure-ai/test-notes.md`。
