# MCP Profiles - Implementation Plan

## Overview

This document outlines the step-by-step implementation plan for adding profile-based MCP configurations to Amazon Q CLI.

## Phase 1: Core Infrastructure (Week 1)

### Task 1.1: Update Profile Data Structure
- **Description**: Extend the existing profile data structure to include MCP server configurations
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/profile.rs`
- **Estimated time**: 1 day
- **Acceptance criteria**:
  - Profile data structure includes MCP server configurations
  - Serialization/deserialization works correctly

### Task 1.2: Implement Migration Strategy
- **Description**: Create a migration strategy to move existing MCP configurations to the default profile
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/profile.rs`
  - `crates/chat-cli/src/mcp_client/client.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - Existing MCP configurations are migrated to the default profile
  - Migration runs only once
  - Migration preserves all existing MCP server settings

### Task 1.3: Add Basic MCP Commands
- **Description**: Implement basic MCP commands (add, remove, list)
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/command.rs`
  - `crates/chat-cli/src/cli/chat/command_handler.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - Commands are properly parsed
  - Commands modify the profile configuration correctly
  - Commands provide appropriate feedback to the user

## Phase 2: Enhanced Features (Week 2)

### Task 2.1: Implement Profile Switching Integration
- **Description**: Update profile switching to handle MCP configurations
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/profile.rs`
  - `crates/chat-cli/src/mcp_client/client.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - MCP configurations switch when profiles switch
  - Connections to MCP servers are properly managed
  - User is notified of MCP configuration changes

### Task 2.2: Implement Enable/Disable Functionality
- **Description**: Add commands to enable/disable MCP servers within a profile
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/command.rs`
  - `crates/chat-cli/src/cli/chat/command_handler.rs`
  - `crates/chat-cli/src/mcp_client/client.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - Enable/disable commands work correctly
  - Disabled servers are not connected to
  - Server state is persisted in the profile configuration

### Task 2.3: Add Show Command
- **Description**: Implement the show command to display currently enabled MCP servers
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/command.rs`
  - `crates/chat-cli/src/cli/chat/command_handler.rs`
- **Estimated time**: 1 day
- **Acceptance criteria**:
  - Show command displays all enabled MCP servers
  - Output is formatted clearly
  - Command works across all profiles

## Phase 3: Testing and Refinement (Week 3)

### Task 3.1: Unit Testing
- **Description**: Write unit tests for all new functionality
- **Files to modify**:
  - `crates/chat-cli/src/cli/chat/profile_test.rs`
  - `crates/chat-cli/src/cli/chat/command_test.rs`
  - `crates/chat-cli/src/mcp_client/client_test.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - All new code has unit test coverage
  - Tests verify correct behavior
  - Edge cases are covered

### Task 3.2: Integration Testing
- **Description**: Write integration tests for the feature
- **Files to modify**:
  - `tests/integration/mcp_profiles_test.rs`
- **Estimated time**: 2 days
- **Acceptance criteria**:
  - Tests verify end-to-end functionality
  - Tests cover all user workflows
  - Tests verify migration works correctly

### Task 3.3: Documentation and Refinement
- **Description**: Update documentation and refine the implementation
- **Files to modify**:
  - `README.md`
  - Various code files for comments and refinements
- **Estimated time**: 1 day
- **Acceptance criteria**:
  - Documentation is updated to reflect new functionality
  - Code is clean and follows project standards
  - All review feedback is addressed

## Timeline

- **Week 1**: Core Infrastructure (Tasks 1.1-1.3)
- **Week 2**: Enhanced Features (Tasks 2.1-2.3)
- **Week 3**: Testing and Refinement (Tasks 3.1-3.3)

## Resources Required

- 1 Developer (full-time)
- 1 Reviewer (part-time)
- 1 Tester (part-time)

## Risks and Contingencies

| Risk | Impact | Likelihood | Contingency |
|------|--------|------------|-------------|
| Breaking existing MCP functionality | High | Medium | Implement feature behind a feature flag that can be disabled |
| Performance issues with many MCP servers | Medium | Low | Optimize connection handling and implement lazy loading |
| Migration issues | High | Medium | Add rollback capability and detailed logging |

## Success Metrics

- All tests pass
- No regression in existing functionality
- User feedback indicates the feature is intuitive and useful
- Performance impact is minimal when switching profiles
