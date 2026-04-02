# history.ts 核心逻辑分析

## 1. 模块概览

`history.ts` 是 Claude Code 中负责聊天记录保存和管理的核心模块，主要功能包括：

- 聊天记录的添加、存储和读取
- 粘贴内容的处理和管理
- 历史记录的持久化存储
- 历史记录的查询和检索

## 2. 导入和常量

```typescript
import { appendFile, writeFile } from 'fs/promises'
import { join } from 'path'
import { getProjectRoot, getSessionId } from './bootstrap/state.js'
import { registerCleanup } from './utils/cleanupRegistry.js'
import type { HistoryEntry, PastedContent } from './utils/config.js'
import { logForDebugging } from './utils/debug.js'
import { getClaudeConfigHomeDir, isEnvTruthy } from './utils/envUtils.js'
import { getErrnoCode } from './utils/errors.js'
import { readLinesReverse } from './utils/fsOperations.js'
import { lock } from './utils/lockfile.js'
import {
  hashPastedText,
  retrievePastedText,
  storePastedText,
} from './utils/pasteStore.js'
import { sleep } from './utils/sleep.js'
import { jsonParse, jsonStringify } from './utils/slowOperations.js'

const MAX_HISTORY_ITEMS = 100
const MAX_PASTED_CONTENT_LENGTH = 1024
```

**常量说明**：
- `MAX_HISTORY_ITEMS`：最大历史记录数量，默认为100
- `MAX_PASTED_CONTENT_LENGTH`：内联存储的最大粘贴内容长度，默认为1024

## 3. 类型定义

### 3.1 StoredPastedContent

```typescript
type StoredPastedContent = {
  id: number
  type: 'text' | 'image'
  content?: string // Inline content for small pastes
  contentHash?: string // Hash reference for large pastes stored externally
  mediaType?: string
  filename?: string
}
```

**字段说明**：
- `id`：粘贴内容的唯一标识
- `type`：内容类型，支持文本或图片
- `content`：内联存储的小文本内容
- `contentHash`：大文本内容的哈希引用
- `mediaType`：媒体类型
- `filename`：文件名

### 3.2 LogEntry

```typescript
type LogEntry = {
  display: string
  pastedContents: Record<number, StoredPastedContent>
  timestamp: number
  project: string
  sessionId?: string
}
```

**字段说明**：
- `display`：显示的文本内容
- `pastedContents`：粘贴的内容，键为ID，值为内容对象
- `timestamp`：时间戳
- `project`：项目路径
- `sessionId`：会话ID

## 4. 粘贴内容处理函数

### 4.1 getPastedTextRefNumLines

```typescript
export function getPastedTextRefNumLines(text: string): number {
  return (text.match(/\r\n|\r|\n/g) || []).length
}
```

**功能**：计算粘贴文本的行数
**参数**：`text` - 粘贴的文本内容
**返回值**：文本的行数（换行符的数量）

### 4.2 formatPastedTextRef

```typescript
export function formatPastedTextRef(id: number, numLines: number): string {
  if (numLines === 0) {
    return `[Pasted text #${id}]`
  }
  return `[Pasted text #${id} +${numLines} lines]`
}
```

**功能**：格式化粘贴文本的引用字符串
**参数**：
- `id` - 粘贴内容的ID
- `numLines` - 文本行数
**返回值**：格式化的引用字符串

### 4.3 formatImageRef

```typescript
export function formatImageRef(id: number): string {
  return `[Image #${id}]`
}
```

**功能**：格式化图片的引用字符串
**参数**：`id` - 图片的ID
**返回值**：格式化的引用字符串

### 4.4 parseReferences

```typescript
export function parseReferences(
  input: string,
): Array<{ id: number; match: string; index: number }> {
  const referencePattern =
    /\[(Pasted text|Image|\.\.\.Truncated text) #(\d+)(?: \+\d+ lines)?(\.)*\]/g
  const matches = [...input.matchAll(referencePattern)]
  return matches
    .map(match => ({
      id: parseInt(match[2] || '0'),
      match: match[0],
      index: match.index,
    }))
    .filter(match => match.id > 0)
}
```

**功能**：解析输入字符串中的粘贴内容引用
**参数**：`input` - 输入字符串
**返回值**：引用对象数组，每个对象包含ID、匹配字符串和索引

### 4.5 expandPastedTextRefs

```typescript
export function expandPastedTextRefs(
  input: string,
  pastedContents: Record<number, PastedContent>,
): string {
  const refs = parseReferences(input)
  let expanded = input
  // Splice at the original match offsets so placeholder-like strings inside
  // pasted content are never confused for real refs. Reverse order keeps
  // earlier offsets valid after later replacements.
  for (let i = refs.length - 1; i >= 0; i--) {
    const ref = refs[i]!
    const content = pastedContents[ref.id]
    if (content?.type !== 'text') continue
    expanded =
      expanded.slice(0, ref.index) +
      content.content +
      expanded.slice(ref.index + ref.match.length)
  }
  return expanded
}
```

**功能**：将输入字符串中的粘贴文本引用替换为实际内容
**参数**：
- `input` - 输入字符串
- `pastedContents` - 粘贴内容对象
**返回值**：替换后的字符串

## 5. 历史记录读取函数

### 5.1 deserializeLogEntry

```typescript
function deserializeLogEntry(line: string): LogEntry {
  return jsonParse(line) as LogEntry
}
```

**功能**：将JSON字符串反序列化为LogEntry对象
**参数**：`line` - JSON字符串
**返回值**：LogEntry对象

### 5.2 makeLogEntryReader

```typescript
async function* makeLogEntryReader(): AsyncGenerator<LogEntry> {
  const currentSession = getSessionId()

  // Start with entries that have yet to be flushed to disk
  for (let i = pendingEntries.length - 1; i >= 0; i--) {
    yield pendingEntries[i]!
  }

  // Read from global history file (shared across all projects)
  const historyPath = join(getClaudeConfigHomeDir(), 'history.jsonl')

  try {
    for await (const line of readLinesReverse(historyPath)) {
      try {
        const entry = deserializeLogEntry(line)
        // removeLastFromHistory slow path: entry was flushed before removal,
        // so filter here so both getHistory (Up-arrow) and makeHistoryReader
        // (ctrl+r search) skip it consistently.
        if (
          entry.sessionId === currentSession &&
          skippedTimestamps.has(entry.timestamp)
        ) {
          continue
        }
        yield entry
      } catch (error) {
        // Not a critical error - just skip malformed lines
        logForDebugging(`Failed to parse history line: ${error}`)
      }
    }
  } catch (e: unknown) {
    const code = getErrnoCode(e)
    if (code === 'ENOENT') {
      return
    }
    throw e
  }
}
```

**功能**：创建一个异步生成器，用于读取历史记录条目
**返回值**：LogEntry对象的异步生成器
**执行流程**：
1. 首先读取内存中的待处理条目（从新到旧）
2. 然后从历史文件中倒序读取（从新到旧）
3. 过滤掉已跳过的条目
4. 解析每行JSON并yield结果

### 5.3 makeHistoryReader

```typescript
export async function* makeHistoryReader(): AsyncGenerator<HistoryEntry> {
  for await (const entry of makeLogEntryReader()) {
    yield await logEntryToHistoryEntry(entry)
  }
}
```

**功能**：创建一个异步生成器，用于读取HistoryEntry对象
**返回值**：HistoryEntry对象的异步生成器

### 5.4 getTimestampedHistory

```typescript
export type TimestampedHistoryEntry = {
  display: string
  timestamp: number
  resolve: () => Promise<HistoryEntry>
}

export async function* getTimestampedHistory(): AsyncGenerator<TimestampedHistoryEntry> {
  const currentProject = getProjectRoot()
  const seen = new Set<string>()

  for await (const entry of makeLogEntryReader()) {
    if (!entry || typeof entry.project !== 'string') continue
    if (entry.project !== currentProject) continue
    if (seen.has(entry.display)) continue
    seen.add(entry.display)

    yield {
      display: entry.display,
      timestamp: entry.timestamp,
      resolve: () => logEntryToHistoryEntry(entry),
    }

    if (seen.size >= MAX_HISTORY_ITEMS) return
  }
}
```

**功能**：获取当前项目的带时间戳的历史记录（用于Ctrl+R搜索）
**返回值**：TimestampedHistoryEntry对象的异步生成器
**特点**：
- 按显示文本去重
- 按时间戳降序排列
- 粘贴内容延迟解析
- 最多返回MAX_HISTORY_ITEMS个条目

### 5.5 getHistory

```typescript
export async function* getHistory(): AsyncGenerator<HistoryEntry> {
  const currentProject = getProjectRoot()
  const currentSession = getSessionId()
  const otherSessionEntries: LogEntry[] = []
  let yielded = 0

  for await (const entry of makeLogEntryReader()) {
    // Skip malformed entries (corrupted file, old format, or invalid JSON structure)
    if (!entry || typeof entry.project !== 'string') continue
    if (entry.project !== currentProject) continue

    if (entry.sessionId === currentSession) {
      yield await logEntryToHistoryEntry(entry)
      yielded++
    } else {
      otherSessionEntries.push(entry)
    }

    // Same MAX_HISTORY_ITEMS window as before — just reordered within it.
    if (yielded + otherSessionEntries.length >= MAX_HISTORY_ITEMS) break
  }

  for (const entry of otherSessionEntries) {
    if (yielded >= MAX_HISTORY_ITEMS) return
    yield await logEntryToHistoryEntry(entry)
    yielded++
  }
}
```

**功能**：获取当前项目的历史记录
**返回值**：HistoryEntry对象的异步生成器
**特点**：
- 优先返回当前会话的条目
- 然后返回其他会话的条目
- 每个会话内部按时间戳降序排列
- 最多返回MAX_HISTORY_ITEMS个条目

## 6. 粘贴内容解析函数

### 6.1 resolveStoredPastedContent

```typescript
async function resolveStoredPastedContent(
  stored: StoredPastedContent,
): Promise<PastedContent | null> {
  // If we have inline content, use it directly
  if (stored.content) {
    return {
      id: stored.id,
      type: stored.type,
      content: stored.content,
      mediaType: stored.mediaType,
      filename: stored.filename,
    }
  }

  // If we have a hash reference, fetch from paste store
  if (stored.contentHash) {
    const content = await retrievePastedText(stored.contentHash)
    if (content) {
      return {
        id: stored.id,
        type: stored.type,
        content,
        mediaType: stored.mediaType,
        filename: stored.filename,
      }
    }
  }

  // Content not available
  return null
}
```

**功能**：解析存储的粘贴内容
**参数**：`stored` - 存储的粘贴内容
**返回值**：完整的PastedContent对象或null
**执行流程**：
1. 如果有内联内容，直接使用
2. 如果有哈希引用，从粘贴存储中获取
3. 如果内容不可用，返回null

### 6.2 logEntryToHistoryEntry

```typescript
async function logEntryToHistoryEntry(entry: LogEntry): Promise<HistoryEntry> {
  const pastedContents: Record<number, PastedContent> = {}

  for (const [id, stored] of Object.entries(entry.pastedContents || {})) {
    const resolved = await resolveStoredPastedContent(stored)
    if (resolved) {
      pastedContents[Number(id)] = resolved
    }
  }

  return {
    display: entry.display,
    pastedContents,
  }
}
```

**功能**：将LogEntry转换为HistoryEntry
**参数**：`entry` - LogEntry对象
**返回值**：HistoryEntry对象
**执行流程**：
1. 解析所有粘贴内容
2. 构建并返回HistoryEntry对象

## 7. 历史记录写入函数

### 7.1 immediateFlushHistory

```typescript
async function immediateFlushHistory(): Promise<void> {
  if (pendingEntries.length === 0) {
    return
  }

  let release
  try {
    const historyPath = join(getClaudeConfigHomeDir(), 'history.jsonl')

    // Ensure the file exists before acquiring lock (append mode creates if missing)
    await writeFile(historyPath, '', {
      encoding: 'utf8',
      mode: 0o600,
      flag: 'a',
    })

    release = await lock(historyPath, {
      stale: 10000,
      retries: {
        retries: 3,
        minTimeout: 50,
      },
    })

    const jsonLines = pendingEntries.map(entry => jsonStringify(entry) + '\n')
    pendingEntries = []

    await appendFile(historyPath, jsonLines.join(''), { mode: 0o600 })
  } catch (error) {
    logForDebugging(`Failed to write prompt history: ${error}`)
  } finally {
    if (release) {
      await release()
    }
  }
}
```

**功能**：将待处理的历史记录写入磁盘
**执行流程**：
1. 检查是否有待处理条目
2. 确保历史文件存在
3. 获取文件锁
4. 将待处理条目转换为JSON行
5. 追加到历史文件
6. 释放锁

### 7.2 flushPromptHistory

```typescript
async function flushPromptHistory(retries: number): Promise<void> {
  if (isWriting || pendingEntries.length === 0) {
    return
  }

  // Stop trying to flush history until the next user prompt
  if (retries > 5) {
    return
  }

  isWriting = true

  try {
    await immediateFlushHistory()
  } finally {
    isWriting = false

    if (pendingEntries.length > 0) {
      // Avoid trying again in a hot loop
      await sleep(500)

      void flushPromptHistory(retries + 1)
    }
  }
}
```

**功能**：尝试将待处理的历史记录写入磁盘
**参数**：`retries` - 重试次数
**执行流程**：
1. 检查是否正在写入或没有待处理条目
2. 检查重试次数是否超过限制
3. 设置写入标志
4. 调用immediateFlushHistory
5. 重置写入标志
6. 如果还有待处理条目，延迟后重试

### 7.3 addToPromptHistory

```typescript
async function addToPromptHistory(
  command: HistoryEntry | string,
): Promise<void> {
  const entry =
    typeof command === 'string'
      ? { display: command, pastedContents: {} }
      : command

  const storedPastedContents: Record<number, StoredPastedContent> = {}
  if (entry.pastedContents) {
    for (const [id, content] of Object.entries(entry.pastedContents)) {
      // Filter out images (they're stored separately in image-cache)
      if (content.type === 'image') {
        continue
      }

      // For small text content, store inline
      if (content.content.length <= MAX_PASTED_CONTENT_LENGTH) {
        storedPastedContents[Number(id)] = {
          id: content.id,
          type: content.type,
          content: content.content,
          mediaType: content.mediaType,
          filename: content.filename,
        }
      } else {
        // For large text content, compute hash synchronously and store reference
        // The actual disk write happens async (fire-and-forget)
        const hash = hashPastedText(content.content)
        storedPastedContents[Number(id)] = {
          id: content.id,
          type: content.type,
          contentHash: hash,
          mediaType: content.mediaType,
          filename: content.filename,
        }
        // Fire-and-forget disk write - don't block history entry creation
        void storePastedText(hash, content.content)
      }
    }
  }

  const logEntry: LogEntry = {
    ...entry,
    pastedContents: storedPastedContents,
    timestamp: Date.now(),
    project: getProjectRoot(),
    sessionId: getSessionId(),
  }

  pendingEntries.push(logEntry)
  lastAddedEntry = logEntry
  currentFlushPromise = flushPromptHistory(0)
  void currentFlushPromise
}
```

**功能**：将聊天记录添加到待处理队列
**参数**：`command` - 聊天记录（字符串或HistoryEntry对象）
**执行流程**：
1. 转换输入为HistoryEntry对象
2. 处理粘贴内容：
   - 过滤掉图片内容
   - 小文本直接内联存储
   - 大文本计算哈希并存储引用
3. 创建LogEntry对象
4. 添加到待处理队列
5. 触发异步写入

### 7.4 addToHistory

```typescript
export function addToHistory(command: HistoryEntry | string): void {
  // Skip history when running in a tmux session spawned by Claude Code's Tungsten tool.
  // This prevents verification/test sessions from polluting the user's real command history.
  if (isEnvTruthy(process.env.CLAUDE_CODE_SKIP_PROMPT_HISTORY)) {
    return
  }

  // Register cleanup on first use
  if (!cleanupRegistered) {
    cleanupRegistered = true
    registerCleanup(async () => {
      // If there's an in-progress flush, wait for it
      if (currentFlushPromise) {
        await currentFlushPromise
      }
      // If there are still pending entries after the flush completed, do one final flush
      if (pendingEntries.length > 0) {
        await immediateFlushHistory()
      }
    })
  }

  void addToPromptHistory(command)
}
```

**功能**：添加聊天记录到历史中
**参数**：`command` - 聊天记录（字符串或HistoryEntry对象）
**执行流程**：
1. 检查环境变量，决定是否跳过
2. 首次调用时注册清理函数
3. 调用addToPromptHistory

## 8. 辅助函数

### 8.1 clearPendingHistoryEntries

```typescript
export function clearPendingHistoryEntries(): void {
  pendingEntries = []
  lastAddedEntry = null
  skippedTimestamps.clear()
}
```

**功能**：清除待处理的历史记录条目

### 8.2 removeLastFromHistory

```typescript
export function removeLastFromHistory(): void {
  if (!lastAddedEntry) return
  const entry = lastAddedEntry
  lastAddedEntry = null

  const idx = pendingEntries.lastIndexOf(entry)
  if (idx !== -1) {
    pendingEntries.splice(idx, 1)
  } else {
    skippedTimestamps.add(entry.timestamp)
  }
}
```

**功能**：撤销最近的addToHistory调用
**执行流程**：
1. 检查是否有最近添加的条目
2. 尝试从待处理队列中移除
3. 如果已写入磁盘，添加到跳过时间戳集合

## 9. 模块状态变量

```typescript
let pendingEntries: LogEntry[] = []
let isWriting = false
let currentFlushPromise: Promise<void> | null = null
let cleanupRegistered = false
let lastAddedEntry: LogEntry | null = null
// Timestamps of entries already flushed to disk that should be skipped when
// reading. Used by removeLastFromHistory when the entry has raced past the
// pending buffer. Session-scoped (module state resets on process restart).
const skippedTimestamps = new Set<number>()
```

**状态变量说明**：
- `pendingEntries`：待处理的历史记录条目
- `isWriting`：是否正在写入磁盘
- `currentFlushPromise`：当前的写入Promise
- `cleanupRegistered`：是否已注册清理函数
- `lastAddedEntry`：最近添加的条目
- `skippedTimestamps`：已跳过的时间戳集合

## 10. 核心流程总结

### 10.1 记录添加流程

1. 调用 `addToHistory` 添加聊天记录
2. 检查环境变量，决定是否跳过
3. 首次调用时注册清理函数
4. 调用 `addToPromptHistory` 处理记录
5. 处理粘贴内容（过滤图片，小文本内联存储，大文本哈希引用）
6. 创建 `LogEntry` 对象并添加到待处理队列
7. 触发 `flushPromptHistory` 异步写入

### 10.2 记录存储流程

1. `flushPromptHistory` 检查写入状态和待处理条目
2. 调用 `immediateFlushHistory` 执行实际写入
3. 确保历史文件存在
4. 获取文件锁
5. 将待处理条目转换为JSON行并追加到文件
6. 释放锁
7. 如果还有待处理条目，延迟后重试

### 10.3 记录读取流程

1. 调用 `getHistory` 或 `getTimestampedHistory` 获取历史记录
2. 内部调用 `makeLogEntryReader` 读取条目
3. 首先读取内存中的待处理条目
4. 然后从历史文件中倒序读取
5. 过滤掉已跳过的条目
6. 将 `LogEntry` 转换为 `HistoryEntry` 并返回

## 11. 性能和安全考虑

### 11.1 性能优化

1. **内存缓冲**：使用 `pendingEntries` 数组缓冲记录，减少磁盘I/O次数
2. **异步写入**：后台异步写入，不阻塞主线程
3. **大文本处理**：大文本通过哈希存储到 paste store，减少历史文件大小
4. **批量写入**：将多个记录一次性写入，提高效率
5. **延迟解析**：粘贴内容延迟解析，提高读取速度

### 11.2 安全考虑

1. **文件权限**：历史文件使用 `0o600` 权限，确保只有所有者可读写
2. **并发安全**：使用文件锁避免并发写入冲突
3. **错误处理**：写入失败时记录日志但不影响主流程
4. **隐私保护**：敏感信息（如API密钥）不会存储在历史记录中

## 12. 代码优化建议

1. **错误处理增强**：
   - 在 `immediateFlushHistory` 中增加更详细的错误日志
   - 考虑添加重试机制的指数退避策略

2. **性能优化**：
   - 考虑使用更高效的JSON序列化/反序列化库
   - 对于频繁的历史记录操作，考虑使用内存缓存

3. **代码可读性**：
   - 增加更多的JSDoc注释
   - 提取重复的代码片段为单独的函数

4. **健壮性**：
   - 增加对历史文件大小的限制和清理机制
   - 考虑添加历史记录的备份和恢复功能

## 13. 总结

`history.ts` 是 Claude Code 中负责聊天记录管理的核心模块，通过以下设计实现了高效、安全的历史记录功能：

1. **分层存储**：小文本内联存储，大文本哈希引用，图片单独存储
2. **异步处理**：后台异步写入，提高用户体验
3. **并发安全**：使用文件锁确保多进程安全
4. **智能读取**：优先返回当前会话的记录，支持按项目过滤
5. **性能优化**：内存缓冲、批量写入、延迟解析等措施

这种设计既保证了聊天记录的完整保存，又兼顾了性能和安全性，为用户提供了良好的历史记录体验。