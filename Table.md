
# data.table 完整知识点总结

## 一、什么是 data.table

data.table 是 R 语言中用于高效数据操作的扩展包，它**继承自 data.frame**，但在语法和性能上做了大幅改进。

简单说：**data.table 就是增强版的 data.frame**。

---

## 二、为什么用 data.table

|痛点|data.table 的解决方案|
|---|---|
|data.frame 大数据集操作慢|底层用 C 实现，速度快 10-100 倍|
|语法冗长（反复写 `df$col`）|列名直接当变量用|
|分组聚合要用 `aggregate()`，不直观|`[, j, by]` 一行搞定|
|修改数据会复制整个表|`:=` 原地修改，省内存|
|读写大文件慢|`fread()` / `fwrite()` 极快|

---

## 三、适用场景

- 数据量大（百万行以上）
- 需要频繁分组、聚合、筛选
- 内存紧张，需要原地修改
- 需要快速读写 CSV 文件
- 复杂的表连接操作

**不太需要的场景**：小数据集、简单分析、团队都用 tidyverse

---

## 四、与 list、data.frame 的对比

### 继承关系

```
list
  └── data.frame
        └── data.table
```

### 核心区别

|特性|list|data.frame|data.table|
|---|---|---|---|
|结构|任意元素的容器|等长向量的列表，二维表|同 data.frame，但有增强|
|访问方式|`[[i]]` 或 `$name`|`[i, j]` 或 `$col`|`[i, j, by]`|
|列是否等长|不要求|必须|必须|
|行名|无|有|不推荐用，用 key 代替|
|性能|-|一般|极快|
|修改语义|引用|**复制**|**引用**（`:=`）|

### `[,]` 行为差异

```r
# data.frame
df[1:3, ]           # 前3行
df[, "col"]         # 返回向量
df[df$x > 5, ]      # 筛选

# data.table
dt[1:3]             # 前3行，不需要逗号
dt[, col]           # col 是表达式，返回向量
dt[, .(col)]        # 返回 data.table
dt[x > 5]           # 筛选，不需要 dt$
```

### 引用 vs 复制（重要！）

```r
# data.frame：赋值后独立
df2 <- df
df2$x <- 999        # df 不受影响

# data.table：赋值后共享
dt2 <- dt
dt2[, x := 999]     # dt 也变了！
dt2 <- copy(dt)     # 用 copy() 才能独立
```

---

## 五、核心语法：`dt[i, j, by]`

这是 data.table 的灵魂，三个位置各有用途：

|位置|作用|示例|
|---|---|---|
|`i`|筛选行（哪些行）|`dt[x > 5]`|
|`j`|操作列（做什么）|`dt[, .(sum(x))]`|
|`by`|分组（按什么分）|`dt[, .(sum(x)), by = "group"]`|

---

## 六、常用操作方法

### 1. 筛选行

```r
dt[x > 5]                    # 条件筛选
dt[1:10]                     # 前10行
dt[order(x)]                 # 排序
dt[order(-x)]                # 降序
```

### 2. 选择/计算列

```r
dt[, x]                      # 返回向量
dt[, .(x)]                   # 返回 data.table
dt[, .(x, y)]                # 多列
dt[, .(total = sum(x))]      # 计算
dt[, .(mean_x = mean(x), sd_x = sd(x))]  # 多个统计量
```

### 3. 添加/修改列（`:=`）

```r
dt[, new_col := x^2]                    # 添加列
dt[, c("a", "b") := .(x+1, x-1)]        # 添加多列
dt[x > 5, new_col := 999]               # 条件修改
dt[, old_col := NULL]                   # 删除列
```

### 4. 分组聚合

```r
dt[, .(total = sum(x)), by = "group"]           # 单列分组
dt[, .(total = sum(x)), by = c("g1", "g2")]     # 多列分组
dt[, .N, by = "group"]                          # 计数
```

### 5. 特殊变量

|变量|含义|示例|
|---|---|---|
|`.N`|行数|`dt[, .N, by = "group"]`|
|`.SD`|子表|`dt[, .SD[1], by = "group"]`|
|`.SDcols`|限制 .SD 的列|`dt[, lapply(.SD, mean), .SDcols = 1:3]`|
|`.I`|行索引|`dt[, .(rows = list(.I)), by = "group"]`|
|`.GRP`|组编号|`dt[, .GRP, by = "group"]`|
|`.BY`|当前分组值|`dt[, .BY[[1]], by = "group"]`|

### 6. 表连接

```r
# 基本连接
X[Y, on = "key"]                         # Y left join X
X[Y, on = c("a" = "b")]                  # 列名不同

# 连接类型
X[Y, on = "key", nomatch = NULL]         # 内连接
X[!Y, on = "key"]                        # 反连接

# 滚动连接
X[.(value), on = "col", roll = "nearest"]
```

### 7. 键与索引

```r
setkey(dt, col)              # 设置键（会排序）
setkeyv(dt, "col")           # 同上
key(dt)                      # 查看键
dt["value"]                  # 快速查找

setindex(dt, col)            # 设置索引（不排序）
indices(dt)                  # 查看索引
```

### 8. 数据重塑

```r
# 宽 → 长
melt(dt,
  id.vars = "id",
  measure.vars = c("col1", "col2"),
  variable.name = "var",
  value.name = "val"
)

# 多组变量
melt(dt,
  id.vars = "id",
  measure.vars = patterns(price = "^price", shipping = "^ship")
)

# 长 → 宽
dcast(dt, row_var ~ col_var, value.var = "value")
dcast(dt, row_var ~ col_var, value.var = "value", fun.aggregate = sum)
```

### 9. 其他实用函数

|函数|用途|
|---|---|
|`rbindlist(list(...))`|合并多个表|
|`copy(dt)`|深拷贝|
|`CJ(a, b)`|交叉连接（所有组合）|
|`uniqueN(x)`|唯一值个数|
|`fread()` / `fwrite()`|快速读写文件|
|`setnames(dt, old, new)`|重命名|
|`setorder(dt, col)`|排序|
|`setnafill(dt, fill = 0)`|填充 NA|
|`shift(x, n)`|滞后/超前|

---

## 七、课程例题回顾

### 例题 1：温度转换

```r
# 将华氏度转为摄氏度
ski.temps[unit == "Fahrenheit", temp := (temp - 32) * 5 / 9]
ski.temps[, unit := "Celsius"]
```

**知识点**：条件修改 + `:=`

### 例题 2：分组取最值

```r
# 每个国家最高温的滑雪场
ski.temps[, .SD[which.max(temp)], by = "country"]

# 或者只取名字
ski.temps[, .(name = name[which.max(temp)]), by = "country"]
```

**知识点**：`.SD` + `which.max()` + `by`

### 例题 3：表连接计算总价

```r
# 每个部门的总票价
destinations[ticket.prices, on = c(dest = "name")][,
  .(total = sum(people * ticket)), by = "department"]
```

**知识点**：`on` 连接 + 分组聚合

### 例题 4：melt + join 计算收入

```r
# 先 melt 成长格式
prices.long <- melt(itemshop.prices,
  id.vars = "item",
  measure.vars = patterns(price = "^price\\.", shipping = "^shipping\\."),
  variable.name = "channel"
)

# 再 join 并计算
prices.long[sales, on = c("item", "channel")][,
  .(revenue = sum((price + shipping) * quantity)), by = "channel"]
```

**知识点**：`melt()` + `patterns()` + 连接 + 分组

---

## 八、注意事项

### 1. 引用语义陷阱

```r
dt2 <- dt          # 不是复制！
dt2[, x := 0]      # dt 也变了
# 正确做法
dt2 <- copy(dt)
```

### 2. `:=` 不返回值

```r
result <- dt[, new := x^2]   # result 是 dt 本身，不是新表
# 如果想链式操作，用 []
dt[, new := x^2][]           # 加 [] 才会打印
```

### 3. `j` 位置的表达式

```r
dt[, "col"]        # 返回 data.table（和 data.frame 不同）
dt[, col]          # 返回向量
dt[, .(col)]       # 返回 data.table
```

### 4. `by` 会改变结果结构

```r
dt[, sum(x)]                    # 返回一个数
dt[, sum(x), by = "g"]          # 返回 data.table
```

### 5. `on` vs `key`

- 有 `key`：可以直接 `dt["value"]`
- 无 `key`：必须用 `on = "col"`
- 推荐：**优先用 `on`**，更明确

### 6. `melt` 的 `variable` 是 factor

```r
dt.long$channel               # 是 factor 类型
levels(dt.long$channel)       # 查看/修改水平
```

### 7. `CJ()` 用于补全缺失组合

```r
# 确保每个 item-channel 组合都有记录
dt[CJ(item, channel, unique = TRUE), on = c("item", "channel")]
```

---

## 九、速查表

```r
# 读数据
dt <- fread("file.csv")

# 筛选
dt[condition]

# 选列
dt[, .(col1, col2)]

# 计算
dt[, .(result = func(col))]

# 修改
dt[, new_col := expr]

# 分组
dt[, .(result = func(col)), by = "group"]

# 连接
X[Y, on = "key"]

# 宽→长
melt(dt, id.vars, measure.vars)

# 长→宽
dcast(dt, row ~ col, value.var)
```

---
# 补充知识点

这个练习补充了一些上面没有详细讲的内容：

---

## 1. 创建 data.table 的两种方式

```r
# 方式一：as.data.table()（推荐用于内置数据集）
carsdt <- as.data.table(cars)

# 方式二：setDT()（原地转换，更省内存）
carsdt <- cars
setDT(carsdt)
```

**注意**：`setDT()` 不能直接用在内置数据集上，因为它们是锁定的。需要先赋值给新变量。

---

## 2. `%between%` 操作符

比 `>=` 和 `<=` 组合更简洁：

```r
# 传统写法
dt[speed >= 10 & speed <= 20]

# data.table 写法
dt[speed %between% c(10, 20)]
```

---

## 3. 批量添加多列的写法

```r
# 写法一：`:=`() 函数形式
dt[, `:=`(speed.3 = speed^3, speed.4 = speed^4)]

# 写法二：c() + .() 形式
dt[, c("speed.3", "speed.4") := .(speed^3, speed^4)]

# 写法三：动态列名 + lapply
dt[, (paste0("speed.", 3:4)) := lapply(3:4, function(x) speed ^ x)]
```

**注意**：写法三中列名要加 `()`，否则会被当作变量名而非表达式。

---

## 4. 删除列的两种方式

```r
# data.table 方式（推荐）
dt[, speed := NULL]

# base R 方式（也可以）
dt$speed <- NULL
```

---

## 5. 不推荐的写法

练习里标注了一些"可行但不推荐"的写法：

```r
# 不推荐：用 cbind 添加列
cbind(dt, speed.2 = dt$speed^2)   # 创建新表，浪费内存

# 不推荐：用 $ 赋值
dt$speed.2 <- dt$speed^2          # 可能触发复制

# 推荐：用 :=
dt[, speed.2 := speed^2]          # 原地修改，高效
```

---

## 总结表格

| 操作             | 推荐写法                          | 不推荐              |
| -------------- | ----------------------------- | ---------------- |
| 转换为 data.table | `as.data.table()` 或 `setDT()` | -                |
| 范围筛选           | `%between%`                   | `>= & <=` 组合     |
| 添加单列           | `dt[, new := expr]`           | `dt$new <- expr` |
| 添加多列           | `dt[, c(...) := .(...)]`      | `cbind()`        |
| 删除列            | `dt[, col := NULL]`           | -                |