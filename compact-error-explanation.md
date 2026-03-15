# /compact 报错与远端 compact 能力边界说明

## 流程

1. 在 Codex 执行 `/compact`。
2. Codex 发送 `POST /v1/responses/compact`，请求体包含 `model + instructions + input(history)`。
3. 服务端返回 `{"output":[...items]}` 后，Codex 会用这组 `output` 替换本地历史。
4. 下一轮对话时，Codex 会把这组历史作为 `POST /v1/responses` 的 `input` 再发给上游。
5. 如果历史中含有 `{"type":"compaction","encrypted_content":"..."}`，上游会尝试解密和校验该字段。

## 当前报错的直接原因

- 当前实现把“明文摘要”写进了 `encrypted_content`。
- 上游在下一轮请求中会把该字段当作密文解析；明文无法通过解密/校验。
- 因此出现错误：`The encrypted content ... could not be decrypted or parsed`。
- `Context compacted` 仅表示 compact 请求本身返回成功，不代表后续携带该历史的请求一定成功。

## 为什么无法做“完整原生支持”

- `encrypted_content` 不是普通字符串，而是由上游服务端生成并可验证的受保护载荷。
- 当前代理（GitHub Copilot 兼容链路）没有能力签发 OpenAI 原生可验证的 compaction 密文。
- 本地伪造或自定义“加密”都无法通过上游校验。

## 第二方案（兼容版）能支持到什么程度

- 可以继续支持远端 `/responses/compact` 调用流程。
- 但返回应改为普通 summary message，不返回 `type: "compaction"` 的伪密文项。
- 这属于“兼容版远端 compact”，不是 OpenAI 原生加密 compact。

## 若要实现完整原生能力

1. 直连真实支持原生 `POST /v1/responses/compact` 且可签发 `encrypted_content` 的后端，并透传返回值。
2. 或继续使用当前代理方案，但接受兼容版（summary message）而非原生加密 compaction。

