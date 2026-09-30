# 听证会逐字稿网页校订台

供听证会**记录员**与**复核员**（及**参与者**）使用的网页校订台：导入带段号的 UTF-8 逐字稿后，
记录员可在段内选定字符范围提交“替换文字 + 依据”的修订提议；复核员可采纳、驳回提议并发布。
发布形成不可变版本；因发布而过期的提议可由记录员**续提**（按逐次发布的修订把原选区换算到当前正文后重提）；
参与者可对原文范围留下异议引用，修订使被引文字消失时保留原文快照并提示重新定位。

技术栈：Node.js（Express）+ SQLite（better-sqlite3）+ 无构建浏览器原生前端。

---

## 一、快速启动（Docker Compose 一键启动）

前置要求：已安装 Docker（含 Compose v2）。

```bash
# （可选）复制环境变量文件并按需修改端口、种子账户密码
cp .env.example .env

# 构建并后台启动
docker compose up -d --build

# 查看日志
docker compose logs -f
```

启动后浏览器访问：

> **http://localhost:3000**  （如修改了 `HOST_PORT`，则为 `http://localhost:<HOST_PORT>`）

停止与清理：

```bash
docker compose down          # 停止（保留数据卷 hearing-revision-data）
docker compose down -v       # 停止并删除数据卷（将清空全部数据）
```

SQLite 数据库保存在命名卷 `hearing-revision-data`（容器内 `/data/hearing.sqlite`）中。
如需用宿主机目录，可把 compose 中的卷改为 `./data:/data`。

### 演示账户（首次启动自动初始化）

| 角色 | 用户名 | 密码 | 可执行操作 |
| --- | --- | --- | --- |
| 记录员 | `recorder` | `recorder123` | 导入逐字稿、提交修订提议 |
| 复核员 | `reviewer` | `reviewer123` | 采纳/驳回提议、发布新版本、重新定位异议 |
| 参与者 | `participant` | `participant123` | 对原文范围提出异议、重新定位自己的异议 |

种子账户仅在数据库用户表为空时创建；可通过环境变量覆盖用户名与密码（见下文配置）。

---

## 二、本地直接运行（不用 Docker）

前置要求：Node.js ≥ 18（原生自带全局 `fetch`；开发环境为 Node 20）。better-sqlite3 为原生模块，
从 npm 安装时通常自动下载预编译二进制；若需源码编译，请准备 Python 3、make、g++。

```bash
npm install
npm start                 # 等价于 node src/server.js
```

然后访问 **http://localhost:3000**。

仅初始化数据库（建表 + 种子账户）不启动服务：

```bash
npm run init-db
```

运行测试（纯逻辑单元测试 + 启动临时服务的端到端测试）：

```bash
npm test
```

---

## 三、配置（环境变量）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `3000` | HTTP 监听端口（容器内）。宿主机端口用 compose 的 `HOST_PORT` 映射 |
| `HOST` | `0.0.0.0` | 监听地址 |
| `DATA_DIR` | `./data`（本地）/ `/data`（容器） | SQLite 文件目录 |
| `DB_PATH` | `$DATA_DIR/hearing.sqlite` | 数据库文件完整路径 |
| `SESSION_TTL_MS` | `43200000` | 登录会话有效期（毫秒，默认 12 小时） |
| `SEED_RECORDIST_USER` / `SEED_RECORDIST_PASS` | `recorder` / `recorder123` | 记录员种子账户 |
| `SEED_REVIEWER_USER` / `SEED_REVIEWER_PASS` | `reviewer` / `reviewer123` | 复核员种子账户 |
| `SEED_PARTICIPANT_USER` / `SEED_PARTICIPANT_PASS` | `participant` / `participant123` | 参与者种子账户 |

> 安全提示：演示使用 HTTP 与种子密码。生产部署请置于 HTTPS 反向代理之后，并修改全部种子密码；
> 种子账户仅在首次初始化（用户表为空）时生效。

---

## 四、使用流程

1. **记录员登录 → 导入逐字稿**
   - 粘贴 UTF-8 文本或选择 `.txt` 文件。行首段号支持 `12. 正文`、`12、正文`、`12 正文`；
     无段号的非空行自动顺延编号；段号重复会报错拒绝导入。
   - 导入即发布**不可变的第 1 版**。全场只允许导入一次，重复导入返回 409 且不改变状态。
2. **记录员在正文中拖选段内文字 → “提交修订提议”**，填写替换文字（空串表示删除，禁止换行跨段）与必填依据。
   - 提议**绑定提交时的已发布版本**（基版），并保存基版原文摘录快照。
3. **复核员**在右侧“提议”面板或正文标记上：
   - **采纳**：进入发布暂存区；**驳回**：可附原因，已采纳的也可驳回移出暂存区。
   - 采纳时若与暂存区已采纳修订**范围交叠**，或提议**基版已过期**，该提议被标记为“冲突”并写明原因，**不覆盖任何其他修订**。
4. **复核员发布**：同一基版上互不重叠的采纳结果按段合并，形成**不可变新版本**；
   暂存区为空时发布会被拒绝（防止重复发布）。发布后所有仍待复核的提议一律标记为“基版已过期”冲突，
   需由记录员基于新版本重新提交（可手工重提，也可用下述“续提”）。可随时切换“发布预览”查看合并效果。
   - **发布前须逐条回应被暂存修订触及的异议**（见第 8 节）：触及按“段内 Unicode 码点半开区间与异议
     当前锚点相交”判定，**插入点落在引用范围内也算触及**（边界相接不算）。复核员须为每条受影响异议
     填写**非空回应说明**（回应自动关联异议、相关采纳提议及当时的版本与暂存状态）；缺失或回应已失效时
     发布被拒绝，版本、提议及决议均不改变。没有受影响异议时仍按原规则发布。
5. **续提（基版已过期提议的换算重提）**：记录员可对**本人**因发布被标为“基版已过期”的冲突提议，
   在提议卡片或详情弹窗中点击“续提到当前版本…”发起**续提预览**；非本人或非此类冲突（如范围交叠）的提议会被拒绝。
   - 系统按从原基版到当前版**逐次发布的修订**，把原段内码点半开选区换算到当前正文，
     显示目标选区、当前摘录、原替换文字与依据（纯几何换算，**不凭同文搜索猜测位置**）。
   - 仅当该选区**未被中途修订触及**（零长度选区则其插入点未被任何修订区间触及），
     且换算目标**不与当前已采纳（暂存区）提议交叠**时，确认后生成**关联原提议的新待复核提议**
     （沿用原替换文字与依据，绑定当前版本为新基版）。
   - 中途修订触及选区/插入点、目标段缺失、或预览之后版本/暂存区发生变化时，确认被拒绝并提示
     **重新预览**或在当前正文**手工选区提交**；原冲突提议、决议与历史版本均保持可追溯，
     新提议沿用现有复核与发布规则。
6. **异议引用**：任何登录用户都可拖选原文范围提出异议，系统保存被引原文快照。
   - 之后发布的修订若与引用范围相交，引用不会自动改指新文字：该异议在新版本中显示
     **“待重新定位”**警示，原文快照继续保留；可由提出者或复核员在当前版本上手动重新定位并留痕。
7. **版本查看**：“版本”面板可只读查看任一历史版本的**正文、决议（采纳/驳回/冲突/续提/发布留痕）与异议**；
   历史版本内容不随后续发布改变。
8. **发布前异议回应（本轮新增）**：复核员发布暂存修订前，系统按异议**当前锚点**逐条核对受影响集合。
   - **触及判定**：暂存修订的段内码点半开区间 `[start,end)` 与异议当前锚点 `[aStart,aEnd)` 同段且
     `start < aEnd && aStart < end` 即为触及；零长度修订（纯插入）的**插入点严格落在引用范围内也算触及**，
     `[0,2)` 与 `[2,4)` 这种边界相接不算。待重新定位的异议也按其当前锚点参与判定。
   - **填写回应**（`POST /api/objections/:id/respond`，仅复核员）：`note` 必填非空。回应自动关联该异议、
     本次触及它的**相关采纳提议**集合，并快照回应时的**版本**与**暂存状态指纹**、异议锚点；同一发布周期内
     可反复覆盖更新。
   - **发布时重新核对**：采纳/驳回提议（相关提议集合或暂存状态变化）、重新定位异议（锚点变化）或有新异议
     被触及后，发布事务内会重新计算受影响集合并逐条校验回应绑定：缺失回应（`MISSING_RESPONSE`）、
     回应后暂存区变化（`STAGE_CHANGED`）、相关提议集合变化（`PROPOSALS_CHANGED`）、异议被重新定位
     （`OBJECTION_RELOCATED`）、说明为空（`EMPTY_NOTE`）一律**拒绝发布**（409
     `OBJECTION_RESPONSES_REQUIRED`，响应体 `details` 逐条列出失效原因），事务回滚——
     版本、提议及决议均不改变，已采纳修订仍留在暂存区，重新核对并回应后即可再发。
   - 异议已不再被任何暂存修订触及（如唯一相关修订被驳回）时，原草稿成为**失效孤儿**：网页与
     `GET /api/publish-checks` 会展示其失效原因，它**不阻止发布**；成功发布后孤儿草稿自动丢弃，不写入历史。
   - **没有受影响异议时仍可按原规则发布**（含暂存区为空时的重复发布防护不变）。
   - **成功发布后**：每条受影响异议的有效回应随新版本**冻结只读**，在对应历史版本
     （`GET /api/versions/:id` 的 `responses` 与版本查看弹窗“发布前异议回应”区）展示；
     异议**原文快照保留**、既有“相交即待重新定位”的规则不变。

网页中：发布条会列出待回应项与失效原因（可点击跳到异议卡片），未齐备时“发布新版本”按钮禁用；
“异议”页签对被触及异议显示“发布前待回应 / 已回应·待发布 / 回应已失效”状态与回应入口（仅复核员），
页签角标计入待处理数；历史版本查看器新增“发布前异议回应”只读区。

所有失败场景（无权操作、非法字符范围、重复发布、重复导入、重复采纳/重定位等）均返回明确的
HTTP 状态码与中文原因，且在数据库事务中执行——失败即回滚，不改变任何状态。

### 字符范围口径

范围按 **Unicode 码点**计偏移、半开区间 `[start, end)`（前后端一致，emoji 等增补平面字符按 1 个字符计），
仅允许在同一段落内选择；`[0,2)` 与 `[2,4)` 视为不重叠，可无缝相接。

---

## 五、HTTP API 摘要

会话采用 HttpOnly Cookie（`POST /api/login` 返回 `session`）。除登录、静态资源外均需登录。

| 方法与路径 | 角色 | 说明 |
| --- | --- | --- |
| `POST /api/login` / `POST /api/logout` | 全部 | 登录 / 退出 |
| `GET /api/me` | 全部 | 当前用户 |
| `GET /api/booth` | 登录用户 | 当前版本正文、暂存区、提议、异议、版本列表 |
| `POST /api/doc/import` | 记录员 | 导入逐字稿（仅一次），形成 v1 |
| `POST /api/proposals` | 记录员 | 提交提议 `{seg,start,end,replacement,basis}` |
| `POST /api/proposals/:id/adopt` | 复核员 | 采纳；交叠/过期则返回冲突标记与原因 |
| `POST /api/proposals/:id/reject` | 复核员 | 驳回（可含 `reason`） |
| `POST /api/proposals/:id/rebase-preview` | 记录员（本人） | 续提预览：把“基版已过期”冲突提议的选区逐版换算到当前正文，返回目标选区/摘录/原替换文字与依据及确认凭证 |
| `POST /api/proposals/:id/rebase-confirm` | 记录员（本人） | 确认续提 `{token}`：校验凭证（版本与暂存区未变、选区未被触及、不与暂存区交叠）后生成关联原提议的新待复核提议 |
| `POST /api/publish` | 复核员 | 合并暂存修订并发布新版本 |
| `POST /api/objections` | 登录用户 | 对原文范围提出异议 `{seg,start,end,note}` |
| `POST /api/objections/:id/relocate` | 提出者或复核员 | 对“待重新定位”的异议手动重定位 |
| `POST /api/objections/:id/respond` | 复核员 | 发布前对被暂存修订触及的异议填写/更新非空回应 `{note}`；未被触及返回 409 |
| `GET /api/publish-checks` | 复核员 | 发布前核对（只读）：受影响异议、待回应项、逐条失效原因（`blockers`）、失效孤儿草稿与 `canPublish`。`GET /api/booth` 的 `publishChecks` 字段返回同一结构 |
| `POST /api/publish` | 复核员 | 合并暂存修订并发布；存在缺失/失效异议回应时 409 `OBJECTION_RESPONSES_REQUIRED`（`details` 逐条说明），事务回滚 |
| `GET /api/versions/:id` | 登录用户 | 只读查看不可变版本（正文/决议/异议/`responses` 发布前异议回应） |

---

## 六、数据模型（SQLite）

- `versions`：不可变版本，`segments_json` 为该版全文快照，`decision_watermark` 界定该版可见的决议。
- `proposals`：修订提议；记录段号、码点范围、替换文字、依据、**基版 `base_version_id`**、
  基版原文摘录、状态机（pending → adopted/rejected/conflicted → published）与冲突原因。
  `conflict_kind` 区分冲突类别（`base_expired` 基版已过期 / `overlap` 范围交叠，旧库打开时自动补列并按原因文本回填）；
  `rebased_from_id` 记录续提来源——由哪个“基版已过期”冲突提议换算重提而来，双向可追溯。
- `decisions`：决议审计流（adopt / reject / conflict / rebase / publish），不可变追加。
- `objections` + `objection_anchors`：异议主表（含永久原文快照）与逐版本锚点；
  发布时按修订映射，相交即置 `needs_relocation`。`objection_logs` 记录创建与重定位操作。
- `objection_responses`：发布前异议回应。`status='pending'` 为发布周期内草稿（每异议每版本至多一条，
  部分唯一索引），保存非空说明 `note`、相关采纳提议快照 `proposal_ids_json`、回应时版本
  `base_version_id`、暂存状态指纹 `stage_fingerprint` 与锚点快照；发布重新核对通过后随新版本置
  `frozen` 并记录 `published_version_id`，在该历史版本只读展示；已不触及异议的孤儿草稿在成功发布时删除。

---

## 六之二、发布前异议回应：HTTP 调用示例

```bash
# 1) 复核员登录（Cookie 自动保存在 cookies.txt）
curl -c cookies.txt -X POST http://localhost:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"reviewer123"}'

# 2) 发布前核对：查看受影响异议、待回应项与失效原因
curl -b cookies.txt http://localhost:3000/api/publish-checks
# 未齐备时示例：
# { "versionId":3, "affectedCount":1, "canPublish":false,
#   "blockers":[ {"objectionId":1,"responseId":null,
#                 "reasonCode":"MISSING_RESPONSE",
#                 "reason":"该异议被本次暂存修订触及，尚未填写回应，发布被阻止"} ],
#   "items":[ ... ], "orphanResponses":[] }
# reasonCode 取值：MISSING_RESPONSE / STAGE_CHANGED / PROPOSALS_CHANGED /
#                  OBJECTION_RELOCATED / EMPTY_NOTE / NOT_TOUCHED_ANYMORE

# 3) 对被触及的异议逐条填写非空回应（也可在 /api/booth 的 publishChecks.items 中取 objectionId）
curl -b cookies.txt -X POST http://localhost:3000/api/objections/1/respond \
  -H 'Content-Type: application/json' \
  -d '{"note":"经复核录音00:34:10，修订与原声一致；异议原文快照留存，继续跟踪。"}'
# 201 新建 / 200 覆盖更新 {"ok":true,"id":1,"updated":false}

# 4) 回应缺失或失效时发布会被拒绝（事务回滚，版本/提议/决议不变；details 逐条说明）
curl -b cookies.txt -X POST http://localhost:3000/api/publish \
  -H 'Content-Type: application/json' -d '{"note":"第二次修订"}'
# HTTP/1.1 409
# {"error":"发布被拒绝：有 1 项……缺少有效回应……",
#  "code":"OBJECTION_RESPONSES_REQUIRED",
#  "details":[{"objectionId":1,"responseId":null,"reasonCode":"MISSING_RESPONSE","reason":"……"}]}

# 5) 全部受影响异议均有有效回应后发布成功；历史版本只读查看冻结回应
curl -b cookies.txt -X POST http://localhost:3000/api/publish \
  -H 'Content-Type: application/json' -d '{"note":"第二次修订"}'
# {"ok":true,"versionId":3}
curl -b cookies.txt http://localhost:3000/api/versions/3
# responses: [{ id, objectionId, proposalIds:[7], baseVersionId:2, seg, start, end,
#              note, responder_name, created_at }]
```

---

## 七、目录结构

```
.
├── Dockerfile               # 多阶段构建（原生模块编译 + 精简运行镜像）
├── docker-compose.yml       # 一键启动，含数据卷与健康检查
├── .env.example             # 配置样例（端口、会话、种子账户）
├── package.json
├── sample-transcript.txt    # 示例逐字稿
├── src
│   ├── server.js            # Express 路由与错误出口
│   ├── auth.js              # Cookie 会话与角色鉴权
│   ├── db.js                # SQLite 连接、建表、种子账户
│   ├── store.js             # 业务核心：导入/提议/复核/发布/续提/异议/版本视图
│   └── text.js              # 码点偏移、范围校验、合版、引用映射、续提换算
├── public                   # 浏览器前端（原生 HTML/CSS/JS，无构建步骤）
│   ├── index.html
│   ├── styles.css
│   └── app.js
└── test
    ├── unit.test.js         # 文本算子与解析的单元测试
    └── e2e.test.js          # 临时库 + 真实 HTTP 的端到端测试
```
