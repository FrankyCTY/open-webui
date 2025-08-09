# Message Rendering Pipeline: From Backend to UI

This document explains how chat messages, including "thinking" steps, flow from the backend through streaming to the frontend UI components.

## Overview

The message rendering pipeline consists of:
1. **Backend**: Generates content blocks (including reasoning) as structured data
2. **Transport**: Streams data via Server-Sent Events (SSE) and WebSocket
3. **Frontend**: Parses content through Markdown pipeline and renders UI components

## 1. Backend: Content Generation

### Reasoning Block Generation

The backend processes model responses and wraps reasoning content in structured blocks:

```python
# backend/open_webui/utils/middleware.py:1465-1483
elif block["type"] == "reasoning":
    reasoning_display_content = "\n".join(
        (f"> {line}" if not line.startswith(">") else line)
        for line in block["content"].splitlines()
    )
    
    reasoning_duration = block.get("duration", None)
    
    if reasoning_duration is not None:
        # Completed reasoning with duration
        content = f'{content}\n<details type="reasoning" done="true" duration="{reasoning_duration}">\n<summary>Thought for {reasoning_duration} seconds</summary>\n{reasoning_display_content}\n</details>\n'
    else:
        # In-progress reasoning
        content = f'{content}\n<details type="reasoning" done="false">\n<summary>Thinking…</summary>\n{reasoning_display_content}\n</details>\n'
```

### Stream Conversion (Ollama → OpenAI Format)

For Ollama models, reasoning content is extracted and converted:

```python
# backend/open_webui/utils/response.py:102-128
async def convert_streaming_response_ollama_to_openai(ollama_streaming_response):
    async for data in ollama_streaming_response.body_iterator:
        data = json.loads(data)
        
        reasoning_content = data.get("message", {}).get("thinking", None)
        message_content = data.get("message", {}).get("content", None)
        
        data = openai_chat_chunk_message_template(
            model, message_content, reasoning_content, openai_tool_calls, usage
        )
        
        yield f"data: {json.dumps(data)}\n\n"
```

## 2. Transport: Streaming to Frontend

### Server-Sent Events Processing

The frontend receives streaming data and forwards it through WebSocket:

```javascript
// src/routes/+layout.svelte:344-376
if (form_data?.stream ?? false) {
    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    
    const processStream = async () => {
        while (true) {
            const { done, value } = await reader.read();
            if (done) break;
            
            const chunk = decoder.decode(value, { stream: true });
            const lines = chunk.split('\n').filter((line) => line.trim() !== '');
            
            for (const line of lines) {
                $socket?.emit(channel, line);  // Forward to WebSocket
            }
        }
    };
    
    await processStream();
}
```

## 3. Frontend: Message Rendering Chain

### Component Hierarchy

The message rendering follows this component chain:

```
Messages.svelte
└── Message.svelte (per message)
    └── ResponseMessage.svelte (for assistant messages)
        └── ContentRenderer.svelte
            └── Markdown.svelte
                └── MarkdownTokens.svelte
```

### Messages List Component

```svelte
<!-- src/lib/components/chat/Messages.svelte:423-449 -->
{#each messages as message, messageIdx (message.id)}
    <Message
        {chatId}
        bind:history
        {selectedModels}
        messageId={message.id}
        idx={messageIdx}
        {user}
        <!-- ... other props ... -->
    />
{/each}
```

### Message Type Routing

```svelte
<!-- src/lib/components/chat/Messages/Message.svelte:70-93 -->
{:else if (history.messages[history.messages[messageId].parentId]?.models?.length ?? 1) === 1}
    <ResponseMessage
        {chatId}
        {history}
        {messageId}
        {selectedModels}
        isLastMessage={messageId === history.currentId}
        <!-- ... other props ... -->
    />
```

### Content Rendering

```svelte
<!-- src/lib/components/chat/Messages/ResponseMessage.svelte:799-852 -->
<ContentRenderer
    id={message.id}
    {history}
    {selectedModels}
    content={message.content}
    sources={message.sources}
    done={($settings?.chatFadeStreamingText ?? true) ? (message?.done ?? false) : true}
    {model}
    <!-- ... event handlers ... -->
/>
```

### Markdown Processing

```svelte
<!-- src/lib/components/chat/Messages/ContentRenderer.svelte:130-191 -->
<Markdown
    {id}
    {content}
    {model}
    {save}
    {preview}
    {done}
    sourceIds={/* ... */}
    {onSourceClick}
    {onTaskClick}
    {onSave}
    onUpdate={(token) => {
        // Artifact detection for HTML/SVG
        if ((['html', 'svg'].includes(lang) || (lang === 'xml' && code.includes('svg')))) {
            showArtifacts.set(true);
        }
    }}
/>
```

## 4. Markdown Pipeline: Custom Extensions

### Marked Setup with Extensions

```javascript
// src/lib/components/chat/Messages/Markdown.svelte:30-44
const options = { throwOnError: false, breaks: true };
marked.use(markedKatexExtension(options));
marked.use(markedExtension(options));  // Custom details extension

$: (async () => {
    if (content) {
        tokens = marked.lexer(
            replaceTokens(processResponseContent(content), sourceIds, model?.name, $user?.name)
        );
    }
})();
```

### Details Extension: Tokenization

The custom extension parses `<details>` blocks from the backend:

```typescript
// src/lib/utils/marked/extension.ts:29-60
function detailsTokenizer(src: string) {
    const detailsRegex = /^<details(\s+[^>]*)?>\n/;
    const summaryRegex = /^<summary>(.*?)<\/summary>\n/;
    
    const detailsMatch = detailsRegex.exec(src);
    if (detailsMatch) {
        const endIndex = findMatchingClosingTag(src, '<details', '</details>');
        const fullMatch = src.slice(0, endIndex);
        const detailsTag = detailsMatch[0];
        const attributes = parseAttributes(detailsTag);
        
        let content = fullMatch.slice(detailsTag.length, -10).trim();
        let summary = '';
        
        const summaryMatch = summaryRegex.exec(content);
        if (summaryMatch) {
            summary = summaryMatch[1].trim();
            content = content.slice(summaryMatch[0].length).trim();
        }
        
        return {
            type: 'details',
            raw: fullMatch,
            summary: summary,
            text: content,
            attributes: attributes
        };
    }
}
```

### Token Rendering: Svelte Components

**Important**: The `detailsRenderer` function is NOT used for actual UI rendering. Instead, tokens are rendered by Svelte components:

```svelte
<!-- src/lib/components/chat/Messages/Markdown/MarkdownTokens.svelte:283-301 -->
{:else if token.type === 'details'}
    <Collapsible
        title={token.summary}
        open={$settings?.expandDetails ?? false}
        attributes={token?.attributes}
        className="w-full space-y-1"
        dir="auto"
    >
        <div class="mb-1.5" slot="content">
            <svelte:self
                id={`${id}-${tokenIdx}-d`}
                tokens={marked.lexer(token.text)}
                attributes={token?.attributes}
                {done}
                {onTaskClick}
                {onSourceClick}
            />
        </div>
    </Collapsible>
```

## 5. UI Components: Collapsible Reasoning Display

### Dynamic Title Based on State

The `Collapsible` component shows different titles based on reasoning state:

```svelte
<!-- src/lib/components/common/Collapsible.svelte:114-129 -->
{#if attributes?.type === 'reasoning'}
    {#if attributes?.done === 'true' && attributes?.duration}
        {#if attributes.duration < 60}
            {$i18n.t('Thought for {{DURATION}} seconds', {
                DURATION: attributes.duration
            })}
        {:else}
            {$i18n.t('Thought for {{DURATION}}', {
                DURATION: dayjs.duration(attributes.duration, 'seconds').humanize()
            })}
        {/if}
    {:else}
        {$i18n.t('Thinking...')}
    {/if}
{/if}
```

### Loading States and Animations

While reasoning is in progress, a shimmer effect and spinner are shown:

```svelte
<!-- src/lib/components/common/Collapsible.svelte:102-112 -->
<div class="w-full font-medium flex items-center justify-between gap-2 {attributes?.done &&
    attributes?.done !== 'true' ? 'shimmer' : ''}">
    
    {#if attributes?.done && attributes?.done !== 'true'}
        <div>
            <Spinner className="size-4" />
        </div>
    {/if}
    
    <!-- Title content -->
</div>
```

### Status Indicators During Generation

Before content appears, status indicators show progress:

```svelte
<!-- src/lib/components/chat/Messages/ResponseMessage.svelte:794-799 -->
{#if message.content === '' && !message.error && (message?.statusHistory ?? [...(message?.status ? [message?.status] : [])]).length === 0}
    <Skeleton />
{:else if message.content && message.error !== true}
    <ContentRenderer ... />
{/if}
```

## 6. Styling and Visual Design

### Tailwind CSS Classes

The reasoning UI uses Tailwind utility classes for styling:

- **Layout**: `w-full`, `flex`, `items-center`, `justify-between`
- **Spacing**: `gap-2`, `mb-1.5`, `space-y-1`
- **Typography**: `font-medium`, `text-sm`
- **Interactions**: `cursor-pointer`, `hover:bg-gray-100`
- **Animations**: `shimmer` (custom class), `transition`

### Responsive Design

- **Mobile**: Collapsible headers stack properly on small screens
- **Desktop**: Full expand/collapse functionality with chevron indicators
- **Dark Mode**: Automatic color scheme adaptation via Tailwind dark: variants

## 7. Content Processing and Cleanup

### Removing Details for Copy/TTS

When copying text or using text-to-speech, reasoning details are stripped:

```javascript
// src/lib/utils/index.ts:850-875
export const removeAllDetails = (content) => {
    content = content.replace(/<details[^>]*>.*?<\/details>/gis, '');
    return content;
};

export const processDetails = (content) => {
    content = removeDetails(content, ['reasoning', 'code_interpreter']);
    
    // Convert tool_calls <details> to string results
    const detailsRegex = /<details\s+type="tool_calls"([^>]*)>([\s\S]*?)<\/details>/gis;
    const matches = content.match(detailsRegex);
    if (matches) {
        for (const match of matches) {
            const attributes = {};
            // Parse attributes and replace with result
            content = content.replace(match, `"${attributes.result}"`);
        }
    }
    
    return content;
};
```

## Summary

The message rendering pipeline transforms backend reasoning blocks into interactive UI components:

1. **Backend** generates `<details type="reasoning">` with state attributes
2. **Streaming** delivers content incrementally via SSE/WebSocket
3. **Markdown** parsing extracts details tokens with custom extension
4. **Svelte components** render tokens as interactive `Collapsible` elements
5. **Styling** applies via Tailwind classes and custom animations
6. **State management** shows "Thinking..." → "Thought for X seconds" progression

The key insight is that the backend's HTML `<details>` blocks are parsed as structured data and rendered through Svelte components, not as raw HTML, providing better control over styling and interactivity.
