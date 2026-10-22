# 给这条管道写脚本

skill 本身**不打包脚本**——管道脚本是项目资产，统一在
`<PROJECT_ROOT>/scripts/`，由本文件规定怎么写。
新脚本一律落在项目 `scripts/` 下，命名 `kjx_<动作>_<对象>.py`（`.js` 用于浏览器驱动）。

## 0. 写之前先确认要不要写

```
有个动作要反复执行 →
├─ Mavis 原生 tool 已支持？          → 直接用 tool，别包脚本
├─ 涉及多 agent 协作/评审？          → 用后台 Task 子代理，不在脚本里编排
├─ 涉及跨 session 状态传递？          → 用 scratchpad 文件
├─ 是判断性/创造性工作？              → 在 procedure 里直接做，别脚本化
└─ 真的是确定性本地计算？            → 才写脚本
```

**项目 scripts/ 里有 190+ 个文件，但大部分是一次性探针**（`kjx_probe_*` / `kjx_dig_*` / `kjx_find_*`），
写新脚本前先确认没有能复用的活脚本，别重复造。

## 1. 写接口的入参从前端源码挖，不猜

ERP 端点全在 `erp2-api-*.js` 里（快照在 `recon/`），形如
`xx=(e={})=>m('/api/xxx',e)`。**导出名是压缩过的**（`export{Ke as $, ...}`），
要解析导出映射才能对上哪个别名对应哪个 api 函数。

```python
# 从 chunk 里提取全部 swarm 端点（比在压缩 JS 里 grep 可读）
import re, io, glob
s = set()
for f in glob.glob('recon/*.js') + glob.glob('recon/*/*.js'):
    t = io.open(f, encoding='utf-8', errors='replace').read()
    s.update(re.findall(r'/api/shopee_swarm/([A-Za-z0-9_]+)', t))
```

**写接口没有 dry-run**：`publishGoods` 参数一补齐就立刻执行。
探参数时 id 必须是**明确无效**的（999999 / 0 / 负数），
或用浏览器拦截器 `page.route('**/publishGoods', r => { 记 postData; r.abort(); })`。
**读接口随便探。**

## 2. 分页取全量（`scripts/kjx_paging.py` 的 `fetch_all`）

**任何「先取列表再按 id 过滤」的逻辑必须分页取全量。**
ERP 的 `/index`、`/shopProductsList`、`/publishRecord` 都有分页，
一次性取 `limit:200` 在数据涨过 200 条后会**静默漏数据**——不报错、不提示。

```python
from kjx_paging import fetch_all
rows = fetch_all(tok, '/index', {'platform_id': 78}, page_size=200)
```

新脚本直接 import 它，不要自己写翻页循环。
**新写的过滤逻辑若分页取全量后仍没匹配到，必须显式报错**——静默无输出是最难查的失败模式。

## 3. 写操作必须回读校验

```python
res = api('/editClaimGoods', tok, payload)
if res.get('code') != 1000: ...
# 关键：重新 GET 详情比对字段，不能只看 code
after = api('/getClaimGoodsInfo', tok, {'id': gid, 'platform_id': 78})
```

批量写完**逐件回读**。认领后 ERP 还没同步完，前若干件写回会静默失败
（返回 1000 但字段没变）。失败件按 id 单独补跑，做成显式参数（`--claim-ids`），
不要重跑整批。

## 4. 日志风格：默认摘要，异常才详报

**脚本输出的 token 是我 poll 日志时读进上下文的**——不 poll 就零成本。

- 数据落盘 JSON，**stdout 只放进度**
- 逐件明细要 `--verbose` 显式索取
- 长任务只打异常件 + 每 N 件汇总一行（实测 227 行 → 33 行，token -84%）
- 换行输出用 `sys.stdout.reconfigure(encoding='utf-8')`

## 5. 共享 JSON 的并发陷阱

后台脚本每件都重写同一个 JSON，手工补抓的值会被冲掉。
**手工补丁放独立文件**（如 `recon/freight_extra.json`），读的时候统一合并。

多个 worker 共用一个浏览器时，清理标签页**不能关别人的页面**——
只关既不是自己的、也不是任何 worker 持有的；并给 worker 加「页面被关就自己换一个」的自愈。

## 6. 文件与路径

```python
ROOT = r'<PROJECT_ROOT>'          # 项目根写死，换机器只改这一处
sys.path.insert(0, os.path.join(ROOT, 'scripts'))   # 让 import kjx_rules / kjx_paging 可用
tok = io.open(os.path.join(ROOT, '<TOKEN_FILE>'), encoding='utf-8').read().strip()
```

**token 不进上下文、不打印、不写进记忆。**
共享常量一律从 `kjx_rules.py` 取（`DROP_DESC_KEYS`/`VLM_HARD_ROLE`/`MARKUP`…），
不要在脚本里另写一份——历史上终检与写回规则不同步就是这么来的。

## 7. PowerShell 陷阱（win32）

| 坑 | 做法 |
|---|---|
| 传长 id 列表被转义坏 | 脚本支持 `--ids-file` / `--claim-ids` 文件形式 |
| `python -c "..."` 内嵌三引号/双引号被解析坏 | **写补丁到文件再跑**，不要内联 |
| `Remove-Item` 被安全策略禁止 | 用 Python `os.remove` |
| `mcode-tools` 是 .cmd 包装，subprocess 直接 exec 报 FileNotFoundError | 必须 `shell=True` |
| `Get-Content` 不加 `-Encoding UTF8` 静默损坏中文 | 一律加 `-Encoding UTF8` |
| 浏览器脚本 | 走 CDP 9222，**9222 是用户的 Chrome 不是 Edge** |

## 8. 浏览器驱动脚本（.js）

用 `playwright-core` 连已登录的 Chrome（CDP 9222），
profile `C:/Users/cheng/.boss-chrome-profile`，登录态保留。

- **不要另起浏览器实例**，用用户那个
- 定位元素优先**按文本 + `el.click()`**，坐标法在可滚动列表里必然点空
- 1688 滑块整域拦截时调 `kjx_block_alert.js` 通知用户解锁，不要自己绕过

## 9. 写完必做的两件事

1. **改规则/判定逻辑后**：回扫全量确认命中列表没变（零回归）。
   例：改 `VLM_HARD_ROLE` 后跑 `kjx_ip_audit_claimed.py --all`，
   命中列表应与改前完全一致，无新增误伤。
2. **改了脚本逻辑后**：确认已启动的进程是不是还在跑旧代码——
   看进程启动时间，否则会误判「改了没生效」。

## 10. 改完刷新 skill

管道加了新脚本或新坑后，**同步更新本 skill**：
- 新环/新命令 → `references/pipeline-commands.md`
- 新规则常量 → `references/rules-and-ip.md`
- 新踩的坑 → `references/traps.md`
- 新接口 → `references/api-reference.md`
- 改完跑一次 `node <skill-creator>/scripts/lint-skill.js <skill目录>` 确认 references 链接没断

skill 不带脚本，所以**管道能力的唯一来源是项目 scripts/ + 这份规约**。
两处不同步 = 下一批按旧规则跑。
