# Grok 空闲探活修复与低 token 验证（2026-11-11）

## 根因

生产原始 2.19.5 的 V3 监控经 `RunPelicanBackground` 生成探活题目，但 Grok 文本测试分支调用 `buildGrokQuotaProbeBody`，正文固定为 `hi`。因此上游正常返回问候语后，被监控按期望的数字答案判为失败。

探活模型名也必须匹配账号的模型映射键。`Grok 4.5` 不匹配，`grok-4.5` 可以选到账号 #23。该上游的 `/models` 已实际返回 `grok-4.5`、`grok-4.6`、`grok-4.7`、`grok-4.7-build-fast`。

## 修改

- Grok 测试分支读取真实测试题目，并保留模型映射和原有响应解析。
- V3 空闲探活去掉输入示例。Grok 只要求回复随机的短数字，例如 `Reply only 7.`，仍核对预期答案。
- 仅 Grok 空闲探活请求设置 `max_output_tokens=16`。可识别为支持推理参数的模型发送 `reasoning.effort=none`，参数兼容性继续使用现有 Grok 归一化逻辑。
- 普通测试继续保留其题目和推理设置；原有账号连通测试和额度探测仍使用原来的请求。
- 保留空闲触发、每格只探测一次和真实请求充足时跳过的条件。

这些源文件在修改前与 `v2.19.5` 完全一致，可以作为原始 2.19.5 的独立补丁；不需要部署当前分支已有的融合生图功能。

## 验证

本地回归先复现 API Key 账号的出站 `input=hi`，再验证修复后 API Key / OAuth 两类账号的真实题目传递、模型映射、监控预算、普通测试参数和监控答案校验。以下定向回归通过：

```powershell
go test -tags=unit ./internal/service -run '^Test(AccountTestService.*Grok|ObserveGrokTestResponse|GrokQuotaServiceProbeUsage|ChannelMonitorV3|PelicanProbeKind|PelicanPayloads|ValidateChallenge)' -count=1 -timeout=3m
```

通过账号 #23 的现有上游进行一次短算术请求，使用 `grok-4.5`、`max_output_tokens=16`、`reasoning.effort=none`。HTTP 200，正确返回 `52`，但耗时 94.727 秒，上游上报：

| 项目 | token |
| --- | ---: |
| 输入 | 2166 |
| 其中缓存输入 | 2048 |
| 输出 | 102 |
| 其中推理 | 101 |

请求体只有短题目，以上额外输入与推理用量是上游上报的；网关不能保证该上游严格执行 token / 推理限制。不能将请求中的 16 当成实际计费上限，也不能将可见的一个数字当成总输出用量。

用户选择保留真实生成探活、尽量减少 token。随后对 `grok-4.7-build-fast` 发送 `Reply only 7.`，同样设置 `max_output_tokens=16`；按现有兼容逻辑，该模型不发送未声明支持的推理参数。HTTP 200，正确返回 `7`，耗时 63.378 秒，上游上报输入 3341（其中缓存 3328）、输出 144（其中推理 143）。

两次请求分别使用不同模型和提示，不能由此严格判断模型的固定成本差异；但没有证据表明名字含 `fast` 就更省 token，因此不自动将生产探活切换为该模型。可以先使用已匹配账号的 `grok-4.5`；新短数字请求在该模型的实际用量尚未另行验证。额外用量的组成与限制参数的执行需要上游确认。

Grok 首次短算术响应超过目前 V3 的 90 秒超时。此次未放宽超时；若上游响应超过 90 秒，探活仍会按超时处理。

截至本文记录时，生产配置、容器和运行二进制尚未修改。原始 2.19.5 的独立补丁通过临时 Git index 的 `git apply --cached --check` 验证，未修改真实暂存区。
