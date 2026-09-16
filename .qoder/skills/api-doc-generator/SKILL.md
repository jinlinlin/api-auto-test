---
name: api-doc-generator
description: Scan backend code for Java, Go, Python, and Node.js projects to extract HTTP API endpoints and generate interface documentation in JSON or Markdown. Use when the user asks to scan a module, interface URL, or the whole project to auto-generate API docs.
---

# API Doc Generator

面向 Java、Go、Python、Node.js 多技术栈后端项目，从源码中扫描 HTTP 接口定义，并根据用户指定范围**默认同时生成 OpenAPI 3.x JSON 与 Markdown 两份文档**，两者数据同源、一一对应。

## 使用时机

在当前项目中，当用户有类似需求时使用本 Skill：

- “生成接口文档 / API 文档 / 接口说明”
- “根据代码扫描所有接口”
- “按模块生成接口文档，比如 `backend/user-service`”
- “只针对某个 URL 或一组 URL 生成文档”
- “替代或补充 Swagger、OpenAPI 文档”

## 快速开始

1. **确定范围与输出格式**  
   用户会以自然语言指定：
   - 范围：
     - 整个项目：例如 “扫描整个项目的接口”
     - 某个模块目录：例如 “只扫描 `backend/user-service` 目录”
     - 某个接口 URL 或 URL 前缀：例如 “只生成 `/api/user` 相关接口”
   - 输出格式：
     - `json`：仅生成 OpenAPI 3.x JSON 文件
     - `md`：仅生成 Markdown 文档（从 OpenAPI JSON 派生渲染）
     - 未指定：**默认双输出**，同时生成 `api-docs.json`（OpenAPI 3.x）与 `api-docs.md`（从 JSON 派生）

2. **推断后端技术栈与后端根目录**  
   使用工具（如 `Glob`、`Read`）检测项目特征文件与目录结构，推断：
   - 使用的语言/框架（Java / Go / Python / Node.js）
   - 主要后端目录（优先尝试：`backend/`、`server/`、`api/`，或根据框架标准结构推断）

3. **按语言规则扫描接口定义**  
   对每种识别到的语言/框架，使用对应规则在源码中查找路由/接口定义（见后文“语言/框架识别与扫描规则”）。

4. **构建统一接口模型**  
   将不同语言扫描到的接口信息统一映射到 OpenAPI 3.x 结构（见"统一模型 → OpenAPI 3.x 映射规则"），便于后续统一输出。

5. **按用户指定格式输出并写入文件**  
   - 解析得到一个 `outputFormat`（`"json"`、`"md"` 或 `"both"`）以及输出文件路径（见下文"输出文件写入约定"）；  
   - **JSON 输出必须符合 OpenAPI 3.x 规范**（含 `openapi`、`info`、`paths`、`components/schemas` 顶层结构），不得使用自定义 JSON 模型；  
   - **Markdown 从 OpenAPI JSON 派生渲染**（同源），两文件的路径/方法/参数/字段必须一一对应，禁止两套独立生成逻辑；  
   - 若用户未指定格式，默认双输出（`both`）：同时写入 `docs/api/api-docs.json` 与 `docs/api/api-docs.md`；  
   - **必须**：将生成内容写入实际文件（而不是只返回到对话中），然后同时在回答中告知用户文件路径；  
   - 如有需要，仍可在回答中附带一份内容预览（例如前几行或某个服务的小节），但真正的完整结果以文件为准。

---

## 统一模型 → OpenAPI 3.x 映射规则

扫描得到的统一接口模型最终输出为 **OpenAPI 3.x** 规范的 JSON 文件。Markdown 文档从该 JSON 派生渲染，保证同源一致。

每个 endpoint 映射为 `paths.{path}.{method}` 下的 operation，`operationId`/`summary`/`description`/`tags`/`deprecated` 直接对应 OpenAPI 同名字段；`pathParams`/`queryParams`/`headerParams` 分别映射为 `parameters` 中 `in: path`/`query`/`header` 的条目；`requestBody` 和 `responses` 的 schema 均通过 `$ref` 引用 `components/schemas` 中注册的 DTO/VO 定义。

> 若项目已开启 springdoc，可参考 `/v3/api-docs` 的输出结构作为实际样例。

### 关键约束

1. **$ref 不内联**：所有 DTO/VO 类型在 `components/schemas` 中定义一次，请求体、响应体均通过 `$ref` 引用，禁止在多处重复内联相同结构。
2. **统一响应包装**：若项目使用 `Result<T>` 等统一包装类，在 `components/schemas` 中为每个 `Result<XxxVO>` 生成对应的包装 schema，其 `data` 字段通过 `$ref` 引用实际业务 VO。
3. **分页响应**：分页接口的响应 schema 中需包含 `total` 字段（`type: integer`），并在 description 中标注"分页专用"。
4. **参数合并**：`@Parameters/@Parameter` 显式声明的参数与 `@RequestParam`/`@PathVariable` 解析到的参数合并到 OpenAPI 的 `parameters` 数组中，不得遗漏。

---

## 用户输入解析约定

### 支持的典型指令形式

- “扫描整个项目，生成接口文档（md）”
- “只扫描 `backend/user-service` 模块，输出 json”
- “只生成 `/api/user` 和 `/api/admin` 相关接口的文档，格式 md”
- “扫描所有 Java + Go 后端服务，生成统一 JSON 接口清单”

### 解析规则

1. **范围解析**  
   - 若用户提供 **目录路径**（如 `backend/user-service`），仅在该目录下扫描。  
   - 若用户仅提到 “整个项目 / 所有接口”，则：
     - 优先识别标准后端目录（如 `backend/`、`server/`、`api/`、`services/`）；
     - 若无法确定，则在项目根目录下按语言特征文件（如 `pom.xml`、`build.gradle.kts`、`go.mod`、`package.json`、`requirements.txt` 等）推断后端子目录，再在这些目录中扫描。

2. **URL 过滤**  
   - 若用户指定一个或多个 URL / URL 前缀（如 `/api/user`、`/internal/*`），则在构建统一模型后，对 `path` 字段进行过滤，仅保留匹配的 endpoint。  
   - 匹配方式可以是：
     - 完整相等：`path === 指定路径`
     - 前缀匹配：`path` 以指定前缀开头
     - 简单通配（如以 `*` 结尾时表示前缀）

3. **输出格式**  
   - 若用户显式指定 `json`，仅生成 `api-docs.json`（OpenAPI 3.x）；  
   - 若用户显式指定 `md`，仅生成 `api-docs.md`（从 OpenAPI JSON 派生渲染）；  
   - **若用户未指定，默认双输出**：同时生成 `api-docs.json` 与 `api-docs.md`。

---

## 输出文件写入约定

为避免“只在对话中展示结果但没有真实文件”的情况，本 Skill 在实现时需要遵守以下约定：

1. **默认输出目录**  
   - 统一目录 `docs/api/`，Markdown 与 JSON 均写入此目录；
   - 默认双输出文件名：`api-docs.json`（OpenAPI 3.x）+ `api-docs.md`（从 JSON 派生）；
   - 当目录不存在时，应先创建目录，再写入文件。

2. **文件命名规则**  
   - 若用户没有指定文件名，建议按以下规则自动生成：
     - 基础名：`api-docs`；
     - 若用户指定了模块目录（如 `backend/user-service`），可在文件名中带上模块标识，例如：
       - Markdown：`api-docs-user-service.md`
       - JSON：`api-docs-user-service.json`
     - 若存在多次生成或需要区分时间，可在文件名后追加时间戳，例如：`api-docs-20260311-101500.md`。
   - 若用户显式指定了文件名，则优先使用用户指定的名称（并自动补齐扩展名与目录，如仅给出 `user-api`，则写入为 `docs/api/user-api.md`）。

3. **写入与返回策略**  
   - 生成文档后，**必须真正写文件** 到磁盘；  
   - 默认双输出时，`api-docs.json` 与 `api-docs.md` 必须同时写入；  
   - 在回答中至少包含：
     - 实际写入的文件路径（相对项目根，如 `docs/api/api-docs.json` + `docs/api/api-docs.md`）；  
     - 如有必要，附带一小段内容预览或结构摘要（例如服务数量、接口数量）；  
   - 可以同时将完整内容作为字符串返回，但文件写入是强制要求。

4. **覆盖 / 追加策略**  
   - 默认行为可以是 **覆盖写入**（每次生成重写同名文件）；  
   - 若希望保留历史版本，可在实现中采用时间戳命名或版本号命名；  
   - 无论采用哪种策略，都应在回答中明确提示当前生成对应的文件名与是否覆盖了旧文件。

---

## 语言/框架识别与扫描规则

下面是针对各技术栈的识别与接口扫描要点，仅作为 Skill 使用时的指导原则，不要求实现完全精确的语法解析，更侧重于通过模式匹配快速提取大部分接口信息。

### 1. Java（Spring MVC / Spring Boot）

#### 项目识别

- 通过以下信号推断使用 Spring：
  - 存在 `pom.xml`、`build.gradle` 或 `build.gradle.kts`；
  - `pom.xml` / `build.gradle` / `build.gradle.kts` 中包含如 `spring-boot-starter-web`、`spring-web`、`spring-mvc` 等依赖；
  - 源码目录包含 `src/main/java/` 结构。

#### 源文件范围

- 使用 `Glob`：`**/*.java`，在推断的后端根目录（如 `backend/xxx` 或 `src/main/java`）下扫描。

#### 接口识别模式

- **控制器类级注解**：
  - `@RestController`
  - `@Controller`
  - 可能组合 `@RequestMapping("/xxx")`
- **方法级注解**：
  - `@RequestMapping(...)`
  - `@GetMapping(...)`
  - `@PostMapping(...)`
  - `@PutMapping(...)`
  - `@DeleteMapping(...)`
  - `@PatchMapping(...)`
- **路径组合规则**：
  - 类级 `@RequestMapping("/user")` + 方法级 `@GetMapping("/info")` => `/user/info`
  - 若方法级注解使用 `value` 或 `path` 属性，需要解析其字符串值：
    - `@GetMapping("/info")`
    - `@GetMapping(path = "/info")`
    - `@RequestMapping(value = "/info", method = RequestMethod.GET)`
- **HTTP 方法推断**：
  - 由方法级注解名直接推断（`GetMapping` -> `GET`，`PostMapping` -> `POST` 等）；
  - 或由 `@RequestMapping(method = RequestMethod.GET)` 中的 `RequestMethod` 推断。

#### 参数与请求体/响应模型

- **路径参数**：
  - 参数注解 `@PathVariable`，结合方法签名中的参数名与类型。
- **查询参数**：
  - 参数注解 `@RequestParam`。
- **请求体**：
  - 参数注解 `@RequestBody`；
  - **必须**追踪到 DTO 类型定义文件，展开所有字段并记录每个字段的类型、必填性、校验约束与描述。
- **响应模型**：
  - 方法返回类型（如 `Result<LiteratureVO>`、`ResponseEntity<UserDto>`）；
  - **必须**追踪到 VO/DTO 类型定义文件，展开所有字段并记录类型与描述（见下文"响应模型字段展开规则"）。

#### 参数校验约束提取（必须）

对 DTO / Query 类中的每个字段，**必须**读取以下校验注解并完整记录到文档：

| 注解 | 文档中记录方式 | 示例 |
| --- | --- | --- |
| `@NotNull` | 必填=是 | — |
| `@NotBlank` / `@NotEmpty` | 必填=是，不可为空字符串 | — |
| `@Min(n)` | 最小值=n | `@Min(1)` → "最小值: 1" |
| `@Max(n)` | 最大值=n | `@Max(100)` → "最大值: 100" |
| `@Size(min, max)` | 长度/大小范围 | `@Size(min=1, max=100)` → "长度: 1~100" |
| `@Pattern(regexp)` | 格式约束 | `@Pattern(regexp="^\\d{8}$")` → "格式: 8位数字" |
| `@Valid` | 嵌套对象校验 | 递归展开内嵌对象字段 |
| 自定义业务校验（如 `Assert.notNull`） | 在接口备注中说明 | `Assert.notNull(dto.getType(), "type")` → "type 不能为空" |

**重要**：不要仅标注"必填/非必填"，必须将取值范围、长度限制、格式要求等约束信息完整写入参数字段说明。

#### 业务行为补充（Service 层）

对于关键业务接口（如创建、导入、删除等），仅看 Controller 注解无法完整描述业务行为。此时应：

1. 从 Controller 方法体中找到调用的 Service 方法；
2. 读取 Service 方法实现，提取关键业务逻辑步骤；
3. 在接口"接口说明"或"业务行为备注"中补充核心流程，例如：
   - "新增文献后会自动触发异步 RAG 解析任务"
   - "删除前检查文献是否被引用，被引用时返回错误"
   - "导入时先检查 DOI 是否已存在，存在则更新，不存在则新增"

**注意**：不需要逐行翻译代码，只需提取影响用户理解的关键业务规则和副作用。

#### 响应模型字段展开规则（必须）

**核心原则**：文档中的每个请求体和响应体，都必须展开到具体字段级别，不能只写类型名。

1. **请求体 DTO**：
   - 找到 `@RequestBody` 对应的 DTO 类文件，读取并展开所有字段；
   - 每个字段记录：字段名、类型、是否必填（根据校验注解）、约束条件、描述。

2. **响应体 VO/DTO**：
   - 找到方法返回类型中泛型参数对应的 VO/DTO 类文件，读取并展开所有字段；
   - 每个字段记录：字段名、类型、描述；
   - 若 VO 中嵌套了其他 VO/DTO 类型（如 `List<LiteratureTagVO>`），也需要展开该嵌套类型的字段定义，可放在附录或独立章节中。

3. **统一响应包装结构**：
   - 若项目使用了统一响应包装类（如 `Result<T>`、`R<T>`、`ApiResponse<T>`），**必须**：
     - 在文档开头的"通用说明"章节中完整描述包装类的字段结构（如 `code`、`msg`、`data`、`time` 等）；
     - 每个接口的响应描述中，说明 data 字段对应的实际类型。

4. **分页响应特殊处理**：
   - 若接口返回分页数据，响应结构与普通接口不同，需明确说明：
     - `data` 字段为列表类型（如 `List<XxxVO>`），包含当前页数据；
     - `total` 字段为总记录数（独立于 `data` 的顶层字段）；
   - 在"通用说明"中给出分页响应的完整 JSON 示例。

5. **错误码与异常响应**：
   - 扫描项目中的错误码枚举类（如 `ResultCode`、`ErrorCode`、`ApiCode` 等），提取所有错误码及其含义；
   - 扫描 `BusinessException` / `ServiceException` 等自定义异常类，了解业务异常的抛出模式；
   - 在文档"通用说明"中列出所有可能的错误码及对应场景；
   - 在关键接口中，根据 Service 层逻辑标注可能返回的业务错误码（如"文献不存在(16001)"）。

6. **附录组织**：
   - 将所有 VO/DTO 的字段定义汇总到文档末尾的"附录：数据模型定义"章节；
   - 按模块分组，每个 VO/DTO 一个小节；
   - 包含枚举值说明（如 `type: 0=PDF, 1=EPUB, 2=CAJ`）。

#### Query 类处理规则（必须）

- `query/` 包下的查询条件对象（通过 `@Valid` 绑定到 Controller 方法参数）应**展开为 URL 查询参数表**（query params），**不得**当作 requestBody 处理。
- 仅 `@RequestBody` 注解标注的 `dto/` 包下对象才作为请求体（requestBody）。
- 典型分包约定：`dto/`（请求体 DTO）、`query/`（查询条件 Query）、`vo/`（响应视图 VO）。

#### @RequirePermission 数组语义说明

- 本项目权限注解形式为 `@RequirePermission(value = {KNOWLEDGE_ALL, KNOWLEDGE_MANAGE})`，`value` 为字符串数组。
- **默认语义为 OR**（满足其一即可）：用户拥有数组中任意一个权限即可访问。
- 若项目实际为 AND 语义（需同时满足所有权限），需在文档权限描述中注明组合方式，例如：`需同时具备 KNOWLEDGE_ALL 和 KNOWLEDGE_MANAGE 权限`。
- 扫描时应检查权限常量定义类（如 `PermissionConstants`）以获取权限标识的完整含义。

#### 上下文自动注入参数识别

某些参数虽然出现在 DTO/Query 中，但实际由框架从登录上下文自动注入，前端无需传递。识别规则：

- 在 Controller 方法体中，若某字段通过 `WorkspaceContextHolder` 等工具类从上下文获取并手动 set 到 DTO 中，则该字段为**上下文自动注入参数**。
- 文档中应标注：`（由系统从上下文自动注入，前端无需传递）`。
- 常见模式（按项目实际上下文工具类适配）：
  - `query.setWorkspaceId(WorkspaceContextHolder.getWorkspaceId())`
  - `dto.setUserId(GlobalUserContextHolder.getUserId())`

#### 权限描述规范

- 若接口使用了自定义权限注解（如 `@RequirePermission`、`@PreAuthorize` 等），**必须**记录所需权限标识。
- 若接口**没有**权限注解，统一描述为：`无特殊权限要求（需登录）`。不要写"无特殊权限注解"等技术实现措辞。
- 权限描述应放在"接口说明"之后、参数表之前的独立段落。

#### 服务分组规则

- **service 名**取模块目录名：如 `uniplore-server-xxx` 中的 `xxx` 部分（例如 `uniplore-server-uniresearch` → service 名为 `uniresearch`）。
- 若模块下还有子分组（如按 controller 包名 `literature`、`workspace` 等），可作为 `module` 字段进一步细分。
- **多模块聚合**：所有 `uniplore-server-xxx` 模块的接口聚合到同一份文档中，按 service 名（模块段名）分组展示。

#### 描述信息

- 优先使用（按优先级从高到低）：
  1. OpenAPI 注解：`@Operation(summary, description, operationId)`、`@Tag(name, description)`、`@Schema(description)`、`@Parameter(description, example)`；
  2. 方法上的 JavaDoc 注释；
  3. 类名 + 方法名作为退化描述。
- **@Parameters/@Parameter 显式声明的参数**（含 `in=QUERY`、`required`、`example` 等属性）**必须并入参数表**，与 `@RequestParam` / `@PathVariable` 解析到的参数合并展示，不得遗漏。

#### 可选：复用 springdoc 已有能力

- 若项目已集成 springdoc / swagger 基础设施（如本项目有 `uniplore-spring-boot3-starter-swagger`），**优先**从运行中服务的 `/v3/api-docs` 端点导出 OpenAPI JSON 作为基底，再按需补全缺失项（如业务行为备注、上下文注入参数、权限常量含义等）。
- 源码扫描的 JSON 作为 springdoc 不可用时的**兜底来源**。
- **无论来源如何，最终输出的 JSON 必须严格符合 OpenAPI 3.x 规范**，保证两份产出（JSON 与 MD）格式一致、可互相替换。
- 源码扫描同时作为校验手段，确保导出内容与代码实际实现一致。

---

### 1b. Java（Spring WebFlux）

#### 项目识别

- 通过以下信号推断使用 Spring WebFlux：
  - Gradle 依赖中包含 `spring-boot-starter-webflux` 或自定义 webflux starter（如 `uniplore-spring-boot3-starter-common-webflux`、`uniplore-spring-boot3-starter-ai-mcp-server-webflux`）；
  - 源码中存在 `RouterFunction`、`HandlerFunction`、`ServerRequest`、`ServerResponse` 等类型。

#### 接口识别模式

- **函数式路由（RouterFunction）**：
  - 常见形式：
    - `RouterFunctions.route().GET("/path", handler)` 或 `route(GET("/path"), handler)`
    - `@Bean RouterFunction<ServerResponse>` 方法中定义路由
  - 链式路由：
    - `route().GET("/path", handler).POST("/path", handler).build()`
  - 路径组合：通过 `.path("/prefix", () -> route().GET("/sub", handler).build())` 嵌套。
- **注解式控制器**：与 Spring MVC 相同（`@RestController` + `@GetMapping` 等），扫描规则不变。

#### 响应式返回类型展开规则

- `Mono<T>` 返回类型：展开为内层类型 `T`（如 `Mono<Result<UserVO>>` → 响应体为 `Result<UserVO>` 的 JSON 结构）。
- `Flux<T>` 返回类型：展开为 `T` 的数组（如 `Flux<ItemVO>` → 响应为 `ItemVO[]` 列表）。
- `Mono<Void>` / `Mono<Boolean>`：无响应体或简单布尔值。
- 对于 SSE（Server-Sent Events）流式接口（返回 `Flux<ServerSentEvent<T>>`），需在文档中标注为流式响应并说明事件格式。

#### 参数与请求体

- 与 Spring MVC 规则一致：`@PathVariable`、`@RequestParam`、`@RequestBody` 等注解用法相同。
- 函数式路由的 handler 中通过 `request.pathVariable()`、`request.queryParam()`、`request.bodyToMono()` 等方式提取参数，需追踪到对应的 DTO/VO 类型。

---

### 2. Go（net/http、Gin 等）

#### 项目识别

- 存在 `go.mod`；
- 源码中含有 `package main` 或标准 Go 目录结构（如 `cmd/`, `internal/`, `pkg/` 等）。

#### 源文件范围

- 使用 `Glob`：`**/*.go`，过滤测试文件（如 `*_test.go`）。

#### 接口识别模式

- **Gin**：
  - 常见形式：
    - `router.GET("/path", handler)`
    - `router.POST("/path", handler)`
    - `group.PUT("/path", handler)`
    - `r.DELETE("/path", handler)`
  - 分组路由：
    - `userGroup := router.Group("/user")`
    - `userGroup.GET("/info", handler)` => `/user/info`
- **net/http**：
  - `http.HandleFunc("/path", handler)`
  - `mux.HandleFunc("/path", handler)`

#### 参数与请求体/响应模型（简化）

- Go 中类型信息多通过 handler 内部解析（如 `c.BindJSON(&req)` 或 `json.NewDecoder(r.Body).Decode(&req)`），可选择性：
  - 搜索 handler 内部的 `BindJSON`、`Decode` 调用，尝试获取绑定的 struct 类型；
  - 若成本过高，可仅记录 handler 函数名与请求体类型名（如果明显）。

#### 描述信息

- 优先使用 handler 函数上方的注释，作为 `summary` 或 `description`。

---

### 3. Python（Django / Flask / FastAPI）

#### 项目识别

- 通过以下信号：
  - 存在 `manage.py`、`settings.py`（Django）；
  - 依赖文件（如 `requirements.txt`、`pyproject.toml`、`Pipfile`）中包含：
    - `django`
    - `flask`
    - `fastapi`
  - 源码目录里存在 `app.py`、`main.py`、`asgi.py`、`wsgi.py` 等典型入口。

#### 源文件范围

- 使用 `Glob`：`**/*.py`，排除迁移等可选目录（如 `migrations/`）视需要调整。

#### 接口识别模式

- **Flask**：
  - 装饰器形式：
    - `@app.route("/path", methods=["GET", "POST"])`
    - `@blueprint.route("/path", methods=["GET"])`
  - 从 `methods` 参数中读取 HTTP method 列表，未指定默认 `GET`。

- **FastAPI**：
  - 装饰器形式：
    - `@app.get("/path")`
    - `@app.post("/path")`
    - `@router.put("/path")`
    - `@router.delete("/path")`
  - HTTP 方法直接由装饰器名推断。
  - 参数类型注解（如 `id: int`, `q: str | None = Query(None)`）可用于构建 path/query 参数描述。

- **Django**（基础支持）：
  - 在 `urls.py` 或其它 URL 配置中查找：
    - `urlpatterns = [ path("path/", view_func, name="..."), ... ]`
    - `re_path(r"^path/$", view_func, ...)`
  - 进一步可关联到 view 函数或 class-based view，但首版可只记录 path、view 名称与 HTTP method（若难以精确，则默认 `GET`，并在描述中标注“method 未精确识别”）。

#### 描述信息

- 优先来源：
  - 视图函数 / 处理函数的 docstring；
  - 装饰器参数（若有描述类扩展）；
  - 与路由同文件中的注释。

---

### 4. Node.js（Express / Koa / NestJS 等）

#### 项目识别

- 存在 `package.json`；
- `package.json` 中依赖包含：
  - `express`
  - `koa`, `@koa/router`
  - `@nestjs/common`, `@nestjs/core`

#### 源文件范围

- 使用 `Glob`：
  - JavaScript：`**/*.js`
  - TypeScript：`**/*.ts`
  - 可排除前端目录（如 `frontend/`、`src/client/`），重点扫描后端目录（如 `src/`, `server/`, `backend/` 中与 Node 服务相关部分）。

#### 接口识别模式

- **Express**：
  - 典型形式：
    - `app.get("/path", handler)`
    - `app.post("/path", handler)`
    - `router.put("/path", handler)`
    - `router.delete("/path", handler)`
  - 支持链式调用或变量别名（如 `const r = express.Router(); r.get("/path", ...)`）。

- **Koa + Router**：
  - `router.get("/path", handler)`
  - `router.post("/path", handler)`

- **NestJS**：
  - 控制器类：
    - `@Controller('/users')` class `UserController` {...}
  - 方法装饰器：
    - `@Get('/list')`
    - `@Post('/')`
    - `@Put(':id')`
  - 路径组合与 Spring 类似：类级路径前缀 + 方法级路径。

#### 参数与请求体/响应模型（简化）

- Node 通常在 handler 中解析 `req.params`、`req.query`、`req.body` 等：
  - 若直接解析字段（如 `const { id } = req.params`），可在同一函数中简单收集字段名；
  - 若使用 DTO 或 schema 校验库（如 `Joi`、`zod`、`class-validator` 等），可进一步读取 schema 定义，构建字段描述（首版可选）。

#### 描述信息

- 优先来源：
  - handler 上方注释；
  - NestJS 中类或方法上的装饰器（如 `@ApiOperation({ summary: '...' })`）；
  - 若无，则使用 controller + handler 函数名作为退化描述。

---

## 扫描与构建模型的流程建议

实现本 Skill 时，可按以下顺序执行（可根据项目实际情况调整）：

1. **定位后端根目录**（如存在多个服务，可多次执行或多服务并行扫描）。  
2. **检测语言/框架**：  
   - 根据特征文件与依赖判断哪些语言/框架实际存在；  
   - 仅对存在的语言执行扫描，避免无用遍历。
3. **按语言扫描接口定义**：  
   - 使用 `Glob` 定位候选源码文件集；  
   - 使用 `Grep` 或等价工具，按上述模式搜索注解、装饰器、路由调用等；  
   - 在需要更高语义理解时，可在单文件范围内用 `SemanticSearch` 或 AST 方案，但需注意性能。
4. **提取字段并填充统一 Endpoint 模型**：  
   - 每发现一个接口定义，就构建一个 Endpoint 实例；  
   - 尽量填充路径、方法、控制器、operationId、参数、请求体、响应等字段；  
   - 无法确定的信息可留空或用描述性占位（例如 `type: "unknown"`）。
5. **按服务/模块分组**：  
   - 根据目录结构、命名约定（如 `user-service`, `order-service`）推断 `service` 字段；  
   - 可按 package/module/controller 进行次级分组，便于 Markdown 展示。
6. **应用用户指定的 URL / 模块过滤**：  
   - 根据用户提供的目录/URL 条件过滤 Endpoint 列表。  
7. **生成最终输出**：  
   - 将统一模型映射为 OpenAPI 3.x JSON（见下文"输出格式模板"）；  
   - 若需 Markdown，从 OpenAPI JSON 派生渲染，保证同源一致；  
   - 默认双输出：同时写入 `api-docs.json` 与 `api-docs.md`。

---

## 输出格式模板

### JSON 输出模板（OpenAPI 3.x）

JSON 输出**必须**符合 OpenAPI 3.x 规范。完整结构示例见上文"统一模型 → OpenAPI 3.x 映射规则"章节。

扩展字段可通过 `info.x-*` 或顶层 `x-*` 扩展属性添加，例如：

- `x-generatedAt`：扫描时间戳；
- `x-project`：项目名称；
- `x-scanConfig`：扫描配置信息。

### Markdown 输出模板

Markdown 从 OpenAPI JSON 派生渲染（同源），结构如下：

```markdown
# 接口文档（自动生成）

生成时间：2026-03-11T10:00:00Z  
项目：example-project

---

## 通用说明

### Base URL 与认证方式

| 服务 | Base URL 前缀 | 认证方式 |
| --- | --- | --- |
| uniplore-server-uniresearch | `/uniplore-server-uniresearch` | Header `Authorization: Bearer {token}` |
| ... | ... | ... |

> 实际 Base URL 前缀从网关路由配置（如 gateway 的 `bootstrap-routes.yml`）与各服务 `application.yml` 的 `context-path` 中提取；认证方式从网关过滤器（如 AuthFilter）或安全配置中提取。

### 统一响应结构

所有接口均使用统一响应包装，结构如下：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| code | integer | 状态码，200 表示成功 |
| msg | string | 提示信息 |
| time | long | 服务器时间戳 |
| data | T | 业务数据（具体类型见各接口说明） |
| total | long | 分页接口专用，总记录数（非分页接口不返回此字段） |

### 分页响应结构示例

```json
{
  "code": 200,
  "msg": "success",
  "time": 1712345678000,
  "total": 100,
  "data": [
    { "id": "...", "name": "..." }
  ]
}
```

### 错误码说明

| 错误码 | 含义 | 场景 |
| --- | --- | --- |
| 200 | 成功 | 请求正常处理 |
| 400 | 业务失败 | 业务逻辑校验不通过 |
| 403 | 拒绝访问 | 无权限 |
| 404 | 未找到 | 资源不存在 |
| 500 | 服务异常 | 服务端内部错误 |
| 4000 | 参数错误 | 请求参数校验失败 |

（以上为示例，实际错误码应从项目源码中提取）

---

## user-service 用户服务

### GET /api/users/{id}

**接口说明**  
获取用户详情。

**权限要求**  
无特殊权限要求（需登录）

**所属控制器**  
`UserController.getUserInfo`

**路径参数**

| 名称 | 类型   | 必填 | 约束 | 说明     |
| ---- | ------ | ---- | ---- | -------- |
| id   | string | 是   | —    | 用户 ID  |

**查询参数**

| 名称   | 类型    | 必填 | 约束 | 说明             |
| ------ | ------- | ---- | ---- | ---------------- |
| verbose | boolean | 否   | —    | 是否返回扩展信息 |

**请求头参数**

| 名称        | 类型   | 必填 | 约束 | 说明         |
| ----------- | ------ | ---- | ---- | ------------ |
| X-Request-Id | string | 否   | —    | 请求追踪 ID  |

**请求体（Request Body）**

Content-Type: `application/json`

| 字段 | 类型    | 必填 | 约束 | 说明   |
| ---- | ------- | ---- | ---- | ------ |
| name | string  | 是   | 长度: 1~50 | 用户名 |
| age  | integer | 否   | 最小值: 0 | 年龄   |

**响应（Responses）**

- `200 OK` — data 类型: `UserVO`

  | 字段 | 类型   | 说明    |
  | ---- | ------ | ------- |
  | id   | string | 用户 ID |
  | name | string | 用户名  |

**请求示例**

```http
GET /api/users/123?verbose=true
```

**响应示例**

```json
{
  "code": 200,
  "msg": "success",
  "time": 1712345678000,
  "data": {
    "id": "123",
    "name": "张三"
  }
}
```

**可能的错误响应**

| 错误码 | 场景 |
| --- | --- |
| 404 | 用户 ID 不存在 |
| 4000 | id 参数格式错误 |

---
```

实现时可：

- 按 `service` 分节；  
- 每个 Endpoint 一个子小节，标题统一为 `METHOD path`；  
- 缺失的字段用"（暂无信息）"或直接省略对应表格部分；
- **每个接口必须包含"请求示例"和"响应示例"段落**，示例内容应基于实际参数结构和响应结构构造合理的示例值；
- **每个接口应包含"可能的错误响应"段落**，列出该接口可能返回的业务错误码及场景；
- 参数表中增加"约束"列，用于记录取值范围、长度限制、格式要求等；
- 文档末尾必须包含"附录：数据模型定义"章节，汇总所有 VO/DTO 的完整字段定义。

---

## 使用 Cursor 工具的建议

实现或使用本 Skill 时，可以配合 Cursor 提供的工具，以在不同语言下高效扫描：

- **Glob**：  
  - 用于快速定位代码文件：
    - Java：`**/*.java`
    - Go：`**/*.go`（排除 `*_test.go`）
    - Python：`**/*.py`
    - Node.js：`**/*.js`, `**/*.ts`
- **Grep**（或同等 ripgrep 工具）：  
  - 用于匹配典型注解/装饰器/路由模式，例如：
    - Java：`@RestController`, `@GetMapping`, `@PostMapping` 等；
    - Go：`\\.GET\\(\"/`, `http.HandleFunc\\(\"/` 等；
    - Python：`@app\\.route\\(\"/`, `@router\\.get\\(\"/`, `urlpatterns` 等；
    - Node.js：`app\\.get\\(\"/`, `router\\.post\\(\"/`, `@Controller\\(`, `@Get\\(\"/` 等。
- **SemanticSearch**：  
  - 在大型文件或复杂 handler 内，帮助理解与接口关联的 DTO、schema 定义等；
  - 在无法通过简单字符串搜索确定请求体/响应结构时，可以对相关类型/结构体进行语义查找。

性能建议：

- 尽量先用 **特征文件 + 目录结构** 限定搜索范围，再在指定目录下做 `Glob` + `Grep`；  
- 避免在整个仓库做无筛选的全文搜索，尤其是当前端/文档/依赖目录较大时。

---

## 最佳实践与质量检查清单

生成接口文档后，应对照以下清单进行自检，确保文档质量：

### 完整性检查

- [ ] 所有 VO/DTO 的字段是否已展开到具体字段级别（而非只写类型名）？
- [ ] 统一响应包装类（如 `Result<T>`）的字段结构是否在"通用说明"中描述？
- [ ] 分页接口是否明确说明了 `data` + `total` 的响应结构？
- [ ] 错误码是否从源码中提取并在"通用说明"中列出？
- [ ] 每个接口是否包含请求示例和响应示例？
- [ ] 每个接口是否标注了权限要求？

### 准确性检查

- [ ] 参数校验约束是否完整（取值范围、长度限制、格式要求等）？
- [ ] 上下文自动注入的参数（如 workspaceId、userId）是否已标注"前端无需传递"？
- [ ] 权限描述是否统一使用业务语言（而非"无特殊权限注解"等技术措辞）？
- [ ] 关键业务接口的业务行为描述是否准确（是否读了 Service 层逻辑）？
- [ ] 枚举值是否列出了所有可选值及含义？

### 可读性检查

- [ ] 文档结构是否清晰（通用说明 → 各模块接口 → 附录数据模型）？
- [ ] 参数表是否包含"约束"列？
- [ ] JSON 示例中的值是否合理（不是简单的 "xxx" 占位符）？
- [ ] 跨模块依赖是否有交叉说明（如"上传文件需先调用文件上传接口"）？

---

## 扩展与自定义

- **新增语言/框架支持**：  
  - 在后续需要支持新的后端技术（如 PHP Laravel、Ruby on Rails、Rust Axum/Actix 等）时，可以：
    - 添加对应的“项目识别”规则（特征文件、依赖名、目录结构）；
    - 定义新的“接口识别模式”（路由、装饰器、宏等）；
    - 将提取到的信息映射到 OpenAPI 3.x 的 paths / components / parameters 结构。

- **项目特定约定**：  
  - 若项目有自定义的注解/装饰器/路由封装（例如自定义 `@ApiGet`、`defineRoute("/path", "GET")` 等）：
    - 可在本 Skill 的实现中加入额外的匹配规则；
    - 或在项目本身维护一个简单的“映射说明”（如单独文档），由使用者手动补充接口说明。

- **与现有文档体系集成**：  
  - 生成的 JSON 本身就是 OpenAPI 3.x 规范文件，可直接导入 Swagger UI、Stoplight、Postman 等工具；  
  - 生成的 Markdown 可以直接放入文档站（如 Docusaurus、VuePress、MkDocs）中，或进一步加工。

