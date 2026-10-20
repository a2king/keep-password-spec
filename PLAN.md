# Keep Password Spec — 计划书

> 版本目标：与桌面 `KeepPassword.Core` **当前 FormatVersion = 3**、服务端 `/api/*` 对齐。  
> 状态：调研落地文档，供后续编写规范与向量时按章节填充。

---

## 1. 目标与原则

| 原则 | 说明 |
|------|------|
| 单一真相 | 加密格式与同步契约只在本仓定义；各端实现不得私自发散 |
| 比特兼容 | 云端同一份 encrypted blob，PC / Android / iOS 均可解密 |
| 可验证 | 所有密码学步骤必须有黄金向量；CI 跑向量，不靠「感觉一致」 |
| 渐进披露 | 先冻结「能同步 + 能解锁 + 能填登录」；运维类条目字段先只读复制 |

**不在本仓做的事：** UI、系统 Autofill 实现、业务文案。

---

## 2. 建议目录结构（落地后）

```text
keep-password-spec/
  README.md
  PLAN.md                 ← 本文件
  VERSION                 ← 规范版本，如 0.1.0
  docs/
    vault-format.md       ← 保险库信封 + payload schema
    sync-api.md           ← 或直接维护 openapi.yaml
    crypto.md             ← KDF / AES-GCM / AAD / wrap 说明
    item-kinds.md         ← 条目类型与字段字典
    mobile-scope.md       ← 移动端首期功能边界（对桌面裁剪）
  openapi/
    sync-api.yaml         ← 从 keep-password-server 导出并人工校对
  vectors/
    README.md
    kdf/
    aes-gcm/
    password-wrap/
    short-unlock/
    vault-roundtrip/      ← 完整小保险库样例（无真实密码）
  scripts/
    export-openapi.sh     ← 可选：从 Flask 生成草稿
    verify-vectors/       ← 可选：多语言校验入口说明
```

---

## 3. 工作分期

### Phase S0 — 清单冻结（约 1 周，文档为主）

- [ ] 从桌面代码整理 **Envelope JSON** 字段表（`format`, `version`, `account`, `kdf`, `passwordWrap`, `shortKey`, `cipher`, `keyFileSaved`…）
- [ ] 整理 **Payload**（entries / labels）与 `VaultItemKind`、账号、自定义字段
- [ ] 列出 AAD 字符串规则（payload v1/v2/v3、password-wrap、short-unlock 等）
- [ ] 导出服务端路由为 `openapi/sync-api.yaml` 初稿
- [ ] 写 `docs/mobile-scope.md`：P0/P1/P2 功能边界

### Phase S1 — 向量与样例

- [ ] Argon2id 派生向量（固定 password/salt/profile → key）
- [ ] AES-256-GCM 加解密向量（含 AAD）
- [ ] 主密码 wrap / 短密钥 wrap 向量
- [ ] 最小可解锁 vault 文件样例（合成数据，可公开）
- [ ] 文档：如何用 .NET 测试生成向量，再让 Kotlin/Swift 复现

### Phase S2 — 规范正式化

- [ ] `VERSION` + changelog
- [ ] OpenAPI 与 server 实现 diff 检查说明
- [ ] 兼容策略：version 2 可读、version 3 为新默认；禁止静默写旧格式

---

## 4. Vault 格式要点（待写入 vault-format.md）

来源：`KeepPassword.Core` / `VaultStore`

| 常量 | 值 | 含义 |
|------|-----|------|
| LegacyPasswordVersion | 1 | 主密码直接派生数据密钥（旧） |
| IndependentKeyVersion | 2 | 独立 data key + 主密码 wrap |
| FormatVersion | **3** | 本地密钥保险库（可无主密码层） |

信封：

- `format`: `"keep-password-vault"`
- `cipher.algorithm`: `"aes-256-gcm"`
- KDF / ShortKey：`argon2id` + memoryKb / iterations / parallelism / salt（及 shortKey.hash）

Payload AAD（实现必须一致）：

- v1: `keep-password/v1|{account}`（以桌面 `VaultCipher.AssociatedData` 为准）
- v2: `keep-password/payload/v2|{account}`
- v3: `keep-password/payload/v3|{account}`

密码 wrap AAD：`keep-password/password-wrap/v2|{account}`

**.quick 设备解锁** 为桌面本机机制；移动端应用 **Keystore/Keychain + 生物识别**，不要求兼容 `.quick` 文件格式。

---

## 5. Sync API 要点（待写入 openapi）

来源：`keep-password-server` `url_prefix=/api`

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/health` | 健康检查 |
| GET | `/auth/captcha` | 验证码 |
| POST | `/auth/register` | 注册 |
| POST | `/auth/verify-email` | 邮箱验证 + 绑定密钥 |
| POST | `/auth/resend-verify` | 重发验证 |
| POST | `/auth/login` | 登录 |
| POST | `/auth/forgot-password` | 忘记密码 |
| POST | `/auth/reset-password` | 重置密码 |
| GET | `/me` | 个人中心 |
| POST | `/me/request-code` | 敏感操作验证码 |
| POST | `/me/devices/:id/revoke` | 解绑设备 |
| POST | `/me/binding-key/regenerate` | 重发绑定密钥 |
| POST | `/me/delete` | 注销 |
| POST | `/sync/device/request-code` | 设备绑定验证码 |
| POST | `/sync/device/bind` | 设备绑定 |
| POST | `/sync/device/unbind-self` | 本机解绑 |
| GET | `/sync/status` | 同步状态 |
| GET/PUT | `/sync/vault` | 下载/上传加密保险库（冲突 409） |
| GET/PUT | `/sync/wrapped-key` | 官网密码包裹的密钥 |
| GET | `/admin/overview` | 管理端（移动端可不实现） |

原则：服务器只存密文 blob，无法解密内容。

---

## 6. 条目类型（移动端消费）

与桌面一致的 kind（名称以桌面 `VaultItemKinds` 为准）：

- Login（P0 完整：编辑 + Autofill）
- Server / Database / Cluster / ApiCredential / SshKey（P0：**展示 + 复制**；P2：完整编辑）

字段字典写入 `docs/item-kinds.md`，避免三端 label/key 漂移。

---

## 7. 与各仓协作约定

1. **改格式 / 改 API**：先 PR 本仓 → 合并 → 再改 server / desktop / mobile。  
2. **向量失败**：禁止发版。  
3. **桌面先实现的行为**：以 Core 为准回写文档；发现文档错误改文档，不要让手机「猜」。  
4. **密钥材料**：向量仓不得提交真实用户保险库；仅合成样例。

---

## 8. 验收标准（Spec 仓本身）

- [ ] 新人只读本仓即可实现「解锁样例 vault + 调用 sync 上传下载」
- [ ] OpenAPI 与 server 路由一致（抽样手工或脚本）
- [ ] 至少 1 组 vault-roundtrip 向量被 .NET 测试引用；Kotlin/Swift 后续对齐

---

## 9. 参考代码位置

| 主题 | 仓库路径 |
|------|----------|
| FormatVersion / Envelope | `keep-password/src/KeepPassword.Core/Vault/VaultStore.cs` |
| Payload / Entries | `VaultSession` / `VaultEntry` |
| Sync 客户端（桌面） | `keep-password/src/KeepPassword.Core/Sync/` |
| Sync 服务端 | `keep-password-server/backend/app/api/` |
| 默认官网 URL | `SyncDefaults.ProductionBaseUrl` |
