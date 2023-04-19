---
title: "Xdebug使用Dbgp协议与PHPStorm通信过程"
date: 2023-03-29T09:40:02+08:00
draft: false
categories: ["PHP", "Debug"]
tags: ["Xdebug", "Debug", "Dbgp", "PHPStorm"]
---

1. 向 IDE 发起连接请求

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<init appid="1" idekey="PHPSTORM" language="PHP" protocol_version="1.0" fileuri=""
    xmlns="urn:debugger_protocol_v1">
    <engine version="1.0.0">
        <![CDATA[SDB]]>
    </engine>
    <author>
        <![CDATA[Chance]]>
    </author>
</init>
```

2. feature_set -i 1 -n show_hidden -v 1

- feature_set：命令名称，用于设置调试器的功能。
- -i 1：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -n show_hidden：要设置的功能名称，即显示隐藏变量。
- -v 1：功能的值，表示要显示隐藏变量。
  回复：
  ```xml
  <?xml version="1.0" encoding="ISO-8859-1"?>
  <response xmlns="urn:debugger_protocol_v1" command="feature_set" transaction_id="1" feature="show_hidden" success="1"/>
  ```

3. stdout -i 8 -c 1

- stdout：命令名称，用于将输出发送到调试器的控制台。
- -i 8：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -c 1：输出的内容类型，表示输出的是文本内容。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="stdout" transaction_id="8" success="1"/>
  ```

4. status -i 9

- status：命令名称，用于查询调试器的状态。
- -i 9：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="status" transaction_id="9" status="starting" reason="ok"/>
  ```

5. step_into -i 10

- step_into：命令名称，用于让调试器进入下一步的代码。
- -i 10：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="step_into" transaction_id="10" status="break" reason="ok"/>
  ```

6. eval -i 11 -- aXNzZXQoJF9TRVJWRVJbJ1BIUF9JREVfQ09ORklHJ10p

- eval：命令名称，用于在调试器中执行代码。
- -i 11：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -- aXNzZXQoJF9TRVJWRVJbJ1BIUF9JREVfQ09ORklHJ10p：要执行的代码，base64 编码。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="eval" transaction_id="11"><property type="bool"><![CDATA[0]]></property></response>
  ```
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="eval" transaction_id="13"><property type="string" size="9" encoding="base64"><![CDATA[MTI3LjAuMC4x]]></property></response>
  ```

7. breakpoint_set -i 16 -t line -f file:///home/test.php -n 20

- breakpoint_set：命令名称，用于设置断点。
- -i 16：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -t line：断点的类型，表示设置的是行断点。
- -f file:///home/test.php：要设置断点的文件路径，以 file:// 开头，表示本地文件系统路径。
- -n 20：要设置断点的行号，即在指定文件的第 20 行设置断点。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="breakpoint_set" transaction_id="16" id="0"/>
  ```

8. stack_get -i 16

- stack_get：命令名称，用于获取当前调用栈的信息。
- -i 16：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="stack_get" transaction_id="19"><stack where="Chance\SwowDebug\Debugger::breakPointHandler" level="0" type="file" filename="file:///home/test.php" lineno="20"/></response>
  ```

9. context_names -i 20

- context_names：命令名称，用于获取当前上下文环境的名称列表。
- -i 20：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
  回复：
  ```xml
  <?xml version="1.0" encoding="ISO-8859-1"?>
  <response xmlns="urn:debugger_protocol_v1" command="context_names" transaction_id="20"><context name="Local" id="1"/></response>
  ```

10. context_get -i 21 -d 0 -c 1

- context_get：命令名称，用于获取指定上下文环境的变量信息。
- -i 21：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -d 0：要获取变量信息的上下文环境的编号，0 表示当前上下文环境。
- -c 1：要获取变量信息的层数，表示要获取当前上下文环境下的一层变量信息。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="context_get" transaction_id="21"><property name="_GET" fullname="$_GET" type="array" children="1" numchildren="0"/><property name="_POST" fullname="$_POST" type="array" children="1" numchildren="0"/><property name="_COOKIE" fullname="$_COOKIE" type="array" children="1" numchildren="0"/><property name="_FILES" fullname="$_FILES" type="array" children="1" numchildren="0"/><property name="argv" fullname="$argv" type="array" children="1" numchildren="1"/><property name="argc" fullname="$argc" type="integer"><![CDATA[1]]></property><property name="_SERVER" fullname="$_SERVER" type="array" children="1" numchildren="26"/><property name="a" fullname="$a" type="integer"><![CDATA[1]]></property><property name="__composer_autoload_files" fullname="$__composer_autoload_files" type="array" children="1" numchildren="6"/></response>
  ```

11. property_get -i 22 -n $a -d 0 -c 1 -p 0

- property_get：命令名称，用于获取指定变量的属性信息。
- -i 22：命令的唯一标识符，用于在调试器和 IDE 之间进行通信。
- -n $a：要获取属性信息的变量名，以 $ 开头表示变量名。
- -d 0：要获取属性信息的上下文环境的编号，0 表示当前上下文环境。
- -c 1：要获取属性信息的层数，表示要获取当前上下文环境下的一层属性信息。
- -p 0：要获取的属性的编号，0 表示获取所有属性信息。
  回复：
  ```xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <response xmlns="urn:debugger_protocol_v1" command="property_get" transaction_id="22"><property name="$a" fullname="$a" type="array" children="1" numchildren="4"><property name="0" fullname="$a['0']" type="integer"><![CDATA[1]]></property><property name="1" fullname="$a['1']" type="integer"><![CDATA[2]]></property><property name="2" fullname="$a['2']" type="integer"><![CDATA[3]]></property><property name="3" fullname="$a['3']" type="integer"><![CDATA[4]]></property></property></response>
  ```

12. step_over -i 21
    run -i 22

- step_over -i 21：这个命令让调试器执行一行代码，跳过函数调用。其中 -i 21 表示这个命令的标识符是 21，用于在调试器和客户端之间唯一标识这个命令。
- run -i 22：这个命令让调试器开始运行代码，并在遇到断点或程序结束时停止。其中 -i 22 表示这个命令的标识符是 22，用于在调试器和客户端之间唯一标识这个命令。
