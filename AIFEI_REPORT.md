# Aifei Framework — 综合源码分析报告

**项目地址**: `~/lab/aifei`  
**版本**: 1.0.0  
**描述**: "the world's first AI coding framework" — 由 JFinal 作者詹波开发  
**许可证**: Apache 2.0  
**编译**: Java 8 (保留 `-parameters` 编译选项用于运行时反射读取方法参数名)

---

## 1. 项目整体结构

### 模块划分 (`pom.xml`)

```
aifei-parent (pom)
 ├── aifei          — 核心框架 (core, config, server, router, argument, aop, util, proxy, log)
 ├── aifei-log      — 日志适配层 (slf4j, log4j2)
 ├── aifei-json     — JSON 序列化 (fastjson2 适配)
 ├── aifei-proxy    — AOP 代理实现 (cglib, javassist)
 ├── aifei-db       — 数据库 ORM (Db+Row, Model, 事务, 分页, SQL 指令)
 ├── aifei-enjoy    — 模板引擎 (Enjoy, 表达式解析, AST)
 ├── aifei-undertow — Undertow HTTP 服务器集成
 └── aifei-all      — 聚合模块 (uber JAR)
```

**关键外部依赖**: fastjson2, druid, HikariCP, cglib-nodep, javassist, undertow-core, slf4j, log4j2

---

## 2. AifeiConfig / Settings / Routes / Plugins

### AifeiConfig 接口
- **源码**: `aifei/src/main/java/cn/aifei/config/AifeiConfig.java`
- **签名**: `public interface AifeiConfig<I extends Input, O extends Output>`
- **方法**:
  - `void config(Settings<I, O> settings)` — 配置 Settings
  - `void config(Routes routes)` — 配置路由
  - `void config(Plugins plugins)` — 配置插件
  - `default void onStart()` — 启动回调（插件已启动）
  - `default void onStop()` — 关闭回调（Server 已停，插件尚未停）

**设计思路**: 泛型 `I, O` 贯穿全框架，保障 Input/Output 类型统一。用户实现该接口，框架回调三个 config 方法。

### Aifei 启动入口
- **源码**: `aifei/src/main/java/cn/aifei/core/Aifei.java`
- **核心方法**: `public static <I, O> void start(AifeiConfig<I, O> aifeiConfig, String[] args)`
- **启动流程**:
  1. 解析命令行 `--key=value` 参数为系统属性
  2. 调用 `PathUtil.init(aifeiConfig)` 初始化项目路径
  3. 创建 `Settings`, `Routes`, `Plugins` 实例
  4. 依次回调 `aifeiConfig.config(settings)`, `aifeiConfig.config(routes)`, `aifeiConfig.config(plugins)`
  5. 校验 Server 和 Handler 已配置
  6. 启动所有 Plugin
  7. 回调 `aifeiConfig.onStart()`
  8. 构建 Handler 责任链
  9. 初始化 Dispatcher
  10. 启动 Server
  11. 注册 JVM ShutdownHook 调用 `Aifei.stop()`

### Settings
- **源码**: `aifei/src/main/java/cn/aifei/config/Settings.java`
- **签名**: `public class Settings<I extends Input, O extends Output>`
- **链式 API**: 几乎所有 setter 返回 `this`
- **关键方法**:
  - `setServer(Server<P1,P2>, Dispatcher<P1,P2,I,O>)` — 配置 Server + Dispatcher
  - `addHandler(Handler<I,O>)` — 添加 Handler 到链
  - `addGlobalInterceptor(Interceptor)` — 添加全局拦截器
  - `setLogFactory(LogFactory)` — 配置日志
  - `setProxyFactory(ProxyFactory)` — 配置 AOP 代理
  - `setUploadPath / setDownloadPath` — 文件路径
  - `configArgument(Consumer<ArgumentKit>)` — 配置参数注入

### Routes
- **源码**: `aifei/src/main/java/cn/aifei/config/Routes.java`
- **方法**:
  - `add(String path, Class<?> target)` / `add(path, target, Interceptor...)` — 手动添加路由
  - `scan(String basePackage)` / `scan(basePackage, Interceptor...)` / `scan(basePackage, Interceptor[], Predicate<Class<?>>)` — 自动扫描
  - `setActionOverload(boolean)` — 是否允许 action 重载

### Plugins
- **源码**: `aifei/src/main/java/cn/aifei/config/Plugins.java`
- **方法**: `add(Plugin)` — 添加插件
- **Plugin 接口**: `aifei/src/main/java/cn/aifei/plugin/Plugin.java` — `start()` / `stop()`

---

## 3. Path 注解 & 路由机制

### @Path 注解
- **源码**: `aifei/src/main/java/cn/aifei/core/Path.java`
- **目标**: `ElementType.TYPE, ElementType.METHOD`
- **属性**: `String value()`
- **设计**:
  - 类级别必须以 `/` 打头
  - 方法级别以 `/` 打头为绝对路径（忽略类级别）
  - 方法级别空字符串 `""` = 入口方法（类似 JFinal 的 index）
  - 不支持 `@Inherited`（提升确定性）

### @Para 注解
- **源码**: `aifei/src/main/java/cn/aifei/core/Para.java`
- **属性**: `path()`, `name()`, `defaultValue()`, `match()`
- 用于配置 action 方法参数的路由匹配行为

### Router 路由核心
- **源码**: `aifei/src/main/java/cn/aifei/router/Router.java`
- **数据结构**: `Map<String, Object> mapping` — key 为 actionPath，value 为 `Action` 或 `ActionGroup`
- **扫描流程**: `scan(basePackage, interceptors, skip)` → `Scanner` 扫描包 → 对每个 `@Path` 类构建路由
- **构建路由 `buildRoute`**:
  - 遍历类的 public 方法（排除 `@NoPath`）
  - 解析方法级 `@Path` 或使用入口方法名
  - 组合 `classPath + methodPath` 为 `actionPath`
  - 创建 `Interceptor[]` (Routes级 → Class级 → Method级)
  - 构建 `Argument[]` 参数数组
  - 创建 `Action` 并调用 `addAction()`
- **路由匹配 `getAction(path, input)`**:
  - 先精确匹配 `mapping.get(path)`
  - 未命中则尝试 `path.substring(0, lastIndexOf('/'))` 并提取 `pathPara`
  - `Action` 单一路由: 验证 pathPara 存在性
  - `ActionGroup` 多重重载: 调用 `matchParameter()` 按参数数量优先匹配

### Action
- **源码**: `aifei/src/main/java/cn/aifei/router/Action.java`
- **属性**: `actionPath`, `targetPath`, `interceptors`, `targetClass`, `method`, `arguments`
- **提供**: `getMatchParaCount()`, `getPathParaCount()`, `getMethodParaCount()`

### ActionGroup
- **源码**: `aifei/src/main/java/cn/aifei/router/ActionGroup.java`
- 用于 action 重载场景，多个 Action 共享同一 path

### RouterKit
- **源码**: `aifei/src/main/java/cn/aifei/router/RouterKit.java`
- 单例模式，持有 Router 实例

---

## 4. Out 统一返回对象

项目 **源码中不包含 `Out` 类**，Out 是一个用户/业务层的概念。框架通过 **`RollbackDecision` 接口** 支持事务根据返回值自动回滚：

- **源码**: `aifei-db/src/main/java/cn/aifei/db/transaction/RollbackDecision.java`
- **签名**: `public interface RollbackDecision { boolean shouldRollback(); }`
- **设计**: 事务 `transaction()` 的返回值如果实现了 `RollbackDecision`，框架在提交前调用 `shouldRollback()`，若返回 true 则自动回滚事务，无需显式调用 `tx.rollback()`

框架文档中示例的 `Out` 类（用户项目中定义）：
```java
public class Out implements RollbackDecision {
    Code code;
    public boolean shouldRollback() { return code != Code.OK; }
    // ...
}
```

使用 `Db.transaction(tx -> { return Out.ok("成功"); })` — 返回 `Out.ok` 不触发回滚，返回 `Out.fail` 自动回滚。

---

## 5. AifeiRow / Model 体系 & sql() 链式查询 & 动态 SQL

### AifeiRow — 行数据基类
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/AifeiRow.java`
- **签名**: `public class AifeiRow<R extends AifeiRow<R>> implements Iterable<Map.Entry<String, Object>>, Serializable`
- **核心属性**:
  - `String table` — 表名
  - `String[] primaryKey` — 主键名
  - `Map<String, Object> data` — 数据存储
  - `Set<String> change` — 变更追踪集合
- **核心方法**:
  - `set(field, value)` — 设置值并记入 change 集合
  - `put(key, value)` — 设置值但不记入 change
  - `get(field)`, `getStr`, `getInt`, ... — 类型安全的 getter
  - `table()`, `primaryKey()` — 元数据访问
  - `columnDefined(field)` — 字段是否在表中定义（子类可覆盖）
  - `to(Class<T> modelClass)` — 转换为子类类型（共享 data Map）

### Row — 通用行
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/Row.java`
- **签名**: `public class Row extends AifeiRow<Row>`
- **静态工厂**: `Row.of(String table)`, `Row.of(table, primaryKey)`, `Row.of(table, pk1, pk2)`
- **实例方法**: `id(Object)`, `id()`, `compositeId()`, `insert()`, `delete()`, `update()`

### AifeiDao — 数据访问基类
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/AifeiDao.java`
- **签名**: `public class AifeiDao<D extends AifeiDao<D, R>, R extends AifeiRow<R>>`
- **核心属性**: `config`, `sqlPara`, `rowType`, `select`, `hasGroupBy`
- **链式 SQL**: 所有方法返回 `(D) this` 实现链式调用

### Dao — 通用 DAO
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/Dao.java`
- **签名**: `public class Dao extends AifeiDao<Dao, Row>` — 专用于 Row 的 DAO

### sql() 链式查询
**入口**: `Db.sql(String sql, Object... paras)` → 返回 `Dao` 对象

**完整链式示例**:
```java
Db.sql("select * from user where id = ?", 123).find();
Db.sql("select * from article").paginate(1, 10);
User.sql("select * from user where age > ?", 18).find();
```

**执行器架构** (包: `aifei-db/src/main/java/cn/aifei/db/executor/`):
- `FindExecutor` — 查询返回 Row/Model 列表
- `QueryExecutor` — 查询返回标量值
- `InsertExecutor` — 插入
- `UpdateExecutor` — 更新
- `DeleteExecutor` — 删除
- `PaginateExecutor` — 分页
- `BatchInsertExecutor` / `BatchUpdateExecutor` / `BatchExecutor` — 批量
- `FunExecutor` — JDBC 回调

### Enjoy SQL 与动态 SQL 指令

**#where 指令**
- **源码**: `aifei-db/src/main/java/cn/aifei/db/sql/WhereDirective.java`
- **用法**: `#where(field, operator, para)` — 条件值不为 null 时才生成 SQL
- **输出**: 自动生成 `where` 关键字 + 第一个条件不含 `and` 前缀

**#and 指令**
- **源码**: `aifei-db/src/main/java/cn/aifei/db/sql/AndDirective.java`
- **用法**: `#and(name, '=', name)` — 仅在参数值不为 null 时生成 `and name = ?`
- **扩展操作符**: `contains`, `notContains`, `startsWith`, `endsWith`

**#orderBy 指令**
- **源码**: `aifei-db/src/main/java/cn/aifei/db/sql/OrderByDirective.java`
- **用法**: `#orderBy(f1, f2, ...)` — 白名单机制防 SQL 注入
- **前端参数格式**: `{field: 'name', order: 'desc'}` 或 `[{...}, {...}]`
- **字段映射**: `#orderBy(updated_time:updateTime)` 映射后端字段到前端字段

**Condition 条件引擎**
- **源码**: `aifei-db/src/main/java/cn/aifei/db/sql/Condition.java`
- 解析 `#where/#and` 的三个参数（field, operator, para），当 para 为 null 时跳过 SQL 生成
- 支持 like 扩展运算符的自动值包装

---

## 6. Db 工具类 — transaction() 事务 & batch() 批量

### Db 工具类
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/Db.java`
- 所有方法为 `static`，委托给 `Dao` 或 `Batch` 实例
- **入口**: `Db.use()` → `Dao` / `Db.batch()` → `Batch`

### transaction() 事务
- **源码**: `Db.java:533` → 委托到 `AifeiDao.transaction()`
- **AifeiDao.transaction**: `aifei-db/.../core/AifeiDao.java:833`
- **签名**: `<T> T transaction(Atom<T> atom)` / `transaction(Isolation, Atom<T>)`
- **事务设计** (Transaction.java):
  - **隐式提交**: 一切正常才提交，否则回滚
  - **回滚触发**: `rollback()`, `rollbackIf(true)`, `RollbackDecision.shouldRollback()` 返回 true, 抛出异常
  - **嵌套事务**: 隔离级别只升不降
  - **onException**: 可配置异常回调函数
  - **onCommitSuccess**: 事务提交成功后的回调列表
- **TransactionExecutor**: 执行事务的 begin/commit/rollback/end 全流程
- **TransactionKit** (`aifei-db/.../transaction/TransactionKit.java`):
  - `ThreadLocal<Transaction<?>>` 管理当前事务
  - `setOnException()` / `setOnBeforeCommit()` 全局回调

### batch() 批量操作
- **源码**: `aifei-db/src/main/java/cn/aifei/db/core/Batch.java`
- **三种模式**:
  1. Row/Model 列表批量 insert/update
  2. 单条 sql + 多条参数 execute
  3. 多条 sql 直接执行
- **配置**: `batchSize()`, `commitOnBatchSize()`, `getGeneratedKeys()`, `putUpdateCountsToRow()`
- **BatchResult**: 返回批量操作结果

---

## 7. 参数注入系统

### Argument 抽象基类
- **源码**: `aifei/src/main/java/cn/aifei/argument/Argument.java`
- **签名**: `public abstract class Argument<I extends Input, O extends Output, T>`
- **核心方法**: `public abstract T getValue(I input, O output)`
- **元数据**: `name`, `type`, `parameterizedType`, `defaultValue`, `pathPara`, `index`, `match`, `matchCount`

### Argument 子类体系
- `aifei/.../argument/BasicArguments.java`:
  - `BasicArgument` — 基础类型 (String, Integer, Long, Double 等)
  - `InputArgument` — 注入 Input 对象
  - `OutputArgument` — 注入 Output 对象
  - `DateArgument` — 日期类型
- `BeanArgument` — Java Bean 自动装配
- `MapArgument` — Map 注入
- `ListArgument` — List 注入
- `ArrayArgument` — 数组注入
- `EnumArgument` — 枚举注入
- `NoMatch` — 标记接口，实现类不参与路由匹配

### ArgumentKit 管理
- **源码**: `aifei/src/main/java/cn/aifei/argument/ArgumentKit.java`
- **方法**: `register(type, argumentClass)`, `remove(type)`, `registerInputArgumentFun()`, `registerOutputArgumentFun()`, `registerBeanArgumentFun()`, `setFactory()`

### ArgumentFactory
- **源码**: `aifei/src/main/java/cn/aifei/argument/ArgumentFactory.java`
- 根据 Parameter 类型分派到对应的 Argument 实现类

### 设计思路
Argument 本质是 Method 级依赖注入，但注入时可利用请求上下文 Input/Output：
- 可注入当前登录用户
- 可注入 Sse 实现流式交互
- 可注入 Out 实现文件导出
- 通过 lambda 延迟执行消除对 Servlet API 的依赖

---

## 8. 链式 API 实现

框架广泛使用链式 API 模式，核心模式是 **返回 `this`**。

### Row.of() 静态工厂链
```java
Row.of("user").set("name", "james").id(123).update();
// Row.of("article", "article_id").set("title", "...").insert().getId();
```

### Db.sql() 链
```java
Db.sql("select * from user where id > ?", 5).find();
Db.sql("select * from article").paginate(1, 10);
```

### Settings 链
```java
settings.setServer(server, dispatcher)
        .addHandler(handler)
        .addGlobalInterceptor(myInterceptor);
```

### Routes 链
```java
routes.add("/path", Target.class, interceptor1)
      .setActionOverload(true);
```

### 实现模式
所有返回 `this` 的方法都在 `AifeiRow`, `AifeiDao`, `Settings`, `Batch`, `Routes`, `Plugins` 等类中。`AifeiDao` 使用 CRTP：`public D sql(...) { return (D) this; }`，确保子类链式调用返回子类类型。

---

## 9. Dispatcher / Server / Handler 核心流程

### 三层架构
```
Server (Undertow/NETTY/...)
    ↓ P1, P2 (如 HttpServerExchange, Void)
Dispatcher<P1, P2, I, O>
    ↓ I, O (如 Input, Output)
Handler<I, O> 责任链
    ↓
最终 Handler (路由派发 + 拦截器 + Action 执行)
```

### Server 接口
- **源码**: `aifei/src/main/java/cn/aifei/server/Server.java`
- `void init(Dispatcher<P1,P2,?,?> dispatcher)`
- `void start()`
- `void stop()`

### Dispatcher 接口
- **源码**: `aifei/src/main/java/cn/aifei/server/Dispatcher.java`
- **设计意图**: 解耦 Server 与 Handler，支持任意底层 Server 切换
- `void init(Handler<I,O> handler)`
- `void dispatch(P1 p1, P2 p2)` — 将 Server 请求转化为 I/O 后传递给 Handler

### Handler 责任链
- **源码**: `aifei/src/main/java/cn/aifei/core/Handler.java`
- **签名**: `public abstract class Handler<I, O>` 持有 `Handler<I, O> next`
- **方法**: `public abstract void handle(String path, I input, O output)`
- **构建**: `Aifei.makeHandlerChain()` 将 List 通过 `next` 指针串联

### Undertow 实现
- **UndertowServer**: `aifei-undertow/.../UndertowServer.java` — 实现 `Server<HttpServerExchange, Void>`
- **UndertowHandler**: `aifei-undertow/.../UndertowHandler.java` — 实现 Undertow 的 `HttpHandler`
  - 先处理静态资源（含 `.` 的路径）
  - IO 线程中短的请求切换到 Worker 线程
  - 调用 `startBlocking()` 启用阻塞模式
  - 调用 `dispatcher.dispatch(exchange, null)`
- **UndertowConfig**: 端口、SSL、Gzip、资源路径等配置

### 完整请求流程
```
Undertow → UndertowHandler.handleRequest()
         → dispatch(exchange, null)
         → Handler.handle(path, input, output)
         → Router.getAction(path, input)
         → 执行拦截器链 (Routes级 → 全局 → Class级 → Method级)
         → Action.method.invoke(target, args)
```

---

## 10. forEachPage 分批处理

- **定义**: `AifeiDao.java:644`
- **签名**: `public void forEachPage(int pageSize, Function<Page<R>, Boolean> fun)`
- **重载**: `forEachPage(startPageNum, endPageNum, pageSize, Function<Page<R>, Boolean> fun)`

### 实现 (PaginateExecutor.java:159-181)
```java
public <T extends AifeiRow<T>> void forEachPage(AifeiDao<?, T> dao, int pageSize, 
        Boolean hasGroupBy, Function<Page<T>, Boolean> fun) {
    forEachPage(dao, 1, Integer.MAX_VALUE, pageSize, hasGroupBy, fun);
}

public <T extends AifeiRow<T>> void forEachPage(AifeiDao<?, T> dao, int startPageNum, 
        int endPageNum, int pageSize, Boolean hasGroupBy, Function<Page<T>, Boolean> fun) {
    // 缓存 totalRows，只查一次数据库
    TotalRows totalRows = new TotalRows() {
        private long totalRows = -1;
        @Override public long get(SqlPara sqlPara, TotalRowsQuery totalRowsQuery) {
            if (totalRows == -1) { totalRows = totalRowsQuery.execute(); }
            return totalRows;
        }
    };
    for (int i = startPageNum; i <= endPageNum; i++) {
        Page<T> page = execute(dao, i, pageSize, hasGroupBy, totalRows);
        if (page.getRows().isEmpty() || !fun.apply(page)) break;
    }
}
```

### 设计要点
- **totalRows 只查一次**：通过 `TotalRows` 缓存机制，后续分页复用第一次查询的结果
- **可提前终止**：`fun.apply(page)` 返回 `false` 时停止遍历
- **应用场景**：多表关联查询后写入 Elasticsearch、Redisearch 等搜索引擎
- **用法**:
```java
User.sql("select * from user").forEachPage(10, page -> {
    for (User user : page.getRows()) {
        System.out.println(user.getName());
    }
    return true;
});
```

---

## 附录：关键文件路径汇总

| 组件 | 路径 |
|------|------|
| Aifei 启动入口 | `aifei/.../core/Aifei.java` |
| AifeiConfig | `aifei/.../config/AifeiConfig.java` |
| Settings | `aifei/.../config/Settings.java` |
| Routes | `aifei/.../config/Routes.java` |
| Plugins | `aifei/.../config/Plugins.java` |
| @Path | `aifei/.../core/Path.java` |
| @Para | `aifei/.../core/Para.java` |
| Input/Output | `aifei/.../core/Input.java`, `Output.java` |
| Handler | `aifei/.../core/Handler.java` |
| Dispatcher | `aifei/.../server/Dispatcher.java` |
| Server | `aifei/.../server/Server.java` |
| Router | `aifei/.../router/Router.java` |
| RouterKit | `aifei/.../router/RouterKit.java` |
| Action | `aifei/.../router/Action.java` |
| ActionGroup | `aifei/.../router/ActionGroup.java` |
| Argument | `aifei/.../argument/Argument.java` |
| ArgumentKit | `aifei/.../argument/ArgumentKit.java` |
| AifeiRow | `aifei-db/.../core/AifeiRow.java` |
| Row | `aifei-db/.../core/Row.java` |
| AifeiDao | `aifei-db/.../core/AifeiDao.java` |
| Dao | `aifei-db/.../core/Dao.java` |
| Db | `aifei-db/.../core/Db.java` |
| DbConfig | `aifei-db/.../core/DbConfig.java` |
| DbKit | `aifei-db/.../core/DbKit.java` |
| AifeiDb | `aifei-db/.../core/AifeiDb.java` |
| Batch | `aifei-db/.../core/Batch.java` |
| Transaction | `aifei-db/.../transaction/Transaction.java` |
| TransactionKit | `aifei-db/.../transaction/TransactionKit.java` |
| Atom | `aifei-db/.../transaction/Atom.java` |
| RollbackDecision | `aifei-db/.../transaction/RollbackDecision.java` |
| #where | `aifei-db/.../sql/WhereDirective.java` |
| #and | `aifei-db/.../sql/AndDirective.java` |
| #orderBy | `aifei-db/.../sql/OrderByDirective.java` |
| Condition | `aifei-db/.../sql/Condition.java` |
| PaginateExecutor | `aifei-db/.../executor/PaginateExecutor.java` |
| UndertowServer | `aifei-undertow/.../UndertowServer.java` |
| UndertowHandler | `aifei-undertow/.../UndertowHandler.java` |
