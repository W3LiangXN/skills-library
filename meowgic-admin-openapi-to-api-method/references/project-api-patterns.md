# Meowgic Admin 项目模式与示例

以下依据 meowgic-admin 源码整理，用于定位和理解差异；执行时以目标工作树的实现为准。示例中的 widgets 是说明用途的假设接口，不代表真实后端已有该接口。

## 源码定位

| 要确认的行为 | 优先阅读 |
| --- | --- |
| 请求签名、解包、鉴权 | `src/utils/request.ts` |
| 分页与公共参数 | `src/types/global.ts` |
| 列表、详情、写操作整包返回 | `src/pages/tools/service.ts`、`typing.ts`、`index.tsx` |
| 从组合入参中分离 path 与 body | `src/pages/user/service.ts`、`src/pages/email/service.ts` |
| DELETE 配置及操作响应 | `src/pages/email/service.ts`、`typing.ts` |
| 跨页面复用服务 | `src/pages/subscription/service.ts` |
| mock 接入 | 目标目录的 `service.ts`、`mock.ts`、`typing.ts` 和调用点 |
| Discover 独立客户端 | `src/pages/app-manage/discover-service/request.ts`、`service.ts` |

普通领域以 `src/pages/<module>/service.ts` + `typing.ts` 组织。部分模块（如 `credits`、`user-app-create`、`app-manage/template`、`connector`）当前仍有 mock 服务，不能把 mock 返回值当成 API 文档。方法名通常为 `getXxxList`、`getXxxDetail`、`createXxx`、`updateXxx`、`deleteXxx`，沿用已有导出名和参数形式。

## 普通列表与整包写操作

假设文档定义列表 query 只有可选 `page/pageSize` 和必填 `status`，列表 `data` 为必填 `count/list`，删除响应仅有必填 `code/message`：

`typing.ts`：

```ts
import type { IListResult, TCommonParams } from "@/types/global";

/** 组件状态 */
export enum WidgetStatus {
  /** 已启用 */
  ENABLED = "enabled",
  /** 已停用 */
  DISABLED = "disabled",
}

/** 组件列表参数 */
export interface WidgetListParams
  extends Pick<TCommonParams, "page" | "pageSize"> {
  /** 状态 */
  status: WidgetStatus;
}

/** 组件列表项 */
export interface WidgetListItem {
  /** 组件 ID */
  id: string;
  /** 名称 */
  name: string;
}

/** 组件列表返回值 */
export type WidgetListResult = IListResult<WidgetListItem>;

/** 删除组件返回值（整包） */
export interface DeleteWidgetResponse {
  /** 业务状态码 */
  code: number;
  /** 返回消息 */
  message: string;
}
```

`service.ts`：

```ts
import { DELETE, GET } from "@/utils/request";
import type {
  DeleteWidgetResponse,
  WidgetListParams,
  WidgetListResult,
} from "./typing";

/** 获取组件列表。 */
export const getWidgetList = async (
  params: WidgetListParams,
): Promise<WidgetListResult> =>
  GET<WidgetListResult>("/admin/v1/widgets", params);

/** 删除组件。 */
export const deleteWidget = async (
  id: string,
): Promise<DeleteWidgetResponse> =>
  DELETE<DeleteWidgetResponse>(`/admin/v1/widgets/${id}`, {
    completeResponse: true,
  });
```

后台 `completeResponse` 只影响运行时，不调整泛型；例如 `POST<IResponse<CreatedData>>(..., { completeResponse: true })` 的结果才有正确的 `code/message/data` 类型。`IResponse` 从 `@/utils/request` 按类型导入。仅写 `POST<CreatedData>` 不会得到主站 `http.post` 那样的整包返回类型推导。

`src/utils/request.ts` 的成功拦截器也不会因为 HTTP 成功响应里的业务 `code` 非零就自动 reject，因此页面已有的 `response.code` 检查需要保留。

## 条件分支

### query、body 与数组

- POST/PUT 同时带 query 与 body：`POST<Response>(url, body, { params: query })`。
- DELETE 同时带 query 与 body：`DELETE<Response>(url, { params: query, data: body })`；需要整包时在同一配置中追加 `completeResponse: true`。
- GET 第二参通过 `qs.stringify` 序列化，数组默认不是主站的逗号拼接方式。公共 `TCommonParams.ids` 是 `string[]`，不无依据扩成 `string[] | string`。
- 文档明确要求重复键、逗号或其他数组格式时，为当前接口选择对应序列化。可使用 `GET<Response>(url, undefined, { params: query, paramsSerializer: ... })`，使 Axios 的序列化设置生效；第三参 `paramsSerializer` 无法改变第二参已经拼好的 query。不要同时在两处传同一组 query。

### PATCH 或其他封装未导出的方法

先确认目标工作树是否已新增 helper。当前普通封装只导出 GET/POST/PUT/DELETE；必要时复用导出的 `request` 实例，不新建 axios 客户端。默认解包情况下可使用 Axios 第二泛型显式描述拦截后的返回值，例如 `request.patch<BusinessData, BusinessData>(url, body)`，并声明方法为 `Promise<BusinessData>`；若要整包则两处都用完整响应类型，配置通过 `CustomConfig` 类型提供 `completeResponse`。核对本地 Axios 签名后再使用。

### multipart 与二进制

普通客户端默认设置 JSON Content-Type。multipart 应使用 `FormData`，并按本地 Axios 的浏览器适配方式覆盖默认 JSON header（例如设为 `multipart/form-data`，由 Axios/浏览器补 boundary），不能照搬主站 `http.post` 的自动识别规则，也不能手写 boundary。没有上传需求时无需添加此分支代码。

下载文件等非 envelope 响应必须避免 `data?.data` 解包。对 Blob 响应配置 `responseType: "blob"` 与 `completeResponse: true`，泛型写 `Blob`；这里的“整包”指 HTTP 响应体，不是 `AxiosResponse<Blob>`。

### Discover 独立服务

`src/pages/app-manage/discover-service/request.ts` 导出 `discoverRequest`，有独立 `DISCOVER_BASE_URL` 与 `x-revalidate-secret` 配置，没有普通客户端的 data 解包。现有服务从 Axios 响应取 `response.data`，并针对 `success: false` 做有限重试。

仅当目标属于此服务时沿用该客户端及相关约定；不要把 `/api/revalidate/discover` 改接普通后台封装，或将其重试策略扩散到其他写接口。技能不读取或输出密钥值。

### mock 与 UI 契约差异

先检查调用方是否依赖整包 `code`、内部 `data`，或直接读取 `list/count`，再替换请求。已有 UI 可能包含表单、筛选或展示用派生字段，应与真实 API 参数区分。API 文档不足以构造某个 UI 字段时明确报告缺口，不伪造默认值掩盖差异；必要的适配仅限用户要求接入的接口。
