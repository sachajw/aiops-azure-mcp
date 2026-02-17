# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Microsoft MCP (Model Context Protocol) servers provide AI agents with structured access to Azure, Microsoft Fabric, and other Microsoft services. This repository contains:
- **Azure MCP Server** (`servers/Azure.Mcp.Server/`) - 100+ tools across 42+ Azure services
- **Microsoft Fabric MCP Server** (`servers/Fabric.Mcp.Server/`) - Fabric workspace and data platform operations
- **Core libraries** (`core/`) - Shared infrastructure for command patterns, authentication, and MCP protocol
- **Toolsets** (`tools/Azure.Mcp.Tools.{Service}/`) - Individual Azure service implementations

## Build Commands

```powershell
# Basic build
dotnet build

# Single project build (preferred for faster feedback)
dotnet build tools/Azure.Mcp.Tools.Storage/src

# Full verification before PR
./eng/scripts/Build-Local.ps1 -UsePaths -VerifyNpx

# AOT compatibility check
./eng/scripts/Build-Local.ps1 -BuildNative
```

## Testing

```powershell
# Run unit tests for specific toolset
./eng/scripts/Test-Code.ps1 -Paths Storage

# Run specific test class
dotnet test --filter "FullyQualifiedName~StorageAccountListCommandTests"

# Run all unit tests
./eng/scripts/Test-Code.ps1

# Live tests (requires Azure authentication)
./eng/scripts/Deploy-TestResources.ps1 -Paths Storage
./eng/scripts/Test-Code.ps1 -TestType Live -Paths Storage
```

## Code Quality

```powershell
# Format code
dotnet format

# Format specific files
dotnet format --include="tools/Azure.Mcp.Tools.Storage/**/*.cs"

# Spelling check
.\eng\common\spelling\Invoke-Cspell.ps1
```

## Key Architecture Patterns

### Toolset Structure
```
tools/Azure.Mcp.Tools.{Service}/
├── src/
│   ├── Commands/{Resource}/{Resource}{Operation}Command.cs
│   ├── Services/I{Service}Service.cs
│   ├── Options/{Service}OptionDefinitions.cs
│   └── {Service}Setup.cs              # DI registration
└── tests/
    ├── Azure.Mcp.Tools.{Service}.UnitTests/
    ├── Azure.Mcp.Tools.{Service}.LiveTests/
    ├── test-resources.bicep           # Test infrastructure
    └── test-resources-post.ps1
```

### Command Naming
Commands follow `azmcp <service> <resource> <operation>` pattern:
```bash
azmcp storage account get
azmcp sql database show
azmcp keyvault secret get
```

## Critical Coding Standards

- **Use primary constructors** for all C# classes with dependency injection
- **Use `System.Text.Json`** - never Newtonsoft.Json
- **Make command classes sealed** unless designed for inheritance
- **Make members static** when possible for AOT compatibility
- **Separate files** for each class and interface
- **AOT-safe code** - all response models must be registered in JsonSerializerContext

### Parameter Naming
- Use `subscription` (never `subscriptionId`)
- Use `resourceGroup` (not `resourceGroupName`)
- Use singular nouns (`--account` not `--account-name`)

### Option Handling Pattern
```csharp
protected override void RegisterOptions(Command command)
{
    base.RegisterOptions(command);
    command.Options.Add(OptionDefinitions.Common.ResourceGroup.AsRequired());
}

protected override OptionsType BindOptions(ParseResult parseResult)
{
    var options = base.BindOptions(parseResult);
    options.ResourceGroup ??= parseResult.GetValueOrDefault<string>(OptionDefinitions.Common.ResourceGroup.Name);
    return options;
}
```

## Adding New Commands

1. Create command: `tools/Azure.Mcp.Tools.{Service}/src/Commands/{Resource}/{Resource}{Operation}Command.cs`
2. Create service method if needed
3. Register in `{Service}Setup.cs`
4. Add unit tests following existing patterns
5. Create live test infrastructure (`test-resources.bicep`, `test-resources-post.ps1`)
6. Submit one tool per PR

See `servers/Azure.Mcp.Server/docs/new-command.md` for detailed step-by-step guide.

## Server Modes

- **Default (Namespace)**: `azmcp server start` - Collapses tools by namespace
- **Consolidated**: `--mode consolidated` - Groups related operations
- **Namespace filter**: `--namespace storage --namespace keyvault`
- **Single tool**: `--mode single` - Single "azure" tool with internal routing
- **All tools**: `--mode all` - Expose all 800+ individual tools

## Documentation

- `AGENTS.md` - Comprehensive AI agent instructions (authoritative reference)
- `servers/Azure.Mcp.Server/docs/new-command.md` - Step-by-step command creation
- `servers/Azure.Mcp.Server/docs/azmcp-commands.md` - Complete command reference
- `CONTRIBUTING.md` - Contribution guidelines and workflows
- `.github/copilot-instructions.md` - GitHub Copilot coding instructions
