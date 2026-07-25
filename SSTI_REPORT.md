# SSTI（服务器端模板注入）漏洞检测与修复报告

**项目名称：** 用户信息管理平台  
**评估日期：** 2026年7月25日  
**报告作者：** chilianzhe1470  
**仓库地址：** https://github.com/chilianzhe1470/Classrepair  
**密级：** 内部 — 仅限教学使用

---

## 1. 总体摘要

对 Flask 用户管理应用中的**欢迎页**（`/welcome`）和**反馈功能**（`/feedback`）进行了 SSTI（服务器端模板注入）漏洞检测。两处功能均使用 `render_template_string` 并直接将用户输入拼接到模板字符串中，存在**严重的 SSTI 漏洞**，可导致远程代码执行（RCE）。

**检测结果：**

| 漏洞类型 | 严重程度 | CWE 编号 | 修复版状态 |
|---------|:--------:|:--------:|:----------:|
| SSTI — 欢迎页表达式注入 | 🔴 严重 | [CWE-1336](https://cwe.mitre.org/data/definitions/1336.html) | ✅ 已修复 |
| SSTI — 欢迎页读取配置 | 🔴 严重 | [CWE-1336](https://cwe.mitre.org/data/definitions/1336.html) | ✅ 已修复 |
| SSTI — 反馈页表达式注入 | 🔴 严重 | [CWE-1336](https://cwe.mitre.org/data/definitions/1336.html) | ✅ 已修复 |

---

## 2. 新增功能说明

### 2.1 欢迎页 `/welcome`

| 属性 | 漏洞版(5000) | 修复版(8080) |
|------|:-----------:|:-----------:|
| 渲染方式 | `render_template_string(f"...{name}...")` | `render_template_string("...{{ name }}...", name=name)` |
| 用户输入处理 | 直接拼接到模板字符串 | 作为变量传递给模板引擎 |
| 是否存在 SSTI | ❌ 存在 | ✅ 已修复 |

### 2.2 反馈功能 `/feedback`

| 属性 | 漏洞版(5000) | 修复版(8080) |
|------|:-----------:|:-----------:|
| 渲染方式 | `render_template_string(f"...{name}...{message}...")` | `render_template_string("...{{ name }}...", name=name, message=message)` |
| 注入点 | `name` 和 `message` 两个参数 | 无（变量传递） |
| 是否存在 SSTI | ❌ 存在 | ✅ 已修复 |

---

## 3. 漏洞详情

### 3.1 V-SSTI-01：欢迎页 SSTI — 基础表达式注入

| 属性 | 内容 |
|------|------|
| **CWE 编号** | [CWE-1336](https://cwe.mitre.org/data/definitions/1336.html) |
| **CVSS 3.1** | 9.8（严重） |
| **风险描述** | 欢迎页将用户输入的 `name` 参数直接拼接到 `render_template_string` 的模板字符串中。Flask 的 Jinja2 引擎会解析 `{{ ... }}` 语法，攻击者可注入任意模板表达式。 |

#### 攻击场景 1：基础算术表达式探测

```bash
# 输入 {{7*7}}
curl -s "http://target:5000/welcome?name=%7B%7B7*7%7D%7D"
# 页面显示：欢迎你，49！
# 说明 SSTI 漏洞存在（服务器计算了 7*7=49）
```

#### 攻击场景 2：读取 Flask 配置

```bash
# 输入 {{config}}
curl -s "http://target:5000/welcome?name=%7B%7Bconfig%7D%7D"
# 页面泄露：SECRET_KEY = "dev-key-2025" 等配置信息
```

#### 攻击场景 3：远程代码执行（RCE）

```bash
# 通过 Python 对象链执行系统命令
curl -s "http://target:5000/welcome?name=%7B%7B''.__class__.__mro__[2].__subclasses__()%7D%7D"
```

#### 漏洞代码

```python
@app.route("/welcome")
def welcome():
    name = request.args.get("name", "亲爱的用户")
    # 漏洞：直接拼接用户输入到模板字符串
    return render_template_string(f"<h1>欢迎你，{name}！</h1>")
```

---

### 3.2 V-SSTI-02：反馈页 SSTI

| 属性 | 内容 |
|------|------|
| **CWE 编号** | [CWE-1336](https://cwe.mitre.org/data/definitions/1336.html) |
| **CVSS 3.1** | 9.8（严重） |
| **风险描述** | 反馈页的 `name` 和 `message` 参数均直接拼接到模板字符串中，攻击者可通过任一个参数注入模板代码。 |

#### 攻击场景

```bash
# 在留言内容中注入模板表达式
curl -s -X POST http://target:5000/feedback \
  --data-urlencode "name=test" \
  --data-urlencode "message={{7*7}}"
# 页面显示：49（7*7 的计算结果）
```

---

## 4. 检测过程与结果

### 4.1 测试工具

- **工具：** `curl` 命令行
- **目标漏洞版：** http://192.168.145.130:5000
- **目标修复版：** http://192.168.145.130:8080

### 4.2 测试结果

| # | 测试项 | 漏洞版(5000) | 修复版(8080) |
|:-:|:-------|:-----------:|:-----------:|
| 1 | 正常访问 `/welcome?name=张三` | ✅ 正常显示 | ✅ 正常显示 |
| 2 | SSTI 探测 `{{7*7}}` | ✅ 返回 `49`（注入成功） | ✅ 返回原文 `{{7*7}}`（安全） |
| 3 | 读取配置 `{{config}}` | ✅ 泄露 `SECRET_KEY` | ✅ 显示原文（安全） |
| 4 | 反馈 SSTI `{{7*7}}` | ✅ 返回 `49`（注入成功） | ✅ 显示原文（安全） |

---

## 5. 代码对比

### 5.1 漏洞版（5000 端口）

```python
@app.route("/welcome")
def welcome():
    name = request.args.get("name", "亲爱的用户")
    # ❌ 直接拼接用户输入到模板字符串
    return render_template_string(f"<h1>欢迎你，{name}！</h1>")


@app.route("/feedback", methods=["GET", "POST"])
def feedback():
    if request.method == "POST":
        name = request.form.get("name", "")
        message = request.form.get("message", "")
        # ❌ 直接拼接用户输入到模板字符串
        return render_template_string(f"<h2>{name} 的反馈：</h2><p>{message}</p>")
```

### 5.2 修复版（8080 端口）

```python
@app.route("/welcome")
def welcome():
    name = request.args.get("name", "亲爱的用户")
    # ✅ 通过变量传递，不拼接用户输入
    return render_template_string("<h1>欢迎你，{{ name }}！</h1>", name=name)


@app.route("/feedback", methods=["GET", "POST"])
def feedback():
    if request.method == "POST":
        name = request.form.get("name", "")
        message = request.form.get("message", "")
        # ✅ 通过变量传递，不拼接用户输入
        return render_template_string(
            "<h2>{{ name }} 的反馈：</h2><p>{{ message }}</p>",
            name=name, message=message
        )
```

### 5.3 修复要点

| 安全措施 | 漏洞版 | 修复版 |
|:---------|:-----:|:-----:|
| 渲染方式 | f-string 拼接 | 变量传递 |
| 用户输入位置 | 模板字符串中 | `render_template_string` 参数中 |
| Jinja2 解析 | 解析用户输入的模板语法 | 只解析预定义的 `{{ name }}` |
| `{{7*7}}` 注入 | 返回 `49`（执行） | 返回 `{{7*7}}`（原文） |
| `{{config}}` 注入 | 泄露配置 | 显示原文 |

---

## 6. 安全建议

1. **禁止拼接用户输入到模板字符串** — 永远不要使用 `render_template_string(f"...{user_input}...")`
2. **使用变量传递** — 所有用户输入应作为 `render_template_string` 或 `render_template` 的变量参数传递
3. **优先使用静态模板文件** — 用 `render_template` 配合独立的 .html 模板文件，而非 `render_template_string`
4. **避免 `render_template_string`** — 除非完全确定模板内容是静态的，否则优先使用 `render_template`
5. **最小权限原则** — 应用使用最小系统权限运行，减少 SSTI 被利用后的影响范围
6. **沙箱模式** — 如必须使用动态模板，考虑使用 Jinja2 的沙箱模式（`SandboxedEnvironment`）

---

## 7. 结论

本次评估发现并修复了 2 处 SSTI 漏洞：

| 漏洞类型 | CWE | CVSS | 状态 |
|---------|:---:|:----:|:----:|
| 欢迎页 SSTI 表达式注入 | 1336 | 9.8（严重） | ✅ 已修复 |
| 反馈页 SSTI 表达式注入 | 1336 | 9.8（严重） | ✅ 已修复 |

修复前，攻击者可通过 `{{7*7}}` 探测漏洞，通过 `{{config}}` 读取应用配置（含 Secret Key），通过 Python 对象链实现远程代码执行，完全控制服务器。修复后，用户输入仅作为普通变量传递，即使在输入中包含 `{{7*7}}` 也不会被 Jinja2 解析执行，仅作为纯文本显示。

---

*本报告作为网络安全实训课程作业提交。所有测试均在授权环境下进行。*
