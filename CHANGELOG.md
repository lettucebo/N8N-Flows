# CHANGELOG

## [1.0.4] - 2026-07-29
### Documentation
- Rewrote **Slack to Google Calendar AI Assistant** documentation to match the exported workflow JSON:
  - Replaced the outdated 9-node, webhook-based description with the actual 14-node graph (Slack Trigger → Filter Valid Messages → Analyze Message with AI → Parse AI Response → Has Valid Event → Check Confidence Score → …).
  - Documented the shared error sink: four nodes route `onError: continueErrorOutput` into `Send Error Notification`.
  - Documented the Slack Block Kit contract (`blocks` must be a `JSON.stringify` string) and the two block builder nodes.
  - Documented the duplicated 0.7 confidence threshold in `Parse AI Response` and `Check Confidence Score`.
  - Documented workflow settings, credential types, and the Slack Trigger (its production Webhook URL must still be registered in the Slack app's Event Subscriptions).
  - Kept `README.md` and `README.zh-tw.md` section-for-section aligned.

### Repository tooling
- Added `.mcp.json` for GitHub Copilot CLI MCP server configuration.
- Expanded `.github/copilot-instructions.md` with export-file contract, local checks, cross-node contracts, and credential handling rules.

### Corrections
- `sendUpdates` is `none`, not `all` as stated in 1.0.3; no reminders are configured in `additionalFields`. The 1.0.3 entry describes changes that are not present in the exported workflow.
- `attendees` is computed by `Parse AI Response` but is never mapped into `Create Calendar Event`, so events are always created without guests.

## [1.0.3] - 2025-06-01
### Enhancements
- Updated **Slack to Google Calendar AI Assistant**:
  - Improved AI prompt for better detection and processing of complex event types.
  - Enhanced structured event descriptions with detailed formatting.
  - Configured reminders for both all-day and timed events.
  - Improved notification messages with better formatting and details.
  - Changed calendar events to auto-send invites (sendUpdates: all).
  - Enhanced user experience with emoji and structured information.

### Fixes
- Resolved issues with Slack message filtering logic.
- Fixed incorrect handling of all-day event date formats.

## [1.0.2] - 2025-06-01
### Enhancements
- Improved **Slack to Google Calendar AI Assistant**:
  - Updated AI prompt to better detect and process complex event types.
  - Added structured event descriptions with detailed formatting.
  - Configured reminders for both all-day and timed events.
  - Improved notification messages with better formatting and details.
  - Set calendar events to not auto-send invites (sendUpdates: none).
  - Enhanced user experience with emoji and structured information.
- Added VS Code settings and configuration files

## [1.0.1] - 2025-05-29
### Updated
- Enhanced **README.md** to include a detailed project overview and directory structure.
- Improved **README.zh-tw.md** with additional information on system maintenance and update considerations.

## [1.0.0] - 2025-05-28
### Added
- Initial release of the **N8N-Flows** project.
- Added workflows for **RaindropKnowledgeManagement**, integrating Raindrop.io, Azure AI, and Notion.
- Included project documentation:
  - **README.md**: Overview and usage instructions.
  - **README.zh-tw.md**: Chinese technical documentation for node configurations and functionalities.