---
title: 记一次漫长的Debug - Gorm中, 使用sqlite的内存数据库测试时出现table not found
date: 2020-07-20 18:50:54
tags:
  - Go
  - Gorm
  - 数据库
---
**TL;DR:** `:memory:` 会 **为每个链接打开一个新的数据库**. 
使用 `file::memory:?cache=shared` 也可能碰到问题, 配置 `MaxOpenConnection` 为 1 即可. (`db.DB().SetMaxOpenConns(1)`)


`Gorm`这一`ORM`框架很好的屏蔽了不同的`RDMS`之间的差异, 在运行单元测试时使用sqlite的内存数据库 `:memory:` 显得尤为方便.

在我的 [EduOJ](github.com/leoleoasd/EduOJBackend) 项目中, 数据库相关的测试没有使用 `sqlmock` 等, 而是使用了基于`SQLite`内存数据库. 但是, 在我开始编写代码的过程中出现了玄学问题: 一旦开启单元测试的 `Parallel` 模式, 就会随机的出现 `table xxx not found` 错误. 甚至, 在运行代码的过程中, 通过调试模式可以看到, 上一行还存在数据表, 到了下一行就不存在了. 

以为这是一个 `Race condition`, `db`的值被替换了, 拿着
`race detector` 看了半天啥也没有. 又经过各种测试, 发现这个问题 **当且仅当两个需要数据库的测试同时运行** 时发生. 

绝望之中, 搜索了一下 `gorm sqlite race condition`, 找到了 `go-sqlite3` 的 [#204](https://github.com/mattn/go-sqlite3/issues/204) 这个 issue. issue中提到, **sqlite 会给每一个数据库连接单独开启一个内存数据库**. 现在看起来, 这一点也非常合理: 连接与连接之间本身就是应该不 share 任何信息的. issue中给出的解决方案是, 使用 `file::memory:?cache=shared` 替换 `:memory:`. 

替换为 `:memory:` 后, sqlite又报错说 "table xxx is locked". 根据[Django文档](https://docs.djangoproject.com/en/dev/ref/databases/#database-is-locked-errorsoption), 可以发现这个问题源自 sqlite 对于并发请求的处理能力偏弱. 因此, 在使用 sqlite 进行单元测试的时候, 相对好的解决方案还是配置连接池的最大连接数为1.

```go
db, _ := gorm.Open("sqlite3",":memory:")
db.DB().SetMaxOpenConns(1)
```

那么, 为什么我在调试的过程中会见到如此之诡异的现象呢? 这得从库的设计说起. 由于数据库连接的建立和关闭开销较大, 是一种较为"昂贵"的资源, 因此成熟的数据库客户端库都会 **使用连接池**. 在需要连接的时候, 从连接池中取出一个已经建立好的连接, 使用完连接后, 放回连接池中, 就能花很大的加快应用程序的速度.

在我的使用情境中, 初始化数据库/建立数据表的`SQL`仅在第一个连接中执行了. 如果关闭单元测试的`Parallel`模式, 那么所有的单元测试都是顺序执行的, 始终会用到同一个连接, 不会出现上文的错误. 一旦开启`Parallel`模式, 每一次`sql`指令就都可能在不同的连接上进行, 而正如上文所述, 每一个连接对应了一个不同的内存数据库, 因此也就会出现 "上一行还正常, 下一行就不存在数据表" 的情况了.
