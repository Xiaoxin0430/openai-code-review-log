# Code Review 评审报告

## 📋 变更概述

本次变更将 `ChatCompletionRequestDTO` 的请求体结构从 OpenAI 标准的 `messages`（`List<Map<String, String>>`）简化为单一的 `input`（`String`），并同步修改了调用方。

---

## 🔴 严重问题（Critical）

### 1. API 协议兼容性破坏

OpenAI 及兼容接口（包括 Qwen 的 Chat Completions 接口）要求的请求体格式为：

```json
{
  "model": "qwen-plus",
  "messages": [
    {"role": "user", "content": "..."}
  ]
}
```

将字段名从 `messages` 改为 `input` 后，序列化出的 JSON 将变为：

```json
{
  "model": "qwen-plus",
  "input": "..."
}
```

**这会导致 API 调用直接失败**，除非下游的 HTTP 请求层做了额外的字段映射转换。请确认调用链路上是否有对应的适配逻辑，否则这是一个 **线上阻断性 Bug**。

### 2. 丢失 Role 语义

原代码中 `setMessages(String content)` 内部封装了 `role: "user"` 的语义：

```java
// 旧代码 - 明确指定了 role
message.put("role", "user");
message.put("content", content);
```

新代码的 `input` 字段是一个纯字符串，**无法表达 role 信息**。这意味着：
- 无法设置 `system` 角色来注入系统级 Prompt（当前 system prompt 是直接拼在 user 消息里的，本身就不规范）
- 无法区分用户消息和系统指令

---

## 🟡 设计问题（Design Concerns）

### 3. 多轮对话能力丧失

原结构 `List<Map<String, String>>` 天然支持多轮对话上下文：

```java
// 旧结构可以承载完整对话历史
messages = [
  {"role": "system", "content": "你是一个代码评审专家"},
  {"role": "user", "content": "请评审以下代码..."},
  {"role": "assistant", "content": "这段代码有以下问题..."},
  {"role": "user", "content": "请针对第2点展开说明"}
]
```

改为 `String input` 后，**SDK 被限制为只能做单轮请求**，丧失了多轮对话和上下文记忆的能力。作为一个 SDK 的 DTO，这种设计过于短视。

### 4. DTO 职责退化

原来的 `setMessages(String content)` 虽然方法名不太准确（参数是 String 但方法名叫 setMessages），但它实际上承担了 **将简单输入适配为标准协议格式** 的职责，是一种合理的防腐层设计。

现在这个适配逻辑被移除了，DTO 退化成了一个纯粹的 POJO，协议适配的压力被转移到了调用方或者下游。

---

## 🟢 改进建议

### 方案一：恢复标准 messages 结构（推荐）

```java
public class ChatCompletionRequestDTO {

    private String model = Model.QWEN3_7_PLUS.getCode();

    private List<Message> messages;

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class Message {
        private String role;
        private String content;
    }

    // 保留便捷方法，同时支持灵活设置
    public void setUserMessage(String content) {
        this.messages = Collections.singletonList(new Message("user", content));
    }

    public void setSystemAndUserMessage(String systemPrompt, String userContent) {
        this.messages = Arrays.asList(
            new Message("system", systemPrompt),
            new Message("user", userContent)
        );
    }
}
```

调用方改为：

```java
chatCompletionRequestDTO.setSystemAndUserMessage(
    "你是一个高级编程架构师，精通各类场景方案、架构设计和编程语言，请根据 git diff 记录，对代码做出评审。",
    diffCode
);
```

这样做的好处：
- ✅ 符合 OpenAI / Qwen API 标准协议
- ✅ System Prompt 和 User Content 分离，语义清晰
- ✅ 保留多轮对话扩展能力
- ✅ 用强类型 `Message` 类替代 `Map<String, String>`，避免魔法字符串

### 方案二：如果下游确实使用 `input` 协议

如果确认使用的是 DashScope 的旧版文本生成接口（非 Chat 格式），则至少应该：

```java
public class ChatCompletionRequestDTO {

    private String model;

    // 使用内部类封装 input 结构
    private InputBody input;

    @Data
    public static class InputBody {
        private List<Message> messages;
        // 或者 private String prompt; 视具体 API 而定
    }
}
```

保持 DTO 结构与 API 协议严格对齐，而不是用一个扁平的 `String` 来糊弄。

---

## 📝 总结

| 维度 | 评价 |
|------|------|
| **功能正确性** | ❌ 大概率导致 API 调用失败 |
| **协议规范性** | ❌ 偏离 OpenAI/Qwen 标准协议 |
| **可扩展性** | ❌ 丧失多轮对话和角色控制能力 |
| **代码简洁性** | ✅ 代码量减少，但属于过度简化 |
| **类型安全** | ❌ 从结构化数据退化为裸字符串 |

**结论：建议打回重构。** 当前的简化是以牺牲协议兼容性和扩展性为代价的，属于"为了简单而简单"的反模式。推荐采用方案一，用强类型的 `Message` 列表来替代 `Map<String, String>`，同时保留便捷设置方法。