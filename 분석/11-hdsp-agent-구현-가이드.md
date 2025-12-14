# hdsp-agent 구현 가이드

> **문서 목적**: Void 분석 결과를 바탕으로 한 hdsp-agent 코드 분석/생성 에이전트 구현 권장사항

---

## 1. 개요

이 문서는 Void 프로젝트에서 추출한 패턴을 hdsp-agent에 적용하기 위한 실용적인 구현 가이드입니다.

### 1.1 hdsp-agent 목표

| 기능 | 설명 | Void 참조 |
|------|------|-----------|
| 코드 분석 | 코드베이스 이해, 심볼 추적, 의존성 분석 | `contextGatheringService.ts` |
| 코드 생성 | LLM 기반 코드 생성 및 변환 | `editCodeService.ts` |
| 도구 실행 | 파일 읽기/쓰기, 터미널 명령 실행 | `toolsService.ts` |
| 대화 관리 | 멀티턴 대화, 컨텍스트 유지 | `chatThreadService.ts` |

---

## 2. 권장 아키텍처

### 2.1 계층 구조

```
┌─────────────────────────────────────────────────────────────┐
│                      Agent Layer                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ AgentLoop   │ │ TaskPlanner │ │ ResponseGenerator   │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                      Service Layer                           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ LLMService  │ │ ToolService │ │ ContextService      │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ FileService │ │ EditService │ │ ConversationService │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                      Core Layer                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ DI Container│ │ EventSystem │ │ DisposableManager   │   │
│  └─────────────┘ └─────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 서비스 매핑

| hdsp-agent 서비스 | Void 대응 서비스 | 역할 |
|-------------------|------------------|------|
| `IAgentService` | `IChatThreadService` | 에이전트 루프 관리 |
| `ILLMService` | `ILLMMessageService` | LLM 통신 추상화 |
| `IToolService` | `IToolsService` | 도구 등록/실행 |
| `ICodeService` | `IEditCodeService` | 코드 변환/적용 |
| `IContextService` | `IContextGatheringService` | 컨텍스트 수집 |
| `IFileService` | `IFileService` (VSCode) | 파일 시스템 접근 |
| `IConfigService` | `ISettingsService` | 설정 관리 |

---

## 3. 핵심 서비스 구현

### 3.1 의존성 주입 시스템

```typescript
// 서비스 식별자 정의
const ILLMService = createServiceIdentifier<ILLMService>('ILLMService');
const IToolService = createServiceIdentifier<IToolService>('IToolService');
const ICodeService = createServiceIdentifier<ICodeService>('ICodeService');

// 서비스 식별자 생성 함수
function createServiceIdentifier<T>(id: string): ServiceIdentifier<T> {
  const identifier = Symbol(id) as ServiceIdentifier<T>;
  (identifier as any).toString = () => id;
  return identifier;
}

// 서비스 컨테이너
class ServiceContainer {
  private services = new Map<symbol, any>();
  private factories = new Map<symbol, () => any>();

  register<T>(id: ServiceIdentifier<T>, instance: T): void {
    this.services.set(id as symbol, instance);
  }

  registerFactory<T>(id: ServiceIdentifier<T>, factory: () => T): void {
    this.factories.set(id as symbol, factory);
  }

  get<T>(id: ServiceIdentifier<T>): T {
    if (this.services.has(id as symbol)) {
      return this.services.get(id as symbol);
    }

    const factory = this.factories.get(id as symbol);
    if (factory) {
      const instance = factory();
      this.services.set(id as symbol, instance);
      return instance;
    }

    throw new Error(`Service not found: ${String(id)}`);
  }
}
```

### 3.2 이벤트 시스템

```typescript
interface IDisposable {
  dispose(): void;
}

interface Event<T> {
  (listener: (e: T) => void): IDisposable;
}

class Emitter<T> implements IDisposable {
  private listeners: Set<(e: T) => void> = new Set();
  private disposed = false;

  get event(): Event<T> {
    return (listener: (e: T) => void) => {
      if (this.disposed) {
        return { dispose: () => {} };
      }

      this.listeners.add(listener);

      return {
        dispose: () => {
          this.listeners.delete(listener);
        }
      };
    };
  }

  fire(event: T): void {
    if (this.disposed) return;

    for (const listener of this.listeners) {
      try {
        listener(event);
      } catch (error) {
        console.error('Event listener error:', error);
      }
    }
  }

  dispose(): void {
    this.disposed = true;
    this.listeners.clear();
  }
}
```

### 3.3 Disposable 관리

```typescript
class DisposableStore implements IDisposable {
  private disposables: Set<IDisposable> = new Set();
  private disposed = false;

  add<T extends IDisposable>(disposable: T): T {
    if (this.disposed) {
      disposable.dispose();
      return disposable;
    }

    this.disposables.add(disposable);
    return disposable;
  }

  delete(disposable: IDisposable): void {
    this.disposables.delete(disposable);
  }

  clear(): void {
    for (const disposable of this.disposables) {
      try {
        disposable.dispose();
      } catch (error) {
        console.error('Dispose error:', error);
      }
    }
    this.disposables.clear();
  }

  dispose(): void {
    if (this.disposed) return;
    this.disposed = true;
    this.clear();
  }
}

// 사용 예시
abstract class BaseService implements IDisposable {
  protected readonly _disposables = new DisposableStore();

  dispose(): void {
    this._disposables.dispose();
  }
}
```

---

## 4. LLM 통합 서비스

### 4.1 프로바이더 추상화

```typescript
interface ILLMProvider {
  readonly id: string;
  readonly displayName: string;

  sendMessage(request: LLMRequest): AsyncIterable<LLMFragment>;
  validateConfig(): Promise<boolean>;
  getAvailableModels(): Promise<ModelInfo[]>;
}

interface LLMRequest {
  messages: ChatMessage[];
  model: string;
  tools?: ToolDefinition[];
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
}

interface LLMFragment {
  type: 'text' | 'tool_use' | 'done' | 'error';
  content?: string;
  toolCall?: ToolCall;
  error?: Error;
}

// 멀티 프로바이더 서비스
class LLMService extends BaseService implements ILLMService {
  private providers = new Map<string, ILLMProvider>();

  registerProvider(provider: ILLMProvider): IDisposable {
    this.providers.set(provider.id, provider);

    return {
      dispose: () => {
        this.providers.delete(provider.id);
      }
    };
  }

  async *sendMessage(
    providerId: string,
    request: LLMRequest
  ): AsyncIterable<LLMFragment> {
    const provider = this.providers.get(providerId);
    if (!provider) {
      throw new Error(`Provider not found: ${providerId}`);
    }

    yield* provider.sendMessage(request);
  }
}
```

### 4.2 스트리밍 응답 처리

```typescript
interface IStreamingHandler {
  onText(text: string): void;
  onToolCall(toolCall: ToolCall): Promise<ToolResult>;
  onComplete(): void;
  onError(error: Error): void;
}

async function processLLMStream(
  stream: AsyncIterable<LLMFragment>,
  handler: IStreamingHandler
): Promise<void> {
  const pendingToolCalls: ToolCall[] = [];

  for await (const fragment of stream) {
    switch (fragment.type) {
      case 'text':
        handler.onText(fragment.content!);
        break;

      case 'tool_use':
        pendingToolCalls.push(fragment.toolCall!);
        break;

      case 'done':
        // 대기 중인 도구 호출 처리
        for (const toolCall of pendingToolCalls) {
          await handler.onToolCall(toolCall);
        }
        handler.onComplete();
        break;

      case 'error':
        handler.onError(fragment.error!);
        break;
    }
  }
}
```

---

## 5. 도구 시스템

### 5.1 도구 정의

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  parameters: JSONSchema;
  requiresApproval?: boolean;
  category?: 'file' | 'terminal' | 'web' | 'analysis';
}

interface ToolResult {
  success: boolean;
  output?: string;
  error?: string;
  metadata?: Record<string, any>;
}

interface ITool {
  readonly definition: ToolDefinition;
  execute(params: Record<string, any>): Promise<ToolResult>;
  validate?(params: Record<string, any>): ValidationResult;
}
```

### 5.2 핵심 도구 구현

```typescript
// 파일 읽기 도구
class ReadFileTool implements ITool {
  readonly definition: ToolDefinition = {
    name: 'read_file',
    description: 'Read the contents of a file',
    parameters: {
      type: 'object',
      properties: {
        path: { type: 'string', description: 'File path to read' },
        startLine: { type: 'number', description: 'Start line (optional)' },
        endLine: { type: 'number', description: 'End line (optional)' }
      },
      required: ['path']
    },
    category: 'file'
  };

  constructor(private fileService: IFileService) {}

  async execute(params: { path: string; startLine?: number; endLine?: number }): Promise<ToolResult> {
    try {
      const content = await this.fileService.readFile(params.path);
      const lines = content.split('\n');

      const start = params.startLine ?? 0;
      const end = params.endLine ?? lines.length;
      const selectedLines = lines.slice(start, end);

      return {
        success: true,
        output: selectedLines.join('\n'),
        metadata: {
          totalLines: lines.length,
          selectedRange: [start, end]
        }
      };
    } catch (error) {
      return {
        success: false,
        error: `Failed to read file: ${error}`
      };
    }
  }
}

// 파일 편집 도구
class EditFileTool implements ITool {
  readonly definition: ToolDefinition = {
    name: 'edit_file',
    description: 'Edit a file using search/replace blocks',
    parameters: {
      type: 'object',
      properties: {
        path: { type: 'string', description: 'File path to edit' },
        searchReplaceBlocks: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              search: { type: 'string' },
              replace: { type: 'string' }
            },
            required: ['search', 'replace']
          }
        }
      },
      required: ['path', 'searchReplaceBlocks']
    },
    requiresApproval: true,
    category: 'file'
  };

  constructor(
    private fileService: IFileService,
    private diffService: IDiffService
  ) {}

  async execute(params: {
    path: string;
    searchReplaceBlocks: Array<{ search: string; replace: string }>;
  }): Promise<ToolResult> {
    try {
      const originalContent = await this.fileService.readFile(params.path);
      let modifiedContent = originalContent;

      for (const block of params.searchReplaceBlocks) {
        if (!modifiedContent.includes(block.search)) {
          return {
            success: false,
            error: `Search string not found: "${block.search.substring(0, 50)}..."`
          };
        }
        modifiedContent = modifiedContent.replace(block.search, block.replace);
      }

      // Diff 생성
      const diff = this.diffService.computeDiff(originalContent, modifiedContent);

      // 파일 저장
      await this.fileService.writeFile(params.path, modifiedContent);

      return {
        success: true,
        output: `Applied ${params.searchReplaceBlocks.length} changes`,
        metadata: { diff }
      };
    } catch (error) {
      return {
        success: false,
        error: `Failed to edit file: ${error}`
      };
    }
  }
}
```

### 5.3 도구 서비스

```typescript
class ToolService extends BaseService implements IToolService {
  private tools = new Map<string, ITool>();
  private pendingApprovals = new Map<string, PendingApproval>();

  private readonly _onToolExecuted = new Emitter<ToolExecutionEvent>();
  readonly onToolExecuted = this._onToolExecuted.event;

  constructor() {
    super();
    this._disposables.add(this._onToolExecuted);
  }

  registerTool(tool: ITool): IDisposable {
    this.tools.set(tool.definition.name, tool);

    return {
      dispose: () => {
        this.tools.delete(tool.definition.name);
      }
    };
  }

  async executeTool(
    name: string,
    params: Record<string, any>,
    options?: ExecutionOptions
  ): Promise<ToolResult> {
    const tool = this.tools.get(name);
    if (!tool) {
      return { success: false, error: `Tool not found: ${name}` };
    }

    // 유효성 검사
    if (tool.validate) {
      const validation = tool.validate(params);
      if (!validation.valid) {
        return { success: false, error: validation.error };
      }
    }

    // 승인 필요 여부 확인
    if (tool.definition.requiresApproval && !options?.skipApproval) {
      const approved = await this.requestApproval(name, params);
      if (!approved) {
        return { success: false, error: 'Tool execution rejected by user' };
      }
    }

    // 도구 실행
    const result = await tool.execute(params);

    // 이벤트 발생
    this._onToolExecuted.fire({
      tool: name,
      params,
      result,
      timestamp: Date.now()
    });

    return result;
  }

  getToolDefinitions(): ToolDefinition[] {
    return Array.from(this.tools.values()).map(t => t.definition);
  }

  private async requestApproval(
    toolName: string,
    params: Record<string, any>
  ): Promise<boolean> {
    // 승인 UI 로직 구현
    return true; // 기본값
  }
}
```

---

## 6. 코드 분석 서비스

### 6.1 컨텍스트 수집

```typescript
interface IContextService {
  gatherContext(request: ContextRequest): Promise<CodeContext>;
  updateCache(filePath: string): Promise<void>;
  getRelatedFiles(filePath: string): Promise<string[]>;
}

interface ContextRequest {
  filePath: string;
  position?: Position;
  scope: 'file' | 'directory' | 'project';
  maxTokens?: number;
}

interface CodeContext {
  currentFile: FileContext;
  relatedFiles: FileContext[];
  symbols: SymbolInfo[];
  imports: ImportInfo[];
  totalTokens: number;
}

class ContextService extends BaseService implements IContextService {
  private cache = new LRUCache<string, FileContext>(100);

  constructor(
    private fileService: IFileService,
    private symbolService: ISymbolService
  ) {
    super();
  }

  async gatherContext(request: ContextRequest): Promise<CodeContext> {
    const currentFile = await this.getFileContext(request.filePath);

    // 관련 파일 수집
    const relatedPaths = await this.getRelatedFiles(request.filePath);
    const relatedFiles: FileContext[] = [];

    let totalTokens = this.estimateTokens(currentFile.content);

    for (const path of relatedPaths) {
      if (request.maxTokens && totalTokens >= request.maxTokens) {
        break;
      }

      const context = await this.getFileContext(path);
      const tokens = this.estimateTokens(context.content);

      if (request.maxTokens && totalTokens + tokens > request.maxTokens) {
        continue;
      }

      relatedFiles.push(context);
      totalTokens += tokens;
    }

    // 심볼 정보 수집
    const symbols = await this.symbolService.getSymbols(request.filePath);

    // import 분석
    const imports = this.analyzeImports(currentFile.content);

    return {
      currentFile,
      relatedFiles,
      symbols,
      imports,
      totalTokens
    };
  }

  async getRelatedFiles(filePath: string): Promise<string[]> {
    const content = await this.fileService.readFile(filePath);
    const imports = this.analyzeImports(content);

    const relatedPaths: string[] = [];

    for (const imp of imports) {
      const resolvedPath = await this.resolveImportPath(filePath, imp.path);
      if (resolvedPath) {
        relatedPaths.push(resolvedPath);
      }
    }

    return relatedPaths;
  }

  private async getFileContext(filePath: string): Promise<FileContext> {
    const cached = this.cache.get(filePath);
    if (cached) {
      return cached;
    }

    const content = await this.fileService.readFile(filePath);
    const context: FileContext = {
      path: filePath,
      content,
      language: this.detectLanguage(filePath),
      lastModified: Date.now()
    };

    this.cache.set(filePath, context);
    return context;
  }

  private analyzeImports(content: string): ImportInfo[] {
    const imports: ImportInfo[] = [];

    // ES6 imports
    const esImportRegex = /import\s+(?:(?:\{[^}]*\}|\*\s+as\s+\w+|\w+)\s+from\s+)?['"]([^'"]+)['"]/g;
    let match;

    while ((match = esImportRegex.exec(content)) !== null) {
      imports.push({
        path: match[1],
        type: 'esm',
        line: content.substring(0, match.index).split('\n').length
      });
    }

    // CommonJS requires
    const requireRegex = /require\s*\(\s*['"]([^'"]+)['"]\s*\)/g;

    while ((match = requireRegex.exec(content)) !== null) {
      imports.push({
        path: match[1],
        type: 'commonjs',
        line: content.substring(0, match.index).split('\n').length
      });
    }

    return imports;
  }

  private estimateTokens(text: string): number {
    // 대략적인 토큰 추정 (문자 4개당 1토큰)
    return Math.ceil(text.length / 4);
  }

  private detectLanguage(filePath: string): string {
    const ext = filePath.split('.').pop()?.toLowerCase();
    const languageMap: Record<string, string> = {
      ts: 'typescript',
      tsx: 'typescriptreact',
      js: 'javascript',
      jsx: 'javascriptreact',
      py: 'python',
      go: 'go',
      rs: 'rust',
      java: 'java'
    };
    return languageMap[ext || ''] || 'plaintext';
  }

  private async resolveImportPath(
    fromPath: string,
    importPath: string
  ): Promise<string | null> {
    // import 경로 해석 로직
    // 상대 경로, 절대 경로, 패키지 경로 처리
    return null; // 구현 필요
  }
}
```

### 6.2 코드 변환 서비스

```typescript
interface ICodeTransformService {
  applyChanges(changes: CodeChange[]): Promise<TransformResult>;
  previewChanges(changes: CodeChange[]): Promise<DiffPreview[]>;
  rollback(checkpointId: string): Promise<void>;
}

interface CodeChange {
  filePath: string;
  type: 'edit' | 'create' | 'delete';
  content?: string;
  searchReplaceBlocks?: Array<{ search: string; replace: string }>;
}

class CodeTransformService extends BaseService implements ICodeTransformService {
  private checkpoints = new Map<string, CheckpointData>();

  constructor(
    private fileService: IFileService,
    private diffService: IDiffService
  ) {
    super();
  }

  async applyChanges(changes: CodeChange[]): Promise<TransformResult> {
    // 체크포인트 생성
    const checkpointId = this.createCheckpoint(changes);

    const results: ChangeResult[] = [];

    for (const change of changes) {
      try {
        const result = await this.applyChange(change);
        results.push(result);
      } catch (error) {
        // 실패 시 롤백
        await this.rollback(checkpointId);
        return {
          success: false,
          error: `Failed to apply change to ${change.filePath}: ${error}`,
          appliedChanges: results
        };
      }
    }

    return {
      success: true,
      checkpointId,
      appliedChanges: results
    };
  }

  async previewChanges(changes: CodeChange[]): Promise<DiffPreview[]> {
    const previews: DiffPreview[] = [];

    for (const change of changes) {
      const preview = await this.generatePreview(change);
      previews.push(preview);
    }

    return previews;
  }

  async rollback(checkpointId: string): Promise<void> {
    const checkpoint = this.checkpoints.get(checkpointId);
    if (!checkpoint) {
      throw new Error(`Checkpoint not found: ${checkpointId}`);
    }

    for (const [filePath, originalContent] of checkpoint.files) {
      if (originalContent === null) {
        // 파일이 생성되었으면 삭제
        await this.fileService.deleteFile(filePath);
      } else {
        // 원래 내용으로 복원
        await this.fileService.writeFile(filePath, originalContent);
      }
    }

    this.checkpoints.delete(checkpointId);
  }

  private async applyChange(change: CodeChange): Promise<ChangeResult> {
    switch (change.type) {
      case 'create':
        await this.fileService.writeFile(change.filePath, change.content || '');
        return { filePath: change.filePath, type: 'created' };

      case 'delete':
        await this.fileService.deleteFile(change.filePath);
        return { filePath: change.filePath, type: 'deleted' };

      case 'edit':
        if (change.searchReplaceBlocks) {
          let content = await this.fileService.readFile(change.filePath);

          for (const block of change.searchReplaceBlocks) {
            content = content.replace(block.search, block.replace);
          }

          await this.fileService.writeFile(change.filePath, content);
        } else if (change.content) {
          await this.fileService.writeFile(change.filePath, change.content);
        }
        return { filePath: change.filePath, type: 'edited' };
    }
  }

  private createCheckpoint(changes: CodeChange[]): string {
    const id = `checkpoint_${Date.now()}`;
    const files = new Map<string, string | null>();

    // 현재 파일 상태 저장 (동기 버전 필요)
    // 실제 구현에서는 비동기 처리 필요

    this.checkpoints.set(id, { id, files, timestamp: Date.now() });
    return id;
  }

  private async generatePreview(change: CodeChange): Promise<DiffPreview> {
    if (change.type === 'create') {
      return {
        filePath: change.filePath,
        type: 'create',
        content: change.content || ''
      };
    }

    if (change.type === 'delete') {
      const original = await this.fileService.readFile(change.filePath);
      return {
        filePath: change.filePath,
        type: 'delete',
        originalContent: original
      };
    }

    // 편집 미리보기
    const original = await this.fileService.readFile(change.filePath);
    let modified = original;

    if (change.searchReplaceBlocks) {
      for (const block of change.searchReplaceBlocks) {
        modified = modified.replace(block.search, block.replace);
      }
    } else if (change.content) {
      modified = change.content;
    }

    const diff = this.diffService.computeDiff(original, modified);

    return {
      filePath: change.filePath,
      type: 'edit',
      originalContent: original,
      modifiedContent: modified,
      diff
    };
  }
}
```

---

## 7. 에이전트 루프

### 7.1 메인 에이전트 루프

```typescript
interface IAgentService {
  processMessage(message: string): AsyncIterable<AgentEvent>;
  cancel(): void;
}

interface AgentEvent {
  type: 'thinking' | 'text' | 'tool_call' | 'tool_result' | 'done' | 'error';
  content?: string;
  toolCall?: ToolCall;
  toolResult?: ToolResult;
  error?: Error;
}

class AgentService extends BaseService implements IAgentService {
  private abortController: AbortController | null = null;

  constructor(
    private llmService: ILLMService,
    private toolService: IToolService,
    private contextService: IContextService,
    private configService: IConfigService
  ) {
    super();
  }

  async *processMessage(userMessage: string): AsyncIterable<AgentEvent> {
    this.abortController = new AbortController();

    try {
      // 컨텍스트 수집
      yield { type: 'thinking', content: 'Gathering context...' };

      const context = await this.contextService.gatherContext({
        filePath: this.getCurrentFile(),
        scope: 'project',
        maxTokens: 50000
      });

      // 메시지 구성
      const messages = this.buildMessages(userMessage, context);

      // 도구 정의 가져오기
      const tools = this.toolService.getToolDefinitions();

      // LLM 요청
      const request: LLMRequest = {
        messages,
        model: this.configService.getModel(),
        tools,
        signal: this.abortController.signal
      };

      // 에이전트 루프
      let continueLoop = true;
      let iteration = 0;
      const maxIterations = 10;

      while (continueLoop && iteration < maxIterations) {
        iteration++;

        const stream = this.llmService.sendMessage(
          this.configService.getProviderId(),
          request
        );

        const pendingToolCalls: ToolCall[] = [];

        for await (const fragment of stream) {
          if (fragment.type === 'text') {
            yield { type: 'text', content: fragment.content };
          } else if (fragment.type === 'tool_use') {
            pendingToolCalls.push(fragment.toolCall!);
          } else if (fragment.type === 'error') {
            throw fragment.error;
          }
        }

        // 도구 호출이 없으면 루프 종료
        if (pendingToolCalls.length === 0) {
          continueLoop = false;
          break;
        }

        // 도구 실행
        for (const toolCall of pendingToolCalls) {
          yield { type: 'tool_call', toolCall };

          const result = await this.toolService.executeTool(
            toolCall.name,
            toolCall.arguments
          );

          yield { type: 'tool_result', toolCall, toolResult: result };

          // 도구 결과를 메시지에 추가
          messages.push({
            role: 'assistant',
            content: null,
            toolCalls: [toolCall]
          });

          messages.push({
            role: 'tool',
            toolCallId: toolCall.id,
            content: JSON.stringify(result)
          });
        }
      }

      yield { type: 'done' };

    } catch (error) {
      if (error instanceof Error && error.name === 'AbortError') {
        yield { type: 'done', content: 'Cancelled' };
      } else {
        yield { type: 'error', error: error as Error };
      }
    } finally {
      this.abortController = null;
    }
  }

  cancel(): void {
    this.abortController?.abort();
  }

  private buildMessages(userMessage: string, context: CodeContext): ChatMessage[] {
    const systemPrompt = this.buildSystemPrompt(context);

    return [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userMessage }
    ];
  }

  private buildSystemPrompt(context: CodeContext): string {
    let prompt = `You are a code analysis and generation assistant.\n\n`;

    prompt += `## Current File\n`;
    prompt += `Path: ${context.currentFile.path}\n`;
    prompt += `Language: ${context.currentFile.language}\n\n`;
    prompt += `\`\`\`${context.currentFile.language}\n`;
    prompt += context.currentFile.content;
    prompt += `\n\`\`\`\n\n`;

    if (context.relatedFiles.length > 0) {
      prompt += `## Related Files\n`;
      for (const file of context.relatedFiles) {
        prompt += `### ${file.path}\n`;
        prompt += `\`\`\`${file.language}\n`;
        prompt += file.content;
        prompt += `\n\`\`\`\n\n`;
      }
    }

    return prompt;
  }

  private getCurrentFile(): string {
    // 현재 활성 파일 경로 반환
    return '';
  }
}
```

---

## 8. 구현 우선순위

### 8.1 Phase 1: 코어 인프라 (1-2주)

| 순서 | 컴포넌트 | 설명 | 참조 문서 |
|------|----------|------|-----------|
| 1 | DI 시스템 | 서비스 컨테이너, 식별자 | `02-아키텍처-패턴.md` |
| 2 | 이벤트 시스템 | Emitter, Event | `02-아키텍처-패턴.md` |
| 3 | Disposable | 리소스 관리 | `08-에러-처리-복구.md` |
| 4 | 설정 시스템 | 구성 관리 | `07-상태-관리.md` |

### 8.2 Phase 2: 도구 시스템 (1-2주)

| 순서 | 컴포넌트 | 설명 | 참조 문서 |
|------|----------|------|-----------|
| 1 | 도구 레지스트리 | 도구 등록/발견 | `04-도구-시스템.md` |
| 2 | 기본 도구 | read_file, edit_file | `04-도구-시스템.md` |
| 3 | 터미널 도구 | 명령 실행 | `04-도구-시스템.md` |
| 4 | 승인 시스템 | 위험 도구 승인 | `04-도구-시스템.md` |

### 8.3 Phase 3: LLM 통합 (1-2주)

| 순서 | 컴포넌트 | 설명 | 참조 문서 |
|------|----------|------|-----------|
| 1 | 프로바이더 인터페이스 | 추상화 레이어 | `03-LLM-통합-패턴.md` |
| 2 | 스트리밍 처리 | AsyncIterable | `03-LLM-통합-패턴.md` |
| 3 | 에러 처리 | 재시도, 폴백 | `08-에러-처리-복구.md` |
| 4 | 멀티 프로바이더 | OpenAI, Anthropic 등 | `03-LLM-통합-패턴.md` |

### 8.4 Phase 4: 코드 분석 (2-3주)

| 순서 | 컴포넌트 | 설명 | 참조 문서 |
|------|----------|------|-----------|
| 1 | 컨텍스트 수집 | 파일 분석 | `05-코드-분석-생성-패턴.md` |
| 2 | 심볼 분석 | 의존성 추적 | `05-코드-분석-생성-패턴.md` |
| 3 | 코드 변환 | 편집 적용 | `05-코드-분석-생성-패턴.md` |
| 4 | Diff 시스템 | 변경 시각화 | `05-코드-분석-생성-패턴.md` |

### 8.5 Phase 5: 에이전트 루프 (1-2주)

| 순서 | 컴포넌트 | 설명 | 참조 문서 |
|------|----------|------|-----------|
| 1 | 메시지 파이프라인 | 대화 관리 | `03-LLM-통합-패턴.md` |
| 2 | 에이전트 루프 | 도구 호출 처리 | 본 문서 |
| 3 | 취소 처리 | AbortController | `08-에러-처리-복구.md` |
| 4 | 체크포인트 | 롤백 지원 | `07-상태-관리.md` |

---

## 9. 핵심 참조 파일

### 9.1 코어 인프라

| 파일 | 역할 | 줄 수 |
|------|------|-------|
| `src/vs/platform/instantiation/common/instantiation.ts` | DI 코어 | ~200 |
| `src/vs/base/common/event.ts` | 이벤트 시스템 | ~1300 |
| `src/vs/base/common/lifecycle.ts` | Disposable | ~500 |
| `src/vs/base/common/async.ts` | 비동기 유틸 | ~800 |

### 9.2 도구 시스템

| 파일 | 역할 | 줄 수 |
|------|------|-------|
| `src/vs/workbench/contrib/void/browser/toolsService.ts` | 도구 서비스 | ~600 |
| `src/vs/workbench/contrib/void/common/toolsServiceTypes.ts` | 도구 타입 | ~300 |
| `src/vs/workbench/contrib/void/browser/tools/*.ts` | 개별 도구 구현 | 다양 |

### 9.3 LLM 통합

| 파일 | 역할 | 줄 수 |
|------|------|-------|
| `src/vs/workbench/contrib/void/common/sendLLMMessageService.ts` | LLM 인터페이스 | ~200 |
| `src/vs/workbench/contrib/void/electron-main/llmMessage/sendLLMMessage.impl.ts` | 프로바이더 구현 | ~1500 |
| `src/vs/workbench/contrib/void/common/modelCapabilities.ts` | 모델 설정 | ~1600 |

### 9.4 코드 분석

| 파일 | 역할 | 줄 수 |
|------|------|-------|
| `src/vs/workbench/contrib/void/browser/editCodeService.ts` | 코드 편집 | ~2200 |
| `src/vs/workbench/contrib/void/browser/contextGatheringService.ts` | 컨텍스트 | ~400 |
| `src/vs/workbench/contrib/void/browser/voidDiff/` | Diff 시스템 | ~1000 |

---

## 10. 권장 기술 스택

### 10.1 언어 및 런타임

| 기술 | 버전 | 용도 |
|------|------|------|
| TypeScript | 5.x | 주 개발 언어 |
| Node.js | 20.x LTS | 런타임 환경 |
| ESM | - | 모듈 시스템 |

### 10.2 핵심 라이브러리

| 라이브러리 | 용도 | 대안 |
|------------|------|------|
| `openai` | OpenAI API | 직접 구현 |
| `@anthropic-ai/sdk` | Anthropic API | 직접 구현 |
| `zod` | 스키마 검증 | `yup`, `joi` |
| `commander` | CLI 파싱 | `yargs` |

### 10.3 빌드 도구

| 도구 | 용도 |
|------|------|
| `esbuild` | 번들링 |
| `vitest` | 테스트 |
| `eslint` | 린팅 |
| `prettier` | 포맷팅 |

---

## 11. 모범 사례

### 11.1 코드 품질

```typescript
// ✅ 좋음: 명확한 타입, 에러 처리
async function readFileWithFallback(
  path: string,
  fallback: string
): Promise<string> {
  try {
    return await fs.readFile(path, 'utf-8');
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === 'ENOENT') {
      return fallback;
    }
    throw error;
  }
}

// ❌ 나쁨: any 타입, 에러 무시
async function readFile(path: any) {
  try {
    return await fs.readFile(path);
  } catch {
    return '';
  }
}
```

### 11.2 서비스 설계

```typescript
// ✅ 좋음: 인터페이스 분리, 의존성 주입
interface IFileReader {
  read(path: string): Promise<string>;
}

interface IFileWriter {
  write(path: string, content: string): Promise<void>;
}

class FileService implements IFileReader, IFileWriter {
  constructor(private logger: ILogger) {}
  // ...
}

// ❌ 나쁨: 모든 것을 하나의 클래스에
class DoEverything {
  readFile() {}
  writeFile() {}
  sendEmail() {}
  generateReport() {}
}
```

### 11.3 이벤트 사용

```typescript
// ✅ 좋음: 이벤트로 느슨한 결합
class FileWatcher extends BaseService {
  private readonly _onFileChanged = new Emitter<FileChangeEvent>();
  readonly onFileChanged = this._onFileChanged.event;

  constructor() {
    super();
    this._disposables.add(this._onFileChanged);
  }
}

// 사용
const disposable = fileWatcher.onFileChanged(event => {
  console.log('File changed:', event.path);
});

// ❌ 나쁨: 직접 콜백 전달
class FileWatcher {
  constructor(private callback: (event: any) => void) {}
}
```

---

## 12. 체크리스트

### 12.1 구현 전 체크

- [ ] 서비스 인터페이스 정의 완료
- [ ] DI 컨테이너 설정 완료
- [ ] 기본 도구 정의 완료
- [ ] LLM 프로바이더 설정 완료

### 12.2 구현 중 체크

- [ ] 모든 서비스가 IDisposable 구현
- [ ] 이벤트는 DisposableStore에 등록
- [ ] 에러 처리 및 로깅 포함
- [ ] 취소 토큰 지원

### 12.3 구현 후 체크

- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성
- [ ] 메모리 누수 확인
- [ ] 성능 벤치마크

---

## 13. 참조 문서

| 문서 | 핵심 내용 |
|------|-----------|
| [01-프로젝트-개요.md](./01-프로젝트-개요.md) | Void 프로젝트 구조 |
| [02-아키텍처-패턴.md](./02-아키텍처-패턴.md) | DI, 이벤트, IPC 패턴 |
| [03-LLM-통합-패턴.md](./03-LLM-통합-패턴.md) | 멀티 프로바이더, 스트리밍 |
| [04-도구-시스템.md](./04-도구-시스템.md) | 도구 정의, MCP 통합 |
| [05-코드-분석-생성-패턴.md](./05-코드-분석-생성-패턴.md) | 컨텍스트 수집, 코드 변환 |
| [06-확장-시스템.md](./06-확장-시스템.md) | 플러그인 아키텍처 |
| [07-상태-관리.md](./07-상태-관리.md) | 설정, 체크포인트 |
| [08-에러-처리-복구.md](./08-에러-처리-복구.md) | 에러 핸들링, 취소 |
| [09-성능-최적화.md](./09-성능-최적화.md) | 캐싱, 병렬화 |
| [10-테스트-패턴.md](./10-테스트-패턴.md) | 모킹, 비동기 테스트 |

---

*이 문서는 hdsp-agent 개발의 시작점으로 활용하세요. 각 섹션의 코드는 Void에서 추출한 패턴을 기반으로 하며, 프로젝트 요구사항에 맞게 조정이 필요합니다.*
