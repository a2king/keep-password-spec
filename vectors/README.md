# 黄金向量

本目录存放可公开的合成测试向量（禁止真实用户保险库）。

待 `PLAN.md` Phase S1 启动后，按子目录补充：

- `kdf/` — Argon2id
- `aes-gcm/` — AES-256-GCM + AAD
- `password-wrap/` — 主密码包裹 data key
- `short-unlock/` — 短密钥相关
- `vault-roundtrip/` — 完整最小 vault 样例

格式建议：每组一个 `case.json` + `README.md` 说明输入/输出字段与生成方式。
