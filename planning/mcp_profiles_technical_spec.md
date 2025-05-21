# MCP Profiles - Technical Specification

## Overview

This document outlines the technical implementation for adding profile-based MCP (Model Context Protocol) configurations to Amazon Q CLI, integrating with the existing profile system.

## Requirements Summary

Based on the Q&A document, the key requirements are:

1. Integrate MCP server configurations with the existing profile system
2. Allow users to add/remove MCP servers to specific profiles
3. Automatically switch MCP configurations when switching profiles
4. Migrate existing MCP servers to the default profile
5. Enable/disable MCP servers within a profile
6. Show currently enabled MCP servers when profile changes

## System Architecture

### Data Model

We'll extend the existing profile system to include MCP server configurations. The data structure will be:

```json
{
  "profiles": {
    "default": {
      "context_files": [...],
      "mcp_servers": [
        {
          "url": "http://localhost:8080",
          "name": "local-server",
          "enabled": true
        }
      ]
    },
    "dev": {
      "context_files": [...],
      "mcp_servers": [
        {
          "url": "http://dev-server:8080",
          "name": "dev-server",
          "enabled": true
        }
      ]
    }
  },
  "active_profile": "default",
  "global_context_files": [...]
}
```

### File Structure

The MCP profile configurations will be stored in the same location as the existing profiles:
- macOS: `~/Library/Application Support/com.amazon.q/profiles.json`
- Linux: `~/.config/amazon-q/profiles.json`

### Command Structure

We'll implement the following commands:

1. `/mcp add <url>` - Add an MCP server to the current profile
2. `/mcp remove <url>` - Remove an MCP server from the current profile
3. `/mcp list` - List all MCP servers in the current profile
4. `/mcp enable <url>` - Enable an MCP server in the current profile
5. `/mcp disable <url>` - Disable an MCP server in the current profile
6. `/mcp show` - Show currently enabled MCP servers

Profile switching will continue to use the existing `/profile set <name>` command, which will now also switch MCP configurations.

## Implementation Plan

### 1. Update Profile Data Structure

Modify the profile data structure to include MCP server configurations:

```rust
#[derive(Serialize, Deserialize)]
struct McpServer {
    url: String,
    name: String,
    enabled: bool,
}

#[derive(Serialize, Deserialize)]
struct Profile {
    context_files: Vec<String>,
    mcp_servers: Vec<McpServer>,
}

#[derive(Serialize, Deserialize)]
struct ProfileConfig {
    profiles: HashMap<String, Profile>,
    active_profile: String,
    global_context_files: Vec<String>,
}
```

### 2. Migration Strategy

When the system starts up for the first time after the update:
1. Check if the profile configuration already includes MCP servers
2. If not, read the existing MCP configuration from `mcp.json`
3. Add all existing MCP servers to the default profile
4. Save the updated profile configuration

### 3. MCP Command Implementation

Implement the MCP commands in `crates/chat-cli/src/cli/chat/command.rs`:

```rust
enum Command {
    // Existing commands...
    Mcp(McpCommand),
}

enum McpCommand {
    Add(String),
    Remove(String),
    List,
    Enable(String),
    Disable(String),
    Show,
}
```

### 4. Profile Switching Integration

Update the profile switching logic in `crates/chat-cli/src/cli/chat/profile.rs` to:
1. Save the current MCP configuration to the previous profile
2. Load the MCP configuration from the new profile
3. Disconnect from inactive MCP servers
4. Connect to newly active MCP servers
5. Notify the user of the change

### 5. MCP Client Updates

Modify the MCP client in `crates/chat-cli/src/mcp_client/client.rs` to:
1. Accept a list of MCP servers from the profile
2. Connect to all enabled servers
3. Provide methods to enable/disable servers

## Testing Strategy

1. **Unit Tests**:
   - Test profile data structure serialization/deserialization
   - Test MCP command parsing
   - Test profile switching logic

2. **Integration Tests**:
   - Test adding/removing MCP servers to profiles
   - Test switching between profiles with different MCP configurations
   - Test enabling/disabling MCP servers

3. **Migration Tests**:
   - Test migration of existing MCP configurations to the default profile

## Implementation Phases

### Phase 1: Core Infrastructure
- Update profile data structure
- Implement migration strategy
- Add basic MCP commands (add, remove, list)

### Phase 2: Enhanced Features
- Implement enable/disable functionality
- Add show command
- Improve user notifications

### Phase 3: Testing and Refinement
- Comprehensive testing
- Performance optimization
- Documentation updates

## Risks and Mitigations

1. **Risk**: Breaking existing MCP functionality
   **Mitigation**: Thorough testing and a fallback mechanism to use the old configuration

2. **Risk**: Performance issues when switching profiles with many MCP servers
   **Mitigation**: Optimize connection/disconnection logic and implement lazy loading

3. **Risk**: User confusion with the new command structure
   **Mitigation**: Clear documentation and helpful error messages

## Dependencies

- Existing profile system in `crates/chat-cli/src/cli/chat/profile.rs`
- MCP client implementation in `crates/chat-cli/src/mcp_client/`
- Command parsing in `crates/chat-cli/src/cli/chat/command.rs`
