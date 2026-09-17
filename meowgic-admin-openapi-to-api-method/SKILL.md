---
name: meowgic-admin-openapi-to-api-method
description: 将 OpenAPI、Apifox 或 Swagger 文档转换为 meowgic-admin 项目的 TypeScript 类型与 API 方法。用于在该项目中新增或更新 src/pages 下的 service.ts、typing.ts，或将已有 mock 服务接入文档中的真实接口；meowgic 主站的 services 代码使用其专属技能。
---

# Meowgic Admin OpenAPI 到 API 方法

为 meowgic-admin 的 Umi 管理后台生成页面领域服务和类型，保留现有调用契约，按接口文档决定字段与请求位置。

## 工作流程

1. 确认目标仓库及其 `AGENTS.md`。以用户指定的路径或当前工作树为准，检查 `src/utils/request.ts`、`src/types/global.ts` 和目标页面目录；不要根据技能名称切换到 meowgic 主站。
2. 阅读 [项目模式与示例](references/project-api-patterns.md)，再核对目标模块的 `service.ts`、`typing.ts` 及调用方。确认方法是否已存在、是否使用 mock、调用方消费业务数据还是整包响应。
3. 从文档提取 HTTP 方法、原始路径、path/query/header/body 参数、成功状态码、响应 envelope 和实际引用的 schema。解析 `$ref`；缺失的引用或相互矛盾的定义需明确指出，不能凭页面展示字段补造接口。
4. 选择现有领域目录，扩展 `src/pages/<module>/service.ts` 和 `typing.ts`；嵌套模块沿用现有目录，如 `notification/system`、`subscription/bill`。新领域只创建必要的这两个文件，不附带生成页面、菜单或路由。
5. 按下述规则生成类型与方法。若接入 mock，只替换文档覆盖的方法，检查真实响应与已有 UI 类型的差异；保留尚未接入的方法及其 mock，不顺手删除整份 mock 文件。
6. 对照文档逐项检查路径、参数位置、必填性、响应层级与客户端选择，并执行项目可用的类型检查及目标文件 lint。完成后报告文件、方法与返回类型、校验结果，以及未解决的文档缺口。

## 类型映射

- 后端字段保持原名；对象优先 `export interface`，别名、结构联合和派生类型用 `export type`；固定取值的业务字段优先用 `export enum`。方法及字段使用中文 `/** ... */` 注释，纯类型使用 `import type`，运行时枚举使用普通导入。
- 按每层对象的 `required` 决定字段可选性；参数必填性与 `requestBody.required` 分别处理。path 参数必填。可选与可空分别表达为 `?` 和 `T | null`，未知结构用 `unknown` 或 `Record<string, unknown>`，不把“不知道”写成可选。
- 日期时间保持 JSON 的 `string`；数值用 `number`，UUID 用 `string`。ID 与大整数以文档的实际传输类型为准，不仅凭字段名改成字符串；`uint64` 声明与字符串示例冲突时先核对既有接口或澄清。金额同样保留字符串/数值的实际契约。
- 解析数组 `items`、组合 schema 和枚举，仅生成请求/响应可达的类型。认证方式、接入方式、发布状态等固定字符串或数值集合，优先生成 `export enum`，而不是字面量联合 `type`。成员保留后端原始值，命名沿用模块风格（新模块默认大写下划线），并添加中文注释；缺少中文说明时可依据值的明确含义命名，不因此退回字面量联合，也不臆造业务含义。仅在枚举无法准确表达契约或需保持既有调用兼容时使用联合类型，并说明原因。文档未定义的 UI 筛选值（例如 `ALL`）不进入接口允许值，可复用 `Exclude` 收窄已有枚举。
- 接口或嵌套对象字段中的内联固定值联合也按同一规则抽成独立的 `export enum`，即使仅有一个字段使用；字段改为引用枚举，并保留原有的 `?` 和可空性。例如 `userType`、`connectorType`、`authMethod` 分别引用 `ConnectorUserType`、`ConnectorType`、`ConnectorMCPAuthMethod`。枚举名体现业务语义；同义且取值一致时复用已有枚举，不同认证场景按实际含义区分。仅处理本次涉及的类型，不批量改写无关字段。
- `{ count: number; list: T[] }` 形状及必填性匹配时复用 `IListResult<T>`，否则按文档定义。
- 从 `@/types/global` 复用 `TCommonParams`、`EOrderBy`、`ESortBy`、`SystemLang` 等兼容类型。`TCommonParams` 字段都是可选的，且包含多种分页/搜索字段；只接受其中一部分的接口使用 `Pick`，必填字段显式收紧，避免继承出文档不存在的参数。
- 不复制主站的 `CommonPageResponse`、`CommonTimestamp` 或 `TaskState` 到后台。已有类型只有语义和结构均兼容才复用。

## 请求与响应契约

普通后端请求从 `@/utils/request` 导入所需方法，直接使用文档中的 URL。后台通常为 `/admin/v1/...`，但也有 `/api/v1/...`，不能统一替换前缀。

| 方法 | 参数位置 |
| --- | --- |
| `GET<T>(url, query?, config?)` | 第二参为 query，由封装调用 `qs.stringify` |
| `POST<T>(url, body?, config?)` | 第二参为 body，query 放第三参的 `params` |
| `PUT<T>(url, body?, config?)` | 第二参为 body，query 放第三参的 `params` |
| `DELETE<T>(url, config?)` | 第二参是配置，query 放 `params`，body 放 `data` |

- 路径参数插入 URL；若与 body 共用入参对象，先解构分离，避免把 path 字段顺带发进 body。无 body 但需要第三参配置时传 `undefined`。
- 默认拦截器返回 `response.data?.data`，所以泛型是业务 `data` 类型。
- `{ completeResponse: true }` 返回整个 HTTP 响应体；封装仍声明 `Promise<T>`，不会自动把 `T` 包成响应类型。此时泛型和方法返回值必须是完整响应类型。
- 更新已有方法时，按调用方是否读取 `code`、`message`、`data` 保持响应层级。后台多个写操作依赖整包判断业务状态码；不可机械套用“响应只生成 data”。新方法根据文档和同模块调用习惯选择。
- 优先复用兼容的模块 `XxxActionResponse` / `XxxOperateResponse`；文档完整符合必填的 `{ code, message, data }` 时可复用 `IResponse<T>`。缺少 `data` 或 `data` 可选时按实际形状声明，不强行套用它。
- 以实际成功响应为准，支持 200/201 等；204 无响应体用 `void`。非 envelope 的普通 JSON 响应需保留整个响应体，不能让默认解包丢失数据。二进制、multipart、PATCH、数组序列化和 Discover 服务见参考中的条件分支。
- 将鉴权、通用通知、baseURL 留在现有封装；只为文档要求的额外 header 传配置。保持改动局限于接口接入，不新增请求框架或路由常量层。

## 校验与交付

- 检查已有方法/类型和调用点，避免重名、破坏既有签名或引入未使用导入。现存宽松类型不作为新代码使用 `any` 的理由。
- 从 `package.json` 与锁文件确认实际工具。优先运行已有检查；没有 typecheck 脚本时可运行本地 `tsc --noEmit`，并使用本地 ESLint 检查修改的 TS 文件，不执行全仓自动修复。
- Umi 的 `tsconfig.json` 依赖 `src/.umi/tsconfig.json`。缺少生成文件时先运行项目已有的 setup 脚本；依赖或环境不满足时说明阻塞，不为绕过检查修改配置。
- 区分本次引入的问题与既有检查错误；技能生成和静态校验无需调用真实创建、删除、发送等接口。
- 用简短中文列出变更文件、接口对应的方法/返回类型、检查结果及必要的待确认项。
