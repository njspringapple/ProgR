
# R6 面向对象编程完整总结

---

## 一、什么是面向对象编程（OOP）

面向对象编程是一种**以"对象"为中心**的编程范式：

- **对象**：代表某种状态或数据集合（如一个线性模型、一个数据框、一个游戏角色）
- **方法**：可以对对象执行的操作（如用模型做预测、给数据框添加列）
- **类**：对象的"模板"或"蓝图"，定义对象应该有什么字段和方法

### OOP 与其他编程范式对比

|范式|核心思想|R 中的例子|
|---|---|---|
|过程式|把代码组织成函数|普通函数 `mean()`, `sum()`|
|函数式|函数是"一等公民"，无副作用|`lapply()`, `purrr`|
|面向对象|把函数和数据绑定在一起|S3, R6, S4|

---

## 二、R 中的 OOP 系统

|系统|语义|结构|特点|
|---|---|---|---|
|**S3**|复制语义|非常宽松|最常用，简单灵活|
|**R6**|引用语义|明确的类定义|类似 Python/Java|
|S4|复制语义|严格，有类型检查|Bioconductor 常用|
|Reference Classes|引用语义|严格|不推荐，用 R6 代替|
|R7|复制语义|严格|开发中（2023+）|

**本课程重点**：S3 和 R6

---

## 三、R6 详解

### 3.1 为什么用 R6

- 类似 Python/Java 的 OOP 风格
- 引用语义：修改对象时不会复制
- 明确的类结构：`public`、`private`、`active`
- 支持继承

### 3.2 创建类

```r
library(R6)

Cat <- R6Class("Cat",
  public = list(
    name = NULL,
    age = NULL,
    
    # 构造函数
    initialize = function(name, age) {
      self$name <- name
      self$age <- age
    },
    
    # 方法
    talk = function() {
      "meow"
    },
    
    set_age = function(new_age) {
      self$age <- new_age
    }
  )
)
```

### 3.3 创建对象（实例化）

```r
minka <- Cat$new("Minka", 3)
minka$name       # "Minka"
minka$talk()     # "meow"
minka$set_age(4)
minka$age        # 4
```

### 3.4 引用语义（重要！）

```r
# 赋值不会复制！
minka2 <- minka
minka2$name <- "Kitty"
minka$name   # "Kitty" —— 两个变量指向同一个对象！

# 要真正复制，用 clone()
minka3 <- minka$clone(deep = TRUE)
minka3$name <- "Tom"
minka$name   # "Kitty" —— 不受影响
```

**规则**：

- `<-` 赋值：共享同一对象
- `$clone()`：浅拷贝
- `$clone(deep = TRUE)`：深拷贝（推荐）

### 3.5 Private 成员

```r
Queue <- R6Class("Queue",
  public = list(
    add = function(x) {
      private$items <- c(private$items, list(x))
      invisible(self)  # 返回 self 以支持链式调用
    },
    remove = function() {
      if (length(private$items) == 0) return(NULL)
      item <- private$items[[1]]
      private$items <- private$items[-1]
      item
    }
  ),
  private = list(
    items = list()
  )
)

q <- Queue$new()
q$add(1)$add(2)$add(3)  # 链式调用
q$remove()  # 1
q$items     # NULL —— private 成员从外部不可访问
```

### 3.6 Active Bindings（活动绑定）

看起来像字段，但实际上是函数：

```r
Numbers <- R6Class("Numbers",
  public = list(
    x = 100
  ),
  active = list(
    x2 = function(value) {
      if (missing(value)) {
        self$x * 2      # 读取时
      } else {
        self$x <- value / 2  # 赋值时
      }
    }
  )
)

n <- Numbers$new()
n$x2         # 200（读取）
n$x2 <- 1000
n$x          # 500（赋值触发计算）
```

### 3.7 继承

```r
Creature <- R6Class("Creature",
  public = list(
    name = NULL,
    age = NULL,
    initialize = function(name, age) {
      self$name <- name
      self$age <- age
    },
    talk = function() "..."
  )
)

Cat <- R6Class("Cat",
  inherit = Creature,  # 继承
  public = list(
    initialize = function(name, age, furcolor) {
      super$initialize(name, age)  # 调用父类构造函数
      self$furcolor <- furcolor
    },
    furcolor = NULL,
    talk = function() "meow"  # 重写方法
  )
)

kitty <- Cat$new("Kitty", 2, "white")
kitty$talk()  # "meow"
kitty$name    # "Kitty"（继承自 Creature）
```

### 3.8 深拷贝与 deep_clone

当对象包含其他 R6 对象时，需要自定义 `deep_clone`：

```r
Herd <- R6Class("Herd",
  public = list(
    creatures = list(),
    add = function(creature) {
      self$creatures <- c(self$creatures, list(creature))
    }
  ),
  private = list(
    deep_clone = function(name, value) {
      if (name == "creatures") {
        lapply(value, function(x) x$clone(deep = TRUE))
      } else {
        value
      }
    }
  )
)
```

### 3.9 Finalizer（终结器）

对象被垃圾回收时执行：

```r
FileHandler <- R6Class("FileHandler",
  public = list(
    conn = NULL,
    initialize = function(path) {
      self$conn <- file(path, "w")
    }
  ),
  private = list(
    finalize = function() {
      close(self$conn)
      message("File closed!")
    }
  )
)
```

---

## 四、S3 详解

### 4.1 什么是 S3

- R 最早的 OOP 系统
- 非常简单：对象就是带 `class` 属性的任何东西
- 基于函数分发，不是方法绑定

### 4.2 创建 S3 对象

```r
# 方法一：structure()
h1 <- structure(
  list(name = "Alice", age = 25),
  class = "Human"
)

# 方法二：设置 class 属性
h2 <- list(name = "Bob", age = 30)
class(h2) <- "Human"
```

### 4.3 定义泛型函数和方法

```r
# 泛型函数
talk <- function(x) UseMethod("talk")

# 特定类的方法
talk.Human <- function(x) {
  paste("Hello, I'm", x$name)
}

talk.Cat <- function(x) {
  "meow"
}

# 默认方法
talk.default <- function(x) {
  stop("This object can't talk")
}

# 使用
talk(h1)  # "Hello, I'm Alice"
```

### 4.4 print 方法

```r
print.Human <- function(x, ...) {
  cat(sprintf("A human named %s, age %s\n", x$name, x$age))
  invisible(x)  # 必须返回 invisible(x)
}

h1  # 自动调用 print.Human
```

### 4.5 S3 继承

```r
# 多个类，从具体到抽象
class(h1) <- c("Human", "Creature")

talk.Creature <- function(x) "..."

# Human 方法优先，找不到才用 Creature
```

### 4.6 NextMethod()

```r
talk.TalkingCat <- function(x) {
  if (x$age > 3) {
    "Hello, meow!"
  } else {
    NextMethod()  # 调用下一个类的方法
  }
}

c1 <- structure(list(name = "Tom", age = 2), class = c("TalkingCat", "Cat", "Creature"))
talk(c1)  # 调用 talk.Cat（因为 age <= 3）
```

### 4.7 构造函数（推荐）

```r
Human <- function(name, age) {
  assertString(name)
  assertNumber(age, lower = 0)
  structure(
    list(name = name, age = age),
    class = c("Human", "Creature")
  )
}

# 继承的构造函数
Student <- function(name, age, school) {
  obj <- Human(name, age)
  obj$school <- school
  class(obj) <- c("Student", class(obj))
  obj
}
```

---

## 五、S3 vs R6 对比

|特性|S3|R6|
|---|---|---|
|语义|复制|引用|
|类定义|隐式（设置 class 属性）|显式（R6Class）|
|方法调用|`func(obj)`|`obj$func()`|
|私有成员|无|有|
|继承|通过 class 向量|通过 `inherit` 参数|
|类型检查|无|无（可用 checkmate）|
|使用场景|简单对象，base R 风格|复杂状态，需要修改对象|

---

## 六、设计模式：继承 vs 组合

### 继承（is-a 关系）

```r
# Cat "是一个" Creature
Cat <- R6Class("Cat", inherit = Creature, ...)
```

### 组合（has-a 关系）

```r
# ModelEnsemble "包含" 多个 Model
ModelEnsemble <- R6Class("ModelEnsemble",
  public = list(
    models = list(),  # 包含其他模型
    initialize = function(models) {
      self$models <- models
    }
  )
)
```

---

## 七、课程示例回顾

### 示例 1：Creature 和 Cat

```r
Creature <- R6Class("Creature",
  public = list(
    name = NULL,
    age = NULL,
    initialize = function(name, age) {
      self$name <- assertString(name)
      self$age <- assertNumeric(age, lower = 0)
    },
    talk = function() "..."
  )
)

Cat <- R6Class("Cat",
  inherit = Creature,
  public = list(
    furcolor = NULL,
    initialize = function(name, age, furcolor) {
      super$initialize(name, age)
      self$furcolor <- furcolor
      private$.legs <- 4
    },
    talk = function() "meow"
  ),
  active = list(
    legs = function(x) {
      if (!missing(x)) {
        assertInt(x, lower = 0, upper = private$.legs)
        private$.legs <- x
      }
      private$.legs
    }
  ),
  private = list(
    .legs = NULL
  )
)
```

### 示例 2：机器学习模型继承

```r
Model <- R6Class("Model",
  public = list(
    state = NULL,
    train = function(x, y) stop("not implemented"),
    predict = function(x) stop("not implemented"),
    is.trained = function() !is.null(self$state)
  )
)

ModelLinear <- R6Class("ModelLinear",
  inherit = Model,
  public = list(
    train = function(x, y) {
      self$state <- lm(y ~ ., data = cbind(y = y, x))
      invisible(self)
    },
    predict = function(x) predict(self$state, x)
  )
)
```

---

## 八、注意事项

### R6 注意事项

1. **引用语义陷阱**
    
    ```r
    a <- Cat$new("A", 1, "white")
    b <- a         # 不是复制！
    b$name <- "B"
    a$name         # "B" —— a 也变了！
    ```
    
2. **始终用 `clone(deep = TRUE)`**
    
    ```r
    b <- a$clone(deep = TRUE)  # 正确的复制方式
    ```
    
3. **在 `initialize` 中创建引用类型字段**
    
    ```r
    # 错误：所有实例共享同一个 list
    MyClass <- R6Class("MyClass",
      public = list(items = list())
    )
    
    # 正确
    MyClass <- R6Class("MyClass",
      public = list(
        items = NULL,
        initialize = function() self$items <- list()
      )
    )
    ```
    
4. **链式调用返回 `invisible(self)`**
    
    ```r
    add = function(x) {
      private$items <- c(private$items, x)
      invisible(self)  # 支持 obj$add(1)$add(2)
    }
    ```
    

### S3 注意事项

1. **`print` 方法必须返回 `invisible(x)`**
    
2. **继承顺序：子类在前**
    
    ```r
    class(obj) <- c("Student", "Human", "Creature")
    ```
    
3. **用构造函数而不是手动创建**
    

---

## 九、速查表

### R6

```r
# 定义类
MyClass <- R6Class("MyClass",
  public = list(
    field = NULL,
    initialize = function(val) self$field <- val,
    method = function() self$field
  ),
  private = list(
    .hidden = NULL
  ),
  active = list(
    computed = function(val) { ... }
  )
)

# 继承
SubClass <- R6Class("SubClass",
  inherit = MyClass,
  public = list(
    initialize = function(val) {
      super$initialize(val)
    }
  )
)

# 实例化
obj <- MyClass$new(value)

# 克隆
obj2 <- obj$clone(deep = TRUE)
```

### S3

```r
# 构造函数
MyClass <- function(x) {
  structure(list(value = x), class = "MyClass")
}

# 泛型
mymethod <- function(x, ...) UseMethod("mymethod")

# 方法
mymethod.MyClass <- function(x, ...) { ... }

# 继承
SubClass <- function(x) {
  obj <- MyClass(x)
  class(obj) <- c("SubClass", class(obj))
  obj
}

# 调用父类方法
mymethod.SubClass <- function(x, ...) {
  result <- NextMethod()
  # 额外处理
}
```

---

祝考试顺利！🎓