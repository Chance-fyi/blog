---
title: "PHP百万数据导出Excel"
date: 2022-05-19T19:43:58+08:00
draft: false
categories: ["PHP"]
tags: ["PHP","Swoole","xlsWriter","RabbitMQ","协程","导出"]
---

### 背景

公司业务部门经常会需要导出各种数据进行分析，因为需要的数据模板多变，产品一直没有将其规划成系统功能，每次需要的数据都是手写 SQL 语句进行导出。在写了上万行 SQL 语句以后，我对这种体力劳动感觉到了厌烦，于是和领导申请，自己利用空闲时间规划开发了一个灵活配置、适用性强的导表系统。

### 使用技术

+ API 框架 [Hyperf](https://www.hyperf.wiki/2.2/#/zh-cn/)
+ Web 框架 [Ant Design Pro](https://pro.ant.design/zh-CN)
+ Excel 扩展 [xlsWriter](https://xlswriter.viest.me/)

### 优化

为了尽快实现功能，导出 Excel 功能是做的同步导出，但在我的设计中应该是异步导出，因此在后续的优化中引入了 RabbitMQ 消息队列实现异步导出功能。

因为不能将 Db 对象投递到消息队列中，所以投递过去的是 SQL 的预处理语句以及对应的变量，使用`Db::select($sql, $bindings);`执行 SQL 。然后尝试查询百万条数据，直接就内存溢出了。

**vendor/hyperf/database/src/Connection.php 271行**

```php
/**
 * Run a select statement against the database.
 */
public function select(string $query, array $bindings = [], bool $useReadPdo = true): array
{
    return $this->run($query, $bindings, function ($query, $bindings) use ($useReadPdo) {
        if ($this->pretending()) {
            return [];
        }

        // For select statements, we'll simply execute the query and return an array
        // of the database result set. Each element in the array will be a single
        // row from the database table, and will either be an array or objects.
        $statement = $this->prepared($this->getPdoForSelect($useReadPdo)
            ->prepare($query));

        $this->bindValues($statement, $this->prepareBindings($bindings));

        $statement->execute();

        return $statement->fetchAll();
    });
}
```

点进`select`函数查看源码发现，是使用了`fetchAll`来获取数据，`fetchAll`会将查询的所有结果集全部返回，百万条数据量读取到内存中就导致内存溢出了。

在`select`函数下面发现有一个`cursor`函数，两者入参相同，但是是使用`fetch`+`yield`来获取数据，每次只读取一条数据。

**vendor/hyperf/database/src/Connection.php 295行**

```php
/**
 * Run a select statement against the database and returns a generator.
 */
public function cursor(string $query, array $bindings = [], bool $useReadPdo = true): \Generator
{
    $statement = $this->run($query, $bindings, function ($query, $bindings) use ($useReadPdo) {
        if ($this->pretending()) {
            return [];
        }

        // First we will create a statement for the query. Then, we will set the fetch
        // mode and prepare the bindings for the query. Once that's done we will be
        // ready to execute the query against the database and return the cursor.
        $statement = $this->prepared($this->getPdoForSelect($useReadPdo)
            ->prepare($query));

        $this->bindValues(
            $statement,
            $this->prepareBindings($bindings)
        );

        // Next, we'll execute the query against the database and return the statement
        // so we can return the cursor. The cursor will use a PHP generator to give
        // back one row at a time without using a bunch of memory to render them.
        $statement->execute();

        return $statement;
    });

    while ($record = $statement->fetch()) {
        yield $record;
    }
}
```

![image-20220520172431168](https://image.chance.fyi/image-20220520172431168.png)

### 百万数据导出测试

使用此方法配合 xlsWriter 的[固定内存模式](https://xlswriter-docs.viest.me/zh-cn/nei-cun/gu-ding-nei-cun-mo-shi)导出的话，应该就可以完成百万数据的导出了。代码如下:

```php
$config = [
    'path' => BASE_PATH . "/public/export" // xlsx文件保存路径
];
$excel = new Excel($config);
$name = md5(Carbon::now()->toDateTimeString() . Str::random());
$excel = $excel->constMemory("$name.xlsx", "Sheet1", false);
$header = [];
$row = 0;
$sheet = 2;
foreach (Db::connection('export')->cursor($sql, $bindings) as $item) {
    $item = get_object_vars($item);
    $header = $header ?: array_keys($item);
    if ($row === 1048576) { // 超过最大行 新增工作簿
        $excel = $excel->addSheet("Sheet$sheet");
        $sheet++;
        $row = 0;
    }
    if ($row === 0) { // 写入标题
        $excel = $excel->header($header);
        $row++;
    }
    $column = 0;
    foreach ($item as $v) {
        $excel = $excel->insertText($row, $column, $v);
        $column++;
    }
    $row++;
}

$excel->output();
```

尝试导出 140w 数据，发现代码报错了，但是 Excel 文件成功导出了。消费者成功回复了 ack ，但是 MQ 却又一次推送了消费消息，就这样一直死循环的导出文件。在根据报错信息，调试查看源码半天之后，我的 C 盘空间被干满了:cry:，猜测应该是 Docker 日志太多造成的，在将所有容器和镜像删除之后，C 盘占用就下去了，并将镜像文件移至 D 盘保存。

### 发现问题并解决

再次构建好环境复现以后，去查看了一下 MQ 的日志输出，发现很明显的提示了报错信息。原因就是 **MQ 没有如期收到心跳包，消费者错过了心跳，导致 MQ 认为消费者已经不可用了，断开了链接并重新推送了消费消息**。

查看发送心跳的源码发现是使用协程实现的，开了一个协程来定时发送心跳数据包。

**vendor/hyperf/amqp/src/AMQPConnection.php 308行**

```php
protected function heartbeat(): void
{
    if (! $this->enableHeartbeat && $this->getHeartbeat() > 0) {
        $this->enableHeartbeat = true;

        Coroutine::create(function () {
            while (true) {
                if (CoordinatorManager::until(Constants::WORKER_EXIT)->yield($this->getHeartbeat())) {
                    $this->exited = true;
                    $this->close();
                    break;
                }

                try {
                    // PING
                    if ($this->isConnected() && $this->chan->isEmpty()) {
                        $pkt = new AMQPWriter();
                        $pkt->write_octet(8);
                        $pkt->write_short(0);
                        $pkt->write_long(0);
                        $pkt->write_octet(0xCE);
                        $this->chan->push($pkt->getvalue(), 0.001);
                    }
                } catch (\Throwable $exception) {
                    $this->logger && $this->logger->error((string) $exception);
                }
            }
        });
    }
}
```

虽然 Swoole 的[一键协程化](https://wiki.swoole.com/#/runtime?id=swoole_hook_file) hook 掉了文件相关的函数，可以将 PHP 的文件操作函数从同步 IO 转化为异步 IO ，但是我们导出使用的 xlsWriter 是 C 扩展实现的，所以产生的文件 IO 并不能被 Swoole 给 hook 掉，依然是同步 IO ，这将会阻塞起心跳协程，使其不能按时发送心跳数据包，导致 MQ 端口链接。

解决方法也很简单，只要**增加心跳间隔**就可以了。

### 尝试导出千万数据

又导出失败了，内存溢出。看来`cursor`也是有极限的，依然会溢出，多次尝试之后发现还是一百万条数据比较合适。想要导出更多的数据只能分页来查询，代码如下。

```php
$db = Db::connection('export');
$sql = $data['sql'];
$bindings = $data['bindings'];
$pageSize = 1000000;
$total = $db->select("select count(*) as total from (" . $sql . ") as s", $bindings)[0]->total;
$pageNum = ceil($total / $pageSize);

$name = md5(Carbon::now()->toDateTimeString() . Str::random());
$excel = $excel->constMemory("$name.xlsx", "Sheet1", false);
$header = [];
$row = 0;
$sheet = 2;
for ($page = 0; $page < $pageNum; $page++) {
    foreach ($db->cursor("$sql limit " . $page * $pageSize . ",$pageSize", $bindings) as $item) {
        $item = get_object_vars($item);
        $header = $header ?: array_keys($item);
        if ($row === 1048576) {
            $excel = $excel->addSheet("Sheet$sheet");
            $sheet++;
            $row = 0;
        }
        if ($row === 0) {
            $excel = $excel->header($header);
            $row++;
        }
        $column = 0;
        foreach ($item as $v) {
            $excel = $excel->insertText($row, $column, $v);
            $column++;
        }
        $row++;
    }
}

if ($excel->addSheet("Sheet$sheet")
    ->data([[$export->sql]])
    ->output()) {
    $export->status = 1;
} else {
    $export->status = -1;
}
```

再次尝试导出成功了，不出所料 MQ 又断开了，不得不说 xlsWriter 的固定内存模式导出真的厉害，一千万条数据也可以导出来:thumbsup:。
