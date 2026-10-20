# 逐环命令与产物

工作目录固定 `<PROJECT_ROOT>`。先设环境：

```powershell
Set-Location "<PROJECT_ROOT>"
$env:NO_PROXY="127.0.0.1,localhost"; $env:no_proxy="127.0.0.1,localhost"
```

**环 0 之前先体检**：`python scripts\kjx_status.py`

---

## 环 0 · 采集

### 0.1 1688 列表全量（拿 offerId 清单）

```powershell
node scripts\kjx_harvest_mtop.js <店铺memberId> <out.json>
```

<店铺A> memberId `<MEMBER_ID_1>`，<店铺B> `<MEMBER_ID_2>`。
<店铺A>全店 599 件（20 页 × 30）已用 mtop 接口直连采齐。

mtop 签名：`sign = md5(token + "&" + t + "&" + appKey + "&" + data)`，
`token` = 浏览器 cookie `_m_h5_tk` 下划线前半段。**必须**在页面内
`page.evaluate(fetch(url, {credentials:'include'}))` 发请求，纯 Node/Python 直连会被
防盗链拦成 captcha（`FAIL_SYS_USER_VALIDATE`）。

### 0.2 详情页一趟取三样（件重 + 运费 + 入库）

```powershell
node scripts\kjx_fetch_both.js 2 --file recon\b3_ids_v2.txt --gap 12000
```

- 并行数 2（更多会撞风控），`--gap 8000~12000` 是商品间隔
- 断点续跑状态：`recon/b3_state.json`
- 采集走详情页插件「采集本商品」→ `POST api/Collect/add`（**不需要** ERP 的 1688 货源授权）
- 列表页批量采集 `POST /api/collect/addAlibabaProducts` 需要授权，当前失效，不要用
- 产物：`recon/weights.json`（件重）、`recon/freight.json`（运费）

**手工补抓的运费放独立文件** `recon/freight_extra.json`，读的时候合并——
后台脚本每件都重写 `freight.json`，手工补丁会被冲掉。

### 0.3 补抓缺失件

同一命令换 `--file` 只列缺失的 offerId 即可，脚本按已有记录跳过。

---

## 环 1 · IP 预筛（VLM）

```powershell
python scripts\kjx_vlm_review.py --all --imgs 5    # 主图竖拼长图 -> 上传 -> 调 VLM
python scripts\kjx_vlm_retry.py                     # FAIL 件重试，必须跑
python scripts\kjx_vlm_summary.py                  # 角色词频 / 厂牌 Top / IP 件清单
python scripts\kjx_vlm_triage.py                    # A硬IP / B杯子白名单 / C通用动物
python scripts\kjx_final_list.py                    # 产出 final_blocklist.txt / final_passlist.txt
```

产物：`recon/vlm_review.json`（三栏结论）、`recon/final_blocklist.txt`、`recon/final_passlist.txt`、
`recon/sheets/_vlm/<offerId>.jpg`（拼好的长图）。

VLM 调用走 `mcode-tools connector call connector__matrix__describe_images`，
**每批 10 张，返回顺序与入参严格对应**。调用必须 `timeout` + 重试 + 标 FAIL——
外部调用挂死 3 分钟无返回是实测过的。

---

## 环 2 · 认领（破坏性，需用户确认）

```powershell
python scripts\kjx_make_claim_list.py --out wave2          # 预览
python scripts\kjx_make_claim_list.py --out wave2 --apply  # 提交认领
```

分批（每批 30），全部要 `code=1000`。认领后 ERP 还没同步完，
**紧接着的写回会静默失败**，所以认领后要么等，要么写回后回读校验。

---

## 环 3 · 标题

```powershell
python scripts\kjx_check_title.py recon\title_wave2_b1.json   # 自检，逐批
python scripts\kjx_set_title.py --map recon\title_map_wave2_all.json --apply
```

**两个脚本吃的 JSON 格式不一样，务必分清：**

| 脚本 | 格式 | 例 |
|---|---|---|
| `kjx_check_title.py` | `{"wave": "b1", "titles": {"<claim_id>": "标题"}}` | dict，key 是认领 id |
| `kjx_set_title.py` | `[{"id": <CLAIM_ID>, "offerId": "763...", "title": "标题"}]` | **list**，每项一个商品 |

`kjx_check_title.py` 也接受裸 list-of-strings，但**不能**直接吃 set_title 的 list-of-dict。
写完标题先用 check_title 自检，通过后再转成 set_title 的 list 格式。

- `kjx_set_title.py` 会自动重试 3 次（连接被重置）+ 写回后回读校验
- **只提交 `{id,title}` 会覆盖图片/价格**，脚本内部已回填全量字段
- 合并多批用 `kjx_make_title_map.py`（输出 set_title 的 list 格式）

---

## 环 3.5 · 基线快照（改任何东西之前先做）

七环写回会覆盖标题、价格、图片。**改之前先把这一批的全量字段落盘**，
这是唯一的回退依据——没有它，出了问题只能靠记忆手工改回去。

```powershell
# 用 getClaimGoodsInfo 把这批的全量字段导成 JSON
python scripts\kjx_snapshot_claim.py --ids-file recon\wave2_run_ids.txt --out recon\snapshot_before.json
```

快照文件是**只读参照**，写回出错时从它逐字段比对还原。
批量操作前没有快照 = 没有回退路径。

---

## 环 4 · 七环写回

```powershell
# 已认领的（按认领 id）
python scripts\kjx_batch_run.py --apply --no-fetch --claim-ids <CLAIM_ID>,<CLAIM_ID>,<CLAIM_ID>
# 或用文件
python scripts\kjx_batch_run.py --apply --no-fetch --ids-file recon\wave2_run_ids.txt
```

`--no-fetch` 表示跳过抓取直接写回。脚本内置 `--dry` 只认领+体检不改。

**必带回读校验**：批量改完重新 GET 详情比对每个字段，不能只看接口返回 1000。
失败件用 `--claim-ids` 单独补跑，不要重跑整批。

---

## 环 5 · 终检

```powershell
python scripts\kjx_audit.py
python scripts\kjx_audit.py --ids-file recon\本批_claimids.txt   # 只审这一批
```

输出「N 品 / M 项全部通过」。**用的是修复过分页的版本**——修复前它只查 200 条却报「全部通过」。

检查项含 **起订量 = 1**（硬约束，查不到起订量也算不过）。

---

## 环 6 · 发布（破坏性，需用户确认）

```powershell
python scripts\kjx_publish.py --apply --discount 0 --ids-file recon\wave2_publish_ids.txt
```

- `--ids-file` 而不是 `--ids`：PowerShell 传 297 个 id 会转义炸
- `price_discount` 一律 0
- **发布前有起订量闸门**（2026-10-20）：起订量 > 1 的货**一件都不发**，
  闸门不通过直接中止，`--force-moq` 可强推（只在人工逐件确认后用）

### 起订量闸门（硬约束）

```powershell
python scripts\kjx_moq_gate.py --claims-file recon\本批_claimids.txt   # 按认领 id
python scripts\kjx_moq_gate.py --offers-file recon\ids.txt              # 按 offerId
python scripts\kjx_moq_gate.py --published                              # 查已发布商品
```

- 退出码：0 全过 / 1 有起订量违规 / 2 有查不到的（**默认查不到 = 不可发布**）
- 全局起订量表在 `recon/moq_global.json`，由 `kjx_moq.py` 单一维护
- **查不到 ≠ 1 件起订**。手动采的货不在列表数据里，必须单独查。
- 一次发布到「企业」<MERCHANT_ID> = 铺满旗下 7 个 <你的店铺名> 店（<店铺 id 区间>）
- 发布后核对：`python scripts\kjx_publish_records.py`、`python scripts\kjx_wave2_verify.py`

---

## 导出已发布商品报表（只读）

```powershell
python scripts\kjx_export_published_goods.py
```

输出 `recon\已发布商品报表.xlsx`，3 个页签：
「已发布商品」（364 行 × 7 列：offerId / 标题 / 结算价 / 库存(SKU合计) / 重量 / Shopee 状态 / base_item_id）、
「报表说明」（口径 + 状态说明 + 结算价区间 + 重量分布）、「发布失败记录」（9 条历史残留）。

**口径要点（照抄，别自己重新推导）**：
- 「已发布」= `publishRecord.status==1`，按 `base_item_id` 去重取 `publish_time` 最新一条
- 重量只存在于 `getClaimGoodsInfo.weight`（kg），且按**认领 id** 查 → 直接用
  `publishRecord.claim_id` 取，**不要绕 base_item_id join**（认领 id 才是不二键）
- **库存列是各档 SKU 之和**（5 档 = 4440），不是单档 888——列名已标注「(SKU合计)」，别改回去
- 认领已删的件（<CLAIM_ID>）重量留空并报出来，不瞎填

## 状态查询

```powershell
python scripts\kjx_status.py            # 体检 + 全局状态汇总
python scripts\kjx_box_status.py        # 认领箱
python scripts\kjx_claim_to_shop.py     # 认领 → 店铺商品
python scripts\kjx_publish_records.py   # 发布记录
python scripts\kjx_wave2_verify.py      # 第二波专项核对
```

## 导出已发布商品报表（只读）

```powershell
python scripts\kjx_export_published_xlsx.py                        # 默认写 recon\published_goods_report.xlsx
python scripts\kjx_export_published_xlsx.py --out recon\x.xlsx       # 换输出路径
python scripts\kjx_export_published_xlsx.py --no-weight              # 跳过 363 次详情取重量（快，重量列空）
python scripts\kjx_export_published_xlsx.py --verbose                # 打印全部明细行
```

- **纯只读**，只调 `/shopProductsList` + `/index` + `/getClaimGoodsInfo`，不含任何写操作
- 列：`offerId | 标题 | 结算价 | 库存 | 重量 | Shopee 状态 | base_item_id`（另带一个「说明」页记录字段来源）
- 产物：`recon/published_goods_report.xlsx`（openpyxl，首行加粗 + 冻结 + 筛选）

## 撤回已发布商品（发现硬 IP 时）

```powershell
# 1) 删认领记录 + 删采集箱条目（一条命令）
python scripts\kjx_del_claim_full.py <CLAIM_ID> --apply
# 2) 已发布过的，还要处理店铺商品记录
python scripts\kjx_unlist_by_offer.py <OFFER_ID> --apply
```

**注意**：`unlistShopProduct` 在 7 店未同步时是假成功（返回 1000 但商品仍在列表）。
**店铺审核通过前，已发布的商品撤不下来**——所以 IP 必须在发布前拦掉。

## 回溯 IP 筛查（改过规则后回头查已发布的批次）

**管道里原先缺这个能力**——认领前的预筛覆盖不到「规则后来才发现漏」的件。
实测就是靠它查出一件已上架的**美少女战士**漏网品（第一波人工读图复核时漏掉）。

```powershell
# 1) 查已认领/已发布批次里哪些件不通过现行 IP 判定
python scripts\kjx_ip_audit_claimed.py --all
python scripts\kjx_ip_audit_claimed.py --ids-file recon\b3_ids_v2.txt

# 2) 老批次没跑过 VLM 的件，先从 ERP 取主图补预筛
python scripts\kjx_vlm_review.py --ids-file recon\novlm_ids.txt --from-erp-ids recon\novlm_claimids.txt
```

- `kjx_ip_audit_claimed.py` 的**主键是 1688 offerId**（VLM 结论按 offerId 存），
  认领 id 通过 `recon/claim_id_by_offer.json` 反查
- `--from-erp-ids` 让 VLM 预筛**从 ERP 认领详情的 `goods_img` 取主图**，
  不依赖 `recon/shop_*_all.json` 快照（老批次的 offerId 不在快照里）

## 一次性修复脚本

`recon/_fix_paging.py` 跑完即可删。

## 换一家 1688 店开新批次（整店侦察 → 认领 → 写回 → 终检）

**参数化后的跨批次标准流程**。默认路径都指向上一批，跑之前先确认每条命令读的
是不是本批文件（见 traps 31、32）。

```powershell
Set-Location "<PROJECT_ROOT>"
$env:PYTHONIOENCODING="utf-8"
$env:KJX_MEMBER="<MEMBER_ID>"          # 必填：店铺 memberId，缺失直接报错退出

# 1) 整店侦察（mtop 列表接口一次拿全，含起订量 _raw.quantityBegin）
node scripts\kjx_harvest_mtop.js <店铺URL> <out.json> *> recon\_harvest.log

# 2) 同款去重（并查集 + 主图原图 basename + 归一化标题）
python scripts\kjx_dedupe_umbrella.py

# 3) 品类筛（排带电/LED 等）
python scripts\kjx_umbrella_screen.py

# 4) VLM 主图 IP 预筛 + 生成放行/拦截清单
python scripts\kjx_vlm_review.py --ids-file recon\<batch>_vlm_ids.txt --imgs 5
python scripts\kjx_final_list.py --cand recon\<batch>_cands.json --ids-file recon\<batch>_vlm_ids.txt --out-prefix <batch>_

# 5) 采集入库（详情页插件路径，不需 1688 授权）
node scripts\kjx_fetch_both.js 2 --file recon\<batch>_vlm_ids.txt --gap 10000 *> recon\_fetch.log

# 6) 认领（--box 指定本批采集箱映射；破坏性，先问用户）
python scripts\kjx_make_claim_list.py --out <batch> --pass recon\<batch>_final_passlist.txt --box recon\<batch>_collect_map.json --apply

# 7) offerId -> 认领 id 映射
python scripts\kjx_map_claims.py --ids-file recon\<batch>_claim_ids.txt --out recon\<batch>_claimids.txt

# 8) 逐图宣传字样检测（主图+详情图，--out 写本批弃图清单）
python scripts\kjx_img_promo_scan.py --ids-file recon\<batch>_claimids.txt --out recon\<batch>_promo_imgs.json

# 9) 七环写回（--promo 指定本批弃图清单；--missing-weight 兜底件重）
$ids = (Get-Content recon\<batch>_claimids.txt) -join ','
python scripts\kjx_batch_run.py --apply --no-fetch --claim-ids $ids --missing-weight 1.00 --promo recon\<batch>_promo_imgs.json

# 10) 标题写回 + 终检 + 起订量闸门
python scripts\kjx_check_title.py recon\<batch>_titles.json
python scripts\kjx_set_title.py --map recon\<batch>_titles.json --apply
python scripts\kjx_audit.py --ids-file recon\<batch>_claimids.txt
python scripts\kjx_moq_gate.py --claims-file recon\<batch>_claimids.txt
```

**踩过的坑（会静默出错的那种）**：
- `kjx_make_claim_list.py` 认领依据是 `recon/boxid_by_offer.json`，
  **新店必须传 `--box`**，否则匹配数会是 0。
- 认领箱的货源 id **只在 `source_url` 里**，`/index` 没有独立 offerId 字段。
- 终检的件重口径要和写回一致：件重缺失按保守值核对（不判失败），运费缺失才判失败。
- `kjx_audit.py` 只审 `--ids-file` 指定的本批，不要跑全量（认领箱几百条）。
- 标题写回前先 `kjx_check_title.py` 自检（禁词表在 `kjx_rules.py`）。
