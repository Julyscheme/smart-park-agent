# 工单处理工作流 v2 — Dify 配置与实施指南

> **目标**：在 Dify 平台上搭建并发布「智慧园区工单处理工作流 v2」——12 节点工作流，完成报修受理、字段校验、分类、安全检测、派单与标准工单输出。
> **版本**：v2.1 | 2026-09-06 | DSL v0.7.0 | 全部口径经本地 Dify 实机实测（发布版本 2026-09-06 12:05:47）
> **v1 说明**：本指南 v1 版描述的是四场景 ChatBot 设计稿，已由 v2 工单工作流取代；v1 内容仅作演进参考。

---

## 1. 架构总览（12 节点）

| # | 节点 ID | 类型 | 职责 | 关键配置 |
|---|---------|------|------|----------|
| 1 | start | 开始 | 接收报修描述 | user_description（paragraph，必填） |
| 2 | kb_query | 知识检索 | RAG 检索设备手册/应急预案 | top_k=3，**相似度阈值 0.45**（调优见 §10） |
| 3 | extract_info | LLM | 信息提取 | qwen-plus-latest，temp=0.1，结构化输出 4 字段（location/device_type/fault_description/contact） |
| 4 | check_fields | 条件分支 | 必要字段校验 | IF：location 或 fault_description 为空（`empty`）→ 追问 |
| 5 | ask_missing | 代码 | 生成追问话术 | 固定话术，一次性问全缺失字段 |
| 6 | end_ask | 结束 | 输出追问 | follow_up |
| 7 | classify | LLM | 工单分类 | 8 类设备（空调/暖气/照明/电梯/门禁/监控/消防/其他），temp=0.1 |
| 8 | safety_check | 代码 | 安全关键词检测 | 燃气/电路/火灾/电梯困人等 → is_safety=true |
| 9 | assign_dept | LLM | 责任人指派 | 8 类→6 部门路由表 + 故障特征判定规则（"其他"类按描述归电力/安保应急组） |
| 10 | gen_ticket | LLM | 工单内容生成 | 结构化 JSON（语义字段），temp=0.1 |
| 11 | **ticket_finalize** | 代码 | **工单号/时间戳固化** | UUID + 真实时间戳，覆盖写入（见 §9） |
| 12 | end_ticket | 结束 | 输出工单 JSON | 输出选择器 → ticket_finalize.ticket |

```
start → kb_query → extract_info → check_fields ─┬─ 缺字段 → ask_missing → end_ask
                                                └─ 完整 → classify → safety_check → assign_dept
                                                        → gen_ticket → ticket_finalize → end_ticket
```

**设计原则**：LLM 只做语义理解与生成；确定性环节（字段校验、安全规则、工单号、时间戳）全部用分支与代码节点保证。

---

## 2. 前置准备

| 项 | 要求 |
|---|---|
| Dify 实例 | Docker 自部署或 cloud.dify.ai（本指南实测于本地 Docker，DSL v0.7.0） |
| LLM | 通义 qwen-plus-latest（阿里云百炼 DashScope，新用户有免费额度） |
| Embedding | text-embedding-v3（与 LLM 同一百炼 Key；⚠️ DeepSeek 无 embedding 模型，不可单独用） |
| 知识库文档 | `03-knowledge-base/` 下 3 份：equipment-manual.md / emergency-plan.md / park-management-rules.md |
| 测试数据 | `04-mock-data/`（可选） |

---

## 3. 第一步：配置模型供应商

1. Dify 右上头像 → 设置 → 模型供应商 → 通义千问 → 填入百炼 API Key
2. 确认 LLM（qwen-plus-latest）与 Embedding（text-embedding-v3）均可用
3. 自测：任一对话框发一句话，确认模型连通；Embedding 在创建知识库时验证

**失败即停**：模型未连通前不要继续后续步骤。

---

## 4. 第二步：创建知识库

1. 知识库 → 创建 → 名称「智谷产业园知识库」
2. 上传 `03-knowledge-base/` 3 份文档
3. 分段设置：**分段 500 tokens / 重叠 50**；索引方式：**高质量**（向量+关键词混合）
4. 等待 3 份文档全部 `completed`
5. 自测：命中测试输入「电梯困人」，应高相似度命中（实测 0.803）

### ⚠️ 相似度阈值调优（实测案例，必读）

工作流内检索阈值默认 0.7 会导致短查询 0 命中：

| 步骤 | 数据 |
|---|---|
| 现象 | 「暖气不热怎么办？」工作流内 KB 节点 0 命中（阈值 0.7） |
| hit-testing 实测 | 该查询最高相似度仅 **0.494** |
| 结论 | 阈值设 0.5/0.6 仍会漏（0.5 > 0.494） |
| **修正** | **阈值调至 0.45** → 工作流内命中 0 → 1 |

> 方法论：先 hit-testing 实测相似度分布，再定阈值，最后回工作流验证命中数——不要照抄默认值。

---

## 5. 第三步：导入 DSL 并绑定

1. 工作流应用 → 导入 DSL → 选择 `09-smart-park-agent-v2.dify.yml`
2. **重新绑定知识库**：打开「知识库查询」节点，`dataset_ids` 跨实例不通用，必须重新选择你的知识库
3. **核对模型**：4 个 LLM 节点（信息提取/工单分类/责任人指派/工单生成）全部设为 qwen-plus-latest，temperature 0.1
4. 核对画布：12 节点 / 11 条连线；「必要字段判断」仅有 1 个 IF 分支（无空 ELIF）

**已知版本兼容坑**（导入报错时对照）：
- 节点 ID 禁用连字符（`extract-info` 类 ID 的 `{{#...#}}` 引用在 2026 版 graphon 模板解析失败）——本 DSL 已全部使用下划线
- if-else 出边 `sourceHandle` 必须等于 `case_id`；结构化输出引用用三段式 `{{#node.structured_output.field#}}`
- DSL 中 `dataset_ids` 为加密串，跨实例导入后必须手动重绑

---

## 6. 第四步：测试（12 条用例）

| # | 场景 | 输入 | 预期行为（实测口径） |
|---|------|------|---------------------|
| 1 | 正常工单 | A座301空调不制冷，联系方式13800138000 | 工单：空调/设备维修组/中/唯一工单号 |
| 2 | 缺字段追问 | 空调坏了 | 追问：位置+联系方式 |
| 3 | 安全工单 | B座电路跳闸，有焦糊味，疑似电路故障 | 优先级高 + 电力维修组 |
| 4 | 燃气安全 | 一楼食堂闻到燃气味 | 优先级高 + 安保应急组 |
| 5 | 电梯困人 | C座电梯困人，3人被困 | 优先级高 + 电梯维保组（已实测：困人关键词命中安全检测） |
| 6 | 模糊描述 | 办公室太热了，不知道怎么回事 | 追问分支（先验字段后分类） |
| 7 | 多设备 | 会议室灯不亮，空调也不制冷 | 追问/单工单（多设备分次报修，拆分列入演进） |
| 8 | KB 命中 | 暖气不热怎么办？ | KB 命中（0.45 阈值）+ 追问分支 |
| 9 | 重复提交 | A座301空调不制冷 ×2 | 两次均生成工单且工单号不同（无查重，去重列入演进） |
| 10 | 非设备问题 | 快递还没送到 | 追问分支（补全后分类=其他/综合事务组） |
| 11 | 无联系方式 | B座电梯异响 | 生成工单（contact 空，电梯维保组/中）——check_fields 不校验联系方式（已实测；如需强制追问可调整条件） |
| 12 | 非报修输入 | 如何申请会议室？ | 走追问分支（缺位置/联系方式；非报修拒识列入演进）（已实测） |

**运行方式**：
- 编辑器内：右上「运行」逐条输入（⚠️ 编辑器运行不进日志页）
- Service API（推荐，产生日志）：

```bash
curl -X POST http://<your-dify>/v1/workflows/run \
  -H "Authorization: Bearer app-xxxx" \
  -H "Content-Type: application/json" \
  -d '{"inputs": {"user_description": "A座301空调不制冷，联系方式13800138000"}, "response_mode": "blocking", "user": "test-user"}'
```

> 常见 400：`Arg user must be provided` —— body 必须带 `user` 字段。

**通过标准**：语义字段（分类/部门/优先级/追问）符合上表；工单号唯一、created_at 为真实时间。

> **覆盖状态（2026-09-06 20:28）**：12/12 用例已全部实测（Service API·发布版本 2026-09-06 12:05:47），全部 succeeded 且符合上表口径。补跑记录：用例 5 电梯困人（电梯/高/电梯维保组）、用例 10 复测（追问，与首测一致）、用例 11（生成工单 contact 空）、用例 12（追问分支）。

---

## 7. 第五步：发布

1. 编辑器右上「发布」→ 生成版本（版本号含时间戳，可在「发布版本列表」回看）
2. 发布后 Service API 与 WebApp 均使用已发布版本；草稿修改不影响线上
3. 变更流程：改草稿 → 测试运行验证 → 再发布（不要跳过中间验证）

---

## 8. 日志与追踪

- 左侧「日志」：每次**已发布版本**运行的记录（状态/耗时/Tokens/用户）
- 点击记录 → 「追踪」页签：节点级耗时、token 消耗、各节点输入输出——排障定位到节点
- ⚠️ 编辑器草稿运行（draft/run）不产生日志记录，属调试性质

---

## 9. 工单号与时间戳设计（ticket_finalize 代码节点）

**为什么不用 LLM 生成**：实测发现 LLM 生成的 ticket_id 对相同输入倾向于重复（两次运行同号），created_at 出现训练数据年份幻觉（2023 年）。工单号需要强唯一性、时间戳需要真实性——这类确定性需求收敛给代码节点，LLM 只负责语义字段。

```python
import uuid
from datetime import datetime, timezone, timedelta

def main(ticket: dict) -> dict:
    tz = timezone(timedelta(hours=8))
    now = datetime.now(tz)
    ticket['ticket_id'] = 'WO-' + now.strftime('%Y%m%d') + '-' + uuid.uuid4().hex[:8].upper()
    ticket['created_at'] = now.strftime('%Y-%m-%dT%H:%M:%S+08:00')
    return {'ticket': ticket}
```

> ⚠️ Dify code 节点 `main()` 返回 dict 的**键必须与声明的输出变量名一致**（本节点输出变量为 `ticket`，故返回 `{'ticket': ticket}`），否则报 `Output ticket is missing.`

实测：同一输入连续两次运行 → `WO-20260906-E9789E96` / `WO-20260906-2EA0B392`，时间戳均为真实当前时间。

---

## 10. 常见问题排查

| 症状 | 原因 | 处理 |
|------|------|------|
| KB 节点 0 命中 | 相似度阈值高于查询实际得分 | hit-testing 实测相似度 → 按分布下调阈值（本案 0.7→0.45） |
| 节点报 `Output xxx is missing.` | code 节点返回 dict 的键与输出变量名不一致 | 返回 `{'输出变量名': 数据}` |
| 变量引用不生效/下游幻觉 | 节点 ID 含连字符 | 节点 ID 全部改下划线；引用用三段式 |
| 发布检查报「ELIF 不能为空」 | if-else 被解析出空 ELIF case | 删除空 case，仅保留 IF/ELSE |
| API 调用 415 | JSON POST 未显式 Content-Type | 加 `Content-Type: application/json` |
| API 调用 400 | body 缺 `user` 字段 | 补 `"user": "xxx"` |
| 自部署 API 改草稿被旧图覆盖 | 编辑器标签页持有草稿（内存自动回写） | 改草稿前先把 /workflow 标签页导航离开 |
| DeepSeek 做 LLM 时建不了知识库 | DeepSeek 无 embedding 模型 | 另配 embedding（推荐 text-embedding-v3） |

---

## 11. 参考文件

- 演示分镜：[05-demo-script.md](./05-demo-script.md)
- DSL：[09-smart-park-agent-v2.dify.yml](./09-smart-park-agent-v2.dify.yml)（21322 字符，与发布版本一致）
