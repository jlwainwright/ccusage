# ccusage Architecture Patterns & Design Decisions
*Generated: 2025-06-28*

## Overview

This document analyzes the architectural patterns and design decisions implemented in the ccusage TypeScript CLI tool. The codebase demonstrates several sophisticated patterns that contribute to its maintainability, type safety, and extensibility.

---

## 🏗️ Core Architectural Patterns

### 1. **Branded Types Pattern** 
*Implementation: `src/_types.ts:9-84`*

**Pattern Description**: Uses Zod's brand functionality to create type-safe wrappers around primitive strings, preventing category errors and ensuring data integrity.

```typescript
export const modelNameSchema = z.string()
    .min(1, 'Model name cannot be empty')
    .brand<'ModelName'>();

export type ModelName = z.infer<typeof modelNameSchema>;
export const createModelName = (value: string): ModelName => modelNameSchema.parse(value);
```

**Benefits**:
- Compile-time prevention of mixing different string types
- Runtime validation ensures data meets expected formats
- Self-documenting code with clear domain semantics

**Usage Pattern**: Applied to 9 core domain concepts (ModelName, SessionId, RequestId, timestamps, etc.)

### 2. **Command Registry Pattern**
*Implementation: `src/commands/index.ts:13-18`*

**Pattern Description**: Map-based command registry with pluggable subcommands and shared configuration.

```typescript
const subCommands = new Map();
subCommands.set('daily', dailyCommand);
subCommands.set('monthly', monthlyCommand);
subCommands.set('session', sessionCommand);
subCommands.set('blocks', blocksCommand);
subCommands.set('mcp', mcpCommand);
```

**Benefits**:
- Easy addition of new commands without modifying core CLI logic
- Shared argument configuration through `sharedCommandConfig`
- Clear separation between command logic and CLI framework

### 3. **Functional Data Pipeline Pattern**
*Implementation: `src/data-loader.ts`, `src/calculate-cost.ts`*

**Pattern Description**: Immutable data transformations through a series of pure functions with clear input/output contracts.

```typescript
// Data flow: Raw JSONL → Parsed Entries → Aggregated Usage → Calculated Costs
loadDailyUsageData() → calculateTotals() → createTotalsObject()
```

**Benefits**:
- Predictable data transformations
- Easy testing of individual pipeline stages
- Composable operations for different aggregation needs

### 4. **Schema-First Validation Pattern**
*Implementation: Throughout codebase with Zod schemas*

**Pattern Description**: Define data schemas first, then derive TypeScript types and validation functions.

```typescript
export const usageDataSchema = z.object({
    timestamp: isoTimestampSchema,
    model: modelNameSchema,
    inputTokens: z.number().int().min(0),
    outputTokens: z.number().int().min(0),
    // ... additional fields
});

export type UsageData = z.infer<typeof usageDataSchema>;
```

**Benefits**:
- Single source of truth for data contracts
- Runtime validation matches compile-time types
- Automatic type derivation reduces maintenance burden

---

## 🔧 Design Decisions Analysis

### 1. **Multi-Path Claude Directory Support**
*Implementation: `src/data-loader.ts:66-110`*

**Decision**: Support multiple Claude data directories with environment variable override capability.

**Rationale**: 
- Handles breaking changes in Claude Code (moved from `~/.claude` to `~/.config/claude`)
- Supports multiple installations or custom configurations
- Maintains backward compatibility

**Implementation Strategy**:
```typescript
export function getClaudePaths(): string[] {
    // 1. Check environment variable (comma-separated)
    // 2. Try new default path (~/.config/claude)
    // 3. Fall back to old default path (~/.claude)
    // 4. Deduplicate using normalized paths
}
```

**Trade-offs**:
- ✅ Flexible configuration options
- ✅ Backward compatibility maintained
- ⚠️ Increased complexity in path resolution logic

### 2. **Cost Calculation Mode Strategy**
*Implementation: `src/_types.ts:87-97`*

**Decision**: Three distinct cost calculation modes (auto, calculate, display) for handling pre-calculated vs. computed costs.

**Modes**:
- `auto`: Use pre-calculated when available, compute from tokens otherwise
- `calculate`: Always compute from token counts using model pricing
- `display`: Always use pre-calculated costs, show 0 for missing

**Rationale**: 
- Handles inconsistent cost data availability in Claude logs
- Provides user control over cost calculation accuracy vs. completeness
- Supports different use cases (billing validation vs. quick estimates)

**Benefits**:
- Flexibility for different data scenarios
- Clear user expectations about cost calculations
- Enables validation of Claude's cost calculations

### 3. **In-Source Testing Strategy**
*Implementation: Vitest with `import.meta.vitest` guards*

**Decision**: Place tests directly in source files using conditional imports.

```typescript
if (import.meta.vitest != null) {
    const { describe, it, expect } = await import('vitest');
    // Test implementations
}
```

**Benefits**:
- Tests remain close to implementation
- Reduced file count and navigation overhead
- Automatic tree-shaking excludes tests from production builds

**Trade-offs**:
- ✅ Better test maintenance proximity
- ✅ Reduced project complexity
- ⚠️ Less conventional approach may confuse some developers

### 4. **MCP Integration Architecture**
*Implementation: `src/mcp.ts`*

**Decision**: Dual-transport MCP server (stdio and HTTP) with comprehensive tool exposure.

**Architecture**:
```typescript
// Expose core functionality as MCP tools
const mcpTools = {
    'daily': loadDailyUsageData,
    'monthly': loadMonthlyUsageData,
    'session': loadSessionData,
    'blocks': loadSessionBlockData
};
```

**Benefits**:
- Enables integration with multiple MCP clients
- Provides programmatic access to all CLI functionality
- Maintains consistency between CLI and MCP interfaces

---

## 🔍 Pattern Usage Analysis

### 1. **Error Handling Patterns**

#### Current State: Mixed Approaches
```typescript
// Pattern A: Silent catching (src/debug.ts:206)
catch {
    // Skip invalid JSON
}

// Pattern B: Structured logging (src/data-loader.ts)
catch (error) {
    logger.debug(`Error parsing file: ${error.message}`);
}
```

#### Recommended Standardization:
```typescript
// Unified error handling pattern
interface ErrorContext {
    operation: string;
    file?: string;
    line?: number;
    metadata?: Record<string, unknown>;
}

function handleParsingError(error: Error, context: ErrorContext): void {
    logger.debug(`${context.operation} failed: ${error.message}`, context);
}
```

### 2. **Module Organization Patterns**

#### Current Structure:
```
src/
├── _*.ts          # Internal utilities (underscore prefix)
├── commands/      # CLI command implementations
├── *.ts          # Public API modules
```

#### Pattern Analysis:
- ✅ Clear public vs. internal API separation with underscore prefix
- ✅ Commands grouped in dedicated directory
- ⚠️ Some modules grow large (data-loader.ts: 3,676 lines)

#### Recommended Evolution:
```
src/
├── core/          # Core domain logic
│   ├── types/     # Type definitions
│   ├── validation/# Schema definitions
│   └── processing/# Data processing logic
├── infrastructure/# External integrations
├── commands/      # CLI implementations
└── utils/         # Shared utilities
```

### 3. **Configuration Patterns**

#### Environment Variable Handling:
```typescript
// Current pattern: Direct process.env access with defaults
const envPaths = (process.env[CLAUDE_CONFIG_DIR_ENV] ?? '').trim();

// Recommended: Configuration object pattern
interface Config {
    claudeConfigDir: string[];
    debugMode: boolean;
    logLevel: LogLevel;
}

function loadConfig(): Config {
    return {
        claudeConfigDir: parsePathList(process.env.CLAUDE_CONFIG_DIR),
        debugMode: process.env.DEBUG === 'true',
        logLevel: parseLogLevel(process.env.LOG_LEVEL)
    };
}
```

---

## 🚀 Architectural Evolution Opportunities

### 1. **Plugin Architecture Pattern**
*For future extensibility*

**Current**: Hardcoded command implementations
**Opportunity**: Dynamic plugin loading for custom analysis commands

```typescript
interface AnalysisPlugin {
    name: string;
    command: CommandDefinition;
    process(data: UsageData[]): AnalysisResult;
}

class PluginManager {
    register(plugin: AnalysisPlugin): void;
    execute(pluginName: string, data: UsageData[]): Promise<AnalysisResult>;
}
```

### 2. **Event-Driven Architecture Pattern**
*For real-time monitoring*

**Current**: Synchronous data processing
**Opportunity**: Event-driven processing for live monitoring

```typescript
interface UsageEvent {
    type: 'file_added' | 'usage_recorded' | 'session_started';
    timestamp: Date;
    payload: unknown;
}

class UsageEventBus {
    subscribe(eventType: string, handler: EventHandler): void;
    publish(event: UsageEvent): void;
}
```

### 3. **Repository Pattern**
*For data source abstraction*

**Current**: Direct file system access
**Opportunity**: Abstract data sources for testing and flexibility

```typescript
interface UsageRepository {
    findUsageData(criteria: SearchCriteria): Promise<UsageData[]>;
    watchForChanges(callback: ChangeCallback): void;
}

class FileSystemUsageRepository implements UsageRepository {
    // Current implementation
}

class MockUsageRepository implements UsageRepository {
    // Test implementation
}
```

---

## 📊 Pattern Effectiveness Metrics

### Type Safety Coverage
```
✅ Branded types: 9 core domain concepts
✅ Schema validation: 100% of external data
✅ Runtime type checking: All API boundaries
⚠️ Test type safety: Some 'any' usage remains
```

### Code Organization
```
✅ Clear module boundaries: 28 TypeScript files
✅ Consistent naming: Underscore prefix for internals
⚠️ Module size: 4 files exceed 800 lines
⚠️ Complexity concentration: Some functions > 100 lines
```

### Error Handling Consistency
```
⚠️ Mixed patterns: Silent catches vs. structured logging
⚠️ Error context: Inconsistent metadata capture
⚠️ Recovery strategies: Varies by module
```

---

## 🎯 Architectural Recommendations

### 1. **Immediate Improvements** (Week 1)
- Standardize error handling patterns across all modules
- Extract configuration object from environment variable access
- Decompose large functions using method extraction

### 2. **Structural Enhancements** (Month 1)
- Implement module reorganization with clearer boundaries
- Add dependency injection for external services
- Create unified data access layer

### 3. **Advanced Features** (Quarter 1)
- Consider plugin architecture for extensibility
- Evaluate event-driven patterns for real-time features
- Implement repository pattern for data source abstraction

---

## 🏆 Conclusion

The ccusage architecture demonstrates sophisticated use of TypeScript's type system and modern JavaScript patterns. The core strengths lie in type safety, functional data processing, and clear separation of concerns. The main opportunities involve standardizing patterns that have grown organically and preparing the architecture for future scalability requirements.

**Key Architectural Assets:**
- Robust branded type system preventing category errors
- Functional data pipeline enabling predictable transformations
- Flexible configuration supporting multiple environments
- Comprehensive schema validation ensuring data integrity

**Investment Priorities:**
1. **Consistency**: Standardize error handling and configuration patterns
2. **Modularity**: Break down large modules into focused components
3. **Extensibility**: Prepare architecture for plugin-based enhancements
4. **Scalability**: Implement patterns supporting larger datasets and real-time processing

The architecture is well-positioned for continued evolution while maintaining the excellent type safety and developer experience that characterizes the current implementation.

---

*This document analyzes architectural patterns and design decisions as of 2025-06-28. Regular review recommended as codebase evolves.*