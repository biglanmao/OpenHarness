# QQ 通道接入问题记录

日期：2026-11-01

## 结果

QQ 机器人「机器人1906215804」已连通，WebSocket 正常，消息收发均已验证。
模型 provider 已从 `codex` 切到 `deepseek`。

## 问题 1：QQ 通道代码 bug（已修复）

`src/openharness/channels/impl/qq.py:69,84` 读取 `self.config.secret`，
但 `QQConfig`（`src/openharness/config/schema.py:79`）定义的字段是 `app_secret`。

报错：

```
AttributeError: 'QQConfig' object has no attribute 'secret'
```

影响：无论是否填写密钥，通道都会在 `start()` 第一步崩溃。

修复：

```diff
- if not self.config.app_id or not self.config.secret:
+ if not self.config.app_id or not self.config.app_secret:
- await self._client.start(appid=self.config.app_id, secret=self.config.secret)
+ await self._client.start(appid=self.config.app_id, secret=self.config.app_secret)
```

状态：已修改，未提交。

## 问题 2：配置向导不含 QQ（未修复）

`ohmo/cli.py:54` 的 `_INTERACTIVE_CHANNELS` 只有：

```python
_INTERACTIVE_CHANNELS = ("telegram", "slack", "discord", "feishu")
```

影响：`ohmo gateway config` 选不到 QQ，只能手改 `~/.ohmo/gateway.json`。

状态：未修改。

## 问题 3：机器人在线但不回话（已修复）

实际原因是模型鉴权失败，与 QQ 无关。QQ 消息已收到、session 已建立，
但处理管道抛出：

```
ValueError: No external auth binding found for codex_subscription.
  Run 'oh auth codex-login' first.
SystemExit: 1
```

根因：`gateway.json` 的 `provider_profile` 为 `codex`（订阅制鉴权），
本机无 codex 登录凭证。

修复：改为 `deepseek`，复用已有配置：

- `~/.openharness/settings.json` 里已有 profile `deepseek`
  （`provider: deepseek`、`model: deepseek-chat`、`base_url: https://api.deepseek.com/v1`）
- `~/.openharness/credentials.json` 里已有 `profile:deepseek` 的 api_key

## 当前配置

`~/.ohmo/gateway.json`：

```json
{
  "provider_profile": "deepseek",
  "enabled_channels": ["qq"],
  "channel_configs": {
    "qq": {
      "app_id": "1906215804",
      "app_secret": "<redacted>",
      "allow_from": ["*"]
    }
  }
}
```

## 依赖

已安装 `qq-botpy==1.2.1`（附带 `apscheduler==3.11.3`、`tzlocal==5.4.5`）。

## 遗留风险

1. `allow_from: ["*"]` —— 任何 QQ 用户都能私聊并消耗 DeepSeek 额度，
   建议从日志获取自己的 openid 后收紧白名单。
2. `app_secret` 曾在对话中明文出现，建议到 q.qq.com 重置。
3. 配置向导缺口未修，下次重配仍需手改 JSON。
4. `qq.py` 的修复未提交。

## 能力缺口

QQ 通道 `send()` 只调用 `post_c2c_message(msg_type=0)`，仅支持纯文本，
不支持发送文件。发文件需要 `post_c2c_file` 加富媒体上传流程，未实现。
目前仅 Feishu 通道实现了文件上传。

## 备注

`git status` 中另有 `src/openharness/mcp/client.py`、`src/openharness/ui/backend_host.py`
两处改动，与本问题无关，非本次修改产生。
