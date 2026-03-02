# Zexa 语言设计规范

> *Zexa* — 渐进精确、声明并发、恐慌隔离、类型演化，编译到 JavaScript 的实用语言。

---

## 0 设计哲学

```
类型可以生长，不必一步到位。
依赖决定顺序，声明胜过命令。
隔离恐慌，容错内建。
版本演化，迁移自动。
```

四条保留特性：**渐进类型（draft/refine/final）**、**声明式配方（recipe）**、**恐慌隔离（isolate）**、**类型版本演化（evolving）**。

---

## 1 基础语法

语法尽量贴近 JavaScript，降低学习成本。使用 `let`/`const`、花括号、箭头函数等 JS 开发者熟悉的形式。

```zexa
// 类型定义 —— 代数类型
type Option<T> = None | Some(T)

type User = {
  name: String,
  age: Int,
  email: String,
}

// 函数定义
let greet = (user: User): String => {
  return "Hello, " + user.name
}

// 模式匹配
let describe = (opt: Option<Int>): String => match opt {
  None    => "nothing",
  Some(n) => "got " + show(n),
}

// 管道
let result = data |> parse |> validate |> transform

// Lambda
let add = (x, y) => x + y

// 列表推导
let evens = [x for x in 1..100 if x % 2 == 0]

// 解构
let { name, age } = user
let [first, ...rest] = items
```

### 类型系统

- 基于 Hindley-Milner 推断，大多数类型签名可省略
- 泛型采用 `<T>` 语法
- 行多态支持灵活的 Record 类型
- 代数效果通过 `IO`、`Async` 等标记

```zexa
// 泛型
let identity = <T>(x: T): T => x

// 行多态
type HasName<R> = { name: String, ...R }

let getName = <R>(obj: HasName<R>): String => obj.name

// 效果标注
let readFile = (path: String): IO<String> => { /* ... */ }
let pureCalc = (x: Int): Int => x * 2  // 无标注 = 纯函数
```

---

## 2 `draft` / `refine` / `final` — 渐进类型

> 类型可以从草稿逐步完善，不必一开始就完整。

### 草稿类型

```zexa
// 草稿类型：... 表示还有未定义的字段
draft type Config = {
  host: String,
  port: Int,
  ...
}

// 已知字段正常使用
let url = (config: Config): String => config.host + ":" + show(config.port)  // ✓

// 访问未定义字段：编译警告（不报错，编译通过）
let x = config.timeout  // ⚠ timeout 未在 Config 中定义 (draft)
                         // 推断为 Unknown 类型
```

### 逐步细化

```zexa
// 补充字段（只允许添加，不允许删除或改变已有字段类型）
refine type Config = {
  host: String,
  port: Int,
  timeout: Duration,
  ...   // 仍然开放
}

// 此时 config.timeout 不再警告

// 定稿：关闭类型
final type Config = {
  host: String,
  port: Int,
  timeout: Duration,
  tls: Bool,
}
// 此后访问不存在的字段为编译错误 ✗
```

### 草稿函数

```zexa
draft let processOrder = (order: Order): Result => {
  let validated = validate(order)
  todo("应用折扣")        // 占位符，运行时抛 TodoError
  todo("生成发票")
  return Ok(validated)
}

// todo 是内建关键字，接受描述字符串
// 编译器收集所有 todo 并在构建时报告
```

### 草稿传播规则

```zexa
// draft 有传染性：依赖 draft 的函数/模块也必须标记 draft
final let main = (): IO<()> => {   // ✗ 编译错误：依赖草稿函数 processOrder
  processOrder(someOrder)
}

draft let main = (): IO<()> => {   // ✓
  processOrder(someOrder)
}
```

### 编译到 JS

```zexa
draft type Config = { host: String, port: Int, ... }
```

编译为：

```javascript
// Config 运行时表示为普通对象
// draft 字段访问生成带警告的 Proxy（开发模式）或直接属性访问（生产模式）

// 开发模式
function __draftAccess(obj, typeName, knownFields) {
  return new Proxy(obj, {
    get(target, prop) {
      if (typeof prop === 'string' && !knownFields.has(prop)) {
        console.warn(`[Zexa] Accessing unknown field '${prop}' on draft type '${typeName}'`);
      }
      return target[prop];
    }
  });
}

// 生产模式：零开销，直接属性访问
```

`todo()` 编译为：

```javascript
function todo(description) {
  throw new TodoError(description);
}
```

---

## 3 `recipe` — 声明式配方

> 声明依赖而非顺序，编译器自动推导执行顺序和并行机会。

```zexa
recipe let buildReport = (userId: UserId): Async<Report> => {
  let user     = fetchUser(userId)       // 独立
  let orders   = fetchOrders(userId)     // 独立
  let reviews  = fetchReviews(userId)    // 独立
  let profile  = computeProfile(user)    // 依赖 user
  let stats    = computeStats(orders)    // 依赖 orders
  let summary  = summarize(profile, stats, reviews) // 依赖 profile, stats, reviews
  return Report({ user, profile, stats, summary })
}

// 编译器推导的执行图：
//   fetchUser ──→ computeProfile ──┐
//   fetchOrders ─→ computeStats ───┼→ summarize → Report
//   fetchReviews ──────────────────┘
// 前三个自动并行
```

### 规则

- `recipe` 块内的 `let` 绑定**不隐含顺序**
- 编译器分析变量引用关系，构建依赖 DAG
- 无依赖关系的节点自动并发（`Promise.all`）
- 有依赖的节点按拓扑序串行

### 配合 `isolate` 做容错

```zexa
recipe let resilientReport = (userId: UserId): Async<Report> => {
  let user    = isolate { fetchUser(userId) }      // Result<User, Panic>
  let orders  = isolate { fetchOrders(userId) }    // Result<List<Order>, Panic>
  let reviews = isolate { fetchReviews(userId) }   // Result<List<Review>, Panic>
  return buildFromAvailable(user, orders, reviews)
}
```

### 编译到 JS

```zexa
recipe let buildReport = (userId: UserId): Async<Report> => {
  let user     = fetchUser(userId)
  let orders   = fetchOrders(userId)
  let reviews  = fetchReviews(userId)
  let profile  = computeProfile(user)
  let stats    = computeStats(orders)
  let summary  = summarize(profile, stats, reviews)
  return Report({ user, profile, stats, summary })
}
```

编译为：

```javascript
async function buildReport(userId) {
  // Layer 0: 无依赖 — 并行
  const [user, orders, reviews] = await Promise.all([
    fetchUser(userId),
    fetchOrders(userId),
    fetchReviews(userId),
  ]);

  // Layer 1: 依赖 layer 0 — 并行
  const [profile, stats] = await Promise.all([
    computeProfile(user),
    computeStats(orders),
  ]);

  // Layer 2: 依赖 layer 1
  const summary = await summarize(profile, stats, reviews);

  return Report({ user, profile, stats, summary });
}
```

编译器按拓扑层分组，同层用 `Promise.all`，层间 `await`。

---

## 4 `isolate` — 恐慌隔离域

> 任何表达式可以在隔离域中求值，异常不传播，返回 `Result`。

```zexa
let result = isolate {
  let x = mightPanic()
  let y = alsoRisky(x)
  x + y
}  // : Result<Int, Panic>

// 使用结果
match result {
  Ok(value)   => console.log("Got: " + show(value)),
  Err(panic)  => console.log("Failed: " + panic.message),
}
```

### 带选项

```zexa
// 超时
let result = isolate({ timeout: 5000 }) {
  potentiallyInfinite()
}  // : Result<Value, Panic | Timeout>

// 嵌套隔离
let outer = isolate {
  let a = safeWork()
  let b = isolate { riskyWork() }  // 内层失败不影响外层
  combine(a, b)
}
```

### 与 `recipe` 组合

```zexa
recipe let loadDashboard = (userId: UserId): Async<Dashboard> => {
  let profile  = isolate { fetchProfile(userId) }
  let activity = isolate { fetchActivity(userId) }
  let badges   = isolate { fetchBadges(userId) }

  return Dashboard({
    profile:  profile  |> unwrapOr(defaultProfile),
    activity: activity |> unwrapOr([]),
    badges:   badges   |> unwrapOr([]),
  })
}
// 任何一个服务挂掉，其他的仍然正常返回
```

### 编译到 JS

```zexa
let result = isolate {
  let x = mightPanic()
  let y = alsoRisky(x)
  x + y
}
```

编译为：

```javascript
const result = (() => {
  try {
    const x = mightPanic();
    const y = alsoRisky(x);
    return { tag: "Ok", value: x + y };
  } catch (e) {
    return { tag: "Err", value: { tag: "Panic", message: e.message, stack: e.stack } };
  }
})();
```

带超时版本：

```javascript
const result = await (async () => {
  try {
    const value = await Promise.race([
      (async () => potentiallyInfinite())(),
      new Promise((_, reject) =>
        setTimeout(() => reject({ tag: "Timeout" }), 5000)
      ),
    ]);
    return { tag: "Ok", value };
  } catch (e) {
    if (e.tag === "Timeout") return { tag: "Err", value: e };
    return { tag: "Err", value: { tag: "Panic", message: e.message, stack: e.stack } };
  }
})();
```

---

## 5 `evolving` — 类型版本演化

> 类型可以声明版本演化历史，编译器自动生成迁移链。

```zexa
evolving type UserProfile {
  v1 = {
    name: String,
    email: String,
  }

  v2 = {
    name: String,
    email: String,
    avatar: Url,
  }
  migrate(v1 => v2) = (old) => ({ ...old, avatar: defaultAvatar })

  v3 = {
    firstName: String,
    lastName: String,
    email: String,
    avatar: Url,
  }
  migrate(v2 => v3) = (old) => ({
    firstName: old.name |> split(" ") |> head,
    lastName:  old.name |> split(" ") |> last,
    email:     old.email,
    avatar:    old.avatar,
  })
}
```

### 使用

```zexa
// 显式迁移
let current: UserProfile@v3 = oldData |> migrateTo(@v3)

// 序列化自带版本号
let json = serialize(profile)
// => { "_v": 3, "firstName": "Alice", "lastName": "Smith", ... }

// 反序列化自动迁移到最新版本
let loaded: UserProfile@v3 = deserialize(json)
// 如果 json 是 v1，自动执行 v1→v2→v3 链式迁移
```

### 链式迁移

```zexa
// 编译器自动组合迁移函数
// migrateTo(@v3) 对 v1 数据 = migrate(v2=>v3) ∘ migrate(v1=>v2)
// 对 v2 数据 = migrate(v2=>v3)
// 对 v3 数据 = identity
```

### 编译规则

- 每个版本的迁移函数必须提供（除 v1 外）
- `migrate(vN => vN+1)` 只允许相邻版本
- 编译器自动组合非相邻版本的迁移链
- 类型引用默认指向最新版本
- `@v1`、`@v2` 语法显式引用历史版本

### 编译到 JS

```zexa
evolving type UserProfile {
  v1 = { name: String, email: String }
  v2 = { name: String, email: String, avatar: Url }
  migrate(v1 => v2) = (old) => ({ ...old, avatar: defaultAvatar })
}
```

编译为：

```javascript
const UserProfile = {
  currentVersion: 2,

  migrations: {
    "1_to_2": (old) => ({ ...old, avatar: defaultAvatar }),
  },

  migrateTo(data, targetVersion) {
    let v = data._v || 1;
    let current = { ...data };
    while (v < targetVersion) {
      const key = `${v}_to_${v + 1}`;
      const fn = this.migrations[key];
      if (!fn) throw new Error(`No migration from v${v} to v${v+1}`);
      current = fn(current);
      v++;
    }
    current._v = targetVersion;
    return current;
  },

  serialize(obj) {
    return JSON.stringify({ _v: this.currentVersion, ...obj });
  },

  deserialize(json) {
    const data = JSON.parse(json);
    return this.migrateTo(data, this.currentVersion);
  },
};
```

---

## 6 类型系统总览

```zexa
// 基础代数类型
type Bool = True | False
type Option<T> = None | Some(T)
type Result<E, T> = Err(E) | Ok(T)

// 行多态
type HasName<R> = { name: String, ...R }

// 效果标注
type IO<T>    // 同步副作用
type Async<T> // 异步

// isolate 相关
type Panic = { message: String, stack: String }
type Timeout = { duration: Int }

// draft 相关
type Unknown  // 草稿类型中未知字段的类型

// 核心 trait
trait Show<T> {
  let show: (T) => String
}

trait Eq<T> {
  let eq: (T, T) => Bool
}
```

---

## 7 模块系统

```zexa
module MyApp.Users exposing { User, createUser, findUser }

import Data.List { map, filter }
import Network.HTTP { get, post }
import Shop.Types { Order, Customer }

// 草稿模块
draft module MyApp.Billing exposing { * }
```

### 模块编译到 JS

```zexa
module MyApp.Users exposing { User, createUser }
```

编译为 ES Modules：

```javascript
// MyApp/Users.js
export function createUser(name, email) { /* ... */ }
export const User = { /* type metadata */ };
```

导入：

```zexa
import MyApp.Users { createUser }
```

编译为：

```javascript
import { createUser } from "./MyApp/Users.js";
```

---

## 8 工具链

```bash
$ zexa build
  ✓ 12 modules compiled → dist/
  ⚠ 2 draft modules, 5 todos
  ✓ Target: ES2022 JavaScript

$ zexa build --mode production
  ✓ 10 final modules compiled (2 draft modules excluded)
  ✓ Draft proxies removed (zero overhead)
  ✓ Tree-shaken, minified → dist/

$ zexa test
  ✓ 89 tests passed

$ zexa draft-report
  Types:   Config (2 unknown fields)
  Funcs:   processOrder (1 todo)
  Modules: 1 of 5 draft
  Versions: UserProfile at v3 (2 migrations)

$ zexa migrate --type UserProfile --from v1 --to v3 data.json
  ✓ Migrated 150 records: v1 → v2 → v3

$ zexa check
  ✓ All evolving types have complete migration chains
  ✓ All recipe DAGs are acyclic
  ✓ No draft dependencies in final modules
```

---

## 9 完整示例

```zexa
module Shop.Orders

import Shop.Types { Order, Customer, Product, Money }
import Shop.DB { findCustomer, findProduct, saveOrder }

// === 渐进类型 ===

draft type OrderRequest = {
  customerId: String,
  items: List<{ productId: String, quantity: Int }>,
  ...
}

final type OrderResponse = {
  orderId: String,
  total: Money,
  status: OrderStatus,
}

type OrderStatus = Pending | Confirmed | Shipped | Delivered

// === 类型演化 ===

evolving type OrderRecord {
  v1 = {
    orderId: String,
    customerId: String,
    total: Float,
  }

  v2 = {
    orderId: String,
    customerId: String,
    total: Float,
    currency: String,
    createdAt: DateTime,
  }
  migrate(v1 => v2) = (old) => ({
    ...old,
    currency: "USD",
    createdAt: DateTime.now(),
  })
}

// === 业务逻辑：recipe 自动并行 + isolate 容错 ===

recipe let processOrder = (request: OrderRequest): Async<Result<OrderError, OrderResponse>> => {
  // recipe: 编译器分析依赖，无依赖项自动并行
  let customer = isolate { findCustomer(request.customerId) }
  let products = isolate {
    request.items |> map((item) => findProduct(item.productId))
                  |> Promise.all
  }

  // customer 和 products 并行获取
  // 下面的逻辑依赖二者，串行执行

  match (customer, products) {
    (Err(e), _) => return Err(CustomerNotFound(e)),
    (_, Err(e)) => return Err(ProductNotFound(e)),
    (Ok(cust), Ok(prods)) => {
      let total = request.items
        |> zip(prods)
        |> map(([item, product]) => product.price * item.quantity)
        |> sum
        |> applyDiscount(cust.tier)

      let orderId = saveOrder({
        customer: cust,
        items: zip(request.items, prods),
        total: total,
      })

      return Ok({
        orderId: orderId,
        total: Money(total),
        status: Pending,
      })
    }
  }
}

// === 多服务聚合：部分失败优雅降级 ===

recipe let loadOrderDashboard = (userId: String): Async<Dashboard> => {
  let recentOrders = isolate { fetchRecentOrders(userId) }
  let recommendations = isolate { fetchRecommendations(userId) }
  let notifications = isolate { fetchNotifications(userId) }

  // 三个服务并行，任何一个挂掉都不影响其他
  return Dashboard({
    orders:          recentOrders    |> unwrapOr([]),
    recommendations: recommendations |> unwrapOr([]),
    notifications:   notifications   |> unwrapOr([]),
  })
}
```

### 上述示例编译结果（简化）

```javascript
// Shop/Orders.js
import { findCustomer, findProduct, saveOrder } from "./Shop/DB.js";

// evolving type — 迁移注册
const OrderRecord = {
  currentVersion: 2,
  migrations: {
    "1_to_2": (old) => ({ ...old, currency: "USD", createdAt: new Date() }),
  },
  migrateTo(data, v) { /* chain migration */ },
  serialize(obj) { return JSON.stringify({ _v: 2, ...obj }); },
  deserialize(json) {
    const d = JSON.parse(json);
    return this.migrateTo(d, this.currentVersion);
  },
};

// recipe + isolate
async function processOrder(request) {
  // Layer 0: 并行 + 隔离
  const [customer, products] = await Promise.all([
    (() => {
      try { return { tag: "Ok", value: findCustomer(request.customerId) }; }
      catch (e) { return { tag: "Err", value: { tag: "Panic", message: e.message } }; }
    })(),
    (async () => {
      try {
        const ps = await Promise.all(
          request.items.map((item) => findProduct(item.productId))
        );
        return { tag: "Ok", value: ps };
      } catch (e) {
        return { tag: "Err", value: { tag: "Panic", message: e.message } };
      }
    })(),
  ]);

  // Layer 1: 依赖 customer, products
  if (customer.tag === "Err") return { tag: "Err", value: { tag: "CustomerNotFound" } };
  if (products.tag === "Err") return { tag: "Err", value: { tag: "ProductNotFound" } };

  const cust = customer.value;
  const prods = products.value;

  const total = applyDiscount(
    cust.tier,
    request.items.reduce((sum, item, i) => sum + prods[i].price * item.quantity, 0)
  );

  const orderId = await saveOrder({
    customer: cust,
    items: request.items.map((item, i) => [item, prods[i]]),
    total,
  });

  return { tag: "Ok", value: { orderId, total, status: "Pending" } };
}

async function loadOrderDashboard(userId) {
  const [recentOrders, recommendations, notifications] = await Promise.all([
    (async () => { try { return { tag:"Ok", value: await fetchRecentOrders(userId) }; } catch(e) { return { tag:"Err", value:e }; } })(),
    (async () => { try { return { tag:"Ok", value: await fetchRecommendations(userId) }; } catch(e) { return { tag:"Err", value:e }; } })(),
    (async () => { try { return { tag:"Ok", value: await fetchNotifications(userId) }; } catch(e) { return { tag:"Err", value:e }; } })(),
  ]);

  return {
    orders:          recentOrders.tag === "Ok" ? recentOrders.value : [],
    recommendations: recommendations.tag === "Ok" ? recommendations.value : [],
    notifications:   notifications.tag === "Ok" ? notifications.value : [],
  };
}

export { processOrder, loadOrderDashboard, OrderRecord };
```

---

## 10 特性协同矩阵

|            | draft | recipe | isolate | evolving |
|------------|:-----:|:------:|:-------:|:--------:|
| **draft**    | —     | recipe 内可引用草稿类型 | isolate 内可调用草稿函数 | 草稿类型可声明为 evolving |
| **recipe**   |       | —      | 步骤可单独隔离 | recipe 可处理多版本数据 |
| **isolate**  |       |        | —       | 迁移函数可隔离执行 |
| **evolving** |       |        |         | — |

关键组合模式：

- **recipe + isolate**：并行步骤各自隔离，部分失败优雅降级
- **draft + evolving**：草稿阶段的类型也能声明版本演化
- **isolate + evolving**：数据迁移在隔离域中执行，迁移失败不崩溃

---

## 11 编译目标与运行时

### 编译目标

- **主目标**：ES2022+ JavaScript（ESM）
- 输出可直接在 Node.js / Deno / Bun / 浏览器中运行

### 运行时库

编译器附带轻量运行时（`zexa-runtime.js`），提供：

```javascript
// Result 类型工具
export const Ok = (value) => ({ tag: "Ok", value });
export const Err = (value) => ({ tag: "Err", value });
export const unwrapOr = (result, fallback) =>
  result.tag === "Ok" ? result.value : fallback;

// isolate 支持
export function isolate(fn, options = {}) { /* try-catch wrapper */ }
export function isolateAsync(fn, options = {}) { /* with timeout */ }

// evolving 支持
export class EvolvingType { /* migration registry + chain execution */ }

// draft 支持（仅开发模式）
export function draftProxy(obj, typeName, knownFields) { /* Proxy wrapper */ }
```

### 代数类型编译

```zexa
type Option<T> = None | Some(T)

let x = Some(42)
match x {
  None    => "empty",
  Some(n) => "value: " + show(n),
}
```

编译为：

```javascript
const None = { tag: "None" };
const Some = (value) => ({ tag: "Some", value });

const x = Some(42);
const __match_0 = x.tag === "None"
  ? "empty"
  : x.tag === "Some"
    ? "value: " + show(x.value)
    : (() => { throw new Error("Non-exhaustive match"); })();
```

---

## 12 实现路线图

```
Phase 1 — 基础
  ├─ JS-like 语法解析器
  ├─ HM 类型推断
  ├─ 代数类型 + 模式匹配
  ├─ 管道运算符
  └─ 编译到 ES Module

Phase 2 — 核心特性
  ├─ draft / refine / final（渐进类型）
  ├─ isolate（try-catch 包装 + 超时）
  ├─ todo 关键字
  └─ 开发模式 vs 生产模式

Phase 3 — 高级特性
  ├─ recipe（依赖分析 + Promise.all 生成）
  ├─ evolving（迁移链注册与执行）
  └─ 草稿传播检查

Phase 4 — 工具链
  ├─ zexa build / test / check
  ├─ zexa draft-report
  ├─ zexa migrate CLI
  ├─ Source maps
  └─ LSP 语言服务
```
