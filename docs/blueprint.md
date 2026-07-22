# GroupGuard Bot — Bot specification

**Archetype:** custom

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

Automated Telegram group moderation bot that verifies new members with a math captcha, enforces anti-spam rules via configurable thresholds, and provides admin controls for trust management, infractions tracking, and action logging. Sends transparent explanations for all automated actions and offers periodic admin reports.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Telegram group owners
- Moderators
- Community managers

## Success criteria

- New members verified via math captcha within 60s or removed
- Spam infractions trigger configured actions (warn/mute/kick) with public explanations
- Admin commands execute and log actions accurately
- Action log retains last 100 entries
- Reports show join/verification/removal statistics

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open main menu for admins
- **Submit Answer** (button, actor: user, callback: verify:submit) — Submit math captcha response
  - inputs: user answer
  - outputs: verification status
- **/warn** (command, actor: admin, command: /warn) — Warn a user with optional reason
- **/mute** (command, actor: admin, command: /mute) — Mute a user with optional duration
- **/kick** (command, actor: admin, command: /kick) — Kick a user with optional reason
- **/ban** (command, actor: admin, command: /ban) — Ban a user with optional reason
- **/trust** (command, actor: admin, command: /trust) — Mark user as trusted (exempt from enforcement)
- **/log** (command, actor: admin, command: /log) — View recent action log entries
- **/report** (command, actor: admin, command: /report) — Generate moderation statistics report

## Flows

### NewMemberVerification
_Trigger:_ new_member_joined

1. Send welcome message with rules
2. Generate and present math captcha
3. Start 60s verification timer
4. Delete unverified member messages
5. Remove member if verification timeout

_Data touched:_ Member, VerificationChallenge, Infraction, ActionLogEntry

### SpamDetection
_Trigger:_ message_posted

1. Check message against spam thresholds
2. Count infractions if rule breached
3. Trigger configured action (warn/mute/remove)
4. Post explanation message to group

_Data touched:_ Infraction, ActionLogEntry

### AdminModeration
_Trigger:_ /warn /mute /kick /ban

1. Validate admin privileges
2. Execute moderation action
3. Log action with moderator, target, and reason

_Data touched:_ ActionLogEntry, Member

### TrustManagement
_Trigger:_ /trust /untrust

1. Validate admin privileges
2. Update member's trust flag
3. Log trust status change

_Data touched:_ Member, ActionLogEntry

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **Member** _(retention: persistent)_ — Group participant with moderation status
  - fields: user_id, display_name, join_timestamp, trust_flag, verification_status
- **VerificationChallenge** _(retention: session)_ — Time-limited math verification for new members
  - fields: captcha_expression, expected_result, expiry_timestamp
- **Infraction** _(retention: persistent)_ — Record of rule violations
  - fields: violation_type, count, timestamps
- **ActionLogEntry** _(retention: persistent)_ — Audit trail of moderation actions
  - fields: moderator_type, action_type, target_user, reason, timestamp
- **Config** _(retention: persistent)_ — Group-specific moderation settings
  - fields: welcome_message, rules_text, spam_thresholds, exempt_roles, trusted_users

## Integrations

- **Telegram** (required) — Bot API messaging and group management
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Configure welcome message
- Set rules text
- Adjust spam thresholds
- Define exempt roles/users
- Mark users as trusted
- View action log
- Generate moderation reports

## Notifications

- Welcome message with verification prompt
- Spam violation explanation messages
- Admin action confirmation messages
- Periodic moderation summary (pinned or staged)
- Configurable DMs for critical removals

## Permissions & privacy

- Only acts on non-admin users and public messages
- Does not store personal data beyond user_id and display_name
- Automated actions do not target pinned messages
- Action logs include only public moderation data

## Edge cases

- Member answers captcha after timeout (should not count)
- Admin attempts to verify themselves (should bypass captcha)
- Trusted user commits infractions (should not trigger auto-actions)
- Multiple simultaneous spam violations (should aggregate correctly)
- Log retention exceeding 100 entries (should purge oldest)

## Required tests

- End-to-end new member verification flow with timeout
- Spam detection with configured thresholds
- Admin command execution and logging
- Log retention mechanics
- Trusted user exemption validation

## Assumptions

- Math captcha is sufficient for bot detection
- Default thresholds are reasonable for most groups
- Admins will configure rules appropriately for their community
