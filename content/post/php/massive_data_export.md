---
title: "Massive_data_export"
date: 2022-05-19T19:43:58+08:00
draft: true
categories: []
tags: []
---

### 背景

公司业务部门经常会需要导出各种数据进行分析，因为需要的数据模板多变，产品一直没有将其规划成系统功能，每次需要的数据都是手写 SQL 语句进行导出。在写了上万行 SQL 语句以后，我对这种体力劳动感觉到了厌烦，于是和领导申请，自己利用空闲时间规划开发了一个灵活配置、适用性强的导表系统。

### 使用技术

+ API 框架 [Hyperf](https://www.hyperf.wiki/2.2/#/zh-cn/)
+ Web 框架 [Ant Design of React](https://ant.design/docs/react/introduce-cn)
+ Excel 扩展 [xlsWriter](https://xlswriter.viest.me/)

### 优化

为了尽快实现功能，导出 Excel 功能是做的同步导出，但在我的设计中应该是异步导出，因此在后续的优化中引入了 RabbitMQ 消息队列实现异步导出功能。

#### 一

因为不能将 Db 对象投递到消息队列中，所以投递过去的是 SQL 的预处理语句以及对应的变量，使用`Db::select($sql, $bindings);`执行 SQL 。然后尝试查询百万条数据，直接就内存溢出了。

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

点进`select`函数的源码查看发现，使用了`fetchAll`来获取数据，`fetchAll`会将查询的所有结果集全部返回，百万条数据量读取到内存中就导致内存溢出了。在`select`函数下面发现有一个`cursor`函数，两者入参相同，但是是使用`fetch`+`yield`获取数据，每次只读取一条数据。

![image-20220520172431168](https://image.chance.fyi/image-20220520172431168.png)

```php
$name = md5(Carbon::now()->toDateTimeString() . Str::random());
$excel = $excel->constMemory("$name.xlsx", "Sheet1", false);
$header = [];
$row = 0;
$sheet = 2;
foreach (Db::connection('export')->cursor($sql, $bindings) as $item) {
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

if ($excel->addSheet("Sheet$sheet")
    ->data([[$export->sql]])
    ->output()) {
    $export->status = 1;
} else {
    $export->status = -1;
}
```

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

