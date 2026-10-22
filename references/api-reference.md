# 接口与常量速查

## 域名

| 用途 | 域名 |
|---|---|
| ERP API | `https://<ERP_API_HOST>/api/*` |
| ERP 前端 | `https://<ERP_WEB_HOST>` |
| 平台 ID | 轻出海 = **78**（74 TikTok / 76 虾皮TW / 77 Noon / 78 轻出海 / 998 库存订单 / 999 头程集运） |

**合规红线**：只走上面两个域名。跨境X → Shopee 那一跳由平台用用户店铺授权完成，
**绝不直接用 API 连 Shopee 卖家中心**。

## 关键接口

| 用途 | 方法 | 路径 | body |
|---|---|---|---|
| 采集入库（详情页） | POST | `api/Collect/add` | 整页 HTML + head + `.kjx-json-get` JSON（**不需 1688 授权**） |
| 采集入库（列表页批量） | POST | `/api/collect/addAlibabaProducts` | `[{offerId,title,imageUrl,sourceUrl}]`，每批 50（**需授权，当前失效**） |
| 删采集箱 | POST | `/api/Collect/deleteGoods` | `{id}` |
| 认领 | POST | `/api/Claim/claimPlatform` | `{goodsId:[...], platformId:78}` |
| 删认领 | POST | `/api/shopee_swarm/deleteClaimGoods` | `{id}` |
| 七环写回 | POST | `/api/shopee_swarm/editClaimGoods` | 全量字段（只传 `{id,title}` 会覆盖图片/价格） |
| 认领箱列表 | POST | `/api/shopee_swarm/index` | `{platform_id:78, page, limit}` |
| 店铺商品列表 | POST | `/api/shopee_swarm/shopProductsList` | `{platform_id:78, page, limit}` |
| 详情 | POST | `/api/shopee_swarm/getClaimGoodsInfo` | `{id}` |
| 发布 | POST | `/api/shopee_swarm/publishGoods` | `{id:[认领id], merchant_id:[<MERCHANT_ID>], price_discount:0}` |
| 发布记录 | POST | `/api/shopee_swarm/publishRecord` | `{page, limit, platform_id}` |
| 企业列表 | POST | `/api/merchant/getMerchantList` | `{platform_id:78}` |
| 下架 | POST | `/api/shopee_swarm/unlistShopProduct` | `{id, unlist:1}`（**7 店未同步时假成功**） |

**`shopProductsList` 的 `base_item_sku` = 1688 offerId**，
可用它把认领 id ↔ 店铺商品 id 精确对上。

## 字段落点（报表类需求必读）

| 字段 | 在哪个接口 |
|---|---|
| offerId | `shopProductsList.base_item_sku`（=1688 offerId）；缺失时从 `index.source_url` 里 `/offer/(\d+).html` 解析 |
| 标题 | `shopProductsList.base_item_name`（`index.title` 可回退） |
| 结算价 | `shopProductsList.settlement_price` |
| 库存 | `shopProductsList.stock` |
| **重量** | **只在 `getClaimGoodsInfo.weight`**——`shopProductsList` **没有** weight 字段 |
| Shopee 状态 | `shopProductsList.status_text` / `status_code` / `shopee_listed` |
| base_item_id | `shopProductsList.base_item_id`（与 `index.base_item_id` 一一对应，两边各自唯一） |

**要重量就得回认领侧**：`getClaimGoodsInfo` 按**认领 id** 查，认领 id 只在 `index`（认领箱）里。
所以链路是 `shopProductsList` → 按 `base_item_id` join `index` → 拿认领 id → 逐件 `getClaimGoodsInfo` 取重量。
（`getShopProductDetail` 取不到——它要的是 Shopee 的 `item_id`，ERP 侧 `global_item_id=0` 时没有。）

## 写接口入参从前端源码挖

ERP 端点全在 `erp2-api-*.js` 里，形如 `xx=(e={})=>m('/api/xxx',e)`，
导出名是压缩过的（`export{Ke as $, ...}`）。
**要解析导出映射才能对上哪个别名对应哪个 api 函数，不要猜参数名。**

## 标识常量

| 名 | 值 |
|---|---|
| 企业 id | **<MERCHANT_ID>**（<你的公司主体>） |
| 7 个 <你的店铺名> 店 | <轻出海店铺 id 列表> |
| 1688 账号 | `<你的1688账号>`，unb `<你的unb号>` |
| <店铺A> memberId | `<MEMBER_ID_1>` |
| <店铺B> memberId | `<MEMBER_ID_2>` |
| 货代仓 | <你的货代仓所在区> |

**一次发布到「企业」= 同时铺到该企业名下的全部 7 个店**，不需要逐店再发。

## 认证

- token 在 `<PROJECT_ROOT>\<TOKEN_FILE>`，请求头 `token: <值>`
- 成功码 `code=1000`
- token 失效时返回别的 code，脚本会直接抛异常——先确认 `<TOKEN_FILE>` 存在且非空

## 已发布历史（截至 2026-10-20）

| 波次 | 件数 | base_item_id |
|---|---|---|
| 第一批 | 11 | <CLAIM_ID> + <CLAIM_ID>–<CLAIM_ID> |
| 批次2 | 15 | 认领 <CLAIM_ID>–<CLAIM_ID> |
| 批次3 第一波 | 41 | <BASE_ITEM_ID>–<BASE_ITEM_ID>（认领 <CLAIM_ID>–<CLAIM_ID>） |
| 批次3 第二波 | 297 | <BASE_ITEM_ID>–<BASE_ITEM_ID>（认领 <CLAIM_ID>–<CLAIM_ID>） |
| **合计** | **364** | |

发布记录 373 条 = 364 成功 + **9 条 `error.item_already_exist`（全是 <CLAIM_ID> 探针误发，历史残留）**。

## 已知状态说明

**ERP 侧 7 个店显示「未同步」（`shopee_listed=0`、`status_text=处理中`）是正常状态**——
原因是**店铺尚未正式审核通过**，审核通过后会自动同步。
这不是链路 bug，不需要修；在审核通过前，ERP 侧的下架/改价也做不了。
