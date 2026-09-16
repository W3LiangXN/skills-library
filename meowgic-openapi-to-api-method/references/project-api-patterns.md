# Meowgic API 落地模式速览

用于把 OpenAPI/Apifox 文本转成当前项目 `services` 代码时对齐现有风格。

## 服务层结构

- 普通后端接口：`services/<module>/index.ts` + `services/<module>/type.ts`。
- 请求封装：`services/_http.ts` 导出 `http`，会拼接 `NEXT_PUBLIC_API_URL`，自动带 token，并默认把 `{ code, message, data }` 解包为 `data`。
- BFF 接口：`services/_bff-http.ts` 导出 `bffHttp`，只用于 Next 同源 route handler，例如 `app/api/v1/cloud/apps/[appId]/...`。
- 通用类型：`services/api-params.ts`。
- 项目没有 API 路由常量集中表；当前习惯是在方法中直接写 URL。

## HTTP 调用范式

普通列表：

```ts
import { http } from '../_http';
import type { CommonPageResponse } from '../api-params';
import type { CreditUsage } from './type';

/** 获取积分使用情况 */
export const getCreditUsages = (params?: CommonParams) =>
  http.get<CommonPageResponse<CreditUsage>>('/api/v1/credits/usages', params);
```

详情与路径参数：

```ts
/** 获取应用分类详情 */
export function getAppCategoryDetail(id: string) {
  return http.get<AppCategoryItem>(`/api/v1/app-categories/${id}`);
}
```

POST/PUT/DELETE：

```ts
/** 更新用户信息 */
export const updateUserInfo = (params: { nickname?: string; avatar?: string }) =>
  http.put<void>('/api/v1/users', params, { completeResponse: true });
```

BFF：

```ts
import { bffHttp } from '@/services/_bff-http';

/** 列出 app 下所有 Storage bucket */
export const getCloudStorageBuckets = (appId: string) => {
  return bffHttp.get<{ items: CloudStorageBucket[] }>(
    `/api/v1/cloud/apps/${appId}/storage/buckets`,
  );
};
```

multipart：

```ts
export const audioToText = (params: AudioToTextParams) => {
  const form = new FormData();
  form.append('language', params.language ?? 'en');
  return http.post<{ content: string }>('/api/v1/ai/audio/transcriptions', form);
};
```

## `completeResponse` 判断

- 默认不要传第三参，方法返回 OpenAPI 响应里的 `data`。
- 只有既有调用方需要整包 `{ code, message, data }`，或同模块类似创建/认证接口已经使用整包时，才传 `{ completeResponse: true }`。
- 传了 `{ completeResponse: true }` 后，`http` 的返回类型是 `Result<T>`，但服务方法的泛型仍写业务 `data` 类型，例如 `http.post<SignInResponse>(..., { completeResponse: true })`。

## 参数风格

- `GET` query：第二参传对象；没有 query 但要传 options 时第二参写 `undefined`。
- path 参数：使用模板字符串拼进 URL，不保留 `{id}` 占位符。
- body 参数：POST/PUT/DELETE 第二参传 `params`。
- query 中数组会被 `buildQueryUrl` 转为逗号分隔字符串；OpenAPI 中 `ids: string[]` 可以直接用 `string[] | string`，已有 `CommonParams.ids` 就是这个风格。

## 类型声明风格

- 对象结构用 `export interface` 或同文件既有的 `export type` 风格；同目录已有明显偏好时跟随同目录。
- 联合类型、字面量联合、工具类型用 `export type`。
- 枚举用 `export enum`；成员命名跟同目录现有风格，不强制统一大小写。
- 注释使用中文 `/** ... */`，字段注释放在字段上一行。
- 类型导入使用 `import type`。
- 后端字段名保持原样，不把 `snake_case` 改成 `camelCase`。
- 避免 `any`；未知 JSON 用 `Record<string, unknown>` 或 `unknown`。

## 常用复用类型

来自 `services/api-params.ts`：

- `CommonParams`：常见搜索、排序、分页参数。
- `CommonPageResponse<T>`：`{ list: T[]; count: number }`。
- `CommonTimestamp`：常见创建/更新时间字段。
- `TaskState`：任务状态枚举。
- `OrderBy` / `SortBy`：排序字段与排序方向。

列表参数示例：

```ts
import type { CommonParams, CommonTimestamp } from '../api-params';

/** 应用分类参数 */
export interface AppCategoryParams extends CommonParams {
  /** 父分类ID */
  parentId?: string;
}

/** 应用分类项 */
export interface AppCategoryItem extends CommonTimestamp {
  /** 分类ID */
  id: string;
  /** 分类名称 */
  name: string;
}
```

## OpenAPI 到 TS 映射

- `string` -> `string`
- `format: date-time` / RFC3339 -> `string`
- `format: uuid.UUID` / `uint64` / ID 字段 -> `string`
- `integer` / `number` -> `number`
- `boolean` -> `boolean`
- `array` -> `Item[]`
- 空对象但语义未知 -> `Record<string, unknown>`
- `null` 可能出现时 -> `T | null`
- required 外的字段 -> `?`
- OpenAPI 包装字段 `code`、`message` 不进入业务类型。

## 命名建议

- 方法名：`getXxx`、`createXxx`、`updateXxx`、`deleteXxx`、`postXxx`，优先参考同目录。
- 请求参数：`XxxParams`。
- 响应数据：`XxxResponse`、`XxxResult` 或直接业务实体名；同目录已有 `Item` / `Detail` / `Data` 风格时沿用。
- 列表项：`XxxItem`。
- 枚举：用文档描述命名，例如 `SubscriptionStatus`、`IntegrationDiscoverStatus`。

## 质量检查清单

- 新增 API 方法是否导入了正确的 `http` 或 `bffHttp`。
- 普通响应类型是否只描述 `data`。
- 列表响应是否可复用 `CommonPageResponse<T>`。
- 日期和 ID 是否按项目习惯使用 `string`。
- 是否避免生成未被使用的 schema 类型。
- 是否没有顺手重排无关代码。
