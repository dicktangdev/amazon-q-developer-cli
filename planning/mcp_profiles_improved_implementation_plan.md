# MCP Profiles - Improved Implementation Plan

This document outlines a comprehensive implementation plan for adding profile-based MCP configurations to Amazon Q CLI, incorporating the latest information about the MCP server configuration structure.

## Current System Understanding

### Profile System
- Profiles are managed through `ContextManager` in `crates/chat-cli/src/cli/chat/context.rs`
- Each profile has its own directory in the profiles directory (`~/.config/amazon-q/profiles/` on Linux, `~/Library/Application Support/com.amazon.q/profiles/` on macOS)
- Each profile has a `context.json` file that stores its configuration
- Users can create, delete, rename, and switch between profiles using `/profile` commands

### MCP Server Configuration
- MCP servers are configured in `McpServerConfig` in `crates/chat-cli/src/cli/chat/tool_manager.rs`
- Configuration is stored in a JSON file called `mcp.json` under the `.amazonq` folder
- The configuration structure has `mcpServers` as the top-level key (not `mcp_servers` as in the Rust structure)
- Each server configuration includes fields like `command`, `args`, `env`, `autoApprove`, and `disabled`
- MCP servers are loaded when QCLI starts
- No profile-based management exists yet

Example of the current MCP configuration structure:
```json
{
  "mcpServers": {
    "awslabs.aws-diagram-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.aws-diagram-mcp-server"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "autoApprove": [],
      "disabled": false
    }
  }
}
```

## Implementation Plan

### Phase 1: Core Infrastructure (Week 1)

#### Task 1.1: Update ContextManager to Include MCP Configurations
- **Description**: Extend the `ContextManager` to include MCP server configurations
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/context.rs`
- **Implementation Details**:
  - Add `profile_mcp_config` and `global_mcp_config` fields to `ContextManager`
  - Add functions to load and save MCP configurations for profiles
  - Update the profile switching logic to also switch MCP configurations
  - Ensure the configuration structure matches the expected JSON format with `mcpServers` as the top-level key
- **Expected Result**:
  - `ContextManager` can store and manage MCP configurations for each profile
  - MCP configurations are saved in each profile's directory as `mcp.json`
- **Testing**:
  ```bash
  # Build the project
  cargo build
  
  # Create a test profile
  ./target/debug/q chat
  > /profile create test-profile
  
  # Verify the profile directory structure
  ls -la ~/.config/amazon-q/profiles/test-profile/
  # Should show context.json (and eventually mcp.json)
  ```

#### Task 1.2: Implement Migration Strategy
- **Description**: Create a migration strategy to move existing MCP configurations to the default profile
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/context.rs`
  - `crates/chat-cli/src/database/settings.rs`
- **Implementation Details**:
  - Add a migration function to `ContextManager` that:
    1. Checks if migration has already been done using `Setting::McpLoadedBefore`
    2. Reads existing MCP configurations from workspace and global locations
    3. Moves them to the default profile and global profile configurations
    4. Sets `Setting::McpLoadedBefore` to true to prevent re-migration
  - Ensure the migration preserves the correct JSON structure with `mcpServers` as the top-level key
- **Expected Result**:
  - Existing MCP configurations are migrated to the default profile
  - Migration runs only once
- **Testing**:
  ```bash
  # First, set up some MCP servers in the old location
  # Create a mcp.json file with a test server
  
  # Then run the migration (will be triggered on first use after implementation)
  ./target/debug/q chat
  
  # Verify the migration worked
  cat ~/.config/amazon-q/profiles/default/mcp.json
  # Should show the migrated configuration with mcpServers as the top-level key
  ```

#### Task 1.3: Update MCP Commands to Work with Profiles
- **Description**: Modify MCP commands to work with profile-based configurations
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/mcp.rs`
  - `crates/chat-cli/src/cli/chat/command.rs`
- **Implementation Details**:
  - Update MCP commands to use `ContextManager` for configuration management
  - Add `--global` flag to commands to specify whether to modify global or profile-specific configurations
  - Update the `/mcp list` command to show both enabled and disabled servers
- **Expected Result**:
  - MCP commands modify the correct profile's configuration
  - Global flag works correctly
  - `/mcp list` shows all servers with their enabled/disabled status
- **Testing**:
  ```bash
  # Add an MCP server to the current profile
  ./target/debug/q chat
  > /mcp add --name profile-server
  
  # Add an MCP server to the global configuration
  > /mcp add --name global-server --global
  
  # Disable a server
  > /mcp disable profile-server
  
  # List MCP servers in the current profile
  > /mcp list
  # Should show profile-server (disabled) and global-server (enabled)
  
  # List global MCP servers
  > /mcp list --global
  # Should show global-server
  ```

### Phase 2: Enhanced Features (Week 2)

#### Task 2.1: Implement Profile Switching Integration
- **Description**: Update profile switching to handle MCP configurations
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/context.rs`
  - `crates/chat-cli/src/cli/chat/mod.rs`
- **Implementation Details**:
  - Update `switch_profile` method to:
    1. Save the current MCP configuration to the previous profile
    2. Load the MCP configuration from the new profile
  - Update chat initialization to reload MCP servers when profile changes
- **Expected Result**:
  - MCP configurations switch when profiles switch
  - MCP servers are properly reloaded
- **Testing**:
  ```bash
  # Create two profiles with different MCP servers
  ./target/debug/q chat
  > /profile create profile1
  > /mcp add --name server1
  > /profile create profile2
  > /profile set profile2
  > /mcp add --name server2
  
  # Switch between profiles and verify MCP servers
  > /profile set profile1
  > /mcp list
  # Should show server1
  
  > /profile set profile2
  > /mcp list
  # Should show server2
  ```

#### Task 2.2: Implement Enable/Disable Functionality
- **Description**: Add commands to enable/disable MCP servers within a profile
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/mcp.rs`
  - `crates/chat-cli/src/cli/chat/command.rs`
  - `crates/chat-cli/src/cli/chat/command_handler.rs`
- **Implementation Details**:
  - Add `enable` and `disable` subcommands to the MCP command
  - Update the server configuration to set the `disabled` field
  - Implement handlers for the new commands
- **Expected Result**:
  - Users can enable/disable MCP servers within a profile
  - Disabled servers are not connected to
- **Testing**:
  ```bash
  # Add an MCP server
  ./target/debug/q chat
  > /mcp add --name test-server
  
  # Disable the server
  > /mcp disable test-server
  
  # Verify it's disabled
  > /mcp list
  # Should show test-server as disabled
  
  # Enable the server
  > /mcp enable test-server
  
  # Verify it's enabled
  > /mcp list
  # Should show test-server as enabled
  ```

#### Task 2.3: Enhance Show Command
- **Description**: Implement the show command to display currently enabled MCP servers
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/mcp.rs`
  - `crates/chat-cli/src/cli/chat/command.rs`
  - `crates/chat-cli/src/cli/chat/command_handler.rs`
- **Implementation Details**:
  - Add `show` subcommand to the MCP command
  - Implement handler that displays all enabled MCP servers across global and profile configurations
  - Include clear visual indicators for enabled vs. disabled servers
- **Expected Result**:
  - Show command displays all MCP servers with their status
  - Output is formatted clearly with enabled/disabled indicators
- **Testing**:
  ```bash
  # Add some MCP servers
  ./target/debug/q chat
  > /mcp add --name profile-server
  > /mcp add --name global-server --global
  
  # Disable one server
  > /mcp disable profile-server
  
  # Show all servers
  > /mcp show
  # Should show profile-server (disabled) and global-server (enabled)
  ```

### Phase 3: Testing and Refinement (Week 3)

#### Task 3.1: Unit Testing
- **Description**: Write unit tests for all new functionality
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/context.rs` (add tests)
  - `crates/chat-cli/src/cli/chat/mcp.rs` (add tests)
- **Implementation Details**:
  - Add tests for MCP configuration serialization/deserialization
  - Add tests for profile switching with MCP configurations
  - Add tests for enable/disable functionality
  - Ensure tests verify the correct JSON structure with `mcpServers` as the top-level key
- **Expected Result**:
  - All new code has unit test coverage
  - Tests verify correct behavior
- **Testing**:
  ```bash
  # Run unit tests
  cargo test --package chat-cli
  ```

#### Task 3.2: Integration Testing
- **Description**: Write integration tests for the feature
- **Files to modify**:
  - `tests/integration/mcp_profiles_test.rs` (new file)
- **Implementation Details**:
  - Create integration tests that verify end-to-end functionality
  - Test profile switching with MCP configurations
  - Test migration of existing MCP configurations
  - Test the `/mcp list` command shows both enabled and disabled servers
- **Expected Result**:
  - Integration tests verify end-to-end functionality
  - Tests cover all user workflows
- **Testing**:
  ```bash
  # Run integration tests
  cargo test --test mcp_profiles_test
  ```

#### Task 3.3: Documentation and Refinement
- **Description**: Update documentation and refine the implementation
- **Files to modify**:
  - `README.md`
  - Various code files for comments and refinements
- **Implementation Details**:
  - Update documentation to reflect new functionality
  - Add comments to code
  - Refine implementation based on testing feedback
- **Expected Result**:
  - Documentation is updated
  - Code is clean and well-commented
- **Testing**:
  ```bash
  # Verify documentation is accurate
  # Test commands from documentation
  ```

## Detailed Implementation

### Task 1.1: Update ContextManager to Include MCP Configurations

```rust
// Add to crates/chat-cli/src/cli/chat/context.rs

// Add new fields to ContextManager
pub struct ContextManager {
    // Existing fields...
    
    /// MCP server configurations for the current profile
    pub profile_mcp_config: McpServerConfig,
    
    /// Global MCP server configurations (available across all profiles)
    pub global_mcp_config: McpServerConfig,
}

// Update ContextManager::new
pub async fn new(ctx: Arc<Context>, max_context_files_size: Option<usize>) -> Result<Self> {
    // Existing code...
    
    let global_mcp_config = load_global_mcp_config(&ctx).await?;
    let profile_mcp_config = load_profile_mcp_config(&ctx, &current_profile).await?;
    
    Ok(Self {
        // Existing fields...
        global_mcp_config,
        profile_mcp_config,
        // Other fields...
    })
}

// Add new functions
/// Path to the MCP config file for a profile
pub fn profile_mcp_path(ctx: &Context, profile_name: &str) -> Result<PathBuf> {
    Ok(directories::chat_profiles_dir(ctx)?
        .join(profile_name)
        .join("mcp.json"))
}

async fn load_global_mcp_config(ctx: &Context) -> Result<McpServerConfig> {
    let global_path = global_mcp_config_path(ctx)?;
    if ctx.fs().exists(&global_path) {
        let contents = ctx.fs().read_to_string(&global_path).await?;
        let config: McpServerConfig = serde_json::from_str(&contents)
            .map_err(|e| eyre!("Failed to parse global MCP configuration: {}", e))?;
        Ok(config)
    } else {
        Ok(McpServerConfig::default())
    }
}

async fn load_profile_mcp_config(ctx: &Context, profile_name: &str) -> Result<McpServerConfig> {
    let profile_path = profile_mcp_path(ctx, profile_name)?;
    if ctx.fs().exists(&profile_path) {
        let contents = ctx.fs().read_to_string(&profile_path).await?;
        let config: McpServerConfig = serde_json::from_str(&contents)
            .map_err(|e| eyre!("Failed to parse profile MCP configuration: {}", e))?;
        Ok(config)
    } else {
        Ok(McpServerConfig::default())
    }
}

// Update save_config to also save MCP configurations
async fn save_config(&self, global: bool) -> Result<()> {
    // Existing code for saving context configs...
    
    // Save MCP configs
    if global {
        let global_mcp_path = global_mcp_config_path(&self.ctx)?;
        let contents = serde_json::to_string_pretty(&self.global_mcp_config)
            .map_err(|e| eyre!("Failed to serialize global MCP configuration: {}", e))?;
        self.ctx.fs().write(&global_mcp_path, contents).await?;
    } else {
        let profile_mcp_path = profile_mcp_path(&self.ctx, &self.current_profile)?;
        if let Some(parent) = profile_mcp_path.parent() {
            self.ctx.fs().create_dir_all(parent).await?;
        }
        let contents = serde_json::to_string_pretty(&self.profile_mcp_config)
            .map_err(|e| eyre!("Failed to serialize profile MCP configuration: {}", e))?;
        self.ctx.fs().write(&profile_mcp_path, contents).await?;
    }
    
    Ok(())
}

// Update switch_profile to also switch MCP configurations
pub async fn switch_profile(&mut self, name: &str) -> Result<()> {
    // Existing validation code...
    
    // Save current profile MCP config
    self.save_config(false).await?;
    
    // Existing code to update current_profile...
    
    // Load new profile MCP config
    self.profile_mcp_config = load_profile_mcp_config(&self.ctx, name).await?;
    
    // Existing code...
    
    Ok(())
}
```

### Task 1.2: Implement Migration Strategy

```rust
// Add to crates/chat-cli/src/cli/chat/context.rs

/// Migrate existing MCP configurations to the profile system
pub async fn migrate_mcp_configs(&mut self) -> Result<()> {
    // Check if migration has already been done
    let settings = Database::new()?;
    if settings.get_bool(Setting::McpLoadedBefore).unwrap_or(false) {
        return Ok(());
    }
    
    // Load existing MCP configurations
    let workspace_path = workspace_mcp_config_path(&self.ctx)?;
    let global_path = global_mcp_config_path(&self.ctx)?;
    
    // Migrate workspace config to profile config
    if self.ctx.fs().exists(&workspace_path) {
        let contents = self.ctx.fs().read_to_string(&workspace_path).await?;
        let workspace_config: McpServerConfig = serde_json::from_str(&contents)
            .map_err(|e| eyre!("Failed to parse workspace MCP configuration: {}", e))?;
        
        // Merge with profile config
        for (name, server) in workspace_config.mcp_servers {
            self.profile_mcp_config.mcp_servers.insert(name, server);
        }
    }
    
    // Migrate global config to global profile config
    if self.ctx.fs().exists(&global_path) {
        let contents = self.ctx.fs().read_to_string(&global_path).await?;
        let global_config: McpServerConfig = serde_json::from_str(&contents)
            .map_err(|e| eyre!("Failed to parse global MCP configuration: {}", e))?;
        
        // Merge with global config
        for (name, server) in global_config.mcp_servers {
            self.global_mcp_config.mcp_servers.insert(name, server);
        }
    }
    
    // Save the migrated configurations
    self.save_config(true).await?;
    self.save_config(false).await?;
    
    // Mark migration as done
    settings.set_bool(Setting::McpLoadedBefore, true)?;
    
    Ok(())
}
```

### Task 1.3: Update MCP Commands to Work with Profiles

```rust
// Update crates/chat-cli/src/cli/chat/mcp.rs

// Update list_mcp_server to show both enabled and disabled servers
pub async fn list_mcp_server(ctx: &Context, output: &mut SharedWriter, args: McpList) -> Result<()> {
    let mut context_manager = ContextManager::new(Arc::new(ctx.clone()), None).await?;
    
    // Migrate existing configs if needed
    context_manager.migrate_mcp_configs().await?;
    
    let global = args.scope == Some(Scope::Global);
    let config = if global {
        &context_manager.global_mcp_config
    } else {
        &context_manager.profile_mcp_config
    };
    
    if config.mcp_servers.is_empty() {
        writeln!(output, "No MCP server configurations found.\n")?;
        return Ok(());
    }
    
    writeln!(output, "\n{} MCP servers:\n", if global { "Global" } else { "Profile" })?;
    
    for (name, server) in &config.mcp_servers {
        let status = if server.disabled {
            style::SetForegroundColor(style::Color::Red).to_string() + "disabled" + &style::ResetColor.to_string()
        } else {
            style::SetForegroundColor(style::Color::Green).to_string() + "enabled" + &style::ResetColor.to_string()
        };
        
        writeln!(output, "  • {:<20} [{}]", name, status)?;
    }
    
    writeln!(output, "\n")?;
    Ok(())
}
```

### Task 2.2: Implement Enable/Disable Functionality

```rust
// Add to crates/chat-cli/src/cli/chat/mcp.rs

pub async fn enable_mcp_server(ctx: &Context, output: &mut SharedWriter, name: String, global: bool) -> Result<()> {
    let mut context_manager = ContextManager::new(Arc::new(ctx.clone()), None).await?;
    
    // Migrate existing configs if needed
    context_manager.migrate_mcp_configs().await?;
    
    let config = if global {
        &mut context_manager.global_mcp_config
    } else {
        &mut context_manager.profile_mcp_config
    };
    
    // Check if server exists
    if let Some(server) = config.mcp_servers.get_mut(&name) {
        server.disabled = false;
        context_manager.save_config(global).await?;
        writeln!(output, "Enabled MCP server: {}", name)?;
        return Ok(());
    }
    
    Err(eyre!("MCP server '{}' not found", name))
}

pub async fn disable_mcp_server(ctx: &Context, output: &mut SharedWriter, name: String, global: bool) -> Result<()> {
    let mut context_manager = ContextManager::new(Arc::new(ctx.clone()), None).await?;
    
    // Migrate existing configs if needed
    context_manager.migrate_mcp_configs().await?;
    
    let config = if global {
        &mut context_manager.global_mcp_config
    } else {
        &mut context_manager.profile_mcp_config
    };
    
    // Check if server exists
    if let Some(server) = config.mcp_servers.get_mut(&name) {
        server.disabled = true;
        context_manager.save_config(global).await?;
        writeln!(output, "Disabled MCP server: {}", name)?;
        return Ok(());
    }
    
    Err(eyre!("MCP server '{}' not found", name))
}
```

## End-to-End Testing Procedure

To fully test the implementation, follow these steps:

1. **Setup Test Environment**:
   ```bash
   # Build the project
   cargo build
   
   # Create test profiles
   ./target/debug/q chat
   > /profile create dev
   > /profile create prod
   ```

2. **Test MCP Server Management**:
   ```bash
   # Add MCP servers to different profiles
   > /profile set dev
   > /mcp add --name dev-server
   > /profile set prod
   > /mcp add --name prod-server
   > /mcp add --name global-server --global
   
   # Verify servers are profile-specific
   > /profile set dev
   > /mcp list
   # Should show dev-server
   
   > /profile set prod
   > /mcp list
   # Should show prod-server
   
   > /mcp list --global
   # Should show global-server in both profiles
   ```

3. **Test Enable/Disable Functionality**:
   ```bash
   > /profile set dev
   > /mcp disable dev-server
   > /mcp list
   # Should show dev-server as disabled
   
   > /mcp enable dev-server
   > /mcp list
   # Should show dev-server as enabled
   ```

4. **Test Migration**:
   ```bash
   # First, set up MCP servers in the old location
   # Create a mcp.json file with a test server
   
   # Delete any existing profile MCP configurations
   rm -f ~/.config/amazon-q/profiles/default/mcp.json
   
   # Reset the migration flag
   ./target/debug/q settings mcp.loadedBefore false
   
   # Run the migration
   ./target/debug/q chat
   
   # Verify the migration worked
   > /mcp list
   # Should show the migrated servers
   ```

## Expected Results

After implementing this feature, users should be able to:

1. Maintain different sets of MCP servers for different profiles
2. Switch between profiles and have the appropriate MCP servers loaded
3. Enable/disable MCP servers within a profile
4. View all MCP servers (both enabled and disabled) with the `/mcp list` command
5. Manage global MCP servers that are available across all profiles

The implementation should be backward compatible, with existing MCP configurations automatically migrated to the new profile-based system.

## Conclusion

This implementation plan provides a comprehensive approach to adding profile-based MCP configurations to the Amazon Q Developer CLI. By extending the existing profile system, we can provide users with a seamless way to manage different sets of MCP servers for different projects or environments.
