# yujie-dsh-plugin-system-prompt

在每次对话的系统提示词开头注入你自定义的内容，注入内容在**插件管理面板**里编辑。

## 怎么用

1. 打开左侧栏的 **Plugins**（插件管理）页面，顶部 **Installed / Discover** 之后多了一个 **系统提示词** Tab，点它就能直接编辑；也可以从 **Settings → 内置插件** 里的同名 Tab 进入，或者打开 `yujie-dsh-plugin-system-prompt` 这个 bundle 的页面（描述与插件行之间也有同一份配置）。
2. 在配置区填写 **系统提示词**，按「保存」。
3. 从下一个模型步（下一次请求）开始，所有会话、所有代理（含子代理）的系统提示词开头都会带上这段文字。

- 清空文本框并保存会移除注入，恢复默认（不注入）。
- 取消勾选「注入这段提示词」可以临时关闭注入而不删除文字。
- 修改是热生效的：不需要重启 Harness，也不会打断当前会话。

## 配置项

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `prompt` | string | `""` | 注入的系统提示词正文；为空时不注入。 |
| `enabled` | boolean | `true` | 是否注入。 |

这两个字段声明为 volatile，因此会出现在 Host 设置表单里，并持久化到当前 profile 的 `cordis.patch.yml`：

```yaml
- id: yujie-dsh-plugin-system-prompt
  config:
    prompt: '你是一名资深 Go 工程师……'
    enabled: true
```

## 实现方式

- Host 半（`index.js`）：通过 `ctx.systemPrompt.section()` 注册一个 order 为 `-999` 的 section，位置紧跟在固定的 Harness 身份（order `-1000`）之后、内置指引之前。section 文本是函数，每次组装系统提示词时读取 `config.prompt` / `config.enabled` 的当前值。
- Client 半（`client.js`）：同一张卡片注册到三个位置 —— `plugins.tab`（Plugins 页面顶部的 Tab）、`settings.plugins.tab`（Settings → 内置插件 的 Tab）、`plugins.bundle.config`（bundle 页面的配置区）；渲染文本框 + 开关 + 保存按钮，读写通过 `ctx.configForms.get('<entry id>')`，即 Host 设置服务，带版本校验。
- 异步注入的内容对所有 agent 都生效，且不写入用户消息历史，会话日志里只体现为派生的系统提示词节点。

## 依赖

Host 半用 `@deepseek-ai/schemastery` 声明 Config，版本与当前 Harness 内置的 `3.18.5` 保持一致，已安装在本目录的 `node_modules` 里（bundle 以 `link:` 方式装入 profile，消费的是本目录的依赖）。如果这个目录被移动或重新 checkout，请在目录内执行一次：

```sh
pnpm install
```

## 注意事项

- 注入文本不做变量插值（`interpolate: false`），因此可以安全地包含 `{{...}}` 之类的字面量。
- 系统提示词改变会使该前缀之后的 KV cache 复用失效，属于预期成本。
- 只影响系统提示词，不改动用户消息。
