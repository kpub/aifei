**  Aifei 架构师深度解析  **

设计哲学 · 新理念解构 · 优秀代码范式 · 新手全通透

*Just Service. Only Aifei can do.*

### 写在前面 — 用什么眼光看 Aifei

*"当所有人都在讨论 AI 能不能写代码的时候，Aifei 在思考另一个问题：框架应该为 AI 写代码做什么准备。"*

Aifei 不是 JFinal 的下一个版本，也不是 Spring 的简化版。它是一次范式转换——从「服务人类开发者的认知习惯」转向「服务大模型的工作机制」。

理解 Aifei 需要先建立一个认知：大模型写代码不是「会打字的程序员」，而是一个有注意力预算、有上下文窗口限制、有生成模式偏好的信息处理系统。Aifei 的每一个设计决策，都是在为这个系统做优化。

> 🌱 **新手理解** — 你可以把大模型想象成一个聪明但「工作内存有限」的同事。你给他的背景信息越少越清晰，他能输出的内容就越准确。Aifei 就是在框架层面帮你压缩「不必要的背景信息」。

## 第一章  信息论视角下的框架设计

### 1.1 框架的本质是什么

在大多数开发者眼里，框架是「工具集合」——提供数据库、路由、模板等功能。但从更深的角度看，框架的本质是「认知负载的分配系统」。

框架决定了：开发者（或 AI）在写业务代码时，需要同时在脑子里（或上下文里）维持多少个概念？

> 🏛️ **架构洞见** — 架构洞见：框架中每增加一个概念（Controller、Mapper、DTO、VO…），就是在开发者和 AI 的工作内存中增加一个需要维护的状态。传统框架之所以复杂，根本原因不是功能多，而是概念多。

#### ◆ 信息论的极简原则

作者引用了一个深刻的判断原则：

*"当信息被消除掉所有冗余后，你如果再想去掉任何东西，它就无法被还原，造成信息量丢失，在业务代码上就形成 bug。"*

这不是比喻，这是设计框架时真实可用的工程判断标准：

- JFinal/Aifei 已经把要素减到了信息论意义上的最小值
- 再删任何一个概念，业务表达能力就会下降
- Spring 的复杂度大量来自「可以删除但没有删除」的冗余

**❌ 传统写法：框架概念噪音极高**

```java
// Spring Boot 表达「按 ID 查用户」需要的概念数量：
// 1. @RestController（告诉框架这是控制器）
// 2. @GetMapping（告诉框架这是 GET 请求）
// 3. @PathVariable 或 @RequestParam（告诉框架从哪取参数）
// 4. ResponseEntity（包装返回值）
// 5. UserService（不能直接访问数据库，要有服务层）
// 6. UserRepository（不能直接写 SQL，要有 Repository）
@GetMapping("/user/{id}")
public ResponseEntity<User> getUser(@PathVariable int id) {
    return ResponseEntity.ok(userService.findById(id));
}
// 上述代码表达的业务信息量 ≈ 「根据 id 查用户」
// 框架噪音占比：6个外部概念 / 1个业务含义 ≈ 85% 是框架噪音
```

**✅ Aifei 写法：代码即业务，无冗余噪音**

```java
// Aifei 表达「按 ID 查用户」需要的概念数量：
// 1. @Path（告诉框架这是服务入口）
// 2. public 方法（暴露接口）
// 就这两个。参数自动注入，返回值自动序列化。
@Path("/user")
public class UserService {
    public User getById(int id) {
        return User.findById(id);  // 业务语义 100% 清晰
    }
}
// 框架噪音占比：1个概念 / 1个业务含义 ≈ 50% 且完全必要
```

> ✅ **优秀设计** — 这是 Aifei 最核心的设计成就：让代码中信噪比趋近于 1:1。业务代码写多少，框架代码就写多少，没有额外的仪式性代码。

### 1.2 「注意力浓度」——为 AI 设计的新指标

Aifei README 提出了一个传统框架评估体系中从未存在的新指标：Attention 浓度（注意力浓度）。

> 🔭 **类比认知** — 类比：假设你要请一个专家帮你解决一个核心问题，你给他发了一份文件。如果文件里 90% 是公司简介、会议纪要、历史背景，只有 10% 是真正的问题描述，这个专家的精力就被稀释了。大模型面对充满框架样板代码的上下文，就是这种处境。

注意力浓度可以理解为：

*"在给定的上下文 Token 中，与核心业务逻辑直接相关的 Token 占比。"*

> 🏛️ **架构洞见** — 从框架设计角度看，注意力浓度不是锦上添花，而是 AI Coding 时代框架竞争力的核心指标。一个注意力浓度只有 13% 的框架，意味着大模型 87% 的精力花在了「理解框架规则」上，而不是「理解你的业务」。

## 第二章  Just Service — 范式转换的核心

### 2.1 分层架构的「历史债」

Controller → Service → Repository/Dao 的三层架构不是自然规律，它是特定历史时期的产物：

- 早期 Web 开发中，Controller 负责处理 HTTP 协议细节（Servlet API 极繁琐）
- 数据库操作不标准，Repository 层用来隔离 JDBC 的复杂性
- 业务逻辑复杂，Service 层用来做统一编排

但这些理由在今天已经大幅弱化：HTTP 处理可以由框架自动完成，SQL 已经足够表达力强，业务逻辑可以直接写在 Service 里。

> ⚠️ **认知陷阱** — 认知陷阱：很多开发者认为「三层架构是工程最佳实践」。这是历史性误判。三层架构是特定约束条件下的解决方案，当约束消失，方案也应该随之进化。Aifei 不是在反对工程规范，而是在重新审视哪些规范是真正必要的。

### 2.2 Just Service 的三个层次

Just Service 不只是「删掉 Controller」，它在三个层次上重塑了开发范式：

#### ◆ 层次一：结构层面 — 消除非业务性概念

**概念数量：传统 ~15 个  vs  Aifei ~4 个**

```java
// 传统框架的概念图谱（AI 需要理解并生成所有这些）：
//
// HTTP 请求
//   → Filter（过滤器）
//   → Handler/Interceptor（拦截器）
//   → DispatcherServlet（分发器）
//   → Controller（控制器）
//   → @RequestMapping / @GetMapping（路由映射）
//   → @RequestParam / @RequestBody（参数绑定）
//   → Service（业务层）
//   → Repository/Mapper（数据层）
//   → Entity/VO/DTO/PO（数据对象层）
//   → ResultWrapper（返回值包装）
//   → JsonSerializer（序列化）
//
// Aifei 的概念图谱：
//
// HTTP 请求
//   → Handler（处理器，框架级，一次配置）
//   → @Path Service（业务类，AI 只需关注这里）
//   → Row/Model（数据对象，无多重形态）
//   → Out（统一返回）
```

#### ◆ 层次二：认知层面 — 消除多范式并存

传统框架中，开发者（和 AI）需要同时掌握多套数据对象范式：

#### ◆ 层次三：生成层面 — 提供稳定的代码模式

Just Service 给 AI 提供了一个「固定模板」，使代码生成具有高度可预测性：

**Just Service 代码模板 — AI 只需填入 [] 括号内的业务内容**

```java
// Aifei Service 的固定代码模式（AI 只需填入业务部分）：
```

```java
@Path("/[资源名]")                      // ← 固定，填资源名
public class [资源]Service {
```

```java
    // 列表/分页查询（固定模式）
    public Page<[Model]> index(Map<?,?> filter, int pageNum, int pageSize) {
        String sql = "select * from [表] #where(...) #orderBy(...)";  // ← 填业务
        return [Model].sql(sql, filter).paginate(pageNum, pageSize);
    }
```

```java
    // 查详情（固定模式）
    public [Model] detail(int id) {
        return [Model].findById(id);                                    // ← 填业务
    }
```

```java
    // 插入（固定模式）
    public Out insert([Model] model) {
        model.insert();                                                 // ← 填业务
        return Out.ok();
    }
```

```java
    // 更新（固定模式）
    public Out update([Model] model) {
        model.update();                                                 // ← 填业务
        return Out.ok();
    }
```

```java
    // 删除（固定模式）
    public Out delete(int id) {
        [Model].deleteById(id);                                         // ← 填业务
        return Out.ok();
    }
}
```

> ✅ **优秀设计** — 这个模式的精妙之处：结构固定、变化点清晰、AI 每次生成时只需关注「[资源名]」「[Model]」「[表]」「[业务条件]」四个变量，其余全是不变量，生成准确率极高。

## 第三章  数据库设计的五个突破

### 3.1 突破一：链式 sql() 作为统一查询入口

传统 JFinal 的查询 API 存在「多入口」问题：

**❌ JFinal 多入口问题**

```java
// JFinal 的多入口问题（AI 需要选择正确的入口）：
User.dao.find(sql, paras);            // 入口 1：Model 的 dao 对象
User.dao.findFirst(sql, paras);       // 入口 2：同上
User.dao.paginate(p, s, sel, from);   // 入口 3：注意 select 和 from 要分开！
User.dao.template("key", paras);     // 入口 4：SQL 模板方式
Db.find(sql, paras);                  // 入口 5：Db 工具类
Db.findFirst(sql, paras);             // 入口 6：同上
Db.paginate(p, s, sel, from);         // 入口 7：注意 select 和 from 要分开！
Db.getSqlPara("key", paras);         // 入口 8：先拿 SqlPara 再查
// AI 在生成代码时，经常在这 8 种入口之间犹豫并出错
```

**✅ Aifei 统一入口：sql() → 链式执行方法**

```java
// Aifei 的统一入口：sql() 链式调用
// 所有查询都从 sql() 开始，链式决定执行方式
```

```java
Db.sql(sql, params).find();           // 统一入口，链式选择执行方式
Db.sql(sql, params).findFirst();
Db.sql(sql, params).findOne();
Db.sql(sql, params).paginate(p, s);
Db.sql(sql, params).queryLong();
Db.sql(sql, params).update();
```

```java
// Model 版本同样统一：
User.sql(sql, params).find();
User.sql(sql, params).paginate(p, s);
User.sql(sql, params).forEachPage(10, page -> { ... });
```

> 💎 **设计本质** — 设计本质：这不只是 API 风格变化，而是「减少分叉」。AI 在生成代码时，分叉越少，出错概率越低。统一入口让 AI 只需学会一个模式：想查数据，先 sql()，然后链式选择执行方式。

### 3.2 突破二：Model is-a Row（继承而非并列）

这是 Aifei 数据模型最优雅的设计决策之一，值得深入理解。

#### ◆ JFinal 的问题：Model 和 Record 是并列的两套体系

**❌ JFinal：Model 和 Record 是并列的两套体系**

```java
// JFinal 中 Model 和 Record 是完全不同的类型：
User user = User.dao.findById(1);       // 返回 Model<User>
Record r  = Db.findFirst(sql);          // 返回 Record（完全不同的类型！）
```

```java
// 两套 API，两种取值方式：
user.getStr("name");    // Model 取值
r.getStr("name");       // Record 取值（看起来一样，但类型不同）
```

```java
// 问题：当你需要对结果做统一处理时，需要写两套代码
// AI 在生成处理逻辑时，不知道该用 Model 还是 Record
```

**✅ Aifei：Model is-a Row，类型统一**

```java
// Aifei 中 Model is-a Row（Model 继承自 Row）：
public class User extends AifeiRow<User> { ... }
//                              ↑ User 就是 Row，Row 就是 User
```

```java
// 统一类型体系：
User user = User.findById(1);           // 返回 User（也是 Row）
Row  row  = Db.sql(sql).findFirst();    // 返回 Row
```

```java
// 统一 API（因为 User 就是 Row）：
user.getName();                          // 或 user.get("name")
row.get("name");
```

```java
// 统一处理（可以用同一套逻辑处理 User 和 Row）：
void process(Row row) {                  // 接收 Row，User 也可以传入
    String name = row.get("name");
    // ...
}
```

> 🏛️ **架构洞见** — 架构洞见：is-a 关系（继承）比 has-a 关系（组合）在这里更优雅的原因：业务代码不需要类型转换，方法签名可以统一声明为 Row，自然接受所有 Model 子类，多态生效。这是面向对象设计原则的完美应用。

### 3.3 突破三：of() 工厂方法消灭「先查后改」

这是一个小设计，但体现了极深的用户体验思考。

**of() 工厂方法 vs 传统写法**

```java
// 场景：修改 id=123 的用户姓名
```

```java
// Spring JPA 写法（先查再改，两次数据库操作）：
User user = userRepository.findById(123).orElseThrow();
user.setName("james");
userRepository.save(user);
// 问题：为了改一个字段，先发了一条 SELECT，产生不必要的 IO
```

```java
// JFinal 写法（需要先 new 一个 Model，手动设置主键）：
new User().set("id", 123).set("name", "james").update();
// 稍好，但 set("id", 123) 的写法不直观，容易忘
```

```java
// Aifei 写法：
User.of(123).name("james").update();
// of(123) = 创建持有 id=123 的 User 实例（不查库）
// name("james") = 设置 name 字段（流式 setter）
// update() = 只更新被 set 的字段
```

> ✅ **优秀设计** — 优秀设计点：of(id) 方法在不查数据库的情况下创建一个「持有主键」的对象实例，update() 时框架知道这是 WHERE id=? 的更新。一行代码，零多余 SQL，语义完全清晰。

### 3.4 突破四：#where / #and / #orderBy 结构化动态 SQL

这是 Aifei 在 SQL 管理上最具革命性的创新，值得用较多篇幅分析。

#### ◆ 为什么动态 SQL 是 AI 的噩梦

动态 SQL 的本质问题是：SQL 语句的形状在运行时才确定。传统的解决方案是「模板分支」，但这对 AI 生成极不友好：

**❌ 传统动态 SQL：分支多、易出错、有注入风险**

```java
// 传统动态 SQL（JFinal/MyBatis 风格）：
String sql = "select * from user where 1=1";
```

```java
// AI 需要生成这样的 Enjoy 模板（有多种写法，容易出错）：
#sql("findUser")
  select * from user where 1=1
  #if(name)                        ← 容易漏写
    and name like concat('%', #para(name), '%')
  #end                             ← 容易漏写 #end
  #if(status != null)
    and status = #para(status)
  #end
  #if(minAge != null)
    and age >= #para(minAge)
  #end
  #if(orderField != null && !orderField.isEmpty())
    order by #(orderField) #(orderDir)  ← SQL注入风险！orderField 直接拼！
  #end
#end
// 问题：
// 1. 每个条件都要 #if/#end 包裹，AI 容易漏写 #end
// 2. 排序字段直接拼字符串有 SQL 注入风险
// 3. 生成结构不稳定，不同 AI 可能产生不同风格
```

**✅ Aifei #where / #and / #orderBy：结构化，零风险**

```java
// Aifei 新动态 SQL 指令：
// 格式：#where(字段, 操作符, 变量名)  -- 变量为 null 时自动忽略
//       #and(字段, 操作符, 变量名)    -- 同上
//       #orderBy(白名单字段列表)       -- 从 filter 中读取排序意图
```

```java
// 同样的查询需求，Aifei 写法：
String sql = "select * from user" +
    " #where(name, 'like', name)" +
    " #and(status, '=', status)" +
    " #and(age, '>=', minAge)" +
    " #orderBy(created, id, name)";  // 括号内是白名单，只允许这三个字段排序
```

```java
// Java 调用（传 filter Map，null 值自动过滤）：
return User.sql(sql, filter).paginate(pageNum, pageSize);
```

```java
// 前端传 filter：{ name: 'james', status: 1 }  // minAge 不传
// 生成 SQL：select * from user where name like ? and status = ?
// 前端传 filter：{ orderBy: { field: 'created', order: 'desc' } }
// 生成 SQL：... order by created desc  （因为 created 在白名单中）
```

#### ◆ 三大指令的设计精髓

> 💎 **设计本质** — 设计本质：这三个指令把动态 SQL 从「命令式（你告诉我什么时候加条件）」变成了「声明式（你告诉我条件规则，我自动处理）」。声明式代码对 AI 更友好，因为意图清晰，没有隐藏的状态转换。

### 3.5 突破五：批量操作的自动分组

这个功能解决了一个在实际开发中频繁出现但长期被忽视的痛点：批量写入时各行字段不一致。

**✅ Aifei 批量操作：自动分组，字段结构不一致也完美处理**

```java
// 典型场景：从 Excel 导入用户数据，不同行填了不同字段
// 第1行：姓名 + 手机号
// 第2行：姓名 + 邮箱 + 年龄
// 第3行：姓名 + 手机号 + 部门
```

```java
// 传统解决方案（非常繁琐）：
// 1. 先按字段组合分组
// 2. 每组单独构造 PreparedStatement
// 3. 每组单独执行批量
// 或者：每行单独 INSERT（性能极差）
```

```java
// Aifei 解决方案：
List<Row> rows = new ArrayList<>();
rows.add(new Row().set("name","Alice").set("phone","13800000001"));
rows.add(new Row().set("name","Bob").set("email","bob@test.com").set("age",25));
rows.add(new Row().set("name","Carol").set("phone","13800000003").set("dept","研发"));
```

```java
Db.batch().insert("user", rows);  // 框架自动按字段组合分组，分组执行批量插入
// Aifei 内部自动：
// 组1（name+phone）：Alice, Carol → 一次 batch
// 组2（name+email+age）：Bob      → 一次 batch
```

> 🏛️ **架构洞见** — 这个功能的背后是 BatchUpdateExecutor 的设计：它先按字段集合 groupByFields()，然后在事务中按分组执行批量操作。这个设计让批量操作从「需要预处理」变成了「直接丢进去就行」，符合信息论原则——不要让使用者维护本可以自动推导的状态。

## 第四章  AifeiConfig — 集中式配置的哲学

### 4.1 分散式配置 vs 集中式配置

Spring Boot 的配置哲学是「约定优于配置」——尽量不让用户配置，用约定代替。这在小项目中很方便，但在大模型生成代码时是个严重问题。

#### ◆ 约定的副作用：隐式行为

**❌ 约定优于配置：隐式行为对 AI 不友好**

```java
// Spring Boot 的「隐式行为」示例：
// 你加了 spring-boot-starter-data-jpa 依赖，
// 框架自动扫描所有 @Entity 类，
// 自动创建表（如果 ddl-auto=create），
// 自动配置 EntityManager，
// 自动开启事务管理，
// 这些行为都是「隐式」的，在代码里看不见
```

```java
// 对 AI 的影响：
// AI 生成代码时，不知道这些隐式行为的存在
// 生成的代码可能与隐式行为冲突（如手动配置了 DataSource）
// AI 无法通过读代码来理解完整的系统行为
```

**✅ Aifei 显式配置：系统行为完全可见**

```java
// Aifei 的「显式配置」：所有配置都在 AifeiConfig 里，一览无余
public class AppConfig implements AifeiConfig<In, Out> {
```

```java
    public void config(Settings<In, Out> settings) {
        // 服务器：显式指定，一目了然
        settings.setServer(new UndertowServer(), new IoDispatcher());
        // 处理器：显式添加，执行顺序清晰
        settings.addHandler(new AuthHandler());
        settings.addHandler(new LogHandler());
        // 参数注入：显式注册，没有魔法
        settings.configArgument(kit -> kit.register(Account.class, LoginArgument.class));
    }
```

```java
    public void config(Routes routes) {
        // 路由：显式扫描哪个包，带哪个拦截器
        routes.scan("cn.myapp.api", new AuthInterceptor());
    }
```

```java
    public void config(Plugins plugins) {
        // 数据库：显式配置，参数清晰
        AifeiDbPlugin db = new AifeiDbPlugin("main", new DruidSupplier(...));
        db.addModelSet(new ModelSet());
        plugins.add(db);
    }
}
// AI 读这段代码，能完全理解系统做了什么。没有任何隐式行为。
```

> ✅ **优秀设计** — AifeiConfig 的三个 config 重载方法体现了「一个入口，参数类型即语义」的设计哲学：config(Settings) 配系统级设置，config(Routes) 配路由，config(Plugins) 配插件。参数类型本身就是文档。

### 4.2 HIO 模型 — 处理流程的极简化

Aifei 的 HIO（Handler-Input-Output）模型是对 Servlet 生命周期的重新抽象：

> 🌱 **新手理解** — 新手理解：你可以把 HIO 理解为一条「加工流水线」。Input 是原材料（HTTP 请求），Handler 是一道道工序（认证、日志等），Dispatcher 是工厂调度员（找到正确的 Service 方法），Output 是成品（HTTP 响应）。这条流水线的每一段都是显式配置的，没有隐藏工序。

## 第五章  优秀代码范式精讲

### 5.1 范式一：链式 API 的极致运用

Aifei 全面采用链式 API，这不仅仅是风格偏好，而是减少临时变量、降低认知负载的工程决策。

#### ◆ 设计原则：每一步都有意义，每一步都不多余

**链式 API 三要素示例**

```java
// 链式 API 的设计三要素：
// 1. 每个方法返回 this 或下一阶段对象（使链式成为可能）
// 2. 方法名即操作语义（name(v) 比 set("name",v) 更直接）
// 3. 终结方法表明操作类型（find/insert/update/delete）
```

```java
// 示例一：Model 的链式构建与操作
new User()                   // 创建实例
    .name("james")           // 设置字段（方法名 = 字段名）
    .age(25)                  // 设置字段
    .email("j@test.com")     // 设置字段
    .insert();               // 终结：插入数据库
```

```java
// 示例二：查询链
User.sql("select * from user #where(status,'=',status)", filter)
    .paginate(pageNum, pageSize);  // 终结：分页查询
```

```java
// 示例三：修改链（of 工厂 + 链式设置 + 终结）
User.of(123)                // 工厂：持有 id=123 的实例
    .name("newName")         // 设置要更新的字段
    .status(2)               // 设置要更新的字段
    .update();              // 终结：只更新被设置的字段
```

> 💎 **设计本质** — 链式 API 的深层价值：代码的阅读顺序与执行顺序一致（从左到右，从上到下），零跳跃理解成本。这对于 AI 生成代码尤其重要：AI 可以按自然语言描述的顺序直接生成链式代码。

### 5.2 范式二：统一返回对象 Out

Out 是 Aifei 中所有 Service 方法「操作类」接口的统一返回类型（查询方法返回具体数据，操作方法返回 Out）。

**Out 统一返回对象**

```java
// Out 的设计理念：统一成功/失败表达
```

```java
// 静态工厂方法，语义极清晰：
return Out.ok();                  // 成功，无消息
return Out.ok("操作成功");        // 成功，带消息
return Out.ok(user);              // 成功，带数据
return Out.fail("用户不存在");    // 失败，带原因
return Out.fail("余额不足", 402); // 失败，带错误码
```

```java
// 与 Spring 的对比：
// Spring：return ResponseEntity.ok().body(data)  // 冗长
// Spring：return ResponseEntity.status(400).body(error) // 更冗长
// Aifei：return Out.ok(data)  /  return Out.fail(msg) // 简洁
```

#### ◆ 在事务中返回 Out 的妙用

**Out + 事务的完美结合**

```java
// 传统写法：事务和返回值分离，需要额外变量
Out result;
Db.tx(() -> {
    // ... 业务逻辑 ...
    result = n == 1 ? Out.ok("成功") : Out.fail("失败");  // 无法在 Lambda 内赋值外部变量
    return result.isOk();  // JFinal 返回 boolean
});
return result;  // 还需要额外返回
```

```java
// Aifei 写法：事务直接返回业务结果
return Db.transaction(tx -> {
    int n1 = Db.sql("update account set money=money-? where id=?", money, fromId).update();
    int n2 = Db.sql("update account set money=money+? where id=?", money, toId).update();
    return n1==1 && n2==1
        ? Out.ok("转账成功")
        : Out.fail("转账失败");  // 失败时框架自动回滚（Out.fail 触发回滚机制）
});
// 事务和业务结果完美合并，无多余变量，代码意图 100% 清晰
```

> ✅ **优秀设计** — Out 在事务中的设计尤为精彩：transaction() 的 Lambda 直接返回 Out，框架根据 Out.isOk() 决定提交或回滚。业务逻辑和事务控制完全融合，零冗余代码。

### 5.3 范式三：参数自动注入的分层设计

Aifei 的参数注入系统是「可扩展的固定模式」的典范设计：

**参数注入的四层架构**

```java
// 参数注入的分层结构：
```

```java
// 层次1：基础类型（框架内置，无需配置）
public void someAction(int id, String name, long timestamp) { ... }
// → 直接从 URL 参数注入，零配置
```

```java
// 层次2：Map（框架内置，接收所有参数）
public void search(Map<?,?> filter) { ... }
// → 所有 URL/表单参数合并为 Map，零配置
```

```java
// 层次3：Model/Row（框架内置，JSON 反序列化）
public Out save(User user) { ... }
// → 请求体 JSON 自动反序列化为 User，零配置
```

```java
// 层次4：自定义类型（用户扩展，一次注册处处可用）
// 注册阶段（只写一次）：
settings.configArgument(kit -> {
    kit.register(Account.class, LoginAccountArgument.class);
    kit.register(Tenant.class, TenantArgument.class);
});
// 使用阶段（任意 Service 方法直接声明）：
public Page<Order> myOrders(Account account, Tenant tenant, Map<?,?> filter) {
    String sql = "select * from order where user_id=? and tenant_id=?";
    return Order.sql(sql, account.getId(), tenant.getId()).paginate(1, 20);
}
```

> 🏛️ **架构洞见** — 这个设计的精妙：四个层次从简单到复杂，每一层都有明确的职责，扩展点清晰。新手用层次1-3就够了，高级用法在层次4，但接口一致。这是「渐进式复杂度」的体现。

### 5.4 范式四：forEachPage 大数据分批处理

这个 API 设计展示了 Aifei 在「大数据量场景」下的用心：

**forEachPage：大数据量分批处理的最优解**

```java
// 问题场景：对全表用户发送邮件通知
```

```java
// ❌ 危险写法：全量查询
List<User> allUsers = User.sql("select * from user").find();
for (User u : allUsers) { sendEmail(u); }
// 问题：100万用户 = OOM（内存溢出）
```

```java
// ❌ 传统分页写法：繁琐
int page = 1, size = 100;
while (true) {
    Page<User> p = User.sql("select * from user").paginate(page, size);
    for (User u : p.getRows()) { sendEmail(u); }
    if (page >= p.getTotalPage()) break;
    page++;
}
```

```java
// ✅ Aifei forEachPage：优雅分批
User.sql("select * from user").forEachPage(100, (Page<User> page) -> {
    for (User u : page.getRows()) {
        sendEmail(u);
    }
    return true;   // 继续下一页
    // return false; // 提前终止
});
// 自动处理分页逻辑，内存安全，可提前终止，代码极简
```

> ✅ **优秀设计** — forEachPage 的设计亮点：Lambda 返回 boolean 控制是否继续迭代，这让「找到目标即停止」的场景变得优雅。同时框架内部使用游标式分页，每次只加载一批数据到内存，天然内存安全。

## 第六章  Aifei 的设计哲学总结

### 6.1 七个核心设计原则

#### ◆ 原则一：代码即文档

好的代码不需要注释就能理解。Aifei 的 API 命名遵循「读代码 = 读业务描述」的原则：

```java
User.of(123).name("james").update();  // 直读：把 id=123 的用户名字改为 james
Db.batch().insert("user", rows);      // 直读：批量插入 rows 到 user 表
Out.ok("操作成功");                   // 直读：返回成功，消息是「操作成功」
```

#### ◆ 原则二：单一入口，链式分叉

每种操作只有一个入口，执行方式通过链式调用在末端分叉，而不是在入口处分叉：

```java
// 所有查询，唯一入口：sql()
Db.sql(sql, params).find()          // 末端分叉：查多条
Db.sql(sql, params).findFirst()     // 末端分叉：查第一条
Db.sql(sql, params).paginate(p,s)   // 末端分叉：分页
Db.sql(sql, params).queryLong()     // 末端分叉：查单值
// 学一个入口，通过末端方法名选择行为，零歧义
```

#### ◆ 原则三：显式优于隐式

所有系统行为都在代码中可见，没有约定魔法，没有注解黑盒，没有自动配置惊喜。

#### ◆ 原则四：渐进式复杂度

简单场景用简单 API，复杂场景才需要复杂配置，简单场景绝不被复杂场景的存在所污染：

```java
// 简单场景（新手）：
User.findById(1);
```

```java
// 中等场景：
User.sql("select * from user #where(name,'=',name)", filter).find();
```

```java
// 复杂场景（高级）：
User.sql(sql, filter).forEachPage(100, page -> { ...; return true; });
// 每层复杂度独立，新手不需要了解高级用法才能入门
```

#### ◆ 原则五：信息无冗余

每一行代码都承载业务信息，不存在仅用于满足框架要求的样板代码。测量标准：把这行代码删掉，业务信息会丢失吗？会 → 保留；不会 → 删除。

#### ◆ 原则六：类型即约束

用类型系统表达约束，而不是用文档或约定。`#orderBy(created, id)` 的白名单就是类型约束的体现——不在白名单里的字段在类型层面就被排除了。

#### ◆ 原则七：为下一个人（或 AI）设计

代码的最终读者不是写代码的人，而是维护代码的人（人或 AI）。Aifei 的每个设计都在问：「这段代码，AI（或下一个开发者）能一眼看懂吗？」

### 6.2 Aifei 的哲学定位

*"极简不是目标，而是结果。当你把所有不必要的东西都去掉之后，剩下的就是极简的。"*

Aifei 的极简不是「功能少」，而是「概念少」。它功能完整，覆盖了 JFinal 的所有数据库能力，但概念数量减少了 60% 以上。

这就是架构设计中「最难的部分」：不是加功能，而是在加功能的同时不增加概念。Aifei 做到了这一点，这是它最值得学习和尊重的地方。

**  Just Service. Only Aifei can do.  **

*极简是结果，不是目标。3333 行代码，承载 15 年极简设计认知。*
