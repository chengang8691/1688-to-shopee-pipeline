---
name: kjx-shopee-pipeline
description: |
  Run the 1688 → Shopee 轻出海 (cross-border X / 红红火火 ERP) listing pipeline end to end:
  采集 → IP 预筛 → 认领 → 手写标题 → 七环写回 → 终检 → 铺货到 7 个 <你的店铺名> 店.
  Use when the user says 上货/铺货/发布到Shopee/采集1688/认领/改标题/改重量/改结算价/终检/
  轻出海/批次N波/发布失败排查, or asks about ERP 认领箱、发布记录、base_item_id、shopee_listed.
  Also use when a 1688 滑块/验证码 blocks a run, when 运费/件重 is missing, or when an IP
  (蛋仔派对/帕恰狗/Pingu/轻松熊/美少女战士/三丽鸥…) must be judged.
  Also use when the user wants to add a capability to this pipeline (新脚本/加功能/改管道),
  asks where a pipeline script lives, or asks to refresh the pipeline rules after a change.
  Do NOT use for: 妙手 ERP (已弃用), 直接连 Shopee 卖家中心 API (合规红线), 抖音/小红书采集,
  通用 ERP 咨询, or 1688 之外的货源平台.
---

# 1688 → Shopee 轻出海 上货管道

## 硬约束（不可协商）

1. **只走跨境X自家后台**：`<ERP_API_HOST>/api/*` 与 `<ERP_WEB_HOST>/api/*`。
   **绝不直接用 API 连 Shopee 卖家中心**。跨境X → Shopee 那一跳由平台用用户店铺授权完成。
2. **凭证不进上下文**：token 存 `<PROJECT_ROOT>\token.txt`，脚本用 `--token-file token.txt`。
   不要把 token 打印、粘贴进对话或写进记忆。
3. **破坏性操作先问用户**：认领、删除、发布这三类动作执行前必须拿到用户明确确认。
   读操作（体检、终检、状态查询）可直接跑。
4. **规则单一来源**：所有阈值/黑名单从 `scripts/kjx_rules.py` 取，不要在脚本里另写一份。
5. **标题逐条手写**，不用规则模板批量生成。

## 七环管道

| 环 | 做什么 | 关键脚本 |
|---|---|---|
| 0 采集 | 1688 列表全量 → 详情页一趟取「件重+运费+入库」 | `kjx_harvest_mtop.js` → `kjx_fetch_both.js` |
| 1 IP 预筛 | 主图拼长图 → 外部 VLM 判三栏 → 拦/放行清单 | `kjx_vlm_review.py` `kjx_vlm_retry.py` `kjx_final_list.py` |
| 2 认领 | 放行件 ∩ 已入库 ∩ 未认领 → 认领到 platformId=78 → offerId 映射成认领 id | `kjx_make_claim_list.py` `kjx_map_claims.py` |
| 3 标题 | 逐条手写三段式 → 自检 → 写回 + 回读校验 | `kjx_check_title.py` `kjx_set_title.py` |
| 4 七环写回 | 重量/库存888/结算价/描述过滤/详情图裁12/欢迎语 | `kjx_batch_run.py` |
| 5 终检 | 全量认领箱逐条核对 | `kjx_audit.py` |
| 6 发布 | 铺到企业 <MERCHANT_ID> 名下 7 店 | `kjx_publish.py` |

逐环命令、参数、产物文件见 `references/pipeline-commands.md`。

## Set the gates by token cost（越便宜越先跑）

一条波次的执行顺序按 token 成本从低到高设门，每层通过才进下一层：

1. **脚本体检**（零图像开销）：脚本在不在、token 在不在、CDP 在不在、recon 数据齐不齐 →
   `python scripts\kjx_status.py`（在项目根 `<PROJECT_ROOT>` 下跑）
2. **单点检查**：只跑 1 件走通全环，确认字段写对了
3. **每单元代表帧拼 contact sheet**：一张图代替十几张
4. **全量**

**先样张后全量**：长耗时批处理（大批量抓取、渲染、训练）先交一份低成本可审样张，
把决定权交回用户，不要一口气做完再汇报。静帧先行的额外收益是**把 bug 暴露在长任务之前**。

## IP 判定（最容易错的一环）

**判据：只有商品本体上印着明确知名 IP 角色名/品牌字样才是红线。**

- **拦**：Disney/米老鼠/艾莎、ZOOTOPIA 角色、库洛米/大耳狗/美乐蒂/布丁狗/轻松熊、
  美少女战士、小马宝莉、海绵宝宝、蜡笔小新、帕恰狗、Pingu、蛋仔派对 EGGY PARTY、
  元梦之星、奥特曼 ULTRAMAN、胡迪/三眼仔/叉飞侠/扎克大王、
  Scribbled Kitty/Puppy/Kapibara 仿款、BIG MAC
- **放行**：只有通用动物形象（熊猫/兔子/小熊/小狗/水豚/柯基/恐龙/机器人/笑脸花朵…），
  即使带英文短句和假厂牌（PANDA/SHOTAY/ST-YIBEN/TYESO/SITARAYURI/GUodday/VOGUE/CRUHRE/
  GIAVOS/Bichar/FAUNA/Baoluocc/BIGCAR）
- **例外**：DODGE（道奇）按杯子白名单放行（`CUP_WHITELIST`）
- **不要因为「看起来像蹭牌」就扩大化拦截**——用户已明确知情并选择保数量

**扫描面必须四处同扫**（`kjx_rules.vlm_verdict_final(vdb, offer_id, title)`）：
`1688 原标题 + VLM 角色栏 + VLM 英文词栏 + VLM 厂牌栏`。
只扫一边会漏——实测两边各漏 2 件：标题层有 IP 而 VLM 描述没有的（<OFFER_ID> 元梦之星、
<OFFER_ID> 蛋仔派对），以及厂牌栏有 IP 而标题没有的（<OFFER_ID> 帕恰狗）。
**厂牌名 ≠ 普通品牌**：Pochacco / Pingu / EGGY PARTY / ULTRAMAN / CRAYON SHINCHAN 都是 IP。
**匹配一律用完整词组**——用单词会误伤自有厂牌（加 `tom` 会命中 `CHICU TOM`，加 `kitty` 会命中通用拟人词）。

**VLM 全量预筛是刚需层，不是可选优化——人工读图也会漏。**
实证：第一波用人工读 contact sheet 复核 44 件，漏掉 1 件美少女战士硬 IP
（`<OFFER_ID>` / claim <CLAIM_ID> / base_item_id <BASE_ITEM_ID>），
标题层零风险词，**已上架**。是后来靠回溯筛查才抓出来的。
**改过 IP 规则后必须跑一遍 `kjx_ip_audit_claimed.py --all` 回头查已发布的批次。**

完整清单与判定实现见 `references/rules-and-ip.md`。

## Output contract（每轮交付给用户的东西）

1. **一句话结论**：这一波处理了多少件、拦了多少、实际发布多少、零失败还是几件失败
2. **数据表**：关键数字（件重齐不齐、运费齐不齐、终检几品几项、发布成功数、base_item_id 区间）
3. **异常清单**：只列需要用户定夺的件（IP 裁决、缺件重、失败件），不刷长表
4. **落盘文件路径**：报告 md、recon 数据文件
5. **需要用户定夺的开放问题**（如果有）

明细默认**不刷屏**——要明细时用脚本的 `--verbose` 显式索取。
脚本输出风格：默认摘要，异常才详报。

## Failure handling

| 症状 | 处理 |
|---|---|
| 1688 滑块/验证码整域拦截 | 立刻弹窗通知用户解锁（`kjx_block_alert.js` 已实现：置前标签页+红字横幅+桌面气泡+原地等 15 分钟自动续跑）。**不要自己绕过** |
| 件重/运费缺 | 补抓该件（`kjx_fetch_both.js` 支持断点续跑，状态在 `recon/b3_state.json`）。读回的「送至」必须含南宁才采信 |
| 运费显示「包邮」 | 运费记 **0**。旧逻辑只找含「运费」二字的项，找不到就误判 FAIL |
| 外部 VLM 调用挂死 | 必须加 `timeout` + 重试 + 连续失败标 `FAIL`。**静默跳过 = 漏检** |
| 写回后字段没变 | 认领后 ERP 还没同步完，前几件写回会静默失败。**必须回读校验**，失败件按 claim id 补跑 |
| 脚本「跑完了」但什么也没做 | 先查两件事：**分页**（任何「先取列表再按 id 过滤」必须翻页取全量）和**依赖的数据文件**（`recon/候选.json`、`shop_*_all.json`、`weights.json` 等是否覆盖新候选） |
| 探参数时误发商品 | 探针必须用**无效 id**（999999/0/负数）或 `page.route`+`abort`；`publishGoods` 无 dry-run，参数一补齐就立刻执行 |
| ERP 侧 `shopee_listed=0` / 状态「处理中」 | 已知原因：店铺尚未正式审核通过，审核通过后自动同步。**不是链路 bug，不要动它** |

坑的完整清单见 `references/traps.md`，接口清单见 `references/api-reference.md`。

## 写脚本的能力

**本 skill 不打包脚本**——管道脚本是项目资产，统一在
`<PROJECT_ROOT>/scripts/`（约 190 个文件，多数是一次性探针）。
管道新增能力时**在项目里写脚本**，写法见 `references/scripting.md`。

那份规约的要点（写新脚本时直接照做）：

- **写接口的入参从前端 chunk 挖，不猜**（`recon/erp2-api-*.js` 里解析导出映射）；
  探参数用**无效 id** 或 `page.route`+`abort`——`publishGoods` 没有 dry-run，参数一补齐就真发布
- **任何「先取列表再按 id 过滤」必须分页取全量**（用 `kjx_paging.fetch_all`），
  且取全量后仍没匹配到要**显式报错**
- **写操作必须回读校验**，不能只看 `code=1000`；失败件按 id 补跑，不要重跑整批
- **数据落 JSON，stdout 只放进度**，明细要 `--verbose` 才给——
  脚本输出的 token 是 poll 时读进上下文的，不 poll 就零成本
- **共享常量从 `kjx_rules.py` 取**，不在脚本里另写一份
- 手工补抓的补丁放**独立文件**（`freight_extra.json`），别写进会被后台重写的那个 JSON
- 改完规则**回扫全量确认零回归**；改完脚本**确认已启动进程是不是还在跑旧代码**
- 管道里现成的**只读报表**脚本：`kjx_export_published_xlsx.py`
  （导出已发布商品 Excel；重量要从认领侧回取，字段落点见 `references/api-reference.md`）

管道加了新脚本或新坑后，同步更新本 skill 的 references，并跑一次 lint 确认链接没断。
**skill 不带脚本 = 管道能力的唯一来源是项目 scripts/ + 这份规约，两处不同步就会按旧规则跑。**

## 标题写法

三段式：`第一段(材质/工艺/容量) 第二段(品类+功能) 第三段(人群/场景)`，长度 20–30 字。

- **不写任何厂牌词**（DODGE/SHITAY/ST-YIBEN/TYESO/SITARAYURI/BIGCAR/HEYCOOLBY/COISINI/CHICU TOM…）
- **不写动物形象描述**（留给详情图）
- 材质词保留（tritan / 316不锈钢 / 304不锈钢 / 陶瓷内胆 / 内覆陶瓷 / 纯钛 / 双仓 / 双盖 / 茶水分离）
- 写完跑 `kjx_check_title.py` 自检（IP 词/违禁词/B2B 词/长度/段数/重复），不通过就改到过

## 数字规则

- **重量**：`<1KG → 1.00`；`>1KG → 件重 + 500g`；`= 1.00 → 不动`。
  属性表里的「重量 0.3kg」是默认占位值，**永远不可信**。可靠来源是
  `window.context.result.data.productPackInfo.fields.unitWeight`（kg）×1000。
- **结算价**：`(采购价 + 运费 + 货代费 2.5) × 1.35`，逐 SKU 档算，顶层取最小。
  只写 `settlement_price`，**`price` 不动**。运费统一按广西南宁。
- **图片**：主图上限 9 张（**不足不补**，超了截前 9），详情图上限 12 张。
- **库存**：888。
- **发布折扣**：一律 0%。

## Windows (win32) platform notes

```powershell
Set-Location "<PROJECT_ROOT>"
$env:NO_PROXY="127.0.0.1,localhost"; $env:no_proxy="127.0.0.1,localhost"
```

- PowerShell 里 `python -c "..."` 内嵌三引号/双引号极易被解析坏 → **写补丁到文件再跑**
- 命令行传长 id 列表会转义坏 → 脚本用 `--ids-file` / `--claim-ids` 文件形式
- `Remove-Item` 被安全策略禁止 → 用 Python `os.remove`
- `mcode-tools` 是 .cmd 包装，Python `subprocess` 直接 exec 会 FileNotFoundError → 必须 `shell=True`
- 读文件用 `Get-Content -Encoding UTF8`（PS 5.1 默认 ANSI 会静默损坏中文）
- `node` 脚本驱动已登录 Chrome 走 CDP 9222，profile `C:/Users/cheng/.boss-chrome-profile`；
  9222 端口是**用户的 Chrome 不是 Edge**

**项目路径**默认 `<PROJECT_ROOT>`（写死在 `scripts/kjx_status.py` 顶部 `ROOT`）。
换机器或换路径时改这一处即可；管道脚本本身也都在 `ROOT\scripts\` 下。

## Examples

**Input**：「采集完了吗」
**Output**：查 `recon/b3_state.json` + `recon/weights.json` + `recon/freight.json` 计数，
答「350 件处理完，件重 350/350、运费 350/350 全齐，其中 3 件包邮记 0」——不刷长表。

**Input**：「第二波发布完了吗」
**Output**：跑 `scripts/kjx_status.py`，答「297/297 成功，base_item_id <BASE_ITEM_ID>–<BASE_ITEM_ID>，
累计 364 品；发布记录 373 条 = 364 成功 + 9 条历史 <CLAIM_ID> 残留，与本批无关」。

**Input**：「这批能上吗」
**Output**：先跑体检 → 再 VLM 预筛 → 产拦/放行清单 → **把需要用户裁决的件列给用户**，
不自行决定是否放行硬 IP。
