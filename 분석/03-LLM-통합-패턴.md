# LLM 통합 패턴

> **문서 목적**: Void 프로젝트의 멀티 프로바이더 LLM 아키텍처와 메시지 파이프라인 분석
> **대상 독자**: 코드 분석/생성 에이전트 개발자
> **최종 수정**: 2025-12-14

## 개요

Void는 20개 이상의 LLM 프로바이더를 통합하여 코드 분석 및 생성 작업을 수행하는 멀티 프로바이더 아키텍처를 구현한다. 본 문서는 메시지 파이프라인, 스트리밍 처리, 컨텍스트 관리, 그리고 프롬프트 엔지니어링 패턴을 상세히 분석한다.

## 핵심 아키텍처

### 1. 멀티 프로바이더 시스템

#### 1.1 지원 프로바이더 목록 (20+)

**주요 프로바이더**:
- **상용 API**: OpenAI, Anthropic, Google Gemini, xAI (Grok), DeepSeek
- **API 집계 서비스**: OpenRouter, LiteLLM, Azure, AWS Bedrock, Google Vertex
- **로컬/셀프호스팅**: Ollama, vLLM, LM Studio, OpenAI-Compatible

**프로바이더별 특징**:
```typescript
// src/vs/workbench/contrib/void/common/modelCapabilities.ts

const modelSettingsOfProvider: {
  [providerName in ProviderName]: VoidStaticProviderInfo
} = {
  openAI: openAISettings,           // GPT-4.1, o3, o4-mini
  anthropic: anthropicSettings,     // Claude Opus 4, Sonnet 4, 3.7-sonnet
  xAI: xAISettings,                 // Grok-2, Grok-3
  gemini: geminiSettings,           // Gemini 2.5 Pro, Flash
  deepseek: deepseekSettings,       // DeepSeek Chat, Reasoner
  groq: groqSettings,               // Llama, QwQ
  mistral: mistralSettings,         // Codestral, Devstral
  ollama: ollamaSettings,           // 로컬 모델 지원
  // ... 기타 12개 프로바이더
}
```

#### 1.2 프로바이더 추상화 레이어

**공통 인터페이스**:
```typescript
// src/vs/workbench/contrib/void/electron-main/llmMessage/sendLLMMessage.impl.ts

type CallFnOfProvider = {
  [providerName in ProviderName]: {
    sendChat: (params: SendChatParams_Internal) => Promise<void>;
    sendFIM: ((params: SendFIMParams_Internal) => void) | null;  // Fill-in-Middle
    list: ((params: ListParams_Internal<any>) => void) | null;   // 모델 목록 조회
  }
}

export const sendLLMMessageToProviderImplementation = {
  anthropic: {
    sendChat: sendAnthropicChat,
    sendFIM: null,  // Anthropic은 FIM 미지원
    list: null,
  },
  openAI: {
    sendChat: (params) => _sendOpenAICompatibleChat(params),
    sendFIM: null,
    list: null,
  },
  ollama: {
    sendChat: (params) => _sendOpenAICompatibleChat(params),
    sendFIM: sendOllamaFIM,  // FIM 지원
    list: ollamaList,        // 로컬 모델 목록 조회 지원
  },
  // ... 기타 프로바이더
}
```

**프로바이더별 SDK 인스턴스화**:
```typescript
// OpenAI-Compatible 프로바이더 처리
const newOpenAICompatibleSDK = async ({
  settingsOfProvider,
  providerName,
  includeInPayload
}: {
  settingsOfProvider: SettingsOfProvider,
  providerName: ProviderName,
  includeInPayload?: { [s: string]: any }
}) => {
  const commonPayloadOpts: ClientOptions = {
    dangerouslyAllowBrowser: true,
    ...includeInPayload,
  }

  if (providerName === 'openAI') {
    const config = settingsOfProvider[providerName]
    return new OpenAI({ apiKey: config.apiKey, ...commonPayloadOpts })
  }
  else if (providerName === 'ollama') {
    const config = settingsOfProvider[providerName]
    return new OpenAI({
      baseURL: `${config.endpoint}/v1`,
      apiKey: 'noop',
      ...commonPayloadOpts
    })
  }
  else if (providerName === 'openRouter') {
    const config = settingsOfProvider[providerName]
    return new OpenAI({
      baseURL: 'https://openrouter.ai/api/v1',
      apiKey: config.apiKey,
      defaultHeaders: {
        'HTTP-Referer': 'https://voideditor.com',
        'X-Title': 'Void',
      },
      ...commonPayloadOpts,
    })
  }
  // ... 기타 15+ 프로바이더
}
```

### 2. 메시지 파이프라인

#### 2.1 전체 메시지 흐름

```
React UI Component
  ↓ (sendLLMMessage 호출)
ILLMMessageService (Browser 프로세스)
  ↓ (IPC: 'void-channel-llmMessage')
LLMMessageChannel (Main 프로세스)
  ↓ (sendLLMMessageToProviderImplementation)
Provider Implementation (OpenAI/Anthropic/etc)
  ↓ (AsyncIterable 스트림)
Event Listeners (onText, onFinalMessage, onError)
  ↓ (IPC: 이벤트 채널)
React UI Component (스트리밍 업데이트)
```

#### 2.2 서비스 레이어 구현

**ILLMMessageService 인터페이스**:
```typescript
// src/vs/workbench/contrib/void/common/sendLLMMessageService.ts

export interface ILLMMessageService {
  readonly _serviceBrand: undefined;
  sendLLMMessage: (params: ServiceSendLLMMessageParams) => string | null;
  abort: (requestId: string) => void;
  ollamaList: (params: ServiceModelListParams<OllamaModelResponse>) => void;
  openAICompatibleList: (params: ServiceModelListParams<OpenaiCompatibleModelResponse>) => void;
}
```

**LLMMessageService 구현**:
```typescript
export class LLMMessageService extends Disposable implements ILLMMessageService {
  readonly _serviceBrand: undefined;
  private readonly channel: IChannel // LLMMessageChannel

  // 요청 ID별 훅 관리
  private readonly llmMessageHooks = {
    onText: {} as { [eventId: string]: ((params: EventLLMMessageOnTextParams) => void) },
    onFinalMessage: {} as { [eventId: string]: ((params: EventLLMMessageOnFinalMessageParams) => void) },
    onError: {} as { [eventId: string]: ((params: EventLLMMessageOnErrorParams) => void) },
    onAbort: {} as { [eventId: string]: (() => void) },
  }

  constructor(
    @IMainProcessService private readonly mainProcessService: IMainProcessService,
    @IVoidSettingsService private readonly voidSettingsService: IVoidSettingsService,
    @IMCPService private readonly mcpService: IMCPService,
  ) {
    super()

    // IPC 채널 설정
    this.channel = this.mainProcessService.getChannel('void-channel-llmMessage')

    // 이벤트 리스너 설정 (한 번만 설정, 이후 훅으로 라우팅)
    this._register((this.channel.listen('onText_sendLLMMessage'))(e => {
      this.llmMessageHooks.onText[e.requestId]?.(e)
    }))

    this._register((this.channel.listen('onFinalMessage_sendLLMMessage'))(e => {
      this.llmMessageHooks.onFinalMessage[e.requestId]?.(e);
      this._clearChannelHooks(e.requestId)  // 완료 시 정리
    }))

    this._register((this.channel.listen('onError_sendLLMMessage'))(e => {
      this.llmMessageHooks.onError[e.requestId]?.(e);
      this._clearChannelHooks(e.requestId);
    }))
  }

  sendLLMMessage(params: ServiceSendLLMMessageParams) {
    const { onText, onFinalMessage, onError, onAbort, modelSelection, ...proxyParams } = params;

    // 모델 선택 검증
    if (modelSelection === null) {
      const message = `Please add a provider in Void's Settings.`
      onError({ message, fullError: null })
      return null
    }

    // 메시지 유효성 검증
    if (params.messagesType === 'chatMessages' && (params.messages?.length ?? 0) === 0) {
      onError({ message: `No messages detected.`, fullError: null })
      return null
    }

    // MCP 도구 가져오기
    const mcpTools = this.mcpService.getMCPTools()

    // 요청 ID 생성 및 훅 등록
    const requestId = generateUuid();
    this.llmMessageHooks.onText[requestId] = onText
    this.llmMessageHooks.onFinalMessage[requestId] = onFinalMessage
    this.llmMessageHooks.onError[requestId] = onError
    this.llmMessageHooks.onAbort[requestId] = onAbort

    // IPC 채널을 통해 Main 프로세스로 전송
    this.channel.call('sendLLMMessage', {
      ...proxyParams,
      requestId,
      settingsOfProvider: this.voidSettingsService.state.settingsOfProvider,
      modelSelection,
      mcpTools,
    } satisfies MainSendLLMMessageParams);

    return requestId
  }

  abort(requestId: string) {
    this.llmMessageHooks.onAbort[requestId]?.()  // 즉시 호출
    this.channel.call('abort', { requestId } satisfies MainLLMMessageAbortParams);
    this._clearChannelHooks(requestId)
  }

  private _clearChannelHooks(requestId: string) {
    delete this.llmMessageHooks.onText[requestId]
    delete this.llmMessageHooks.onFinalMessage[requestId]
    delete this.llmMessageHooks.onError[requestId]
  }
}
```

#### 2.3 메시지 타입 시스템

**메시지 역할 정의**:
```typescript
// src/vs/workbench/contrib/void/common/sendLLMMessageTypes.ts

// Anthropic 메시지 형식
export type AnthropicLLMChatMessage = {
  role: 'assistant',
  content: string | (
    AnthropicReasoning |
    { type: 'text'; text: string } |
    { type: 'tool_use'; name: string; input: Record<string, any>; id: string; }
  )[];
} | {
  role: 'user',
  content: string | (
    { type: 'text'; text: string; } |
    { type: 'tool_result'; tool_use_id: string; content: string; }
  )[]
}

// OpenAI 메시지 형식
export type OpenAILLMChatMessage = {
  role: 'system' | 'user' | 'developer';
  content: string;
} | {
  role: 'assistant',
  content: string | (AnthropicReasoning | { type: 'text'; text: string })[];
  tool_calls?: {
    type: 'function';
    id: string;
    function: { name: string; arguments: string; }
  }[];
} | {
  role: 'tool',
  content: string;
  tool_call_id: string;
}

// Gemini 메시지 형식
export type GeminiLLMChatMessage = {
  role: 'model'
  parts: (
    | { text: string; }
    | { functionCall: { id: string; name: ToolName, args: Record<string, unknown> } }
  )[];
} | {
  role: 'user';
  parts: (
    | { text: string; }
    | { functionResponse: { id: string; name: ToolName, response: { output: string } } }
  )[];
}

// 통합 메시지 타입
export type LLMChatMessage = AnthropicLLMChatMessage | OpenAILLMChatMessage | GeminiLLMChatMessage
```

**메시지 전송 파라미터**:
```typescript
type SendLLMType = {
  messagesType: 'chatMessages';
  messages: LLMChatMessage[];
  separateSystemMessage: string | undefined;
  chatMode: ChatMode | null;
} | {
  messagesType: 'FIMMessage';  // Fill-in-Middle (자동완성)
  messages: LLMFIMMessage;
  separateSystemMessage?: undefined;
  chatMode?: undefined;
}

export type ServiceSendLLMMessageParams = {
  onText: OnText;
  onFinalMessage: OnFinalMessage;
  onError: OnError;
  logging: { loggingName: string, loggingExtras?: { [k: string]: any } };
  modelSelection: ModelSelection | null;
  modelSelectionOptions: ModelSelectionOptions | undefined;
  overridesOfModel: OverridesOfModel | undefined;
  onAbort: OnAbort;
} & SendLLMType;
```

### 3. 스트리밍 응답 처리

#### 3.1 AsyncIterable 패턴

**OpenAI 스트리밍**:
```typescript
// _sendOpenAICompatibleChat 구현 발췌

let fullReasoningSoFar = ''
let fullTextSoFar = ''
let toolName = ''
let toolId = ''
let toolParamsStr = ''

openai.chat.completions
  .create(options)
  .then(async response => {
    _setAborter(() => response.controller.abort())

    // AsyncIterable 스트림 처리
    for await (const chunk of response) {
      // 메시지 텍스트
      const newText = chunk.choices[0]?.delta?.content ?? ''
      fullTextSoFar += newText

      // 도구 호출
      for (const tool of chunk.choices[0]?.delta?.tool_calls ?? []) {
        const index = tool.index
        if (index !== 0) continue

        toolName += tool.function?.name ?? ''
        toolParamsStr += tool.function?.arguments ?? '';
        toolId += tool.id ?? ''
      }

      // 추론 (Reasoning)
      let newReasoning = ''
      if (nameOfReasoningFieldInDelta) {
        newReasoning = (chunk.choices[0]?.delta?.[nameOfReasoningFieldInDelta] || '') + ''
        fullReasoningSoFar += newReasoning
      }

      // onText 콜백 호출
      onText({
        fullText: fullTextSoFar,
        fullReasoning: fullReasoningSoFar,
        toolCall: !toolName ? undefined : {
          name: toolName,
          rawParams: {},
          isDone: false,
          doneParams: [],
          id: toolId
        },
      })
    }

    // 완료 처리
    if (!fullTextSoFar && !fullReasoningSoFar && !toolName) {
      onError({ message: 'Void: Response from model was empty.', fullError: null })
    } else {
      const toolCall = rawToolCallObjOfParamsStr(toolName, toolParamsStr, toolId)
      const toolCallObj = toolCall ? { toolCall } : {}
      onFinalMessage({
        fullText: fullTextSoFar,
        fullReasoning: fullReasoningSoFar,
        anthropicReasoning: null,
        ...toolCallObj
      });
    }
  })
  .catch(error => {
    if (error instanceof OpenAI.APIError && error.status === 401) {
      onError({ message: invalidApiKeyMessage(providerName), fullError: error });
    } else {
      onError({ message: error + '', fullError: error });
    }
  })
```

**Anthropic 이벤트 기반 스트리밍**:
```typescript
// sendAnthropicChat 구현 발췌

const stream = anthropic.messages.stream({
  system: separateSystemMessage ?? undefined,
  messages: messages as AnthropicLLMChatMessage[],
  model: modelName,
  max_tokens: maxTokens ?? 4_096,
  ...includeInPayload,
  ...nativeToolsObj,
})

let fullText = ''
let fullReasoning = ''
let fullToolName = ''
let fullToolParams = ''

// 이벤트 핸들러
stream.on('streamEvent', e => {
  // 블록 시작
  if (e.type === 'content_block_start') {
    if (e.content_block.type === 'text') {
      if (fullText) fullText += '\n\n'  // 2번째 텍스트 블록
      fullText += e.content_block.text
      runOnText()
    }
    else if (e.content_block.type === 'thinking') {
      if (fullReasoning) fullReasoning += '\n\n'  // 2번째 추론 블록
      fullReasoning += e.content_block.thinking
      runOnText()
    }
    else if (e.content_block.type === 'tool_use') {
      fullToolName += e.content_block.name ?? ''
      runOnText()
    }
  }

  // 델타 (증분 업데이트)
  else if (e.type === 'content_block_delta') {
    if (e.delta.type === 'text_delta') {
      fullText += e.delta.text
      runOnText()
    }
    else if (e.delta.type === 'thinking_delta') {
      fullReasoning += e.delta.thinking
      runOnText()
    }
    else if (e.delta.type === 'input_json_delta') {  // 도구 사용
      fullToolParams += e.delta.partial_json ?? ''
      runOnText()
    }
  }
})

// 완료 핸들러
stream.on('finalMessage', (response) => {
  const anthropicReasoning = response.content.filter(c =>
    c.type === 'thinking' || c.type === 'redacted_thinking'
  )
  const tools = response.content.filter(c => c.type === 'tool_use')
  const toolCall = tools[0] && rawToolCallObjOfAnthropicParams(tools[0])
  const toolCallObj = toolCall ? { toolCall } : {}

  onFinalMessage({ fullText, fullReasoning, anthropicReasoning, ...toolCallObj })
})

// 에러 핸들러
stream.on('error', (error) => {
  if (error instanceof Anthropic.APIError && error.status === 401) {
    onError({ message: invalidApiKeyMessage(providerName), fullError: error })
  } else {
    onError({ message: error + '', fullError: error })
  }
})
```

#### 3.2 콜백 패턴

**OnText/OnFinalMessage/OnError 타입**:
```typescript
export type OnText = (p: {
  fullText: string;
  fullReasoning: string;
  toolCall?: RawToolCallObj
}) => void

export type OnFinalMessage = (p: {
  fullText: string;
  fullReasoning: string;
  toolCall?: RawToolCallObj;
  anthropicReasoning: AnthropicReasoning[] | null
}) => void

export type OnError = (p: {
  message: string;
  fullError: Error | null
}) => void

export type OnAbort = () => void
```

**도구 호출 객체**:
```typescript
export type RawToolParamsObj = {
  [paramName in ToolParamName<ToolName>]?: string;
}

export type RawToolCallObj = {
  name: ToolName;
  rawParams: RawToolParamsObj;
  doneParams: ToolParamName<ToolName>[];
  id: string;
  isDone: boolean;
};

// 파라미터 문자열을 RawToolCallObj로 변환
const rawToolCallObjOfParamsStr = (
  name: string,
  toolParamsStr: string,
  id: string
): RawToolCallObj | null => {
  let input: unknown
  try {
    input = JSON.parse(toolParamsStr)
  } catch (e) {
    return null
  }

  if (input === null) return null
  if (typeof input !== 'object') return null

  const rawParams: RawToolParamsObj = input
  return {
    id,
    name,
    rawParams,
    doneParams: Object.keys(rawParams),
    isDone: true
  }
}
```

### 4. 모델 기능 시스템

#### 4.1 모델 정보 스키마

**VoidStaticModelInfo 타입**:
```typescript
// src/vs/workbench/contrib/void/common/modelCapabilities.ts

export type VoidStaticModelInfo = {
  // 토큰 관리
  contextWindow: number;           // 입력 토큰 수
  reservedOutputTokenSpace: number | null;  // 출력 예약 공간

  // 시스템 메시지 지원
  supportsSystemMessage:
    | false                         // 미지원
    | 'system-role'                 // 'system' 역할 사용
    | 'developer-role'              // 'developer' 역할 사용
    | 'separated';                  // 별도 필드로 전달 (Anthropic)

  // 도구 호출 형식
  specialToolFormat?:
    | 'openai-style'                // OpenAI 표준
    | 'anthropic-style'             // Anthropic 표준
    | 'gemini-style';               // Gemini 표준

  // Fill-in-Middle 지원 (자동완성)
  supportsFIM: boolean;

  // 추가 페이로드 (OpenAI 호환 프로바이더용)
  additionalOpenAIPayload?: { [key: string]: string }

  // 추론(Reasoning) 기능
  reasoningCapabilities: false | {
    readonly supportsReasoning: true;
    readonly canTurnOffReasoning: boolean;  // 추론 비활성화 가능 여부
    readonly canIOReasoning: boolean;       // 추론 출력 가능 여부
    readonly reasoningReservedOutputTokenSpace?: number;
    readonly reasoningSlider?:
      | { type: 'budget_slider'; min: number; max: number; default: number }  // Anthropic
      | { type: 'effort_slider'; values: string[]; default: string }          // OpenAI
    readonly openSourceThinkTags?: [string, string];  // 오픈소스 모델용 <think> 태그
  };

  // 비용 정보 (정보 제공용)
  cost: {
    input: number;
    output: number;
    cache_read?: number;
    cache_write?: number;
  }

  // 다운로드 가능 여부 (로컬 모델)
  downloadable: false | {
    sizeGb: number | 'not-known'
  }
}
```

**모델 기능 조회**:
```typescript
export const getModelCapabilities = (
  providerName: ProviderName,
  modelName: string,
  overridesOfModel: OverridesOfModel | undefined
): VoidStaticModelInfo & (
  | { modelName: string; recognizedModelName: string; isUnrecognizedModel: false }
  | { modelName: string; recognizedModelName?: undefined; isUnrecognizedModel: true }
) => {
  const lowercaseModelName = modelName.toLowerCase()
  const { modelOptions, modelOptionsFallback } = modelSettingsOfProvider[providerName]

  // 사용자 정의 오버라이드 가져오기
  const overrides = overridesOfModel?.[providerName]?.[modelName];

  // 1. modelOptions 객체에서 직접 검색
  for (const modelName_ in modelOptions) {
    const lowercaseModelName_ = modelName_.toLowerCase()
    if (lowercaseModelName === lowercaseModelName_) {
      return {
        ...modelOptions[modelName],
        ...overrides,
        modelName,
        recognizedModelName: modelName,
        isUnrecognizedModel: false
      };
    }
  }

  // 2. 폴백 메커니즘 사용
  const result = modelOptionsFallback(modelName)
  if (result) {
    return {
      ...result,
      ...overrides,
      modelName: result.modelName,
      isUnrecognizedModel: false
    };
  }

  // 3. 기본값 반환
  return {
    modelName,
    ...defaultModelOptions,
    ...overrides,
    isUnrecognizedModel: true
  };
}
```

#### 4.2 프로바이더별 모델 설정 예시

**Anthropic 설정**:
```typescript
const anthropicModelOptions = {
  'claude-3-7-sonnet-20250219': {
    contextWindow: 200_000,
    reservedOutputTokenSpace: 8_192,
    cost: { input: 3.00, cache_read: 0.30, cache_write: 3.75, output: 15.00 },
    downloadable: false,
    supportsFIM: false,
    specialToolFormat: 'anthropic-style',
    supportsSystemMessage: 'separated',
    reasoningCapabilities: {
      supportsReasoning: true,
      canTurnOffReasoning: true,
      canIOReasoning: true,
      reasoningReservedOutputTokenSpace: 8192,
      reasoningSlider: {
        type: 'budget_slider',
        min: 1024,
        max: 8192,
        default: 1024
      },
    },
  },
  'claude-opus-4-20250514': {
    contextWindow: 200_000,
    reservedOutputTokenSpace: 8_192,
    cost: { input: 15.00, cache_read: 1.50, cache_write: 18.75, output: 30.00 },
    downloadable: false,
    supportsFIM: false,
    specialToolFormat: 'anthropic-style',
    supportsSystemMessage: 'separated',
    reasoningCapabilities: {
      supportsReasoning: true,
      canTurnOffReasoning: true,
      canIOReasoning: true,
      reasoningReservedOutputTokenSpace: 8192,
      reasoningSlider: { type: 'budget_slider', min: 1024, max: 8192, default: 1024 },
    },
  },
  // ... 기타 Claude 모델
}
```

**OpenAI 설정**:
```typescript
const openAIModelOptions = {
  'gpt-4.1': {
    contextWindow: 1_047_576,
    reservedOutputTokenSpace: 32_768,
    cost: { input: 2.00, output: 8.00, cache_read: 0.50 },
    downloadable: false,
    supportsFIM: false,
    specialToolFormat: 'openai-style',
    supportsSystemMessage: 'developer-role',
    reasoningCapabilities: false,
  },
  'o3': {
    contextWindow: 1_047_576,
    reservedOutputTokenSpace: 32_768,
    cost: { input: 10.00, output: 40.00, cache_read: 2.50 },
    downloadable: false,
    supportsFIM: false,
    specialToolFormat: 'openai-style',
    supportsSystemMessage: 'developer-role',
    reasoningCapabilities: {
      supportsReasoning: true,
      canTurnOffReasoning: false,  // o3는 항상 추론 모드
      canIOReasoning: false,        // 추론 출력은 안됨
      reasoningSlider: {
        type: 'effort_slider',
        values: ['low', 'medium', 'high'],
        default: 'low'
      }
    },
  },
  // ... 기타 GPT 모델
}
```

**Ollama (로컬 모델) 설정**:
```typescript
const ollamaModelOptions = {
  'qwen2.5-coder:7b': {
    contextWindow: 32_000,
    reservedOutputTokenSpace: null,
    cost: { input: 0, output: 0 },
    downloadable: { sizeGb: 1.9 },
    supportsFIM: true,  // Fill-in-Middle 지원
    supportsSystemMessage: 'system-role',
    reasoningCapabilities: false,
  },
  'deepseek-r1': {
    contextWindow: 128_000,
    reservedOutputTokenSpace: null,
    cost: { input: 0, output: 0 },
    downloadable: { sizeGb: 4.7 },
    supportsFIM: false,
    supportsSystemMessage: 'system-role',
    reasoningCapabilities: {
      supportsReasoning: true,
      canIOReasoning: false,  // 추론 출력 안됨 (로컬에서 수동 파싱 필요)
      canTurnOffReasoning: false,
      openSourceThinkTags: ['<think>', '</think>']  // 수동 파싱용 태그
    },
  },
  // ... 기타 Ollama 모델
}
```

#### 4.3 추론(Reasoning) 기능 처리

**추론 활성화 상태 결정**:
```typescript
export const getIsReasoningEnabledState = (
  featureName: FeatureName,
  providerName: ProviderName,
  modelName: string,
  modelSelectionOptions: ModelSelectionOptions | undefined,
  overridesOfModel: OverridesOfModel | undefined,
) => {
  const { supportsReasoning, canTurnOffReasoning } =
    getModelCapabilities(providerName, modelName, overridesOfModel).reasoningCapabilities || {}

  if (!supportsReasoning) return false

  // Chat 모드이거나 추론을 끌 수 없으면 기본적으로 활성화
  const defaultEnabledVal = featureName === 'Chat' || !canTurnOffReasoning

  const isReasoningEnabled = modelSelectionOptions?.reasoningEnabled ?? defaultEnabledVal
  return isReasoningEnabled
}
```

**추론 페이로드 생성**:
```typescript
export type SendableReasoningInfo = {
  type: 'budget_slider_value',
  isReasoningEnabled: true,
  reasoningBudget: number,
} | {
  type: 'effort_slider_value',
  isReasoningEnabled: true,
  reasoningEffort: string,
} | null

export const getSendableReasoningInfo = (
  featureName: FeatureName,
  providerName: ProviderName,
  modelName: string,
  modelSelectionOptions: ModelSelectionOptions | undefined,
  overridesOfModel: OverridesOfModel | undefined,
): SendableReasoningInfo => {
  const { reasoningSlider } =
    getModelCapabilities(providerName, modelName, overridesOfModel).reasoningCapabilities || {}

  const isReasoningEnabled = getIsReasoningEnabledState(
    featureName, providerName, modelName, modelSelectionOptions, overridesOfModel
  )
  if (!isReasoningEnabled) return null

  // Budget Slider (Anthropic)
  const reasoningBudget = reasoningSlider?.type === 'budget_slider'
    ? modelSelectionOptions?.reasoningBudget ?? reasoningSlider?.default
    : undefined

  if (reasoningBudget) {
    return {
      type: 'budget_slider_value',
      isReasoningEnabled: true,
      reasoningBudget
    }
  }

  // Effort Slider (OpenAI)
  const reasoningEffort = reasoningSlider?.type === 'effort_slider'
    ? modelSelectionOptions?.reasoningEffort ?? reasoningSlider?.default
    : undefined

  if (reasoningEffort) {
    return {
      type: 'effort_slider_value',
      isReasoningEnabled: true,
      reasoningEffort
    }
  }

  return null
}
```

**프로바이더별 추론 페이로드 처리**:
```typescript
// Anthropic
const anthropicSettings: VoidStaticProviderInfo = {
  providerReasoningIOSettings: {
    input: {
      includeInPayload: (reasoningInfo) => {
        if (!reasoningInfo?.isReasoningEnabled) return null

        if (reasoningInfo.type === 'budget_slider_value') {
          return {
            thinking: {
              type: 'enabled',
              budget_tokens: reasoningInfo.reasoningBudget
            }
          }
        }
        return null
      }
    },
  },
  // ...
}

// OpenAI
const openAICompatIncludeInPayloadReasoning = (reasoningInfo: SendableReasoningInfo) => {
  if (!reasoningInfo?.isReasoningEnabled) return null

  if (reasoningInfo.type === 'effort_slider_value') {
    return { reasoning_effort: reasoningInfo.reasoningEffort }
  }
  return null
}

// DeepSeek (OpenAI 호환 + reasoning_content 필드)
const deepseekSettings: VoidStaticProviderInfo = {
  providerReasoningIOSettings: {
    input: { includeInPayload: openAICompatIncludeInPayloadReasoning },
    output: { nameOfFieldInDelta: 'reasoning_content' },  // 추론 출력 필드
  },
  // ...
}

// Ollama (오픈소스 - 수동 파싱)
const ollamaSettings: VoidStaticProviderInfo = {
  providerReasoningIOSettings: {
    input: { includeInPayload: openAICompatIncludeInPayloadReasoning },
    output: { needsManualParse: true },  // <think>...</think> 수동 파싱
  },
  // ...
}
```

### 5. 도구 호출 시스템

#### 5.1 도구 정보 스키마

**내부 도구 정보 타입**:
```typescript
// src/vs/workbench/contrib/void/common/prompt/prompts.ts

export type InternalToolInfo = {
  name: string,
  description: string,
  params: {
    [paramName: string]: { description: string }
  },
  mcpServerName?: string,  // MCP 서버에서 제공되는 경우
}
```

**기본 제공 도구**:
```typescript
export const builtinTools: {
  [T in keyof BuiltinToolCallParams]: {
    name: string;
    description: string;
    params: Partial<{
      [paramName in keyof SnakeCaseKeys<BuiltinToolCallParams[T]>]: { description: string }
    }>
  }
} = {
  // --- 컨텍스트 수집 (읽기/검색/목록) ---
  read_file: {
    name: 'read_file',
    description: `Returns full contents of a given file.`,
    params: {
      uri: { description: `The FULL path to the file.` },
      start_line: {
        description: 'Optional. Do NOT fill this field in unless you were specifically given exact line numbers to search. Defaults to the beginning of the file.'
      },
      end_line: {
        description: 'Optional. Do NOT fill this field in unless you were specifically given exact line numbers to search. Defaults to the end of the file.'
      },
      page_number: {
        description: 'Optional. The page number of the result. Default is 1.'
      },
    },
  },

  ls_dir: {
    name: 'ls_dir',
    description: `Lists all files and folders in the given URI.`,
    params: {
      uri: {
        description: `Optional. The FULL path to the folder. Leave this as empty or "" to search all folders.`
      },
      page_number: { description: 'Optional. The page number of the result. Default is 1.' },
    },
  },

  get_dir_tree: {
    name: 'get_dir_tree',
    description: `This is a very effective way to learn about the user's codebase. Returns a tree diagram of all the files and folders in the given folder.`,
    params: {
      uri: { description: `The FULL path to the folder.` }
    }
  },

  search_pathnames_only: {
    name: 'search_pathnames_only',
    description: `Returns all pathnames that match a given query (searches ONLY file names).`,
    params: {
      query: { description: `Your query for the search.` },
      include_pattern: {
        description: 'Optional. Only fill this in if you need to limit your search because there were too many results.'
      },
      page_number: { description: 'Optional. The page number of the result. Default is 1.' },
    },
  },

  search_for_files: {
    name: 'search_for_files',
    description: `Returns a list of file names whose content matches the given query. The query can be any substring or regex.`,
    params: {
      query: { description: `Your query for the search.` },
      search_in_folder: {
        description: 'Optional. Leave as blank by default. ONLY fill this in if your previous search with the same query was truncated. Searches descendants of this folder only.'
      },
      is_regex: { description: 'Optional. Default is false. Whether the query is a regex.' },
      page_number: { description: 'Optional. The page number of the result. Default is 1.' },
    },
  },

  search_in_file: {
    name: 'search_in_file',
    description: `Returns an array of all the start line numbers where the content appears in the file.`,
    params: {
      uri: { description: `The FULL path to the file.` },
      query: { description: 'The string or regex to search for in the file.' },
      is_regex: { description: 'Optional. Default is false. Whether the query is a regex.' }
    }
  },

  read_lint_errors: {
    name: 'read_lint_errors',
    description: `Use this tool to view all the lint errors on a file.`,
    params: {
      uri: { description: `The FULL path to the file.` },
    },
  },

  // --- 파일 편집 (생성/삭제/수정) ---
  create_file_or_folder: {
    name: 'create_file_or_folder',
    description: `Create a file or folder at the given path. To create a folder, the path MUST end with a trailing slash.`,
    params: {
      uri: { description: `The FULL path to the file or folder.` },
    },
  },

  delete_file_or_folder: {
    name: 'delete_file_or_folder',
    description: `Delete a file or folder at the given path.`,
    params: {
      uri: { description: `The FULL path to the file or folder.` },
      is_recursive: { description: 'Optional. Return true to delete recursively.' }
    },
  },

  edit_file: {
    name: 'edit_file',
    description: `Edit the contents of a file. You must provide the file's URI as well as a SINGLE string of SEARCH/REPLACE block(s).`,
    params: {
      uri: { description: `The FULL path to the file.` },
      search_replace_blocks: {
        description: replaceTool_description  // SEARCH/REPLACE 블록 형식 설명
      }
    },
  },

  rewrite_file: {
    name: 'rewrite_file',
    description: `Edits a file, deleting all the old contents and replacing them with your new contents. Use this tool if you want to edit a file you just created.`,
    // ... params
  },

  // ... 기타 도구들
}
```

#### 5.2 프로바이더별 도구 형식 변환

**OpenAI 형식 변환**:
```typescript
const toOpenAICompatibleTool = (toolInfo: InternalToolInfo) => {
  const { name, description, params } = toolInfo

  const paramsWithType: {
    [s: string]: { description: string; type: 'string' }
  } = {}
  for (const key in params) {
    paramsWithType[key] = { ...params[key], type: 'string' }
  }

  return {
    type: 'function',
    function: {
      name: name,
      description: description,
      parameters: {
        type: 'object',
        properties: params,
      },
    }
  } satisfies OpenAI.Chat.Completions.ChatCompletionTool
}

const openAITools = (chatMode: ChatMode | null, mcpTools: InternalToolInfo[] | undefined) => {
  const allowedTools = availableTools(chatMode, mcpTools)
  if (!allowedTools || Object.keys(allowedTools).length === 0) return null

  const openAITools: OpenAI.Chat.Completions.ChatCompletionTool[] = []
  for (const t in allowedTools ?? {}) {
    openAITools.push(toOpenAICompatibleTool(allowedTools[t]))
  }
  return openAITools
}
```

**Anthropic 형식 변환**:
```typescript
const toAnthropicTool = (toolInfo: InternalToolInfo) => {
  const { name, description, params } = toolInfo
  const paramsWithType: {
    [s: string]: { description: string; type: 'string' }
  } = {}
  for (const key in params) {
    paramsWithType[key] = { ...params[key], type: 'string' }
  }

  return {
    name: name,
    description: description,
    input_schema: {
      type: 'object',
      properties: paramsWithType,
    },
  } satisfies Anthropic.Messages.Tool
}

const anthropicTools = (chatMode: ChatMode | null, mcpTools: InternalToolInfo[] | undefined) => {
  const allowedTools = availableTools(chatMode, mcpTools)
  if (!allowedTools || Object.keys(allowedTools).length === 0) return null

  const anthropicTools: Anthropic.Messages.ToolUnion[] = []
  for (const t in allowedTools ?? {}) {
    anthropicTools.push(toAnthropicTool(allowedTools[t]))
  }
  return anthropicTools
}
```

**Gemini 형식 변환**:
```typescript
const toGeminiFunctionDecl = (toolInfo: InternalToolInfo) => {
  const { name, description, params } = toolInfo
  return {
    name,
    description,
    parameters: {
      type: Type.OBJECT,
      properties: Object.entries(params).reduce((acc, [key, value]) => {
        acc[key] = {
          type: Type.STRING,
          description: value.description
        };
        return acc;
      }, {} as Record<string, Schema>)
    }
  } satisfies FunctionDeclaration
}

const geminiTools = (chatMode: ChatMode | null, mcpTools: InternalToolInfo[] | undefined): GeminiTool[] | null => {
  const allowedTools = availableTools(chatMode, mcpTools)
  if (!allowedTools || Object.keys(allowedTools).length === 0) return null

  const functionDecls: FunctionDeclaration[] = []
  for (const t in allowedTools ?? {}) {
    functionDecls.push(toGeminiFunctionDecl(allowedTools[t]))
  }
  const tools: GeminiTool = { functionDeclarations: functionDecls }
  return [tools]
}
```

#### 5.3 SEARCH/REPLACE 블록 시스템

**SEARCH/REPLACE 블록 형식**:
```typescript
export const ORIGINAL = `<<<<<<< ORIGINAL`
export const DIVIDER = `=======`
export const FINAL = `>>>>>>> UPDATED`

const searchReplaceBlockTemplate = `\
${ORIGINAL}
// ... original code goes here
${DIVIDER}
// ... final code goes here
${FINAL}

${ORIGINAL}
// ... original code goes here
${DIVIDER}
// ... final code goes here
${FINAL}`
```

**edit_file 도구 설명**:
```typescript
const replaceTool_description = `\
A string of SEARCH/REPLACE block(s) which will be applied to the given file.
Your SEARCH/REPLACE blocks string must be formatted as follows:
${searchReplaceBlockTemplate}

## Guidelines:

1. You may output multiple search replace blocks if needed.

2. The ORIGINAL code in each SEARCH/REPLACE block must EXACTLY match lines in the original file. Do not add or remove any whitespace or comments from the original code.

3. Each ORIGINAL text must be large enough to uniquely identify the change. However, bias towards writing as little as possible.

4. Each ORIGINAL text must be DISJOINT from all other ORIGINAL text.

5. This field is a STRING (not an array).`
```

**diff를 SEARCH/REPLACE 블록으로 변환하는 시스템 메시지**:
```typescript
const createSearchReplaceBlocks_systemMessage = `\
You are a coding assistant that takes in a diff, and outputs SEARCH/REPLACE code blocks to implement the change(s) in the diff.
The diff will be labeled \`DIFF\` and the original file will be labeled \`ORIGINAL_FILE\`.

Format your SEARCH/REPLACE blocks as follows:
\`\`\`
${searchReplaceBlockTemplate}
\`\`\`

1. Your SEARCH/REPLACE block(s) must implement the diff EXACTLY. Do NOT leave anything out.

2. You are allowed to output multiple SEARCH/REPLACE blocks to implement the change.

3. Assume any comments in the diff are PART OF THE CHANGE. Include them in the output.

4. Your output should consist ONLY of SEARCH/REPLACE blocks. Do NOT output any text or explanations before or after this.

5. The ORIGINAL code in each SEARCH/REPLACE block must EXACTLY match lines in the original file. Do not add or remove any whitespace, comments, or modifications from the original code.

6. Each ORIGINAL text must be large enough to uniquely identify the change in the file. However, bias towards writing as little as possible.

7. Each ORIGINAL text must be DISJOINT from all other ORIGINAL text.

## EXAMPLE 1
DIFF
\`\`\`
// ... existing code
let x = 6.5
// ... existing code
\`\`\`

ORIGINAL_FILE
\`\`\`
let w = 5
let x = 6
let y = 7
let z = 8
\`\`\`

ACCEPTED OUTPUT
\`\`\`
${ORIGINAL}
let x = 6
${DIVIDER}
let x = 6.5
${FINAL}
\`\`\``
```

### 6. 컨텍스트 수집 시스템

#### 6.1 컨텍스트 수집 서비스

**IContextGatheringService 인터페이스**:
```typescript
// src/vs/workbench/contrib/void/browser/contextGatheringService.ts

export interface IContextGatheringService {
  readonly _serviceBrand: undefined;
  updateCache(model: ITextModel, pos: Position): Promise<void>;
  getCachedSnippets(): string[];
}
```

**컨텍스트 수집 전략**:
```typescript
class ContextGatheringService extends Disposable implements IContextGatheringService {
  private readonly _NUM_LINES = 3;
  private readonly _MAX_SNIPPET_LINES = 7;
  private _cache: string[] = [];
  private _snippetIntervals: IVisitedInterval[] = [];

  public async updateCache(model: ITextModel, pos: Position): Promise<void> {
    const snippets = new Set<string>();
    this._snippetIntervals = [];

    // 1. 주변 코드 수집 (깊이 3)
    await this._gatherNearbySnippets(model, pos, this._NUM_LINES, 3, snippets, this._snippetIntervals);

    // 2. 부모 함수/클래스 수집 (깊이 3)
    await this._gatherParentSnippets(model, pos, this._NUM_LINES, 3, snippets, this._snippetIntervals);

    // 중복 제거 및 캐시 업데이트
    this._cache = Array.from(snippets);
  }

  public getCachedSnippets(): string[] {
    return this._cache;
  }

  // 주변 코드 스니펫 수집 (재귀적 정의 탐색)
  private async _gatherNearbySnippets(
    model: ITextModel,
    pos: Position,
    numLines: number,
    depth: number,
    snippets: Set<string>,
    visited: IVisitedInterval[]
  ): Promise<void> {
    if (depth <= 0) return;

    // 현재 위치 주변 코드 추가
    const startLine = Math.max(pos.lineNumber - numLines, 1);
    const endLine = Math.min(pos.lineNumber + numLines, model.getLineCount());
    const range = new Range(startLine, 1, endLine, model.getLineMaxColumn(endLine));
    this._addSnippetIfNotOverlapping(model, range, snippets, visited);

    // 심볼 찾기 및 정의 탐색
    const symbols = await this._getSymbolsNearPosition(model, pos, numLines);
    for (const sym of symbols) {
      const defs = await this._getDefinitionSymbols(model, sym);
      for (const def of defs) {
        const defModel = this._modelService.getModel(def.uri);
        if (defModel) {
          const defPos = new Position(def.range.startLineNumber, def.range.startColumn);
          this._addSnippetIfNotOverlapping(defModel, def.range, snippets, visited);
          await this._gatherNearbySnippets(defModel, defPos, numLines, depth - 1, snippets, visited);
        }
      }
    }
  }

  // 부모 함수/클래스 수집 (재귀적 상위 컨테이너 탐색)
  private async _gatherParentSnippets(
    model: ITextModel,
    pos: Position,
    numLines: number,
    depth: number,
    snippets: Set<string>,
    visited: IVisitedInterval[]
  ): Promise<void> {
    if (depth <= 0) return;

    // 포함하는 함수/클래스 찾기
    const container = await this._findContainerFunction(model, pos);
    if (!container) return;

    // 컨테이너 코드 추가
    const containerRange = container.kind === SymbolKind.Method
      ? container.selectionRange
      : container.range;
    this._addSnippetIfNotOverlapping(model, containerRange, snippets, visited);

    // 컨테이너 내 심볼 탐색
    const symbols = await this._getSymbolsNearRange(model, containerRange, numLines);
    for (const sym of symbols) {
      const defs = await this._getDefinitionSymbols(model, sym);
      for (const def of defs) {
        const defModel = this._modelService.getModel(def.uri);
        if (defModel) {
          const defPos = new Position(def.range.startLineNumber, def.range.startColumn);
          this._addSnippetIfNotOverlapping(defModel, def.range, snippets, visited);
          await this._gatherNearbySnippets(defModel, defPos, numLines, depth - 1, snippets, visited);
        }
      }
    }

    // 상위 컨테이너 탐색 (재귀)
    const containerPos = new Position(containerRange.startLineNumber, containerRange.startColumn);
    await this._gatherParentSnippets(model, containerPos, numLines, depth - 1, snippets, visited);
  }

  // 중복 방지
  private _addSnippetIfNotOverlapping(
    model: ITextModel,
    range: IRange,
    snippets: Set<string>,
    visited: IVisitedInterval[]
  ): void {
    const startLine = range.startLineNumber;
    const endLine = range.endLineNumber;
    const uri = model.uri.toString();

    if (!this._isRangeVisited(uri, startLine, endLine, visited)) {
      visited.push({ uri, startLine, endLine });
      const snippet = this._normalizeSnippet(
        this._getSnippetForRange(model, range, this._NUM_LINES)
      );
      if (snippet.length > 0) {
        snippets.add(snippet);
      }
    }
  }
}
```

#### 6.2 채팅 변수 시스템

**채팅 변수 타입**:
```typescript
// src/vs/workbench/contrib/chat/common/chatVariables.ts

export interface IChatVariableData {
  id: string;
  name: string;
  icon?: ThemeIcon;
  fullName?: string;
  description: string;
  modelDescription?: string;  // LLM에게 전달되는 설명
  canTakeArgument?: boolean;
}

export type IChatRequestVariableValue =
  | string
  | URI
  | Location
  | unknown
  | Uint8Array
  | IChatRequestProblemsVariable;

export interface IChatVariableResolver {
  (
    messageText: string,
    arg: string | undefined,
    model: IChatModel,
    progress: (part: IChatVariableResolverProgress) => void,
    token: CancellationToken
  ): Promise<IChatRequestVariableValue | undefined>;
}
```

**동적 변수**:
```typescript
export interface IDynamicVariable {
  range: IRange;
  id: string;
  fullName?: string;
  icon?: ThemeIcon;
  prefix?: string;
  modelDescription?: string;
  isFile?: boolean;
  isDirectory?: boolean;
  data: IChatRequestVariableValue;
}
```

**변수 서비스**:
```typescript
export interface IChatVariablesService {
  _serviceBrand: undefined;

  // 동적 변수 가져오기
  getDynamicVariables(sessionId: string): ReadonlyArray<IDynamicVariable>;

  // 컨텍스트 첨부
  attachContext(
    name: string,
    value: string | URI | Location | unknown,
    location: ChatAgentLocation
  ): void;

  // 변수 해석
  resolveVariables(
    prompt: IParsedChatRequest,
    attachedContextVariables: IChatRequestVariableEntry[] | undefined
  ): IChatRequestVariableData;
}
```

### 7. 프롬프트 엔지니어링 패턴

#### 7.1 프롬프트 상수

**컨텍스트 제한**:
```typescript
// src/vs/workbench/contrib/void/common/prompt/prompts.ts

// 디렉토리 구조 정보 제한
export const MAX_DIRSTR_CHARS_TOTAL_BEGINNING = 20_000
export const MAX_DIRSTR_CHARS_TOTAL_TOOL = 20_000
export const MAX_DIRSTR_RESULTS_TOTAL_BEGINNING = 100
export const MAX_DIRSTR_RESULTS_TOTAL_TOOL = 100

// 파일 내용 제한
export const MAX_FILE_CHARS_PAGE = 500_000
export const MAX_CHILDREN_URIs_PAGE = 500

// 터미널 출력 제한
export const MAX_TERMINAL_CHARS = 100_000
export const MAX_TERMINAL_INACTIVE_TIME = 8  // seconds
export const MAX_TERMINAL_BG_COMMAND_TIME = 5

// Prefix/Suffix 컨텍스트 제한
export const MAX_PREFIX_SUFFIX_CHARS = 20_000
```

**코드 블록 래퍼**:
```typescript
// 트리플 백틱 (코드 블록)
export const tripleTick = ['```', '```']
```

#### 7.2 프롬프트 템플릿 예시

**채팅 제안 diff 예시**:
```typescript
const chatSuggestionDiffExample = `\
${tripleTick[0]}typescript
/Users/username/Desktop/my_project/app.ts
// ... existing code ...
// {{change 1}}
// ... existing code ...
// {{change 2}}
// ... existing code ...
// {{change 3}}
// ... existing code ...
${tripleTick[1]}`
```

**터미널 도구 설명 헬퍼**:
```typescript
const terminalDescHelper = `You can use this tool to run any command: sed, grep, etc. Do not edit any files with this tool; use edit_file instead. When working with git and other tools that open an editor (e.g. git diff), you should pipe to cat to get all results and not get stuck in vim.`
```

**파라미터 설명 헬퍼**:
```typescript
const uriParam = (object: string) => ({
  uri: { description: `The FULL path to the ${object}.` }
})

const paginationParam = {
  page_number: { description: 'Optional. The page number of the result. Default is 1.' }
} as const

const cwdHelper = 'Optional. The directory in which to run the command. Defaults to the first workspace folder.'
```

### 8. 에러 처리 패턴

#### 8.1 API 키 오류 처리

```typescript
const invalidApiKeyMessage = (providerName: ProviderName) =>
  `Invalid ${displayInfoOfProviderName(providerName).title} API key.`

// OpenAI 예시
openai.chat.completions.create(options)
  .then(async response => {
    // ... 처리
  })
  .catch(error => {
    if (error instanceof OpenAI.APIError && error.status === 401) {
      onError({ message: invalidApiKeyMessage(providerName), fullError: error });
    } else {
      onError({ message: error + '', fullError: error });
    }
  })

// Anthropic 예시
stream.on('error', (error) => {
  if (error instanceof Anthropic.APIError && error.status === 401) {
    onError({ message: invalidApiKeyMessage(providerName), fullError: error })
  } else {
    onError({ message: error + '', fullError: error })
  }
})
```

#### 8.2 빈 응답 처리

```typescript
// 완료 시 빈 응답 검증
if (!fullTextSoFar && !fullReasoningSoFar && !toolName) {
  onError({ message: 'Void: Response from model was empty.', fullError: null })
} else {
  const toolCall = rawToolCallObjOfParamsStr(toolName, toolParamsStr, toolId)
  const toolCallObj = toolCall ? { toolCall } : {}
  onFinalMessage({
    fullText: fullTextSoFar,
    fullReasoning: fullReasoningSoFar,
    anthropicReasoning: null,
    ...toolCallObj
  });
}
```

#### 8.3 에러 상세 정보 추출

```typescript
export const errorDetails = (fullError: Error | null): string | null => {
  if (fullError === null) {
    return null
  }
  else if (typeof fullError === 'object') {
    if (Object.keys(fullError).length === 0) return null
    return JSON.stringify(fullError, null, 2)
  }
  else if (typeof fullError === 'string') {
    return null
  }
  return null
}

export const getErrorMessage: (error: unknown) => string = (error) => {
  if (error instanceof Error) return `${error.name}: ${error.message}`
  return error + ''
}
```

## 코드 분석/생성 에이전트 개발 가이드

### 1. 멀티 프로바이더 지원 구현

**단계별 프로바이더 통합**:

1. **프로바이더별 SDK 인스턴스화 로직 작성**
   - 각 프로바이더의 인증 방식 처리
   - baseURL 및 헤더 커스터마이징
   - 공통 페이로드 옵션 관리

2. **메시지 형식 변환 레이어 구현**
   - LLMChatMessage 타입을 각 프로바이더 형식으로 변환
   - 시스템 메시지 처리 방식 차이 대응 (role vs separated)
   - 도구 호출 형식 차이 처리 (openai-style vs anthropic-style vs gemini-style)

3. **스트리밍 응답 통합**
   - AsyncIterable 패턴 (OpenAI 계열)
   - Event 기반 패턴 (Anthropic)
   - 공통 콜백 인터페이스로 추상화

4. **모델 기능 메타데이터 관리**
   - VoidStaticModelInfo 스키마 활용
   - 폴백 메커니즘 구현
   - 사용자 정의 오버라이드 지원

### 2. 도구 호출 시스템 구현

**도구 정의 Best Practices**:

1. **명확한 설명 작성**
   - 도구의 목적을 한 문장으로 요약
   - 파라미터 설명에 예시 포함
   - 선택적 파라미터에는 "Optional." 접두사 명시

2. **파라미터 검증**
   - 필수 파라미터 vs 선택적 파라미터 명확히 구분
   - URI 경로는 "FULL path" 강조
   - 페이지네이션 파라미터 일관성 유지

3. **SEARCH/REPLACE 블록 처리**
   - ORIGINAL 코드는 파일 내용과 정확히 일치해야 함
   - 각 ORIGINAL 블록은 고유하게 식별 가능해야 함
   - 여러 블록을 단일 문자열로 전달

### 3. 컨텍스트 수집 전략

**효과적인 컨텍스트 수집**:

1. **계층적 수집**
   - 현재 위치 주변 코드 (±3줄)
   - 심볼 정의 재귀적 탐색 (깊이 3)
   - 부모 함수/클래스 재귀적 탐색 (깊이 3)

2. **중복 방지**
   - IVisitedInterval로 방문한 범위 추적
   - URI + startLine + endLine 조합으로 고유성 검증
   - Set을 사용한 스니펫 중복 제거

3. **스니펫 정리**
   - 빈 줄 및 주석만 있는 줄 제거
   - 여러 줄바꿈을 하나로 정규화
   - 최대 7줄로 스니펫 크기 제한

### 4. 프롬프트 최적화

**토큰 사용량 최적화**:

1. **컨텍스트 제한 설정**
   - 디렉토리 구조: 20,000자
   - 파일 내용: 500,000자/페이지
   - 터미널 출력: 100,000자

2. **페이지네이션 구현**
   - 결과가 많은 경우 page_number 파라미터 활용
   - 첫 페이지로 범위 파악 후 필요시 후속 페이지 요청
   - MAX_CHILDREN_URIs_PAGE (500) 제한

3. **점진적 정보 제공**
   - 초기에는 요약 정보 제공 (파일 이름, 디렉토리 트리)
   - 상세 정보는 필요시에만 요청 (read_file, search_in_file)

### 5. 추론(Reasoning) 기능 활용

**추론 모델 최적화**:

1. **Budget Slider (Anthropic)**
   - 1024 ~ 8192 토큰 범위
   - 간단한 작업: 1024 (기본값)
   - 복잡한 작업: 4096 ~ 8192

2. **Effort Slider (OpenAI)**
   - 'low', 'medium', 'high'
   - 대부분 'low' (기본값)으로 충분
   - 매우 복잡한 디버깅: 'high'

3. **오픈소스 모델**
   - `<think>...</think>` 태그 수동 파싱
   - needsManualParse: true 설정
   - extractReasoningWrapper 활용

### 6. IPC 통신 최적화

**효율적인 이벤트 관리**:

1. **리스너 재사용**
   - 서비스 초기화 시 한 번만 리스너 설정
   - requestId별 훅으로 라우팅
   - 완료/에러 시 훅 정리 (_clearChannelHooks)

2. **Abort 처리**
   - onAbort 훅은 로컬에서 즉시 호출
   - IPC를 통한 abort 메시지 전송
   - 스트림 컨트롤러 abort 메서드 호출

3. **에러 전파**
   - fullError 객체 전체 전달 (직렬화 가능한 경우)
   - 에러 메시지 문자열로 변환
   - 상태 코드별 분기 처리 (401 = API 키 오류)

## 결론

Void의 LLM 통합 패턴은 멀티 프로바이더 지원, 스트리밍 처리, 도구 호출, 컨텍스트 수집의 4가지 핵심 축으로 구성된다. 이 아키텍처는 다음과 같은 장점을 제공한다:

1. **확장성**: 새로운 프로바이더 추가 시 공통 인터페이스만 구현하면 됨
2. **유연성**: 프로바이더별 특수 기능(추론, FIM)을 메타데이터로 관리
3. **성능**: AsyncIterable 스트리밍으로 응답 지연 최소화
4. **안정성**: IPC 채널 분리로 메인 프로세스 안정성 확보
5. **사용자 경험**: 도구 호출 및 컨텍스트 수집으로 정확도 향상

코드 분석/생성 에이전트 개발 시 본 문서의 패턴을 참고하면, 견고하고 확장 가능한 멀티 LLM 시스템을 구축할 수 있다.

## 참고 자료

**핵심 파일**:
- `/src/vs/workbench/contrib/void/common/sendLLMMessageService.ts` - 서비스 레이어
- `/src/vs/workbench/contrib/void/electron-main/llmMessage/sendLLMMessage.impl.ts` - 프로바이더 구현
- `/src/vs/workbench/contrib/void/common/modelCapabilities.ts` - 모델 기능 메타데이터
- `/src/vs/workbench/contrib/void/common/sendLLMMessageTypes.ts` - 타입 정의
- `/src/vs/workbench/contrib/void/common/prompt/prompts.ts` - 프롬프트 및 도구 정의
- `/src/vs/workbench/contrib/void/browser/contextGatheringService.ts` - 컨텍스트 수집

**관련 문서**:
- `/분석/01-아키텍처-개요.md` - 전체 아키텍처 이해
- `/분석/02-핵심-서비스-분석.md` - 서비스 레이어 상세 분석
