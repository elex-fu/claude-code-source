# Claude Code Voice 系统技术分析

## 1. 模块总结介绍

Voice 系统是 Claude Code 的「语音输入中枢」，实现了语音到文本的实时转换功能。该系统支持多种音频捕获方式，包括原生音频模块和外部工具回退。

核心特性：
- **多平台音频捕获**：macOS CoreAudio、Linux ALSA/PulseAudio、Windows
- **静默检测**：自动检测语音结束
- **流式转录**：实时语音识别
- **多后端支持**：本地模型和云端 API

## 2. 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                       Voice System                           │
├─────────────────────────────────────────────────────────────┤
│  Audio Capture                                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  Native     │  │   SoX       │  │      arecord        │ │
│  │  (NAPI)     │  │  (Fallback) │  │    (ALSA Fallback)  │ │
│  │             │  │             │  │                     │ │
│  │ • macOS     │  │ • Cross-plat│  │ • Linux only        │ │
│  │ • Linux     │  │ • Silence   │  │ • Direct ALSA       │ │
│  │ • Windows   │  │   detect    │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│           │                                                   │
│           ▼                                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Speech-to-Text Engine                       │ │
│  │                                                          │ │
│  │  • Streaming STT  • Keyword spotting  • Voice activity  │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. 音频捕获实现

### 3.1 原生音频模块（NAPI）

```typescript
// voice.ts
// 延迟加载原生模块，避免启动时阻塞
let audioNapi: typeof import('audio-capture-napi') | null = null
let audioNapiPromise: Promise<typeof import('audio-capture-napi')> | null = null

function loadAudioNapi(): Promise<typeof import('audio-capture-napi')> {
  audioNapiPromise ??= (async () => {
    const t0 = Date.now()
    const mod = await import('audio-capture-napi')
    // 触发实际加载
    mod.isNativeAudioAvailable()
    audioNapi = mod
    logForDebugging(`[voice] audio-capture-napi loaded in ${Date.now() - t0}ms`)
    return mod
  })()
  return audioNapiPromise
}
```

### 3.2 音频参数配置

```typescript
const RECORDING_SAMPLE_RATE = 16000  // 16kHz - 语音识别最佳采样率
const RECORDING_CHANNELS = 1          // 单声道

// SoX 静默检测参数
const SILENCE_DURATION_SECS = '2.0'   // 2秒静默后停止
const SILENCE_THRESHOLD = '3%'        // 静默阈值
```

### 3.3 平台适配

```typescript
export async function startRecording(
  onData: (chunk: Buffer) => void,
  onEnd: () => void
): Promise<RecordingSession> {
  // 1. 尝试原生模块
  const native = await tryNativeRecording()
  if (native) {
    return native
  }

  // 2. 回退到 SoX
  const sox = await trySoxRecording()
  if (sox) {
    return sox
  }

  // 3. Linux 回退到 arecord
  if (getPlatform() === 'linux') {
    const arecord = await tryArecordRecording()
    if (arecord) {
      return arecord
    }
  }

  throw new Error('No audio capture method available')
}
```

## 4. 录制实现

### 4.1 SoX 录制

```typescript
async function startSoxRecording(
  onData: (chunk: Buffer) => void,
  onEnd: () => void
): Promise<RecordingSession> {
  const args = [
    '-q',                                      // 静默模式
    '-t', 'coreaudio', 'default',             // macOS 输入源
    '-r', String(RECORDING_SAMPLE_RATE),      // 采样率
    '-c', String(RECORDING_CHANNELS),         // 通道数
    '-e', 'signed-integer',                   // 编码
    '-b', '16',                               // 位深
    '-t', 'raw',                              // 输出格式
    '-',                                      // 输出到 stdout
    'silence',                                // 静默检测
    '1', '0.1', SILENCE_THRESHOLD,            // 开始前忽略
    '1', SILENCE_DURATION_SECS, SILENCE_THRESHOLD, // 结束后停止
  ]

  const process = spawn('sox', args)
  
  process.stdout.on('data', onData)
  process.on('close', onEnd)

  return {
    stop: () => process.kill('SIGTERM'),
    isActive: () => !process.killed,
  }
}
```

### 4.2 arecord 录制（Linux）

```typescript
async function startArecordRecording(
  onData: (chunk: Buffer) => void,
  onEnd: () => void
): Promise<RecordingSession | null> {
  // 先探测设备可用性
  const probe = await probeArecord()
  if (!probe.ok) {
    return null
  }

  const args = [
    '-f', 'S16_LE',                    // 16位小端
    '-r', String(RECORDING_SAMPLE_RATE),
    '-c', String(RECORDING_CHANNELS),
    '-t', 'raw',                       // 原始 PCM
    '-',                               // 输出到 stdout
  ]

  const process = spawn('arecord', args)
  
  process.stdout.on('data', onData)
  process.stderr.on('data', (data) => {
    logError('[arecord]', data.toString())
  })
  process.on('close', onEnd)

  return {
    stop: () => process.kill('SIGTERM'),
    isActive: () => !process.killed,
  }
}
```

## 5. 流式语音识别

### 5.1 音频流处理

```typescript
// voiceStreamSTT.ts
export async function* streamSpeechToText(
  audioStream: AsyncIterable<Buffer>
): AsyncGenerator<string, void, unknown> {
  const buffer: Buffer[] = []
  let totalBytes = 0

  for await (const chunk of audioStream) {
    buffer.push(chunk)
    totalBytes += chunk.length

    // 累积足够数据后发送（1秒音频 ≈ 32KB @ 16kHz/16bit/mono）
    if (totalBytes >= 32000) {
      const audioData = Buffer.concat(buffer)
      buffer.length = 0
      totalBytes = 0

      const text = await transcribeChunk(audioData)
      if (text) {
        yield text
      }
    }
  }

  // 处理剩余音频
  if (buffer.length > 0) {
    const text = await transcribeChunk(Buffer.concat(buffer))
    if (text) {
      yield text
    }
  }
}
```

### 5.2 分段转录

```typescript
async function transcribeChunk(audioData: Buffer): Promise<string | null> {
  const response = await fetch(config.sttEndpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'audio/raw',
      'X-Sample-Rate': '16000',
      'X-Channels': '1',
    },
    body: audioData,
  })

  if (!response.ok) {
    logError('STT request failed:', response.statusText)
    return null
  }

  const result = await response.json()
  return result.text || null
}
```

## 6. 关键词检测

### 6.1 唤醒词识别

```typescript
// voiceKeyterms.ts
const WAKE_WORDS = ['hey claude', 'okay claude', 'claude']

export function detectWakeWord(
  transcript: string
): { detected: boolean; confidence: number } {
  const lower = transcript.toLowerCase()
  
  for (const word of WAKE_WORDS) {
    if (lower.includes(word)) {
      return { detected: true, confidence: 1.0 }
    }
    
    // 模糊匹配
    const similarity = calculateSimilarity(lower, word)
    if (similarity > 0.8) {
      return { detected: true, confidence: similarity }
    }
  }

  return { detected: false, confidence: 0 }
}
```

## 7. 与 UI 集成

### 7.1 语音输入组件

```typescript
// components/VoiceInput.tsx
export function VoiceInput({ onTranscript }: VoiceInputProps) {
  const [isRecording, setIsRecording] = useState(false)
  const [transcript, setTranscript] = useState('')

  const startVoiceInput = useCallback(async () => {
    setIsRecording(true)
    setTranscript('')

    const recording = await startRecording(
      // 音频数据回调
      (chunk) => {
        // 流式处理
      },
      // 录制结束
      () => {
        setIsRecording(false)
      }
    )

    // 启动 STT 流
    const sttStream = streamSpeechToText(recording.audioStream)
    
    for await (const text of sttStream) {
      setTranscript((prev) => prev + ' ' + text)
      onTranscript(text)
    }
  }, [onTranscript])

  return (
    <Box>
      <Button onClick={startVoiceInput}>
        {isRecording ? '🔴 Recording...' : '🎤 Voice Input'}
      </Button>
      {transcript && <Text>{transcript}</Text>}
    </Box>
  )
}
```

## 8. 错误处理

### 8.1 设备不可用处理

```typescript
async function tryNativeRecording(): Promise<RecordingSession | null> {
  try {
    const napi = await loadAudioNapi()
    if (!napi.isNativeAudioAvailable()) {
      return null
    }
    return napi.startRecording()
  } catch (error) {
    logForDebugging('[voice] Native recording failed:', error)
    return null
  }
}

async function trySoxRecording(): Promise<RecordingSession | null> {
  if (!hasCommand('sox')) {
    return null
  }
  // ...
}
```

### 8.2 权限错误

```typescript
export async function checkMicrophonePermission(): Promise<boolean> {
  const platform = getPlatform()

  if (platform === 'macos') {
    // macOS 需要麦克风权限
    const status = execSync('tccutil check Microphone', { encoding: 'utf-8' })
    return status.includes('allowed')
  }

  if (platform === 'linux') {
    // Linux 通常无需显式权限检查
    return true
  }

  return false
}
```

---

Voice 系统通过多平台适配、流式处理、关键词检测等设计，实现了无缝的语音输入体验。理解其设计，对于构建语音交互功能具有参考价值。
