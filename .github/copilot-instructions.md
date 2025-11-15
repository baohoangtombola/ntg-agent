---
description: GitHub Copilot Instructions for NTG Agent
---

# GitHub Copilot Instructions for NTG Agent

## AI Persona

You are an experienced Senior .NET Developer. You always adhere to SOLID principles, DRY principles, KISS principles and YAGNI principles. You always follow OWASP best practices. You always break tasks down to the smallest units and approach solving any task in a step by step manner.

## Architecture Overview

This is a **microservices chatbot system** built with **.NET 9 Aspire**, using **Microsoft Agent Framework** for AI capabilities. The architecture follows these core patterns:

### Service Structure (NTG.Agent.AppHost/Program.cs)
```csharp
// Dependency order is critical - services reference their dependencies via WithReference()
var mcpServer = builder.AddProject<Projects.NTG_Agent_MCP_Server>("ntg-agent-mcp-server");
var knowledge = builder.AddProject<Projects.NTG_Agent_Knowledge>("ntg-agent-knowledge");
var orchestrator = builder.AddProject<Projects.NTG_Agent_Orchestrator>("ntg-agent-orchestrator")
    .WithReference(mcpServer)    // Orchestrator depends on MCP server for tool discovery
    .WithReference(knowledge);   // Orchestrator depends on Knowledge for RAG
```

**Service responsibilities:**
- **NTG.Agent.Orchestrator**: Core AI agent orchestration using Microsoft Agent Framework. Manages conversations, agents, and AI tool execution
- **NTG.Agent.Knowledge**: Kernel Memory-based RAG service for document ingestion/retrieval with embeddings
- **NTG.Agent.MCP.Server**: Model Context Protocol server exposing AI tools via HTTP transport
- **NTG.Agent.WebClient**: End-user Blazor WebAssembly + Server UI
- **NTG.Agent.Admin**: Admin portal with YARP reverse proxy for BFF pattern (cookie-based auth)

### Key Architectural Decisions
- **Service Discovery**: Use environment variables `services__<service-name>__https__0` or `services__<service-name>__http__0` for dynamic endpoint resolution (see Orchestrator/Program.cs line 75)
- **Shared Authentication**: Data protection keys stored in `../key/` directory, shared across Admin and Orchestrator for cookie-based auth
- **AI Agent Factory Pattern**: `AgentFactory` creates agents dynamically based on provider (GitHub Models, Azure OpenAI) and loads enabled tools from database
- **Tool Architecture**: Tools come from three sources: 1) Built-in static plugins (DateTimeTools), 2) MCP server via HTTP, 3) Dynamic KnowledgePlugin for RAG

## Essential Developer Workflows

### Starting the Application
```bash
# Run from NTG.Agent.AppHost - this starts ALL services via Aspire orchestration
dotnet run --project NTG.Agent.AppHost
# Access Aspire Dashboard (shows all services, logs, traces, metrics)
# Default endpoints will be shown in the dashboard
```

### Database Migrations (CRITICAL)
```bash
# Admin database (Identity/Users)
cd NTG.Agent.Admin/NTG.Agent.Admin
dotnet ef database update

# Orchestrator database (Agents, Conversations, Documents, Tags)
cd NTG.Agent.Orchestrator
dotnet ef database update

# Add new migration
dotnet ef migrations add MigrationName --project NTG.Agent.Orchestrator
```

**Note**: Both projects have `DesignTimeDbContextFactory` for migration tooling.

### User Secrets Setup (Required for First Run)
```bash
# Orchestrator - GitHub Models token
cd NTG.Agent.Orchestrator
dotnet user-secrets set "GitHub:Models:GitHubToken" "<your_github_token>"

# Knowledge - GitHub token + SQL Server connection
cd NTG.Agent.Knowledge
dotnet user-secrets set "KernelMemory:Services:OpenAI:APIKey" "<github_token>"
dotnet user-secrets set "KernelMemory:Services:SqlServer:ConnectionString" "Server=.;Database=NTGAgent;..."

# MCP.Server - Google Custom Search (for SearchOnlineTool)
cd NTG.Agent.MCP.Server
dotnet user-secrets set "Google:ApiKey" "<google_api_key>"
dotnet user-secrets set "Google:SearchEngineId" "<search_engine_id>"
```

## General Instructions

- Make only high confidence suggestions when reviewing code changes.
- Write code with good maintainability practices, including comments on why certain design decisions were made.
- Handle edge cases and write clear exception handling.
- For libraries or external dependencies, mention their usage and purpose in comments.

## Coding Standards & Patterns

### General Guidelines
- Use **C# 12+** with `<LangVersion>latest</LangVersion>` (test projects) and `<Nullable>enable</Nullable>`
- Primary constructors everywhere: `public class SomeClient(HttpClient httpClient)` - no field declarations
- Follow **async/await** patterns consistently - ALL database and HTTP operations must be async
- Use **dependency injection** throughout - register services in Program.cs, inject via constructors
- **OpenTelemetry tracing**: Agents and chat clients use `.UseOpenTelemetry(sourceName: "NTG.Agent.Orchestrator")` for distributed tracing
- Follow **SOLID principles** and clean architecture patterns

### Naming Conventions  
- Use **PascalCase** for classes, methods, properties, and public members
- Use **camelCase** for local variables and private fields
- Use **kebab-case** for CSS classes and HTML attributes
- Prefix interfaces with 'I' (e.g., `IKnowledgeService`, `IAgentFactory`)
- Use descriptive names that reflect business domain concepts

### File Organization
- **Models**: `Models/Chat/`, `Models/Documents/`, `Models/Agents/` - domain-driven organization
- **DTOs**: Separate `NTG.Agent.Common/Dtos/` project for shared data contracts
- **Services**: `Services/` with interface/implementation pairs (e.g., `IKnowledgeService`, `KernelMemoryKnowledge`)
- **Plugins**: AI function tools in `Plugins/` (e.g., `KnowledgePlugin.cs` for RAG search)
- **Tools**: Separate projects in `AITools/` (e.g., `NTG.Agent.AITools.SimpleTools`, `NTG.Agent.AITools.SearchOnlineTool`)
- Blazor components in `Components/` with subfolder organization, `_Imports.razor` for common usings

## Entity Framework Patterns

### DbContext Usage
- Use `AgentDbContext` as the main database context
- Follow code-first approach with migrations
- Use fluent API configuration when needed
- Implement proper foreign key relationships

### Model Conventions
```csharp
public class ExampleEntity
{
    public ExampleEntity()
    {
        CreatedAt = DateTime.UtcNow;
        UpdatedAt = DateTime.UtcNow;
    }

    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    // Other properties...
}
```

### Common Entities
- **Agent** - AI assistant configuration with instructions
- **Conversation** - Chat session container
- **ChatMessage** - Individual messages with roles (User/Assistant/System)
- **Document** - Uploaded content with metadata and folder association
- **Folder** - Hierarchical document organization
- **Tag** - Content categorization system

## Blazor Component Patterns

### Component Structure
```razor
@using NTG.Agent.Shared.Dtos.SomeNamespace
@inject SomeService SomeService
@inject ILogger<ComponentName> Logger
@rendermode InteractiveServer

<!-- Markup here -->

@code {
    [Parameter] public Guid SomeId { get; set; }
    
    private bool isLoading = true;
    private string? errorMessage;
    
    protected override async Task OnInitializedAsync()
    {
        try
        {
            isLoading = true;
            // Component logic
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Error message");
            errorMessage = "User-friendly error message";
        }
        finally
        {
            isLoading = false;
        }
    }
}
```

### Component Guidelines
- Use `@rendermode InteractiveServer` for server-side interactivity
- Use `@rendermode InteractiveWebAssembly` for client-side components
- Implement proper loading states and error handling
- Use dependency injection for services
- Follow the established parameter naming patterns

## AI Integration Patterns

### Microsoft Agent Framework (Orchestrator/Agents/AgentFactory.cs)
```csharp
// Agent creation with telemetry and function invocation
var chatClient = openAiClient.GetChatClient(modelName)
    .AsIChatClient()
    .AsBuilder()
    .UseFunctionInvocation()  // Required for tool calling
    .UseOpenTelemetry(sourceName: "NTG.Agent.Orchestrator", configure: (cfg) => cfg.EnableSensitiveData = true)
    .Build();

var agent = new ChatClientAgent(chatClient, name: "NTG.Agent", instructions: agent.Instructions, tools: tools)
    .AsBuilder()
    .UseOpenTelemetry(sourceName: "NTG.Agent.Orchestrator")
    .Build();
```

### Tool Architecture (Three Sources)
1. **Built-in Static Tools**: `AIFunctionFactory.Create(DateTimeTools.GetCurrentDateTime)` - simple C# methods decorated with `[Description]`
2. **MCP Server Tools**: Loaded dynamically via `McpClient.ListToolsAsync()` from HTTP transport (see `AgentFactory.GetMcpToolsAsync`)
3. **Dynamic RAG Plugin**: `KnowledgePlugin.AsAITool()` - created per-conversation with specific tags for document filtering

### RAG Implementation (Kernel Memory)
- **Document Ingestion**: Knowledge service uses Kernel Memory for chunking, embedding, and storage
- **Semantic Search**: `KnowledgePlugin.SearchAsync(query, agentId, tags)` - tag-based access control
- **Client Configuration**: `MemoryWebClient` initialized with service discovery endpoint from environment variables

### Agent Configuration
- Agents stored in database with provider configuration (GitHub Models, Azure OpenAI)
- `AgentTools` table tracks enabled tools per agent - only enabled tools loaded
- Basic agents (no tools) created for utility tasks like summarization, conversation naming

## Authentication & Authorization

### Shared Cookie Authentication
- Data protection keys persisted to `../key/` directory (shared between Admin and Orchestrator)
- `.SetApplicationName("NTGAgent")` ensures cookie compatibility across services
- Admin uses YARP reverse proxy (`app.MapReverseProxy()`) to forward authenticated requests to Orchestrator
- **Limitation**: Cookies NOT included in Blazor Server requests - only works with WebAssembly client

### Authorization Patterns
```csharp
[Authorize(Roles = "Admin")]
public class AdminController : ControllerBase
{
    // Admin-only endpoints
}

// In Blazor components
@attribute [Authorize(Roles = "Admin")]
```

### Identity Tables in Orchestrator DbContext
- `AspNetUsers`, `AspNetRoles`, `AspNetUserRoles` excluded from migrations (`.ToTable(name, t => t.ExcludeFromMigrations())`)
- Orchestrator only reads identity data - Admin project owns the schema

## REST API Patterns

### Controller Structure
- Place controllers in `Controllers/` folder within the service project
- Use `[ApiController]` and `[Route("api/[controller]")]` attributes
- Implement proper dependency injection in constructor
- Use async/await patterns for all operations

### API Documentation
- Use XML documentation comments for all public methods
- Document parameters, return types, and exceptions
- Include remarks for complex business logic

## Service Communication

### HTTP Client Patterns
```csharp
public class SomeClient(HttpClient httpClient)
{
    public async Task<List<SomeDto>> GetItemsAsync(Guid id)
    {
        var response = await httpClient.GetAsync($"api/endpoint/{id}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<SomeDto>>() ?? new List<SomeDto>();
    }
}
```

### Service Registration
- Register services in Program.cs using dependency injection
- Use Aspire service defaults for common configuration
- Configure HTTP clients with base addresses
- Implement service discovery for inter-service communication

## Testing Guidelines

### Unit Testing
- Create test projects following the pattern `ProjectName.Tests`
- Use xUnit as the testing framework
- Mock dependencies using appropriate mocking frameworks
- Test business logic in isolation

### Integration Testing
- Test API endpoints with proper authentication
- Verify database operations with test databases
- Test service communication patterns

## Configuration Management

### Application Settings
- Use `appsettings.json` for default configuration
- Use `appsettings.Development.json` for development overrides
- Use user secrets for sensitive data during development
- Support environment-specific configuration

## Error Handling Patterns

### Logging
```csharp
try
{
    // Operation
}
catch (Exception ex)
{
    _logger.LogError(ex, "Descriptive error message with context: {ContextValue}", contextValue);
    throw; // Re-throw if needed or handle appropriately
}
```

### User-Facing Errors
- Provide user-friendly error messages
- Log detailed technical errors for debugging
- Implement proper exception handling in controllers and services
- Use status messages in Blazor components

## Database Migration Patterns

### Adding Migrations
```bash
dotnet ef migrations add MigrationName --project NTG.Agent.Orchestrator
dotnet ef database update --project NTG.Agent.Orchestrator
```

### Migration Guidelines
- Use descriptive migration names
- Include seed data for default entities
- Test migrations in both directions (up and down)
- Document breaking changes

## Performance Considerations

### Best Practices
- Use async/await throughout the application
- Implement proper caching strategies
- Use projection in Entity Framework queries
- Optimize database queries with appropriate indexes
- Use streaming for large file uploads

### Telemetry
- Use OpenTelemetry for distributed tracing
- Implement proper logging at appropriate levels
- Monitor service health and performance
- Use Aspire dashboard for development monitoring

## Common Pitfalls to Avoid

1. **Don't** create synchronous calls in async contexts
2. **Don't** expose internal implementation details in DTOs
3. **Don't** hardcode configuration values - use service discovery environment variables
4. **Don't** forget Entity Framework migrations for BOTH Admin and Orchestrator databases
5. **Don't** miss calling `builder.AddServiceDefaults()` in new services - required for telemetry and service discovery
6. **Don't** forget `.WithReference()` dependencies in AppHost - service endpoints won't resolve without them
7. **Don't** use Blazor Server mode for authenticated API calls via YARP - cookies aren't forwarded (use WebAssembly)

## Build and Verification

### Testing the Application
```bash
# Run all tests from solution root
dotnet test

# Run tests for specific project
dotnet test tests/NTG.Agent.Orchestrator.Tests
```

### Accessing Services
- **Aspire Dashboard**: Auto-opens when running AppHost - shows all service URLs, logs, traces, metrics
- **Admin Portal**: Default credentials `admin@ntgagent.com` / `Ntg@123`
- **OpenTelemetry**: Endpoint configured via `OTEL_EXPORTER_OTLP_ENDPOINT` (defaults to Aspire Dashboard)

When contributing to this project, follow these established patterns and conventions to maintain consistency and quality across the codebase.