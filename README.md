# OCI 永久免费 Arm 实例自动抢购（IntNet-Proxy）

本仓库利用 **GitHub Actions** 定时调用你在 Oracle Cloud（OCI）上**已保存的 Stack（Resource Manager）**，反复尝试创建一台永久免费（Always Free）的 `VM.Standard.A1.Flex`（Ampere/ARM）实例，直到抢到为止。

## 为什么需要"抢"？

OCI 的永久免费 ARM 实例因为太热门，常年处于 `Out of host capacity`（主机容量不足）状态，手动创建很难成功。本项目让 GitHub 每 5 分钟自动帮你跑一次 apply，成功率大大提升。

> 配置区域：**Japan Central (Osaka)**，即 `ap-osaka-1`。
> 实例规格（由你的 Stack 内部固化，本项目不覆盖）：Ampere A1.Flex、4 OCPU / 24 GB、Ubuntu 22.04、200GB 启动盘、主机名 **IntNet-Proxy**。网络、SSH 密钥均已内置在你保存的 Stack 里。

---

## 通知机制（重点）

GitHub Actions 默认**只在工作流运行失败时**给你发邮件通知。本项目正好利用这一特性，让通知完全符合你的要求：

- **没抢到**（apply 作业失败，且经日志确认为无主机容量）→ 工作流保持**成功** → **不打扰你**，后台静默每 5 分钟重试。
- **抢到了**（apply 作业成功）→ 工作流**故意失败**（`exit 1`）→ **触发 GitHub 邮件通知你**。
- **配置/凭据/网络出错**（apply 失败，且日志**不是**"无容量"）→ 工作流同样**故意失败** → **邮件告警**，让你及时知道"不是抢不到，而是坏了"，而不是静默空等。

也就是说：**只有"无容量"这一种情况会静默重试**；真正的成功和真正的异常都会发邮件。

---

## 两个内置的自动化防御

- **失败原因区分**：apply 失败后，工作流会自动拉取该作业的 Terraform 日志，判断失败原因是 `Out of host capacity`（无容量，静默）还是其它错误（告警）。
- **自动防重复创建**：每次运行开头都会先检查当前是否存在名为 `IntNet-Proxy` 的实例。**一旦抢到过（实例已存在），后续运行会自动跳过创建、直接结束**，不会重复建机器、也不会重复发邮件。即使你忘了禁用工作流，它也不会再二次创建。

---

## 使用步骤

### 第 1 步：Fork 本仓库

点击仓库右上角的 **Fork**，在你自己账号下创建一份副本。之后的配置都在你的副本上进行。

### 第 2 步：准备 OCI 信息

登录 OCI 控制台，确认你在 **Japan Central (Osaka)** 区域，并收集以下信息：

1. **Tenancy OCID**：点头像 → **Tenancy: [名字]** → 复制 **OCID**。（本项目中它同时充当 compartment）
2. **User OCID**：点头像 → **User Settings** → 复制 **OCID**。
3. **API 私钥 & 指纹**：见下方"获取 API 密钥与 fingerprint"。
4. **Stack OCID**：进入 **Developer Services** → **Resource Manager** → **Stacks**，打开你保存的那个 Stack，复制它的 **OCID**（形如 `ocid1.stack.oc1.ap-osaka-1.xxx...`）。

### 获取 API 密钥与 fingerprint

指纹（fingerprint）**不是**在向导里现场生成的，而是 OCI 根据你的公钥算出来的，**要等你点"Add"提交之后才会显示**。按以下步骤操作：

1. 进入 **User Settings** → **Resources**（左侧）→ **API Keys** → 点 **Add API Key**。
2. 在向导中选择 **Generate API key pair** → 点 **Generate API key pair** → 点 **Download private key** 下载 `oci_api_key.pem`，并妥善保存。
   - ⚠️ 私钥只会给你下载一次；下载后可顺手 **Download public key** 留作备用。
3. 点向导底部的 **Add** 按钮提交。
4. 提交后 OCI 会弹出 **Configuration File Preview** 窗口，里面有一行：
   ```
   fingerprint=8f:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx
   ```
   把 `8f:...` 这一段复制下来，就是你要填的 `OCI_CLI_FINGERPRINT`。
5. 如果弹窗已关闭：回到 **API Keys** 列表，找到刚才的 key，点右侧 **"..."** → 选 **View Configuration File**（或 Copy Configuration），里面的 `fingerprint=` 值即是它。

> 💡 备选（本地推算 fingerprint）：在有 OpenSSL 的环境（如 Git Bash / WSL）下运行
> `openssl rsa -pubout -outform DER -in oci_api_key.pem | openssl md5 -c`，
> 输出括号里的那串即 fingerprint，与控制台显示一致。

### 配置 GitHub Secrets 的详细说明

在配置 Secret 前，先说明两个最容易出错的字段，避免填错导致鉴权失败：

**`OCI_CLI_FINGERPRINT`**：填上面第 4/5 步拿到的 `8f:xx:...` 那串（含冒号），不含 `fingerprint=` 前缀。

**`OCI_CLI_KEY_CONTENT`**：必须填 `oci_api_key.pem` 文件的**完整内容**，且**必须包含首尾两行**，缺失这两行会让私钥文件结构不完整、解析失败：
```
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDi...
（中间所有字符原样保留，不要省略）
-----END PRIVATE KEY-----
```
从 `-----BEGIN PRIVATE KEY-----` 那一行开始复制，到 `-----END PRIVATE KEY-----` 那一行结束，**整段粘贴**进填写框，前后不要多加空格或空行。

### 第 3 步：配置 GitHub Secrets（敏感信息，不会进入仓库）

> ⚠️ 请确认你的 Stack 是在 Osaka 区创建，且其中的实例配置为：A1.Flex、4 OCPU / 24GB、Ubuntu 22.04、200GB、主机名 IntNet-Proxy。

到你 fork 的仓库：**Settings** → **Secrets and variables** → **Actions** → **New repository secret**，逐个添加以下 6 个：

| Secret 名称 | 值 |
|-------------|----|
| `STACK_OCID` | 上方拿到的 **Stack OCID** |
| `OCI_CLI_REGION` | `ap-osaka-1` |
| `OCI_CLI_USER` | 你的 **User OCID** |
| `OCI_CLI_TENANCY` | 你的 **Tenancy OCID**（同时作为 compartment） |
| `OCI_CLI_FINGERPRINT` | 上文拿到的 `8f:...` 那串 fingerprint |
| `OCI_CLI_KEY_CONTENT` | `oci_api_key.pem` 文件的**完整内容**（**必须含** `-----BEGIN PRIVATE KEY-----` 和 `-----END PRIVATE KEY-----` 两行） |

### 第 4 步：开启失败邮件通知

1. GitHub 右上角点头像 → **Settings** → **Notifications**。
2. 在 **Actions** 相关区域，将邮件通知设置为**仅在失败时通知**（例如只勾选 "Failures"，不要勾选 "Success" 和 "Warnings"）。
3. 确认你 GitHub 绑定的邮箱能正常接收 GitHub 通知邮件。

> 💡 务必保证“成功”不通知、“失败”才通知——这正是本项目“抢到才提醒”的关键前提。

### 第 5 步：如何验证配置是否正常（详细指南）

先手动触发一次，确认整套链路没问题，再交给定时任务。验证分三个层级，从浅到深排查。

#### 5.1 触发一次手动运行

1. 进入仓库 **Actions** 页签，左侧点 **Try to Create OCI VM (IntNet-Proxy)**。
2. 右侧点 **Run workflow** → 再次点 **Run workflow**，就会开始一次运行。
3. 点开这次运行，逐个展开步骤看日志（下文的每一步都能看到对应名字）。

#### 5.2 逐级看日志，判断卡在哪一层

**第 1 层：OCI CLI 是否配置成功**
- 看 **安装 OCI CLI 并写入 OCI 配置** 这一步。
- ✅ 正常：日志末尾出现 `OCI CLI 配置完成，区域：ap-osaka-1`。
- ❌ 报错：多为私钥格式问题或网络。检查 `OCI_CLI_KEY_CONTENT` 是否完整粘贴（包含 `-----BEGIN PRIVATE KEY-----` 与 `-----END PRIVATE KEY-----`）。

**第 2 层：鉴权 / Stack 是否有效（含实例存在性检测）**
- 看 **触发 Stack Apply 作业并轮询结果** 这一步的开头。
- ✅ 正常：首先出现 `未发现已存在的 IntNet-Proxy 实例，继续发起创建。`，随后打印 `Apply 作业 ID: ocid1.job...`。
- ⚠️ 如果开头直接打印 `检测到已存在目标实例（IntNet-Proxy）`，说明之前已创建成功、本次自动跳过防重复——这是正常且符合预期的。
- ❌ 出现 `无法创建 apply 作业`,说明鉴权失败或 `STACK_OCID` 无效。原因排查顺序：
  1. `STACK_OCID` 是否填的是你 Osaka 区 Stack 的 OCID；
  2. `OCI_CLI_REGION` 是否为 `ap-osaka-1`；
  3. `OCI_CLI_USER` / `OCI_CLI_TENANCY` / `OCI_CLI_FINGERPRINT` / `OCI_CLI_KEY_CONTENT` 是否与 **同一个**用户的一整套 API 凭据匹配。

**第 3 层：作业是否真的在抢资源**
- 正常会看到循环输出 `轮询 #N: 作业状态 = ...`。
- 有这行输出，就说明已经发起了一次真实的 OCI apply，正在等结果，链路完全正常。

#### 5.3 判定最终结果（重要）

看最后一层 **通知处理** 的日志和整个运行的"红/绿"状态：

| 现场表现 | 含义 | 你需要做什么 |
|---------|------|-------------|
| 日志打印 `✓ 已成功创建 IntNet-Proxy 实例！`，且**整个运行是红色（失败）** | ✅ **抢到了！** 这就是触发了邮件 | 立刻按"收到成功邮件后必做"禁用工作流 |
| 日志打印 `检测到非容量类错误……`，且**整个运行是红色（失败）** | ❌ **配置/凭据有问题**（不是资源不足） | 尽快按 5.2 排查，否则会反复告警 |
| 日志打印 `未抢到实例……静默等待下一次重试`，且**整个运行是绿色（成功）** | ⏳ 还没抢到，一切都正常 | 什么都不用做，等邮件 |
| 日志打印 `目标实例已存在……静默结束`，且**整个运行是绿色（成功）** | ✅ 说明之前抢到过，本次自动跳过防重复 | 什么都不用做，去 OCI 确认实例 |
| 卡在第 1/2 层报错 | ❌ 配置有问题 | 按 5.2 排查 |

> 🎯 **判定口诀**：
> 【红色=有邮件 = 要么抢到了、要么配置出错；绿色=静默 = 正常等待或防重复跳过】。
> 绿色时再看一眼日志里是"无容量"还是"实例已存在"，就能区分是在等货还是已成功。

#### 5.4（可选）本机快速自测

如果你想在**不用 GitHub 的前提下**先验证凭据和 Stack 是否可用，可以在装了 OCI CLI 的电脑上（或任一处）运行：

```bash
oci resource-manager stack get --stack-id <你的STACK_OCID>
```

能返回该 Stack 的信息，即证明这套 API 凭据+Stack OCID 组合是有效的，问题大概率不在账户而在 GitHub Secrets 拼写。

### 第 6 步：交给定时任务

测试没问题后就不用管了。从这里开始，GitHub 会**每 5 分钟自动跑一次**，默默帮你抢。没抢到就一直静默重试，不打扰你。

---

## 🚨 收到成功邮件后必做

当你收到 GitHub 邮件通知（也就是实例抢到了），建议尽快停止无人值守的自动运行。虽然有内置的自动防重复检测（见上"两个内置的自动化防御"）兜底，但显式禁用仍然最稳妥：

1. 进入仓库 **Actions** 页签，左侧点 **Try to Create OCI VM (IntNet-Proxy)**。
2. 点右上角 **三个点 (…)** 菜单 → **Disable workflow**（禁用工作流）。

> 💡 即便你忘记禁用也没关系：后续定时运行会在开头检测到 `IntNet-Proxy` 实例已存在，自动跳过创建并正常结束，不会重复建机器、也不会重复发邮件。你只需在空闲时顺手禁用即可。

之后去 OCI 控制台的 **Compute → Instances** 确认你的 `IntNet-Proxy` 实例正在启动，用你在 Stack 里配置的 SSH 密钥登录即可。

---

## 常见问题

**Q：为什么总是停在轮询、显示 FAILED？**
正常，说明 OCI 暂时没有空余的 A1.Flex 容量。工作流会自动继续每 5 分钟重试，不用干预。

**Q：不小心没禁用工作流，会不会重复建实例？**
不会。每次运行开头都会先检测是否已存在 `IntNet-Proxy` 实例，存在就直接跳过创建、正常结束，不会重复建。所以抢到后即使忘了禁用也安全，只需在方便时手动禁用以关闭定时任务即可。

**Q：换区域怎么办？**
把 Secrets 里的 `OCI_CLI_REGION` 改成目标区域（如 `ap-singapore-1`），并把 `STACK_OCID` 换成对应区域的 Stack。Stack 本身限制了所在区域，需在对应区域创建。

## License

[MIT](LICENSE)