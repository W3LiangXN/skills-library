---
name: meowgic-openapi-to-api-method
description: Convert OpenAPI/Apifox/Swagger text into Meowgic project service code with matching TypeScript types and API methods. Use in the meowgic repo when adding or updating files under services from API documentation, especially when the user invokes /meowgic-openapi-to-api-method or asks to generate TS types/API interfaces from OpenAPI text.
version: 1.0.1
---

# Meowgic OpenAPI 到 API 方法

## 概览

把 OpenAPI/Apifox 文本落地为 `services` 下的类型声明和 API 调用。目标是最小改动、严格按文档生成、对齐当前 meowgic 的 `http` / `bffHttp` 封装与 TypeScript 风格。

## 工作流程

1. 先阅读 `references/project-api-patterns.md`，确认当前项目的 API、类型、命名和落点约定。
2. 从用户提供的 OpenAPI 文本提取路径、方法、path/query/body 参数、200 响应里的 `data` 结构，以及组件 schema 引用。
3. 只按文档内出现的信息生成类型；缺失字段不猜，含糊字段用 `unknown` 或保留为可选字段，并在回复里说明。
4. 选择落点：
   - 已有领域目录时，优先扩展 `services/<module>/type.ts` 与 `services/<module>/index.ts`。
   - 没有合适目录时，新增 `services/<module>/type.ts` 和 `services/<module>/index.ts`，并只在必要时补导出。
   - Next 同源 route handler 才使用 `bffHttp`；普通 Go 后端接口使用 `http`。
5. 生成类型：
   - 响应类型只描述业务 `data`，不要把 `{ code, message, data }` 包进普通响应类型。
   - 对象结构优先用 `export interface`，结构联合/派生类型用 `export type`，固定取值的业务字段优先用 `export enum`。
   - 字段注释使用中文 `/** ... */`，接口字段名保持后端原样。
6. 生成 API 方法：
   - `GET` 查询参数放第二参：`http.get<Response>(url, params)`。
   - `POST` / `PUT` / `DELETE` 请求体放第二参：`http.post<Response>(url, params)`。
   - 路径参数用模板字符串：`` `/api/v1/items/${id}` ``。
   - 只有调用方确实需要整包响应时才传 `{ completeResponse: true }`。
7. 校验：
   - 运行项目已有的类型检查或 lint 脚本；如果成本过高，至少运行 `npx tsc --noEmit` 或说明未运行原因。
   - 用 `rg` 检查新增类型/方法是否有命名冲突和未使用导入。

## OpenAPI 提取规则

- 优先读取 200 响应 `content.application/json.schema.properties.data`。
- 如果 `data` 是 `{ count, list }`，优先映射为 `CommonPageResponse<Item>`，除非当前模块已有不同习惯。
- `required` 中没有列出的字段标为可选。
- `format: date-time`、RFC3339、时间字段默认用 `string`，不要改成 `Date`，除非同模块既有类型已经这么做。
- `format: uint64`、`uuid.UUID`、ID 类字段默认用 `string`。
- `integer` / `number` 映射为 `number`；decimal 金额/积分如文档示例为字符串或项目既有同类字段为字符串时用 `string`。
- `array` 根据 `items` 映射；复杂未知对象用 `Record<string, unknown>`，不要用裸 `object` 或 `any`。
- 认证方式、接入方式、发布状态等固定字符串或数值集合，优先生成 `export enum`，而不是字面量联合 `type`。成员保留后端原始值，命名沿用同目录风格（新模块默认大写下划线），并添加中文注释；缺少中文说明时可依据值的明确含义命名，不因此退回字面量联合，也不臆造业务含义。仅在枚举无法准确表达契约或需保持既有调用兼容时使用联合类型，并说明原因。
- 接口或嵌套对象字段中的内联固定值联合也按同一规则抽成独立的 `export enum`，即使仅有一个字段使用；字段改为引用枚举，并保留原有的 `?` 和可空性。例如 `userType`、`connectorType`、`authMethod` 分别引用 `ConnectorUserType`、`ConnectorType`、`ConnectorMCPAuthMethod`。枚举名体现业务语义；同义且取值一致时复用已有枚举，不同认证场景按实际含义区分。仅处理本次涉及的类型，不批量改写无关字段。
- `deprecated: true` 字段可保留，但注释里标注“已废弃”。

## 代码生成约束

- 保持改动外科化：只改目标模块相关文件，不顺手重构相邻代码。
- 使用现有导入别名与同目录风格；已有相对导入就相对导入，已有 `@/services` 就沿用。
- 类型导入使用 `import type`。
- 优先复用 `services/api-params.ts` 的 `CommonParams`、`CommonPageResponse`、`CommonTimestamp`、`TaskState`、`OrderBy`、`SortBy`。
- 不新增路由常量文件；meowgic 当前服务层直接在方法里写 URL。
- 不把 OpenAPI 的 `code` / `message` / headers / 非 200 错误响应生成到业务类型里。
- 不引入运行时校验库、请求客户端抽象或复杂泛型工具，除非用户明确要求。
- 不为了“完整”生成未被 API 方法使用的组件 schema；只生成响应、请求体、参数实际引用到的类型。

## 输出偏好

直接修改项目文件。完成后用简短中文说明：

- 新增/修改了哪些 `services` 文件。
- 每个接口生成的方法名与响应类型。
- 运行了什么校验；如果未运行，说明原因。

## 参考

- `references/project-api-patterns.md`
