+++
title = "Context receivers"
author = ["4shen0ne"]
publishDate = 2022-06-06T00:00:00+08:00
tags = ["kotlin"]
draft = false
+++

-   Status: Experimental in 1.6.20-M1
-   Discussion: [KEEP-259](https://github.com/Kotlin/KEEP/issues/259)

<!--more-->


## 介绍 {#介绍}

Kotlin 在 1.6.20-M1 版本中添加了一个实验性的新特性 context receivers，以支持上下文相关的声明（context-dependent declarations）

这个特性是为了方便在 Kotlin 中面向上下文编程，而在此之前主要通过扩展函数和作用域函数实现。关于面向上下文编程可以看看[这篇文章](https://proandroiddev.com/an-introduction-context-oriented-programming-in-kotlin-2e79d316b0a2)

引入 context receivers 的目的[^fn:1]：

> -   Remove all limitations of member extensions for writing contextual abstractions
>     -   Support top-level (non-member) contextual functions and properties
>     -   Support adding contextual function and properties to 3rd party context classes
>     -   Support multiple contexts
> -   Make blocks of code with multiple receivers representable in Kotlin's type system
> -   Separate the concepts of extension and dispatch receivers from the concept of context receivers
>     -   Context receivers should not change the meaning of unqualified this expression
>     -   Multiple contexts should not be ordered during resolution, resolution ambiguities shall be reported
> -   Design a scalable resolution algorithm with respect to the number of receivers
>     -   Call resolution should not be exponential in the number of context receivers

另外需要注意，context receivers 在 1.6.20 版本中还是实验性的，需要显式开启
`-Xcontext-receivers` 才能使用。比如在 Gradle 中使用：

```kotlin
tasks.withType<KotlinCompile> {
    kotlinOptions.jvmTarget = "17"
    kotlinOptions.freeCompilerArgs += "-Xcontext-receivers"
}
```


## 使用场景 {#使用场景}

举个例子，我们来实现一个简单的银行交易逻辑，其中存款和取款必须是事务性的，在失败时进行回滚

{{< figure src="/ox-hugo/account-service.svg" width="600" >}}

我们在 AccountService#transfer 中处理交易逻辑，通过 Transaction 进行事务性操作，所以 transfer 需要传入一个事务实例。常规实现：

```kotlin
class AccountService {
    fun transfer(tx: Transaction, vararg operations: () -> Unit) {
        tx.start()
        try {
            operations.forEach { it.invoke() }
            tx.commit()
        } catch (e: Exception) {
            tx.rollback()
        }
    }
}
```

然后在调用时将 Transaction 实例和 lambda 作为参数传入：

```kotlin
val service = AccountService()
val transaction = Transaction()
val repo = AccountRepo()
service.transfer(
    transaction,
    { repo.credit(account1, 10.5) },
    { repo.debit(account2, 10.5) }
)
```

上面的实现在 Java 中很常见，但是在 Kotlin 我们可以用扩展函数来优化一下：

```kotlin
class AccountService {
    fun Transaction.transfer(vararg operations: () -> Unit) {
        start()
        try {
            operations.forEach { it.invoke() }
            commit()
        } catch (e: Exception) {
            rollback()
        }
    }
}
```

现在 transfer 中的 this 指向了 Transaction 实例，那么就不再需要通过参数传入了，不过需要在 AccountService 实例的上下文中使用 transfer，比如使用 with：

```kotlin
with(service) {
    transaction.transfer(
        { repo.credit(account1, 10.5) },
        { repo.debit(account2, 10.5) }
    )
}
```

使用扩展函数虽然减少了一些代码，但是存在其他问题：在语义上和原来的版本是不同的——transfer 是 Transaction 类的方法，这一点也不符合实际的逻辑，Transaction 应该只包含事务性的操作

那么有没有办法将 Transaction 上下文引入 transfer 的同时，不改变 transfer 的所属类呢？Context receivers 特性就可以实现这一点

我们在 transfer 的声明上通过 `context()` 指定上下文，那么在函数内就会引入一个指向 Transaction 的隐式的 this

```kotlin
class AccountService {
    context(Transaction)
    fun transfer(vararg operations: () -> Unit) {
        start()
        try {
            operations.forEach { it.invoke() }
            commit()
        } catch (e: Exception) {
            rollback()
        }
    }
}
```

然后在 Transaction 实例的上下文中调用 transfer 服务，在语义上也符合逻辑

```kotlin
with(transaction) {
    service.transfer(
        { repo.credit(account1, 10.5) },
        { repo.debit(account2, 10.5) }
    )
}
```


## 思考 {#思考}

由于 context receivers 还处于实验阶段，使用体验和实际案例都还比较欠缺，要感受到它带来的好处恐怕还需要一段时间。不过在实现 DSL 上它肯定比继承更加适合，唯一需要担心的是滥用 context 导致代码晦涩难懂

关于 context receivers 的详细说明建议看 KEEP 上的相关提案：[context-receivers](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md)

[^fn:1]: <https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md#goals>
