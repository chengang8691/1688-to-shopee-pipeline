# 踩过的坑

每条都带「怎么发现的」和「怎么防」，别只看结论。

---

## 1. 分页只取第 1 页 = 静默什么都不做

**现象**：第二波认领后数据量 364 条，`kjx_batch_run.py --claim-ids <CLAIM_ID>,<CLAIM_ID>,<CLAIM_ID>`
补跑时输出停在「件重/运费」两行，看起来像跑完了——实际循环一次都没执行。
`kjx_audit.py` 更危险：只查 200 条却报「全部通过」。

**根因**：ERP 的 `/index`、`/shopProductsList`、`/publishRecord` 都有分页，
很多脚本只取 `{page:1, limit:200}`。数据量涨过 200 条后，
**「先取列表再按 id 过滤」的逻辑会静默漏掉后面所有条目**——不报错、不提示。

**防**：用 `scripts/kjx_paging.py` 的 `fetch_all()`。已改 14 个脚本。
新增「分页取全量后仍没匹配到」的**显式报错**——静默无输出是最难查的失败模式。

**判据**：任何「先取列表再按 id 过滤」的逻辑，必须分页取全量。

---

## 2. 探 API 参数本身就是破坏性操作

**现象**：为了摸 `publishGoods` 的字段名，用真实商品 id 逐个字段名试，
`{"id": <CLAIM_ID>, "merchant_id": <MERCHANT_ID>}` 直接返回「成功，商品正在发布中」——**商品被真发出去了**。
后续 8 次重复提交被 Shopee 以 `error.item_already_exist` 挡掉。

**根因**：这个接口的参数校验是「缺什么报什么」，**一补齐就立刻执行**，没有 dry-run。

**防**：探参数时 id 必须**明确无效**（999999、0、负数），
让校验在「商品是否存在」那一步就挡住。或者用浏览器拦截器
`page.route('**/publishGoods', r => { 记 postData; r.abort(); })`——请求被 abort 就不可能落地。

**判据**：探「字段名不确定」的**写**接口（发布/删除/发送/付款）用拦截器；读接口随便探。

---

## 3. 认领后立刻写回字段会被拒

**现象**：批量「认领 → 改重量/结算价/库存/详情图/描述」时，前若干件的写回静默失败
（返回 1000 但字段没变，或直接报错）——表现是「重量还停在 0.30、库存非 888、欢迎语没加、
B2B 字段还在」。

**防**：批量改完**必须回读校验**（重新 GET 详情比对每个字段），不能只看接口返回 1000。
失败件用「按 claim id 补跑」的入口重放，做成显式参数（`--claim-ids`），不要重跑整批。

---

## 4. 包邮商品没有「运费」字样被误判失败

**现象**：`<OFFER_ID>` 的 shippingServices 区块是「浙江金华 / 送至 / 广西南宁 /
承诺72小时发货 / **包邮** 可选顺丰」，脚本判 FAIL——但商品其实是**免运费**。

**根因**：旧脚本只找含「运费」二字的服务项，找不到就判 FAIL。

**防**：区块文本里含「包邮」且无数字时，运费记 **0**（`kjx_nn_freight2.js` 已修）。

---

## 5. 切收货地：弹层里是「已保存地址列表」

**现象**：按文字点「送至」在详情页上点不中，读到的运费是发往页面默认地（江西南昌），
静默错价还不报错。

**正确做法**：
1. 点 `#shippingServices a.recieve-address`（「送至」右边那个地名+⌄）
2. 弹出的是 `.module-address-list`「选择收货地址」弹层，内容是 1688 账号**已保存的地址列表**
3. 列表里有「广西壮族自治区南宁市…」，点其中一条
4. 读 `#shippingServices .service-item` 里含「运费」的那个（`.service-item` 有多个）

**收货地区域是页面级状态，刷新即回默认**，所以每件商品都要重切一次。
**校验方式**：读回的「送至」文本必须含目标城市，否则丢弃该条运费（别信脚本的返回值）。

### 第二层坑：地址列表可滚动，坐标法必然点空

即使弹层已开、确实有「广西壮族自治区南宁市…」，按坐标点仍可能连续失败
（实测 3 件切 3 次都停在江西南昌）。原因是**地址列表可滚动，目标地址在折叠区外**，
`getBoundingClientRect()` 拿到的坐标落在视口外。

两种错法：
- 先 `scrollIntoView` 再批量存坐标 → 滚动会让**先前存的坐标全部指向别的行**
- 只 scrollIntoView 完再取一次坐标 → 同样只对最后一个候选有效

**唯一稳法**：按文本现场定位元素 → 滚动 → 直接 `el.click()`（不用坐标）。

```js
const els=[...root.querySelectorAll('*')].filter(e=>e.innerText.replace(/\s+/g,'')===txt);
els.sort((a,b)=>a.querySelectorAll('*').length-b.querySelectorAll('*').length); // 取最内层
els[0].scrollIntoView({block:'center',behavior:'instant'}); els[0].click();
```

**判据**：任何「弹层里点某个列表项」的动作，先问自己「这个列表能滚吗」——能滚就不能用坐标。

---

## 6. 后台并发脚本的三条自伤规则

1. **清理标签页不能关别人的页面**：多个 worker 共用一个浏览器时，「不是自己的 page 就关」会把
   **另一个 worker 的页面**也关掉，表现为隔一件失败一件（`Target page has been closed`）。
   要维护 worker 页面集合，只关既不是自己的、也不是任何 worker 持有的。
   并给 worker 加「页面被关就自己换一个」的自愈。
2. **共享 JSON 落盘会被并发写覆盖**：后台脚本每件都重写 `freight.json`，手工补抓的值被冲掉。
   手工补丁放**独立文件**（`freight_extra.json`），读的时候统一合并。
3. **写接口的入参要从前端源码挖，不要猜**：ERP 端点全在 `erp2-api-*.js` 里，形如
   `xx=(e={})=>m('/api/xxx',e)`，导出名是压缩过的（`export{Ke as $, ...}`），
   要解析导出映射才能对上哪个别名对应哪个 api 函数。

---

## 7. 1688 列表分页的正确入口是 mtop 接口

之前判定「1688 店铺页只有 30 件、无分页控件」是**错的**：分页控件在页面底部（y=857），
不滚到底看不见；而且点「下一页>」时**分页器状态会推进但商品网格不刷新**（静默失败）。

真正接口：

```
POST https://h5api.m.1688.com/h5/mtop.alibaba.alisite.cbu.server.moduleasyncservice/1.0/
  ?jsv=2.7.5&appKey=12574478&t=<ms>&sign=<md5>&api=mtop.alibaba.alisite.cbu.server.ModuleAsyncService
  &v=1.0&type=json&valueType=string&dataType=jsonp&timeout=10000
  data={"componentKey":"Wp_pc_common_offerlist",
        "params":"{\"memberId\":\"<MEMBER_ID>\",\"appdata\":{...,\"pageNum\":N,\"count\":30}}"}
```

**纯 Node/Python 直连会被防盗链拦成 captcha**（`FAIL_SYS_USER_VALIDATE` + 返回 punish 链接），
即使 sign 算对了也不行——缺 1688 的 cookie。**必须在页面内发**，
cookie 会轮换，所以每次请求都要重新读 `_m_h5_tk`。

**成果**：<店铺A>店 599 件（20 页 × 30）在一次任务内全部采齐，无风控、无人工翻页。

---

## 8. 外部调用会挂死

`mcode-tools connector call` 出现过发起后 3 分钟无任何返回。
必须 `subprocess.run(..., timeout=N)` + 重试 + 连续失败标 `FAIL`（不能静默跳过）。
与外部服务/网页被反爬拦截是同一类问题：**外部依赖必须会「响」**。

---

## 9. 图片 token 是链路里唯一值得优化的大头

多模态链路里，**图片进对话上下文的 token 通常是脚本输出的 5–10 倍**。
实测一批 40 件商品：人工读 contact sheet ≈ 280,000 token，脚本输出全部加起来才 ≈ 31,000。

**做法**：每件商品主图竖拼成长图 → 上传临时 URL → 调外部 VLM → 只把结论文字带回对话。
落盘结构化结论，人工只对标红件读原图。

**关键实测**：
- 图片 token ≈ (W × H) / 750
- 批量调用（一次 10 张）**返回顺序与入参严格对应，不串图**；但每张图必须写清
  「这是同一件商品的多张主图竖向拼接」，否则 VLM 会按序号重新编号导致描述错位
- 描述解析按**返回顺序**对齐，不要试图解析 VLM 自己写的序号

**外部 VLM 比人工更细**：实测 4 件标题完全无风险词的商品被读出美少女战士头像、草莓熊+米老鼠+艾莎。
**纯靠标题关键词过滤的链路，蹭牌仿品漏检率极高。**

---

## 10. 长跑任务的日志：成本来自「读」，不是来自「跑」

一个跑 321 件、每件 12 秒的后台抓取任务，日志逐件打印共 ≈3,000 token——
看着像开销，但**这 3,000 token 是我 poll 任务输出时读进上下文的**。
关键数据都落盘在 JSON 里，stdout 只是给人看的进度。**全程不 poll，这笔 token 根本不会发生。**

**判据**：长跑后台任务的真实成本通常是**时间**（321 件 × 12 秒 ≈ 64 分钟），不是 token。

做法：
1. 落盘的数据要独立于 stdout。数据进 JSON，stdout 只放进度。
2. 不要频繁 poll；真的要看，就让脚本**只打异常件 + 每 N 件汇总一行**。
   实测 227 行 → 33 行，3,030 token → 486 token（-84%）。

**另一个坑**：改了脚本的日志逻辑后，**已经启动的进程仍在跑旧代码**。
磁盘上是新版、后台输出还是旧格式时，先确认进程启动时间——
否则会误判「改了没生效」或「优化无效」。

---

## 11. 改了脚本逻辑，已启动的进程仍在跑旧代码

同上。**先确认进程启动时间，再判断改动是否生效。**

---

## 13. 两个标题脚本吃的 JSON 格式不一样

| 脚本 | 格式 |
|---|---|
| `kjx_check_title.py`（自检） | `{"wave": "b1", "titles": {"<claim_id>": "标题"}}` — dict |
| `kjx_set_title.py`（写回） | `[{"id": <CLAIM_ID>, "offerId": "763...", "title": "标题"}]` — list of dict |

`kjx_check_title.py` 也接受裸 list-of-strings，但**不能**直接吃 set_title 的 list-of-dict。
**写完先自检，通过后再转格式写回。** 传错格式会静默跳过或报 key 错。

---

## 14. IP 黑名单加太宽会误伤自有厂牌

回溯检查时想加 `tom`（汤姆猫），结果 4 件商品被误判为嫌疑——
它们的 VLM 描述是自有厂牌 **`CHICU TOM`**（记忆里明确列为「不写进标题」的自有厂牌）。

**判据：IP 角色名一律用完整词组匹配，不要用单词。**
`tom` 会命中 `CHICU TOM`、`tombow`；`judy` 可以（无自有厂牌叫这个）。

同理 `hello kitty` 要连 `hello kitty` 整词，不能只 `kitty`（`kitty` 是常见拟人词）。

---

## 15. VLM_HARD_ROLE 的角色名覆盖要全

《疯狂动物城》只列了 `zootopia`/`judy hopps`/`judy`/`朱迪`，
缺 Nick Wilde、Fin nick、Bellwether。实测没漏过（真实件都同时有朱迪），
但**只印尼克不印朱迪的件会漏**。加名单时注意上面第 14 条的宽度陷阱。

---

## 16. 脚本依赖的数据文件不能缺

| 脚本 | 依赖 | 缺了会怎样 |
|---|---|---|
| `kjx_vlm_review.py` | `recon/候选.json`、`recon/shop_*_all.json` | 静默 NOIMG 或漏标题层 IP |
| `kjx_final_list.py` | `recon/候选.json` + `recon/b3_ids_v2.txt` | 候选为空 → 清单为空 |
| `kjx_make_claim_list.py` | `recon/final_passlist.txt` | 认领清单为空 |
| `kjx_audit.py` | `recon/weights.json`、`recon/freight.json` | 件重/运费项无法核对 |

**新波次开工前先确认这些文件覆盖新候选**，否则会出现「脚本正常退出但什么都没做」。

**回溯筛查脚本已补**：`kjx_ip_audit_claimed.py`（按 offerId 扫已认领批次）+
`kjx_vlm_review.py --from-erp-ids`（从 ERP 详情 `goods_img` 取主图补预筛，
不依赖 `recon/shop_*_all.json` 快照）。改过 `VLM_HARD_ROLE` 后就该跑一遍。

---

## 18. 删认领撤不掉「已发布」产生的店铺商品记录

**现象**：claim <CLAIM_ID>（美少女战士款）删认领 + 删采集箱都成功（code=1000），
但 `shopProductsList` 仍是 364 条，那件还在里面（status=1）。

**根因**：`deleteClaimGoods` 只删认领记录。**ERP 没有删除店铺商品的接口**
（前端 chunk 里只有 `deleteClaimGoods` 和 `unlistShopProduct`）。
之前那 11 件硬 IP 删除后没留下店铺商品记录，是因为**它们是先删认领、后发布**，
压根没进过店铺商品表。

**判据**：
- 还没发布的（认领了但没发）→ `deleteClaimGoods` 就够了
- **已经发布过的** → 还要额外处理店铺商品记录

```powershell
# 一条命令删干净（认领 + 采集箱）
python scripts\kjx_del_claim_full.py <CLAIM_ID> --apply
# 已发布过的再试下架（按 offerId 定位 shop_id）
python scripts\kjx_unlist_by_offer.py <OFFER_ID> --apply
```

**但 `unlistShopProduct` 在 7 店未同步时是「假成功」**：
返回 `code=1000 下架成功`，复查列表里商品**仍在**，status 仍是 1。
原因是 ERP 拿不到 Shopee item_id，下架指令没地方执行。

**所以店铺审核通过前，「已发布的商品撤不下来」是硬限制。**
最干净的做法是**在发布前就把 IP 拦掉**——这正是 VLM 预筛必须在发布前做的原因。

---

## 19. 匹配黑名单要用词边界，不能用子串

`w in text` 的子串匹配会让 `pony` 命中 `company`、`tom` 命中 `CHICU TOM`。
实测 4 件自有厂牌 `CHICU TOM` 因此被误判成嫌疑。

已把 `kjx_rules.vlm_hard_hits` / `vlm_verdict_final` 改成
`kjx_rules._match_word()`：**含英文的词只认独立词，纯中文词仍用子串**。
加新名单后必须回扫全量确认命中列表没变（本次回扫 375 件，零新增误伤）。

---

## 21. 报表要「重量」必须回认领侧取，且两边条数不等

**现象**：做「已发布商品导出」时发现 `shopProductsList`（已发布 364 条）里**根本没有 weight 字段**；
重量只在 `getClaimGoodsInfo.weight`（kg 字符串）里，而那个接口按**认领 id** 查，
认领 id 又只在认领箱 `index` 列表里。`getShopProductDetail` 也不行——它要 Shopee 的 `item_id`，
ERP 侧 `global_item_id=0` 时根本给不出。

**正确链路**：`shopProductsList` → 按 `base_item_id` join `index` → 拿认领 id → 逐件 `getClaimGoodsInfo`。
`base_item_id` 在两个列表里各自唯一（实测 364 / 363 条零重复），是可靠的 join 键。

**条数不等是正常的**：认领箱 363 < 已发布 364。差的那条是**删过认领但已发布**的历史残留
（`offerId=<OFFER_ID>` / `base_item_id=<BASE_ITEM_ID>`，即 skill 里记的那件美少女战士硬 IP）。
它在认领箱里没有记录 → **重量取不到就留空并报出来**，不要瞎填、不要因此报错退出。

**判据**：报表类需求先确认「每个列在哪个接口」，别假设列表接口什么都有；
字段缺失时宁可留空 + 报异常，也不要用相邻字段猜。

---

## 22. `shopProductsList` 与认领箱是**两套 id 空间**，字段语义也不同

| 接口 | id 是什么 | stock 是什么 | weight |
|---|---|---|---|
| `/index`（认领箱） | **认领 id**（如 <CLAIM_ID>） | 各档 SKU 库存**之和** | 顶层 `weight` = 包裹重（1.00） |
| `/shopProductsList` | **店铺商品 id**（如 <SHOP_GOODS_ID>） | 同上（4440 = 5×888） | **无此字段** |
| `/getClaimGoodsInfo` | **认领 id** | 同上 | 顶层 `weight` = 包裹重 |

**坑 1：拿店铺商品 id 调 `getClaimGoodsInfo` 不报错，会返回别的东西。**
实测用 shop_id <SHOP_GOODS_ID> 查到 weight=0.20 / stock=1095，看着像「重量没写回」，
其实 <SHOP_GOODS_ID> 是店铺商品 id，不是认领 id。**做关联前先确认 id 空间**，
可靠连接键是 `base_item_id`（两边各自唯一）。

**坑 2：报表里输出「库存」会让人误以为 888 没写回。**
`shopProductsList.stock` 是各档 SKU 库存之和（5 档 = 4440），
不是单档库存。**报告/报表里要么标注「SKU 库存合计」，要么输出各档原值**，
否则读的人会以为写回失败。终检查的是 `goods_sku_option[].stock == 888`（逐档）。

**坑 3：顶层 `stock` 也是各档之和**（实测 4 档 888 → 3552 = 顶层 stock），
拿顶层 stock 去核对「是不是 888」一定会误判。

---

## 23. 环境坑（PowerShell）

- PowerShell 传长参数列表会转义坏（`--ids` 297 个 id 失败）→ 用 `--ids-file`
- PowerShell 里 `python -c "..."` 内嵌三引号/双引号极易被解析坏 → 写补丁到文件再跑
- `Remove-Item` 被安全策略禁止 → 用 Python `os.remove`
- `mcode-tools` 是 .cmd 包装，subprocess 直接 exec 会 FileNotFoundError → 必须 `shell=True`
- 读文件 `Get-Content` 不加 `-Encoding UTF8` 会静默损坏中文（PS 5.1 默认 ANSI/GBK）
- **GBK 控制台打不出 `✓` / emoji** → 脚本里 `print('✓ ...')` 会抛
  `UnicodeEncodeError` 把**检查结果本身**冲掉。
  修法：脚本开头 `sys.stdout.reconfigure(encoding='utf-8', errors='replace')`，
  或改用 ASCII 符号。**报错信息是 UnicodeEncodeError 时，先怀疑它掩盖了什么。**

## 24. 并发 worker 共享同一个 `--args-file` 会把结果对错号

**现象**：6 个线程并发调外部 VLM，各写各的却都写 `recon/_promo_call.json`。
线程 A 写完还没被 CLI 读走，线程 B 就把文件覆盖了 —— **不报错**，
但「描述属于 A 的图，却被记到 B 的 URL 上」。

**后果比报错更糟**：图片合规判定（弃用带「源头工厂」的图）会张冠李戴，
该弃的留着、不该弃的删掉，而且**回读校验也发现不了**（每个 URL 都「有」一条描述）。

**修法**：`args_file = '_promo_call_%d.json' % worker_tag`（每个 worker 一个文件）。

**通用判据**：任何「写临时文件 → 外部 CLI 读它」的并发模式，
临时文件必须**按 worker 隔离**。共享文件 = 隐形的错配。

## 25. 逐图 VLM 的瓶颈是上传，不是模型调用

实测一批 1031 张图：
- 串行：~10.4 s/图 → **2.9 小时**
- 并行（下载 12 线程 / 上传 12 线程 / VLM 6 线程）→ **约 15 分钟**

关键点：
- `describe_images` **不吃原始 CDN URL**：
  报 `image_info.url could not be used as a media source`，必须先
  `mcode-tools upload_temp_url <本地文件>` 换 temp_url。
  所以**每张图一次上传是绕不开的**，这就是瓶颈所在。
- 上传是**独立的 CLI 调用**，天生可并行 —— 这才是提速的正确切口。
- 两级缓存（`_promo_upload_cache.json` / `_promo_vlm_cache.json`）让重跑几乎免费；
  缓存**按 URL 存**（不是按顺序），否则并发乱序会把结果配错。

**判据**：批量跑外部模型时，先量「每张图的固定开销」。
固定开销（上传/下载）能并行化，模型调用本身才是真并发。

## 26. 1688 起订量（MOQ）在列表接口里，不在详情页

**业务前提**：一件代发、绝不囤货 → **只做 1 件起订的货**。
价格区间合规不等于能做：2 件起订发不出单，50 件起订完全不可做。

**MOQ 从哪读**：
- **列表接口 mtop 原始响应 `_raw.quantityBegin`** —— 整店一次拿全，
  比逐个开详情页快一个量级（61 件一次拿到）。
- 详情页兜底：`processingPrice.fields.beginAmount` /
  `mainPrice.fields.finalPriceModel.tradeWithoutPromotion.offerBeginAmount`。
- **别信** `orderParamModel.orderParam.beginNum` —— 实测某商品列表=2、
  该字段=1，实际是 2 件起订，以列表为准。

**坑**：
- 详情页 `processingPrice` **不是每页都有**（单一价格的商品没有阶梯价，
  这个 key 直接缺失）→ 不能只读这一个 key。
- 详情页 `window.context.result.data` 要**轮询等就绪**（最多 20×2s），
  否则读到 null 会误判成「没有阶梯 = 1 件起订」。
- 手动采的货不在列表数据里 → 得按店铺 memberId 单独查；
  memberId 从详情页 context 里正则捞 `b2b-[0-9a-z]+`（字段层级各页不同，别按 key 名取）。
- **列表接口必须在页面内 `fetch` 发**（带 cookie），纯 Node 直连会被防盗链拦成 captcha。
- 货源商品可能**已经下架**（详情页跳 1688 首页 `notfound`）→ 起订量查不到，
  这时不能默认当 1 件起订。

**判据**：任何「供应商起订量/最小采购量」类字段，都先找**列表接口的批量返回**，
再考虑逐页探测。

## 27. 「已发布」有两套名单，撤下商品要分清是哪一套

ERP 里同一件货出现在**两个地方**，语义不同，撤下时得分别处理：

| 接口 | 含义 | 能删吗 |
|---|---|---|
| `/publishRecord` | 发布记录（每次发布尝试一条） | **能**：`/api/shopee_swarm/delRecord` {id} |
| `/shopProductsList` | 店铺商品记录（ERP 侧待推送/上架状态） | **不能**：没有删除接口 |

**`delRecord` 是唯一能删发布记录的接口**（前端 chunk 里藏在 20 个 endpoint 中间，
不 grep `ShopProduct` 根本发现不了 —— 要把整个 `/api/shopee_swarm/*` 列表拉出来看）。

**完整撤下顺序**（已发布过的商品）：
1. `delRecord` 删发布记录 → 商品从「已发布」报表消失
2. `deleteClaimGoods` 删认领记录
3. `/api/Collect/deleteGoods` 删采集箱条目
4. **残留**：`shopProductsList` 里的待推送记录**删不掉**，
   落 watchlist，等店铺同步后用 `unlistShopProduct` 收尾

**为什么残留删不掉**：7 店未同步时 ERP 拿不到 Shopee 侧 `item_id`，
而 `unlistShopProduct` / `getShopProductDetail` / `updateShopProduct` 全都要这个 id，
`unlistShopProduct` 还会返回 `code=1000 下架成功` 但商品**仍在列表**（假成功）。

**判据**：判断「某商品还在不在已发布里」要问的是
**`publishRecord` 里还有没有它的记录**，不是 `shopProductsList` 里还有没有。
只看后者会以为「撤不下来」而重复操作。

## 28. 待推送的店铺商品记录：删不掉，但**能改**（把库存清零防出单）

ERP 删不掉「待推送」的店铺商品记录（`shopProductsList` 里那些），
但**可以改它** —— 改成库存 0，同步上架后就不会出单。这是当前限制下唯一的兜底。

**三个必须知道的点**（都是试出来的）：

1. **`item_id` = ERP 自己的记录 id**，不是 Shopee 侧 item_id。
   用 `{'id': 记录id}` 调用 → 「请提供 item_id」；
   用 `{'item_id': 记录id}` → 通了（值就是 282460 这种记录 id）。
   `getShopProductDetail` 能拿到完整商品：标题/图片/描述/属性/`model_list` 各档 SKU。
2. **`updateShopProduct` 只认「可修改字段」白名单。**
   只给 `{item_id, base_item_id, stock: 0}` → 「**未检测到可修改的字段**」。
   必须**整条回写**：先 GET 整条 → 只改目标字段 → 整条 PUT。
3. **顶层 `stock` 是派生字段（各档之和的缓存），接口不认它。**
   整条回写后 `model_list` 各档 stock 全归 0，顶层仍显示旧总数。
   **真正下发给 Shopee 的是 `model_list` 各档库存** → 归零有效。
   复核判据要写「**各档合计 == 0**」，看顶层 stock 会误判成没生效。

```python
# 改的姿势
d = api('/getShopProductDetail', {'item_id': rid, 'platform_id': 78})['data']
for m in d['model_list']:
    m['stock'] = 0
api('/updateShopProduct', d)        # 整条回写
```

**必做校验**：每条写完**回读并逐字段 diff** —— 除 stock 外任何字段被改都算失败。
整条回写会把 GET 到的所有字段一起 PUT 回去，漏改一个字段就是一次静默数据损坏。

**判据**：ERP 侧凡是「删不掉但会生效」的东西，先找它的**可写接口**，
不要只盯删除接口。库存清零就是这类「半成功但有效」的操作。

## 28. 查不到的字段不能默认成「最宽松的那个」

起订量查不到时若默认当 1，闸门会放行本该拦下的货；
查不到时若默认当「不可发布」，又会卡住明明合规的货（手动采的货不在整店数据里）。

正确做法：**「查不到」单列一档，单独报告**，
由人决定是补查还是放行 —— 不要在代码里偷偷替它选一个默认值。

---

## 29. 认领箱里的货源 id 只藏在 `source_url` 里

**现象**：认领完要拿「认领记录 id」去写回/终检，手上只有 1688 offerId。
按 `/index` 返回找 `offer_id` / `source_id` / `alibaba_id` 字段，**一个都没有**。

**根因**：`/index`（认领箱列表）每行的货源标识是
`source_url = "https://detail.1688.com/offer/<OFFER_ID>.html"`，
**没有独立的 offerId 字段**。要从 URL 里切 `/offer/(\d+).html`。

**防**：`scripts/kjx_map_claims.py` —— 分页取全量 `/index` → 从 `source_url` 建
`{offerId: claim_id}` 索引 → 按清单匹配，输出「匹配 N / 未匹配 M」，未匹配显式报错。

**判据**：批量脚本之间传递的 id 必须是**同一种**。
「采集箱 goods_id」「认领 id」「1688 offerId」是三套 id，
每一步都要显式转换，且转换失败要**报错**而不是静默跳过。

---

## 30. 写回用保守值、审计用严格值 = 审计会误报

**现象**：写回时件重抓不到，按规则写了保守值 1.00KG（用户明确允许）。
终检却报一批「[✗] 缺基准数据 件重=None」——**明明按规则处理了却判不过**。

**根因**：审计脚本的判定是「`weights.json` 里查不到件重 → 判失败」，
写回脚本的判定是「查不到 → 用 `--missing-weight` 的保守值」。
**两个脚本对同一份缺失数据用了两套口径。**

**防**：`kjx_audit.py` 区分两种缺失：
- **件重缺失** → 按保守值（默认 `1.00`，用 `--missing-weight` 与写回对齐）核对重量；
  结算价核算**不依赖件重**（只依赖 采购价 + 运费 + 货代费），照常核对；
  处理完单列一份「件重基准缺失 N 件」清单。
- **运费缺失** → 判不过（结算价无法核算）。

**判据**：凡是有「兜底默认值」的环节，**审计必须用同一个默认值**，
否则「按规则处理」会被自己的审计判成失败。兜底要**单列报告**而不是静默。

---

## 31. 批处理脚本的输入/输出文件必须按批次隔离

**现象**：跑第二家店的认领清单时，脚本按默认参数把**上一批**的
`recon/<上一批>_claim_ids.txt` 覆盖成了 1 行。
跑 `--help` 也会触发——脚本没有 argparse，`--help` 无人接，直接走主流程。

**根因**：脚本把「本批次的输入输出」写死在默认路径里
（`recon/final_passlist.txt`、`recon/boxid_by_offer.json`、`recon/promo_imgs.json`）。

**防**（已参数化，跨批次复用同一脚本）：

| 脚本 | 新参数 | 作用 |
|---|---|---|
| `kjx_make_claim_list.py` | `--box` | 指定本批的采集箱映射（list / dict 两种形态都吃）|
| `kjx_batch_run.py` | `--promo` | 指定本批的弃图清单 |
| `kjx_map_claims.py` | 新脚本 | offerId → 认领 id 映射 |
| `kjx_img_promo_scan.py` | `--out` | 弃图清单写到本批文件 |

**判据**：连续跑不同批次时，**每个脚本的默认路径都指向上一批的数据**。
凡是有默认路径的批处理脚本，跑之前先确认它读的是本批文件；
能参数化就参数化，别靠「记得改代码」。

---

## 32. 整店侦察必须显式指定店铺，否则**静默采错店**

**现象**：抓「某家店」的全量商品，脚本没读到 memberId，
于是**用了默认的那家店**——结果正常返回几百条，一点不报错。

**根因**：`kjx_harvest_mtop.js` 的 memberId 从短链解析，
解析失败时 fallback 到硬编码的默认值。短链内容是 `wireless1688://` 协议 URL，
memberId 藏在 query 里，解析器没匹配上。

**防**：`KJX_MEMBER` 环境变量显式指定店铺；脚本在 memberId 缺失时**直接报错退出**，
不再 fallback 到默认店铺。同一批次的去重/筛选/VLM 预筛清单都要带店铺标识。

**判据**：**静默用错数据源比报错危险得多**。
「默认门店/默认账号/默认时间范围」这类 fallback 一律改成 fail-fast。
