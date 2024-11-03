---
title: "boltdb 介绍"
date: "2024-05-08T20:56:00+08:00"
tags: 
- boltdb
- etcd
- golang
- 数据库
- kubernetes
showToc: true
---


## 介绍
`BoltDB` 是一个用 Go 语言编写的嵌入式键/值数据库。以下是关于 BoltDB 的一些基本介绍：

- **键/值存储**: BoltDB 为应用程序提供了简单的键/值存储接口。
- **事务**: BoltDB 支持完整的 ACID 事务。
- **嵌入式**: 与像 MySQL 或 PostgreSQL 这样的数据库系统不同，BoltDB 不运行在单独的服务器进程中。它作为一个库被直接嵌入到你的应用程序中。
- **单文件存储**: 所有的数据都存储在一个文件中，这使得备份和迁移变得简单。
- **高效的二进制存储**: 数据在磁盘上使用 B+ 树结构存储，这为随机读取提供了高性能。
- **前缀扫描**: 可以很容易地按键的前缀进行扫描，这使得它适用于范围查询。
- **没有外部依赖**: BoltDB 不依赖于任何外部系统或库。
- **线程安全**: BoltDB 是线程安全的，可以在多个 goroutines 中并发地使用。

BoltDB 特别适用于需要一个轻量级、高性能、易于部署和维护的数据库解决方案的场景。

虽然 BoltDB 非常有用，但它也有其局限性。例如，它不支持分布式存储，也不适用于需要多节点复制或分片的场景。但对于许多应用程序，它提供了一个简单且高性能的存储解决方案。

**源码地址：**

https://github.com/boltdb/bolt 这个项目已经不维护了,  [etcd](https://etcd.io/) 官方维护了一个 fork 库，api 是互通的: https://github.com/etcd-io/bbolt

## 谁在使用

[etcd](https://github.com/etcd-io/etcd)、[tidb](https://github.com/pingcap/tidb)、[influxdb](https://github.com/influxdata/influxdb)、[consul](https://github.com/hashicorp/consul) ......

## 架构图

![boltdb](/images/6f10d17d-ca52-4fd0-a600-e92a9fc99d36.png)

## 引入

```bash
go get go.etcd.io/bbolt 
```

## 使用

### open  a database

Bolt 中的顶级对象是 DB。它在你的硬盘上表示为一个单独的文件，并代表了你数据的一个一致性快照。

要打开你的数据库，只需使用 `bolt.Open()` 函数：

```GO
package main

import (
	"log"

	bolt "go.etcd.io/bbolt"
)

func main() {
	db, err := bolt.Open("my.db", 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```

请注意，Bolt 在数据文件上获得一个文件锁，因此多个进程不能同时打开同一个数据库。打开一个已经打开的 Bolt 数据库会导致它挂起，直到另一个进程关闭它。为了防止无限期的等待，你可以向 `Open()` 函数传递一个超时选项：

```GO
db, err := bolt.Open("my.db", 0600, &bolt.Options{Timeout: 1 * time.Second})
```

### Transactions

Bolt一次只允许一个读写 transactions ，但允许您一次进行多个只读 transactions 。每一个 transactions 在开始时都能看到数据的一致视图。

单独的 transactions 和从它们创建的所有对象（例如桶、键）都不是线程安全的。要在多个goroutines中处理数据，您必须为每一个goroutine启动一个 transactions ，或使用锁来确保一次只有一个goroutine访问一个 transactions 。从数据库创建 transactions 是线程安全的。

 transactions 不应相互依赖，并且通常不应在同一goroutine中同时打开。这可能会导致死锁，因为读写 transactions 需要定期重新映射数据文件，但在任何只读 transactions 打开时都不能这样做。即使是嵌套的只读 transactions 也可能导致死锁，因为子 transactions 可能阻止父 transactions 释放其资源。

#### Read-write transactions

要开始一个读写 transactions ，可以使用DB.Update()函数：

```go
err := db.Update(func(tx *bolt.Tx) error {
	...
	return nil
})
```

在闭包中，您可以看到数据库的一致视图。通过在最后返回nil来提交 transactions 。您还可以通过返回错误在任何时候回滚 transactions 。所有的数据库操作都允许在一个读写 transactions 中进行。

始终检查返回错误，因为它会报告任何可能导致您的 transactions 未完成的磁盘故障。如果您在闭包内返回错误，它将被传递。

#### Read-only transactions

要开始一个只读 transactions ，您可以使用DB.View()函数：

```go
err := db.View(func(tx *bolt.Tx) error {
	...
	return nil
})
```

在这个闭包内，您也可以看到数据库的一致视图，但在只读 transactions 中不允许进行变动操作。在一个只读 transactions 中，您只能检索桶、检索值和复制数据库。

#### Batch read-write transactions

每个DB.Update()都会等待磁盘提交写入。这种开销可以通过使用DB.Batch()函数结合多个更新来最小化：

```go
err := db.Batch(func(tx *bolt.Tx) error {
	...
	return nil
})
```

并发的Batch调用会被机会性地组合成更大的 transactions 。只有当有多个goroutines调用它时，Batch才是有用的。

权衡之处在于，Batch在 transactions 的部分失败时可以多次调用给定的函数。该函数必须是幂等的，且只有在成功从DB.Batch()返回后，副作用才会生效。

例如：不要从函数内部显示消息，而是在封闭范围内设置变量：

```go
var id uint64
err := db.Batch(func(tx *bolt.Tx) error {
	// 在桶中查找最后一个键，解码为bigendian uint64，增加
	// 一个，编码回[]byte，并添加新键。
	...
	id = newValue
	return nil
})
if err != nil {
	return ...
}
fmt.Println("Allocated ID %d", id)
```

#### Managing transactions manually

DB.View()和DB.Update()函数是DB.Begin()函数的包装。这些助手函数将启动 transactions 、执行函数，然后在返回错误时安全地关闭您的 transactions 。这是使用Bolt transactions 的推荐方法。

然而，有时您可能希望手动开始和结束您的 transactions 。您可以直接使用DB.Begin()函数，但请确保关闭 transactions 。

```go
// 开始一个可写的 transactions 。
tx, err := db.Begin(true)
if err != nil {
    return err
}
defer tx.Rollback()

// 使用 transactions ...
_, err := tx.CreateBucket([]byte("MyBucket"))
if err != nil {
    return err
}

// 提交 transactions 并检查错误。
if err := tx.Commit(); err != nil {
    return err
}
```

DB.Begin()的第一个参数是一个布尔值，表示 transactions 是否应该是可写的。

### Using buckets

桶是数据库中的键/值对集合。一个桶中的所有键都必须是唯一的。您可以使用Tx.CreateBucket()函数创建一个桶：

```go
db.Update(func(tx *bolt.Tx) error {
	b, err := tx.CreateBucket([]byte("MyBucket"))
	if err != nil {
		return fmt.Errorf("create bucket: %s", err)
	}
	return nil
})
```

您可以使用Tx.Bucket()函数检索现有的桶：

```go
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	if b == nil {
		return fmt.Errorf("bucket does not exist")
	}
	return nil
})
```

您还可以使用Tx.CreateBucketIfNotExists()函数只在桶不存在时创建一个桶。打开数据库后，为所有的顶级桶调用此函数是一种常见的模式，这样您可以保证它们存在于未来的 transactions 中。

要删除一个桶，只需调用Tx.DeleteBucket()函数。

### Using key/value pairs

要将键/值对保存到存储桶，使用`Bucket.Put()`函数：

```go
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	err := b.Put([]byte("answer"), []byte("42"))
	return err
})
```

这会在`MyBucket`存储桶中将"answer"键的值设置为"42"。要检索此值，我们可以使用`Bucket.Get()`函数：

```go
db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	v := b.Get([]byte("answer"))
	fmt.Printf("The answer is: %s\n", v)
	return nil
})
```

`Get()`函数不返回错误，因为其操作保证可以工作（除非有某种系统故障）。如果键存在，它将返回其字节切片值。如果不存在，则返回nil。重要的是要注意，您可以将零长度的值设置为一个键，这与键不存在是不同的。

使用`Bucket.Delete()`函数从存储桶中删除键：

```go
db.Update(func (tx *bolt.Tx) error {
    b := tx.Bucket([]byte("MyBucket"))
    err := b.Delete([]byte("answer"))
    return err
})
```

这将从`MyBucket`存储桶中删除答案键。

请注意，从`Get()`返回的值只在事务打开时有效。如果您需要在事务外部使用值，则必须使用`copy()`将其复制到另一个字节切片。

### Autoincrementing integer for the bucket

通过使用`NextSequence()`函数，您可以让Bolt确定一个序列，该序列可以用作键/值对的唯一标识符。请参阅下面的示例。

```GO
// CreateUser saves u to the store. The new user ID is set on u once the data is persisted.
func (s *Store) CreateUser(u *User) error {
    return s.db.Update(func(tx *bolt.Tx) error {
        // Retrieve the users bucket.
        // This should be created when the DB is first opened.
        b := tx.Bucket([]byte("users"))

        // Generate ID for the user.
        // This returns an error only if the Tx is closed or not writeable.
        // That can't happen in an Update() call so I ignore the error check.
        id, _ := b.NextSequence()
        u.ID = int(id)

        // Marshal user data into bytes.
        buf, err := json.Marshal(u)
        if err != nil {
            return err
        }

        // Persist bytes to users bucket.
        return b.Put(itob(u.ID), buf)
    })
}

// itob returns an 8-byte big endian representation of v.
func itob(v int) []byte {
    b := make([]byte, 8)
    binary.BigEndian.PutUint64(b, uint64(v))
    return b
}

type User struct {
    ID int
    ...
}
```

### Iterating over keys

Bolt在存储桶内按字节排序存储其键。这使得对这些键进行顺序迭代非常快。要迭代键，我们将使用一个Cursor：

```go
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	b := tx.Bucket([]byte("MyBucket"))

	c := b.Cursor()

	for k, v := c.First(); k != nil; k, v = c.Next() {
		fmt.Printf("key=%s, value=%s\n", k, v)
	}

	return nil
})
```

游标允许您移动到键列表中的特定点，并一次向前或向后移动一个键。

以下函数在游标上可用：

- `First()` 移动到第一个键。
- `Last()` 移动到最后一个键。
- `Seek()` 移动到特定键。
- `Next()` 移动到下一个键。
- `Prev()` 移动到上一个键。 每个函数都有一个返回签名（key []byte, value []byte）。当您迭代到游标的末尾时，`Next()`将返回一个nil键。您必须使用`First()`、`Last()`或`Seek()`移动到某个位置，然后再调用`Next()`或`Prev()`。如果您没有移动到某个位置，那么这些函数将返回一个nil键。

在迭代过程中，如果键不是nil，但值是nil，那么意味着键引用的是一个存储桶而不是一个值。使用`Bucket.Bucket()`来访问子存储桶。

#### Prefix scans

要迭代键前缀，您可以结合使用`Seek()`和`bytes.HasPrefix()`：

```go
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	c := tx.Bucket([]byte("MyBucket")).Cursor()

	prefix := []byte("1234")
	for k, v := c.Seek(prefix); k != nil && bytes.HasPrefix(k, prefix); k, v = c.Next() {
		fmt.Printf("key=%s, value=%s\n", k, v)
	}

	return nil
})
```

#### Range scans

另一个常见的用例是扫描范围，例如时间范围。如果您使用可排序的时间编码，例如RFC3339，那么您可以像这样查询特定的日期范围：

```go
db.View(func(tx *bolt.Tx) error {
	// Assume our events bucket exists and has RFC3339 encoded time keys.
	c := tx.Bucket([]byte("Events")).Cursor()

	// Our time range spans the 90's decade.
	min := []byte("1990-01-01T00:00:00Z")
	max := []byte("2000-01-01T00:00:00Z")

	// Iterate over the 90's.
	for k, v := c.Seek(min); k != nil && bytes.Compare(k, max) <= 0; k, v = c.Next() {
		fmt.Printf("%s: %s\n", k, v)
	}

	return nil
})
```

请注意，虽然RFC3339是可排序的，但Golang的RFC3339Nano实现不使用小数点后固定数量的数字，因此不可排序。

#### ForEach()

如果您知道要在存储桶中迭代所有键，也可以使用`ForEach()`函数：

```GO
db.View(func(tx *bolt.Tx) error {
	// Assume bucket exists and has keys
	b := tx.Bucket([]byte("MyBucket"))

	b.ForEach(func(k, v []byte) error {
		fmt.Printf("key=%s, value=%s\n", k, v)
		return nil
	})
	return nil
})
```

请注意，ForEach()中的键和值仅在事务打开时有效。如果您需要在事务之外使用键或值，您必须使用copy()将其复制到另一个字节切片。

### Nested buckets

以下是给定文档的中文翻译：

请注意，ForEach()中的键和值仅在事务打开时有效。如果您需要在事务之外使用键或值，您必须使用copy()将其复制到另一个字节切片。

**嵌套的桶** 您还可以在键中存储一个桶，以创建嵌套的桶。该API与DB对象上的桶管理API相同：

```go
func (*Bucket) CreateBucket(key []byte) (*Bucket, error)
func (*Bucket) CreateBucketIfNotExists(key []byte) (*Bucket, error)
func (*Bucket) DeleteBucket(key []byte) error
```

假设您有一个多租户应用程序，其中根级别的桶是帐户桶。在此桶内是一系列自身为桶的帐户。在这个序列桶里，你可以拥有许多与账户自身相关的桶（如用户、笔记等），将信息隔离到逻辑分组中。

```go
// createUser 在给定账户中创建一个新用户。
func createUser(accountID int, u *User) error {
    // 开始事务。
    tx, err := db.Begin(true)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // 检索帐户的根桶。
    // 假设在设置帐户时已经创建了此桶。
    root := tx.Bucket([]byte(strconv.FormatUint(accountID, 10)))

    // 设置用户桶。
    bkt, err := root.CreateBucketIfNotExists([]byte("USERS"))
    if err != nil {
        return err
    }

    // 为新用户生成ID。
    userID, err := bkt.NextSequence()
    if err != nil {
        return err
    }
    u.ID = userID

    // 序列化并保存编码的用户。
    if buf, err := json.Marshal(u); err != nil {
        return err
    } else if err := bkt.Put([]byte(strconv.FormatUint(u.ID, 10)), buf); err != nil {
        return err
    }

    // 提交事务。
    if err := tx.Commit(); err != nil {
        return err
    }

    return nil
}
```

### 数据库备份

Bolt是一个单一的文件，所以备份很容易。您可以使用Tx.WriteTo()函数将数据库的一致视图写入一个写入器。如果您从只读事务中调用此函数，它将执行一个热备份，并且不会阻止您的其他数据库读取和写入。

默认情况下，它将使用一个常规的文件句柄，该句柄将利用操作系统的页面缓存。请查看Tx文档，了解有关优化大于RAM数据集的信息。

一个常见的用例是通过HTTP进行备份，这样您就可以使用像cURL这样的工具进行数据库备份：

```go
func BackupHandleFunc(w http.ResponseWriter, req *http.Request) {
    err := db.View(func(tx *bolt.Tx) error {
        w.Header().Set("Content-Type", "application/octet-stream")
        w.Header().Set("Content-Disposition", `attachment; filename="my.db"`)
        w.Header().Set("Content-Length", strconv.Itoa(int(tx.Size())))
        _, err := tx.WriteTo(w)
        return err
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

然后你可以使用这个命令备份：

```go
$ curl http://localhost/backup > my.db
```

或者您可以打开您的浏览器访问http://localhost/backup，它将自动下载。

如果您想备份到另一个文件，您可以使用Tx.CopyFile()辅助函数。

**统计信息** 数据库保持对其执行的许多内部操作的持续计数，这样您可以更好地了解正在发生的事情。通过在两个时间点获取这些统计的快照，我们可以看到在这个时间范围内执行了哪些操作。

例如，我们可以启动一个goroutine，每10秒记录一次统计信息：

```go
go func() {
    // 获取初始统计。
    prev := db.Stats()

    for {
        // 等待10秒。
        time.Sleep(10 * time.Second)

        // 获取当前统计并对其进行差异处理。
        stats := db.Stats()
        diff := stats.Sub(&prev)

        // 将统计信息编码为JSON并打印到STDERR。
        json.NewEncoder(os.Stderr).Encode(diff)

        // 为下一个循环保存统计。
        prev = stats
    }
}()
```

将这些统计信息管道到像statsd这样的服务进行监控，或者提供一个HTTP端点来执行固定长度的样本，也很有用。

### 只读模式

有时创建一个共享的、只读的Bolt数据库是很有用的。要做到这一点，打开数据库时设置Options.ReadOnly标志。只读模式使用共享锁，允许多个进程从数据库中读取，但它会阻止任何进程以读写模式打开数据库。

```go
db, err := bolt.Open("my.db", 0600, &bolt.Options{ReadOnly: true})
if err != nil {
    log.Fatal(err)
}
```

希望这可以帮助您理解给定的文档！如果您有任何问题或需要进一步的澄清，请告诉我。

## referance

- https://github.com/etcd-io/bbolt
- https://zhuanlan.zhihu.com/p/377572049
