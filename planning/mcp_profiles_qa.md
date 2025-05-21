# MCP Profiles Feature - Requirements Q&A

## Feature Overview
This document contains questions to clarify the requirements for adding profile-based MCP configurations to Amazon Q CLI, integrating with the existing profile system.

## Questions

### 1. Current Profile System
Q: How does the current profile system work in Amazon Q CLI? What commands are available and what information is stored in profiles?

A: Below is profile and context command intro - 
/profile help


(Beta) Profile Management

Profiles allow you to organize and manage different sets of context files for different projects or tasks.

Available commands
  help                Show an explanation for the profile command
  list                List all available profiles
  create <n>       Create a new profile with the specified name
  delete <n>       Delete the specified profile
  set <n>          Switch to the specified profile
  rename <old> <new>  Rename a profile

Notes
• The "global" profile contains context files that are available in all profiles
• The "default" profile is used when no profile is specified
• You can switch between profiles to work on different projects
• Each profile maintains its own set of context files

Using the /profile command
The /profile command allows you to view and switch between different context profiles in the Amazon Q Developer CLI.

When you run the /profile command without arguments, it displays a list of available profiles:


q chat
> /profile
Available profiles:
* default
  dev
  prod
  staging
The asterisk (*) indicates the currently active profile.

To switch to a different profile, specify the profile name:

q chat
> /profile set dev
Switched to profile: dev
Managing context

Context files are markdown files that contain information you want Amazon Q to consider during your conversations. These can include project requirements, coding standards, development rules, or any other information that helps Amazon Q provide more relevant responses.

Adding context
You can add files or directories to your context using the /context add command:


q chat
> /context add README.md
Added 1 path(s) to profile context.
To add a file to the global context (available across all profiles), use the --global flag:


q chat
> /context add --global coding-standards.md
Added 1 path(s) to global context.
You can also add multiple files at once using glob patterns:


q chat
> /context add docs/*.md
Added 3 path(s) to profile context.

Viewing context
To view your current context, use the /context show command:


q chat
> /context show
Global context:
  /home/user/coding-standards.md

Profile context (terraform):
  /home/user/terraform-project/README.md
  /home/user/terraform-project/docs/architecture.md
  /home/user/terraform-project/docs/best-practices.md
Removing context
To remove files from your context, use the /context rm command:


q chat
> /context rm docs/architecture.md
Removed 1 path(s) from profile context.
To remove files from the global context, use the --global flag:


q chat
> /context rm --global coding-standards.md
Removed 1 path(s) from global context.
To clear all files from your context, use the /context clear command:


q chat
> /context clear
Cleared all paths from profile context.
To clear the global context, use the --global flag:


q chat
> /context clear --global
Cleared all paths from global context.

### 2. MCP Configuration Integration
Q: Should MCP server configurations be added as a new section within existing profiles, or should they be a separate configuration that references profiles?

A: It should be managed like context files with profile, e.g. /mcp add xxx is to add a mcp server to the profile. But mcp servers is only managed by mcp.json file under amazonq folder now. I do not know how to manage multiple

### 3. Command Structure
Q: What should the command structure be for managing MCP configurations within profiles? Should it be `/mcp profile <action>` or something else?

A: /mcp set xxx to change current profile, and all mcp server binding to this profile will be used

### 4. Profile Switching
Q: When a user switches profiles, should the MCP server configurations automatically switch as well?

A: yes

### 5. Default Behavior
Q: What should happen to existing MCP server configurations when this feature is implemented? Should they be migrated to a default profile?

A: Default should have all mcp servers

### 6. Configuration Storage
Q: Where should the MCP profile configurations be stored? In the same location as existing profiles or elsewhere?

A: yes

### 7. Import/Export
Q: Should users be able to import/export MCP configurations separately from profiles, or should they be included in profile import/export?

A: mcp is loaded only while starting the mcp server

### 8. Scope
Q: Besides server URLs, what other MCP-related settings should be included in profile-based configurations?

A: MCP server setting should not be included in command as well, it should be managed by files like mcp.json as default

### 9. Validation
Q: What validation should be performed when adding MCP servers to a profile?

A: no need

### 10. User Experience
Q: How should the user be notified when MCP configurations change due to profile switching?

A: show current enabled MCP server

### 11. Conflict Handling
Q: How should conflicts be handled if an MCP server configuration in one profile has the same name but different settings as in another profile?

A: Each profile should maintain its own separate list of MCP servers. If a server with the same URL exists in multiple profiles, each profile should maintain its own configuration for that server.

### 12. Enable/Disable Servers
Q: Should there be a way to enable/disable specific MCP servers within a profile without removing them completely?

A: Yes, users should be able to enable/disable MCP servers within a profile using a command like `/mcp enable <server-url>` or `/mcp disable <server-url>`. This allows users to temporarily disable servers without losing their configuration.

### 13. Performance Considerations
Q: Are there any performance considerations we should keep in mind when switching between profiles with different MCP configurations?

A: When switching profiles, the system should efficiently disconnect from inactive MCP servers and connect to newly active ones. There should be minimal delay during profile switching, and the system should handle connection failures gracefully.

### 14. Server Limits
Q: Should there be any limits on the number of MCP servers that can be associated with a single profile?

A: There should be no hard limit on the number of MCP servers per profile, but the UI should provide clear information if too many servers might impact performance.

### 15. Profile Deletion Behavior
Q: How should the system behave if a profile is deleted but it contains the currently active MCP configuration?

A: If a profile is deleted while it's active, the system should automatically switch to the default profile and its associated MCP servers. Users should be notified of this change.
