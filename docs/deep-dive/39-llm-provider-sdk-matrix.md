# LLM Provider SDK Selection Matrix

> **Purpose**: Document the SDK strategy for integrating with Claude, OpenAI, Gemini, Grok, and other LLM providers. Supports decision AD-09 (native tool-calling preserved) and AD-27 (backend language selection).

---

## Provider SDK Landscape

| Provider | Official SDK | Best For | Native Tool Support | Streaming | Notes |
|----------|------------|----------|---------------------|-----------|-------|
| **Anthropic (Claude)** | `@anthropic-ai/sdk` | TypeScript/Node | `tool_use` blocks | ✅ SSE | First-class tool calling |
| | `anthropic` | Python | `tool_use` blocks | ✅ SSE | |
| **OpenAI** | `openai` | Both | Function calling | ✅ SSE | Mature, widely adopted |
| **Google (Gemini)** | `@google/generative-ai` | TypeScript | Function declarations | ✅ | Rapidly evolving |
| | `google-generativeai` | Python | Function declarations | ✅ | |
| **xAI (Grok)** | OpenAI-compatible | Both | Function calling | ✅ | Drop-in replacement |
| **Azure OpenAI** | `openai` with Azure options | Enterprise | Function calling | ✅ | Enterprise features |
| **AWS Bedrock** | `@aws-sdk/client-bedrock-runtime` | AWS-native | Model-dependent | ✅ | Cross-model access |

---

## SDK Comparison: TypeScript/Node.js

### Anthropic SDK
```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const response = await client.messages.create({
  model: 'claude-3-7-sonnet-20250219',
  max_tokens: 4096,
  system: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'Hello' }],
  tools: [{
    name: 'get_weather',
    description: 'Get weather for a location',
    input_schema: { type: 'object', properties: { location: { type: 'string' } }, required: ['location'] }
  }]
});
```

| Criteria | Rating | Notes |
|----------|--------|-------|
| Type safety | ⭐⭐⭐⭐⭐ | Full TypeScript definitions |
| Tool calling | ⭐⭐⭐⭐⭐ | Native `tool_use` / `tool_result` |
| Streaming | ⭐⭐⭐⭐⭐ | SSE with token-by-token |
| Error handling | ⭐⭐⭐⭐☆ | Structured error types |
| Documentation | ⭐⭐⭐⭐⭐ | Excellent |

---

### OpenAI SDK
```typescript
import OpenAI from 'openai';

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
  tools: [{
    type: 'function',
    function: {
      name: 'get_weather',
      description: 'Get weather for a location',
      parameters: { type: 'object', properties: { location: { type: 'string' } }, required: ['location'] }
    }
  }],
  stream: true
});
```

| Criteria | Rating | Notes |
|----------|--------|-------|
| Type safety | ⭐⭐⭐⭐⭐ | Full TypeScript definitions |
| Tool calling | ⭐⭐⭐⭐⭐ | Function calling standard |
| Streaming | ⭐⭐⭐⭐⭐ | SSE with choices[0].delta |
| Error handling | ⭐⭐⭐⭐☆ | Standard HTTP errors |
| Documentation | ⭐⭐⭐⭐⭐ | Excellent |

---

### Google Generative AI SDK
```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);
const model = genAI.getGenerativeModel({ 
  model: 'gemini-2.0-flash',
  tools: [{ functionDeclarations: [...] }]
});

const result = await model.generateContentStream('Hello');
```

| Criteria | Rating | Notes |
|----------|--------|-------|
| Type safety | ⭐⭐⭐⭐☆ | Good, occasionally lags behind API |
| Tool calling | ⭐⭐⭐⭐☆ | Function declarations work well |
| Streaming | ⭐⭐⭐⭐☆ | Good support |
| Error handling | ⭐⭐⭐☆☆ | Less detailed than Anthropic/OpenAI |
| Documentation | ⭐⭐⭐⭐☆ | Good, but evolves quickly |

---

## SDK Comparison: Python

### Anthropic Python SDK
```python
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=4096,
    system="You are a helpful assistant.",
    messages=[{"role": "user", "content": "Hello"}],
    tools=[{
        "name": "get_weather",
        "description": "Get weather for a location",
        "input_schema": {"type": "object", "properties": {"location": {"type": "string"}}, "required": ["location"]}
    }]
)
```

| Criteria | Rating |
|----------|--------|
| Type safety | ⭐⭐⭐⭐☆ (Pydantic models) |
| Tool calling | ⭐⭐⭐⭐⭐ |
| Streaming | ⭐⭐⭐⭐⭐ |
| Error handling | ⭐⭐⭐⭐☆ |
| Async support | ⭐⭐⭐⭐⭐ (`AsyncAnthropic`) |

---

### OpenAI Python SDK
```python
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
    tools=[...],
    stream=True
)
```

| Criteria | Rating |
|----------|--------|
| Type safety | ⭐⭐⭐⭐☆ (Pydantic models) |
| Tool calling | ⭐⭐⭐⭐⭐ |
| Streaming | ⭐⭐⭐⭐⭐ |
| Error handling | ⭐⭐⭐⭐☆ |
| Async support | ⭐⭐⭐⭐⭐ (`AsyncOpenAI`) |

---

## Recommended SDK Strategy

### Primary Recommendation: Official SDKs Per Provider

**Decision**: Use each provider's official SDK directly, not an abstraction layer like LangChain or Vercel AI SDK for the core provider adapters.

**Rationale**:
1. **Native tool formats** (AD-09) - Direct SDK access preserves provider-native tool calling
2. **Fastest feature parity** - New model features available immediately
3. **Best error granularity** - Provider-specific error handling
4. **Clean adapter pattern** - Each provider adapter wraps one official SDK

### Temporal Worker Considerations

| Language | Temporal SDK Maturity | LLM SDK Availability | Recommendation |
|----------|----------------------|---------------------|----------------|
| **TypeScript** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ All providers | **Recommended** |
| **Python** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ All providers | Alternative |

### Implementation Pattern

```typescript
// Provider Adapter Interface (AD-09, AD-10)
interface LLMProviderAdapter {
  provider: string;
  
  invoke(params: InvokeParams): Promise<LLMResponse>;
  invokeStreaming(params: InvokeParams): AsyncIterable<LLMStreamChunk>;
  probe(): Promise<ProviderHealth>;  // E-06
  
  // Tool format conversion happens inside adapter
  convertTools(platformTools: Tool[]): ProviderNativeToolFormat;
}

// Anthropic Implementation
class ClaudeAdapter implements LLMProviderAdapter {
  private client: Anthropic;
  
  async invoke(params: InvokeParams): Promise<LLMResponse> {
    const response = await this.client.messages.create({
      model: params.modelConfig.model_name,
      max_tokens: params.modelConfig.max_tokens,
      system: params.systemPrompt,
      messages: this.convertMessages(params.messages),
      tools: this.convertTools(params.tools),
      temperature: params.modelConfig.temperature
    });
    
    return {
      content: response.content,
      toolCalls: response.stop_reason === 'tool_use' ? 
        this.extractToolCalls(response.content) : [],
      usage: {
        inputTokens: response.usage.input_tokens,
        outputTokens: response.usage.output_tokens
      }
    };
  }
}
```

---

## SDK Dependency Versions

Recommended minimum versions for platform implementation:

| SDK | Version | Reason |
|-----|---------|--------|
| `@anthropic-ai/sdk` | `^0.36.0` | Claude 3.7 support, streaming improvements |
| `openai` (TS) | `^4.82.0` | GPT-4o, structured outputs |
| `@google/generative-ai` | `^0.21.0` | Gemini 2.0 support |
| `anthropic` (Python) | `^0.42.0` | Feature parity with TS SDK |
| `openai` (Python) | `^1.60.0` | Latest features, Pydantic v2 |

---

## Error Classification Mapping

Each SDK throws different error types. Map to platform error taxonomy:

| SDK Error | Platform Error | Retry Strategy |
|-----------|---------------|----------------|
| `Anthropic.APIError` (429) | `RateLimitError` | Honor `Retry-After` (AD-28) |
| `Anthropic.APIError` (5xx) | `ProviderUnavailableError` | Exponential backoff |
| `Anthropic.AuthenticationError` | `ConfigurationError` | No retry - alert operator |
| `OpenAI.APIError` (429) | `RateLimitError` | Honor `Retry-After` |
| `OpenAI.APIError` (5xx) | `ProviderUnavailableError` | Exponential backoff |
| `GoogleGenerativeAI.*Error` | Map by HTTP code | Same as above |

---

## Streaming Implementation Notes

### TypeScript SSE Pattern
```typescript
async function* streamResponse(params: InvokeParams): AsyncIterable<StreamChunk> {
  const stream = await client.messages.create({
    ...params,
    stream: true
  });
  
  for await (const event of stream) {
    if (event.type === 'content_block_delta') {
      yield { type: 'token', content: event.delta.text };
    }
    if (event.type === 'content_block_stop' && event.content_block.type === 'tool_use') {
      yield { type: 'tool_call', toolUse: event.content_block };
    }
  }
}
```

### Connection Resilience
- Implement client-side timeout (e.g., 30s for first token)
- Handle `AbortController` for cancellation
- Log stream interruption events for AD-23 (LLM observability interface)

---

## Related Documents

- `01-PRD.md` AD-09: Provider-agnostic interface with native tool-calling
- `01-PRD.md` AD-10: Agent Core minimalism
- `01-PRD.md` AD-27: Backend implementation language decision
- `02-v1-agent-provisioning-foundation.md`: Provider adapter implementation
- `19-provider-rate-limiting.md`: Rate limit error handling

---

> **Document History**
> - Created: Post-review enhancement (agreed item #9), Doc 39
> - Purpose: SDK selection guidance for implementation phase
> - Status: Recommendation pending AD-27 resolution
