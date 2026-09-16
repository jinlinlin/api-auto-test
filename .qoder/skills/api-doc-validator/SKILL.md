---
name: api-doc-validator
description: 根据用户指定的接口文档反查代码（支持 Java、Go、Python、Node.js 等后端），校验文档与真实实现是否一致；在发现不一致时给出详细差异说明与修订后的文档内容（片段或整篇），帮助保持接口文档与代码同步。
---

# API 文档校验与修复工具（api-doc-validator）

## 概述

本技能用于：**根据用户提供的接口文档，反查后端代码，实现“文档 ←→ 代码”的一致性校验与文档修复建议**。  
支持多种后端语言与框架（Java、Go、Python、Node.js 等），不限制具体代码风格，只依赖路由/注解/装饰器等结构信息。

典型能力：

- 根据用户指定的接口文档（如 `doc/用户管理接口文档.md`）和代码目录，自动检查：
  - URL 路径、HTTP 方法
  - 请求参数：名称、类型（粗粒度）、位置（path/query/body/form）、是否必填
  - 返回类型大类（JSON / HTML 视图 / 纯文本等）
  - 权限/认证标识（若能从代码中解析到）
- 发现不一致时：
  - 生成“差异清单”（文档 vs 代码）
  - 给出**建议修订后的接口文档片段**（Markdown）
  - 差异较多时，可以给出“整份文档的修订版草稿”

> 本 Skill 是对 `api-doc-generator` 的补充：  
> - `api-doc-generator`：从代码 → 生成接口文档  
> - `api-doc-validator`：接口文档 ↔ 代码 双向校验与修复

---

## 使用时机

在当前项目中，当用户有类似需求时使用本 Skill：

- “帮我检查 `doc/用户管理接口文档.md` 和代码是否一致”
- “对比接口文档和 Controller 实现，哪里不一致？”
- “重构之后，哪些接口文档已经过期？”
- “代码里多出来的接口，文档没写全，帮我列出来”
- “请直接给出修正后的接口文档内容”

---

## 快速开始

### 1. 命令式触发

推荐使用命令前缀的方式显式触发本 Skill，例如：

- 只指定文档，使用默认代码根目录（适用于当前 RuoYi 项目）：

  ```text
  /api-doc-validator @doc/用户管理接口文档.md
  ```

- 指定文档 + 指定代码根目录：

  ```text
  /api-doc-validator @doc/部门管理接口文档.md codeRoot=ruoyi-admin/src/main/java
  ```

> 约定：  
> - `@doc/xxx.md`：接口文档路径（相对当前仓库根目录）  
> - `codeRoot`：可选，代码扫描根目录，默认值可以按项目约定设置（如本仓库可默认 `ruoyi-admin/src/main/java`）。

### 2. 自然语言触发

当用户以自然语言提出“文档与代码是否一致”“帮我校验接口文档”等需求时，你应当判断是否适合使用本 Skill，并在内部按本说明中的流程执行。

示例：

- “帮我检查用户管理接口文档和代码实现是否一致”
- “根据 `doc/部门管理接口文档.md` 去对照 Controller，看文档有没有写错”

---

## 适用语言与框架（代码风格不限）

本 Skill **不限制代码风格**（缩进、命名等），只依赖各语言/框架的路由与元信息定义方式。当前应重点支持：

- **Java**
  - Spring / Spring Boot / Spring MVC 等
  - 典型路由/元信息来源：
    - `@RequestMapping` / `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping`
    - 权限/认证注解：如 `@PreAuthorize`、`@RequiresPermissions("system:user:list")`、`@RequiresRoles` 等
    - Controller 类命名：`*Controller`, `*Resource`, `*Endpoint` 等
- **Python**
  - Flask / Django / FastAPI / Sanic 等
  - 典型路由定义：
    - Flask：`@app.route("/path", methods=["GET", "POST"])`
    - FastAPI：`@router.get("/path")` / `@router.post("/path")`
    - Django：`urlpatterns = [path("xxx", view), re_path(...)]`
  - 可利用函数签名、Pydantic 模型、Serializer 等推断参数/响应
- **Node.js**
  - Express / Koa / NestJS / Hapi 等
  - 典型路由定义：
    - Express/Koa：`router.get("/path", handler)`、`app.post("/path", ...)`
    - NestJS：`@Controller("/user")` + `@Get`, `@Post`, `@Patch` 等装饰器
- **Go**
  - Gin / Echo / net/http / chi 等
  - 典型路由定义：
    - Gin：`r.GET("/path", handler)`、`r.POST("/path", handler)`
    - Echo：`e.GET("/path", handler)`、`e.POST(...)`
    - 其他：通过 `http.HandleFunc` / `mux.HandleFunc` 等注册

当项目中同时存在多语言服务时，应按目录或文件特征（例如 `pom.xml`, `go.mod`, `package.json`, `requirements.txt` 等）推断各子模块的技术栈，并分别应用对应的解析规则。

---

## 核心工作流程

整体流程可分为四步：

1. **解析接口文档（事实源 A）**
2. **解析后端代码（事实源 B）**
3. **对比文档与代码，生成差异清单**
4. **生成文档修订建议（片段或整篇草稿）**

### 1. 解析接口文档（事实源 A）

从用户指定的接口文档（通常为 Markdown）中提取接口信息，尤其适配以下风格的文档：

- RuoYi 当前已有的：
  - `doc/用户管理接口文档.md`
  - `doc/部门管理接口文档.md`

建议从文档中提取的字段包括（尽量结构化）：

- 模块名 / 分组名（如 `system-user`、`system-dept`）
- 接口标题与说明
- `HTTP 方法 + URL`（如 `GET /system/user/list`）
- 请求参数列表：
  - 名称
  - 类型（string / integer / boolean / array / object 等粗粒度）
  - 位置：path / query / body / form / header
  - 是否必填
  - 简要说明
- 响应信息（概要）：
  - 返回类型大类：JSON 列表、JSON 对象、HTML 视图、文件下载等
  - 关键字段说明（若文档已有）
- 权限/认证信息（若文档中注明，如 `system:user:list` 等）

> 对于当前仓库的文档，可在实现时优先适配其表格格式与章节结构，再逐步扩展到更通用的 Markdown 格式。

### 2. 解析后端代码（事实源 B）

根据语言与框架，采用相应的解析策略，从代码中抽取与文档结构类似的接口模型。  
目标是获得一个统一的中间模型，例如：

- `service/module/controller`（可选）
- `path`：`/system/user/list`
- `method`：`GET` / `POST` / `PUT` / `DELETE` 等
- `summary` / `description`（如有）
- `parameters`：
  - `name` / `type`（粗粒度）/ `in`（path/query/body/form/header）/ `required`
- `responses`：返回类型大类（JSON/HTML/FILE 等），关键包装类型（如 `AjaxResult`、`TableDataInfo` 等）
- `auth` / `permissions`：从注解/中间件/装饰器中提取

解析时可参考（非穷举）：

- Java：扫描 `@RequestMapping` 及其派生注解、方法参数注解、返回类型与权限注解；
- Python：分析路由装饰器 + 函数签名 + 模型类型；
- Node.js：分析路由注册调用（`router.get` 等）+ 中间件链；
- Go：分析路由表注册代码 + Handler 函数签名。

### 3. 文档 vs 代码 对比规则

对比粒度应覆盖以下几个方面：

1. **接口存在性**
   - 文档中声明的接口，在代码中是否存在对应实现：
     - 如果以 `path + method` 为键在代码中找不到对应条目，则标记为：
       - “文档中有，代码中无（可能已废弃或路径/方法有误）”
   - 反向检查（可选）：代码中存在但文档未写的接口，列出为：
       - “代码中有，文档中缺失（建议补充文档）”

2. **URL 与 HTTP 方法一致性**
   - 比较文档中的 `GET /path` 与代码中的实际映射：
     - 路径大小写、前后斜杠
     - Path 变量占位符结构，如：`/user/{id}` vs `/user/{userId}`

3. **参数列表一致性**
   - 按参数名称匹配，比较：
     - 位置：path / query / body / form / header
     - 类型（字符串/数字/布尔/数组/对象等粗粒度）
     - 必填/可选
   - 若文档描述与代码推断不一致（例如文档标记必填，代码参数为可选带默认值），指出具体差异。

4. **权限/认证一致性（如可获取）**
   - 文档中标注的权限标识（如 `system:user:list`）与代码中的权限注解是否一致。
   - 若文档未写权限而代码有严格限制，可以提示“文档缺少权限说明”。

5. **返回类型与包装结构**
   - 只做粗粒度校验，不必完全解析 DTO：
     - JSON 列表 / JSON 对象 / HTML 视图 / 文件流 等类型是否一致
     - 包装类型（如 `AjaxResult`、`TableDataInfo`）是否相符

### 4. 差异输出与文档修订建议

对每一处差异，建议输出内容包括：

- **差异摘要**：一句话说明哪里不一致（例如：“文档声明为 `GET /system/user/list`，代码实际为 `POST /system/user/list`”）。
- **详细对比**：以小表格或对比段落展示“文档值” vs “代码值”。
- **建议修订后的文档片段**：给出修正后的 Markdown 片段，便于用户直接替换。

当差异较多、文档整体质量较差时，可以：

- 生成“整份接口文档的修订版草稿”，内容以代码解析结果为主，并尽量保留原文中的自然语言描述；
- 在回答中明确说明这是草稿，建议用户人工确认后再覆盖原文档。

> 默认行为：**不直接修改项目中的文档文件**，只在对话中输出修订内容与建议。  
> 若用户明确要求“请帮我直接更新 `doc/xxx.md`”，可以在得到确认后使用编辑工具对文件进行修改。

---

## 使用示例

### 示例 1：校验用户管理接口文档

用户输入：

```text
/api-doc-validator @doc/用户管理接口文档.md
```

期望行为：

1. 从 `doc/用户管理接口文档.md` 中解析所有接口定义。
2. 按默认代码根目录（例如当前项目为 `ruoyi-admin/src/main/java`）扫描相关 Controller 代码。
3. 对比文档与代码的一致性，生成：
   - 文档有但代码无的接口列表；
   - 代码有但文档无的接口建议列表；
   - 每个不一致接口的字段级差异说明；
   - 修订后的文档片段（Markdown）。

### 示例 2：指定子模块代码目录进行校验

用户输入：

```text
/api-doc-validator @doc/部门管理接口文档.md codeRoot=ruoyi-admin/src/main/java/com/ruoyi/web/controller/system
```

期望行为：

1. 只在指定 `codeRoot` 目录下查找 Controller/路由实现。
2. 按上述规则进行比对并输出差异与修订建议。

---

## 注意事项

- **多语言、多框架**：本 Skill 设计为多语言通用，不限制代码风格，但具体实现时可优先支持当前项目中实际存在的语言与框架，再逐步扩展。
- **复杂对象与嵌套结构**：对于复杂请求体/响应体（嵌套 DTO、集合等），建议只做字段级的“粗对齐”与类型大类判断，不强制一一精确还原所有嵌套结构。
- **不覆盖测试职责**：本 Skill 只关注“文档 vs 实现”的一致性，与接口测试（如 `api-test-case-generator`）是互补关系。
- **安全信息披露**：在输出差异与示例时，避免泄露敏感配置、密钥或内部实现细节，仅围绕接口契约本身进行说明。

