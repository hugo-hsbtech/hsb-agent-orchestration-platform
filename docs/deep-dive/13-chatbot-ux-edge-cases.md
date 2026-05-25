# Chatbot UX Edge Cases

## Purpose

This document explores edge cases, failure modes, and exceptional scenarios in the chatbot layer. It provides conceptual guidance for handling complex user interactions without prescribing specific UI implementations or frontend code.

---

## Conversation Lifecycle Edge Cases

### 1. Session Interruption

**Scenario**: User closes browser/app mid-conversation, returns later.

**Considerations**:
- **Resume point**: Restore conversation from last persisted turn or allow user to see full history?
- **Context decay**: How many prior turns are relevant? (Token limits, relevance scoring)
- **Background task awareness**: If user left while task was pending, surface result immediately on return

**Conceptual approach**: Conversation state is server-authoritative. Client reconnects with conversation ID; server returns full history plus any pending task updates.

### 2. Multi-Device Conversations

**Scenario**: User starts on mobile, switches to desktop.

**Considerations**:
- **Real-time sync**: Active conversation visible on all devices simultaneously
- **Conflict resolution**: What if user sends different messages from two devices?
- **Notification handling**: Prevent duplicate notifications across devices

**Conceptual approach**: Optimistic concurrency with last-write-wins or user-visible conflict ("You have an active conversation on another device").

### 3. Conversation Timeout

**Scenario**: User abandons conversation for hours/days.

**Considerations**:
- **Auto-close policy**: When to mark conversation complete?
- **Re-engagement**: Greeting on return ("Welcome back" vs. "Continuing our discussion about...")
- **Background task orphans**: Pending tasks from abandoned conversations

**Conceptual approach**: Configurable timeout per chatbot type. Support conversations auto-close after 24h; sales conversations might stay open longer.

---

## Streaming Response Edge Cases

### 4. Connection Drop During Streaming

**Scenario**: User's connection fails while LLM is generating response.

**Considerations**:
- **Partial persistence**: Streamed tokens persisted incrementally or at completion?
- **Reconnect behavior**: Resume from last received token or restart generation?
- **User perception**: Show "Connection lost, reconnecting..." vs. silent recovery

**Conceptual approach**: Tokens persisted incrementally (every N seconds or M tokens). Reconnect returns full response generated so far plus continuation stream.

### 5. Slow Client Backpressure

**Scenario**: Client processes tokens slowly (slow device, busy browser).

**Considerations**:
- **Buffer overflow**: Server buffer size for unconsumed tokens
- **Degradation strategy**: Pause generation vs. drop tokens vs. batch delivery
- **Timeout**: How long to hold connection for slow client?

**Conceptual approach**: Bounded buffer with backpressure signal. If buffer full, server pauses provider stream (if provider supports) or continues buffering with max wait time.

### 6. Rapid User Messages

**Scenario**: User sends multiple messages before bot responds.

**Considerations**:
- **Message queueing**: Process sequentially or cancel in-flight and process latest?
- **UX indication**: Show "thinking..." per message or aggregate?
- **Context handling**: All user messages in context or only latest?

**Conceptual approach**: Queue with max depth (2–3 messages). Show typing indicator per pending response. Include all user messages in context (not just latest) to preserve user intent.

---

## Routing and Specialist Edge Cases

### 7. Routing Confidence Ambiguity

**Scenario**: Router cannot confidently classify user intent.

**Considerations**:
- **Threshold handling**: Low confidence → fallback vs. clarification question?
- **Multi-label classification**: Intent spans multiple specialists (billing + technical)
- **Escalation path**: When to offer human handoff vs. generic response?

**Conceptual approach**: Confidence bands:
- **High confidence (>0.8)**: Route directly
- **Medium confidence (0.5–0.8)**: Route with disclaimer ("I think you want billing, is that right?")
- **Low confidence (<0.5)**: Clarification question or fallback

### 8. Mid-Conversation Topic Shift

**Scenario**: User abruptly changes subject ("Actually, forget that, I need to reset my password").

**Considerations**:
- **Re-routing trigger**: Every user turn or only when specialist signals confusion?
- **Context clearing**: Preserve full history or truncate to relevant portion?
- **Smooth handoff**: Acknowledge shift vs. seamless transition?

**Conceptual approach**: Re-route every turn. Specialist receives full history but system prompt emphasizes current turn. Acknowledge major shifts: "Switching to account settings..."

### 9. Specialist Dead End

**Scenario**: Specialist cannot help (question outside scope, needs human).

**Considerations**:
- **Graceful escalation**: Clear handoff message vs. opaque transfer?
- **Context preservation**: What context transfers to human/agent?
- **Re-routing**: Offer to try different specialist vs. direct to fallback?

**Conceptual approach**: Specialist has explicit "cannot_help" response type. Triggers fallback with context summary. User sees: "I'm not able to help with that specific issue. Let me connect you with [fallback path]."

---

## Background Task Edge Cases

### 10. Task Completion While User Away

**Scenario**: Background task completes, user not in conversation.

**Considerations**:
- **Notification**: Push notification, email, or silent queue for next visit?
- **Urgency levels**: Critical results (payment failure) vs. informational (report ready)
- **Channel selection**: Same channel (chat) vs. alternative (email)?

**Conceptual approach**: Task declares urgency in DSL. High urgency: notification + queue for surfacing. Low urgency: queue only. User returns → immediate surfacing of completed urgent tasks.

### 11. Multiple Pending Tasks

**Scenario**: User has 3+ background tasks pending/completed.

**Considerations**:
- **Surfacing order**: Chronological, priority-based, or user-selected?
- **Aggregation**: Single summary message vs. individual notifications?
- **Overwhelm prevention**: Maximum tasks surfaced per turn?

**Conceptual approach**: Time-batched surfacing. Tasks completing within 5-minute window aggregate into single message: "I have updates on 3 things you asked about..." Ordered by user recency (most recently asked first).

### 12. Task Failure in Background

**Scenario**: Background task fails after user has moved on.

**Considerations**:
- **Failure notification**: Notify user or silent failure?
- **Retry visibility**: User sees retry attempts or only final failure?
- **Recovery path**: Can user re-trigger? Is there a fix path?

**Conceptual approach**: Distinguish retryable vs. permanent failure. Retryable: no user notification until exhausted. Permanent: surface with apology + next steps: "I wasn't able to [task]. You can [alternative] or I can connect you with support."

---

## Content and Safety Edge Cases

### 13. Harmful or Inappropriate User Input

**Scenario**: User inputs toxic, harmful, or off-topic content.

**Considerations**:
- **Detection layer**: Router, specialist, or both?
- **Response strategy**: Refusal, redirection, or end conversation?
- **Persistence**: Log for safety analysis without retaining toxic content?

**Conceptual approach**: Safety check at routing layer. Flagged content → generic refusal: "I'm not able to discuss that. Is there something else I can help you with?" Single refusal, then offer human handoff if pattern continues.

### 14. PII Exposure

**Scenario**: User pastes sensitive data (credit card, SSN) into chat.

**Considerations**:
- **Detection**: PII scanning on input
- **Handling**: Mask in persistence, exclude from LLM context?
- **User guidance**: Educate user not to share sensitive data?

**Conceptual approach**: PII detection masks before persistence. Specialist receives redacted version: "[CREDIT_CARD_REDACTED]". Response includes guidance: "For your security, please don't share full card numbers here."

### 15. Long Input Handling

**Scenario**: User pastes massive text (document, log file) into chat.

**Considerations**:
- **Size limits**: Hard cutoff or truncation?
- **Processing strategy**: Summarize, chunk, or reject?
- **UX feedback**: Inform user of truncation vs. silent handling?

**Conceptual approach**: Size limit with user notification. Options offered: "This looks like a long document. I can [summarize first part / help you upload it for full processing / review specific section]."

---

## Knowledge Base Edge Cases

### 16. KB Lookup Miss

**Scenario**: KB returns no relevant results for user question.

**Considerations**:
- **Fallback behavior**: Admit ignorance vs. generic response?
- **Re-query strategy**: Relax filters, try different query formulation?
- **Feedback loop**: Log miss for KB improvement?

**Conceptual approach**: Specialist acknowledges KB miss explicitly: "I don't have specific information about that in my knowledge base. Let me [check with a backend agent / connect you with support]."

### 17. Stale or Conflicting KB Results

**Scenario**: KB returns outdated information or contradictory results.

**Considerations**:
- **Confidence scoring**: Low-confidence results filtered or flagged?
- **Multi-source reconciliation**: How to handle conflicting snippets?
- **Recency indication**: Show result dates to user?

**Conceptual approach**: Include source metadata (date, version) in KB results. Specialist prompt instructs preference for recent sources. Contradictions → conservative response: "I found conflicting information about this. The most recent source suggests..."

### 18. KB Timeout

**Scenario**: KB query times out or errors.

**Considerations**:
- **Degradation**: Continue without KB context or error to user?
- **Retry**: Immediate retry or fail fast?
- **Fallback content**: Cached/previous results usable?

**Conceptual approach**: Fast timeout (500ms) with graceful degradation. Specialist continues with disclaimer: "I wasn't able to check the latest documentation, but generally speaking..."

---

## System and Integration Edge Cases

### 19. Specialist Unavailable

**Scenario**: Specialist chatbot fails to respond (error, timeout).

**Considerations**:
- **Failover**: Retry specialist, route to fallback, or error message?
- **Circuit breaker**: How many failures before marking specialist unhealthy?
- **User experience**: Transparent retry vs. opaque handling?

**Conceptual approach**: Single retry with timeout. Persistent failure → fallback: "I'm having trouble connecting with the [domain] specialist. Let me try to help or connect you with support."

### 20. Rate Limit Encounters

**Scenario**: User hits rate limit (too many messages too fast).

**Considerations**:
- **Threshold**: Messages per minute, per hour, per conversation?
- **Response**: Hard block with message vs. slow mode (delayed responses)?
- **Differentiation**: Authenticated vs. anonymous users?

**Conceptual approach**: Tiered limits. Soft limit: "I'm receiving messages quickly. I'll respond as fast as I can." Hard limit: "You've reached the message limit. Please try again in [time]." Authenticated users higher limits.

### 21. Concurrent User Load

**Scenario**: Platform under heavy load, chatbot response degrades.

**Considerations**:
- **Quality degradation**: Slower responses vs. shorter responses vs. queueing?
- **Priority handling**: VIP users, critical workflows prioritized?
- **Transparency**: Inform user of delay or seamless queueing?

**Conceptual approach**: Admission control with queue. If queue deep, inform user: "I'm experiencing high demand right now. Your message is queued and I'll respond shortly." Queue position optional.

---

## Edge Case Response Patterns

### Acknowledgment Strategies

| Scenario | Acknowledgment Pattern |
|---|---|
| Topic shift | "Switching to [new topic]..." |
| Background task start | "I'll look into that and get back to you." |
| Background task complete | "About [original request] — here's what I found..." |
| KB miss | "I don't have specific information on that..." |
| Specialist handoff | "Connecting you with [specialist]..." |
| Error recovery | "I had trouble with that. Let me try again..." |

### Graceful Degradation Hierarchy

1. **Full capability**: Streaming response with KB context, tools, background tasks
2. **Reduced capability**: Non-streaming, no KB, basic response
3. **Fallback response**: Pre-written help message, human handoff offer
4. **Error state**: Apology, alternative contact method

---

## Out of Scope (Implementation Detail)

The following are deferred to frontend/UI implementation:
- Specific UI components (typing indicators, message bubbles)
- Frontend frameworks (React, Vue, etc.)
- Real-time transport (WebSocket, SSE, polling)
- Mobile app vs. web specific behaviors
- Accessibility implementation (screen readers, keyboard nav)
- Analytics and user behavior tracking

---

## What to Read Next

These edge cases extend the v5 chatbot layer:

- **[06-v5 — Chatbot Layer](./06-v5-chatbot-layer.md)** — Return to the core chatbot architecture
- **[08-Performance and Scaling](./08-performance-and-scaling.md)** — Streaming backpressure and connection handling
- **[09-Security Model](./09-security-model.md)** — PII handling and content safety
- **[11-Testing Strategy](./11-testing-strategy.md)** — E2E conversation flow testing

Or explore the full documentation:
- [README](./README.md) — Complete documentation index
- [01-PRD](./01-PRD.md) — Project requirements and decisions

---

## Related Documents

- [06-v5 — Chatbot Layer](./06-v5-chatbot-layer.md) — Core chatbot architecture
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Streaming performance, backpressure
- [09-Security Model](./09-security-model.md) — PII handling, content safety
