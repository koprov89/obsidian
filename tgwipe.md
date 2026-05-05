# TGWipe — Project Description

## What it does

TGWipe is a Telegram bot for managing company Telegram accounts in emergencies. It allows
authorized personnel (admins) to remotely terminate active Telegram sessions and wipe account data
for registered employees — without needing physical access to their devices.

**Primary use case:** An employee loses their phone or leaves the company. An admin opens the bot,
selects the employee's account, and executes a session termination or full data wipe in seconds.

---

## Tech stack

| Component | Technology |
|---|---|
| Bot framework | python-telegram-bot ≥ 21 (async) |
| Telegram API client | Telethon (MTProto) |
| Database | SQLite via `sqlite3` |
| Configuration | `.env` file via `python-dotenv` |
| Language | Python 3.10+ |

---

## Configuration (`.env`)

| Variable | Required | Description |
|---|---|---|
| `API_ID` | yes | Telegram API app ID |
| `API_HASH` | yes | Telegram API app hash |
| `BOT_TOKEN` | yes | Telegram bot token |
| `GLOBAL_ADMIN_IDS` | yes | Comma-separated Telegram user IDs of global admins |
| `GLOBAL_EXECUTOR_IDS` | no | Comma-separated Telegram user IDs of global executors |
| `GLOBAL_MANAGER_IDS` | no | Comma-separated Telegram user IDs of global managers |

Session files are stored in `sessions/`. The database file is `tgwipe.db`. Logs go to `tgwipe.log`.

---

## Role system

There are two tiers of roles: **global** (set in `.env`) and **group-scoped** (stored in the database).

### Global roles (env-based)

| Role | Can do |
|---|---|
| `global_admin` | Everything: actions, manage, create/delete groups, assign group roles, approve admin requests |
| `global_executor` | Actions only (terminate sessions, wipe data) on all users |
| `global_manager` | Manage only (add/delete users, manage groups) across all users |

### Group-scoped roles (DB-based, assigned by global_admin)

| Role | Can do |
|---|---|
| `group_admin` | Actions + Manage, but only for users in their assigned groups |
| `group_executor` | Actions only, for their assigned groups |
| `group_manager` | Manage only, for their assigned groups |

**Role precedence:** Global roles take priority over group roles. A user in `GLOBAL_ADMIN_IDS` is
never treated as a group role holder even if DB rows exist.

**Access control:** `get_accessible_group_ids(user_id)` returns `None` for global roles
(meaning: all groups) or a `set[int]` for group-scoped roles. All listing and action handlers
filter through this function.

---

## Database schema

### `groups`
Stores employee groups.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | auto-increment |
| `name` | TEXT UNIQUE | group name |
| `created_at` | TEXT | ISO timestamp |

### `users`
Stores registered employee Telegram accounts.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | auto-increment |
| `phone` | TEXT UNIQUE | international format (+...) |
| `display_name` | TEXT | shown in admin UI |
| `group_id` | INTEGER FK → groups | cascade on group delete |
| `registered_at` | TEXT | ISO timestamp |
| `session_path` | TEXT | path to `.session` file |

### `group_role_assignments`
Stores group-scoped admin roles.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | auto-increment |
| `telegram_user_id` | INTEGER | Telegram user ID of the admin |
| `group_id` | INTEGER FK → groups | `ON DELETE CASCADE` |
| `role` | TEXT | `group_admin` / `group_executor` / `group_manager` |
| `name` | TEXT | display name, shown in the Group Admins list |

`UNIQUE(telegram_user_id, group_id)` — one role per (person, group) pair.

### `admin_requests`
Pending requests from non-admins who want access.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | auto-increment |
| `telegram_user_id` | INTEGER UNIQUE | one request per person |
| `name` | TEXT | name entered by the requester |
| `requested_at` | TEXT | ISO timestamp |

### `audit_log`
Immutable record of all admin actions.

| Column | Type |
|---|---|
| `id` | INTEGER PK |
| `timestamp` | TEXT |
| `admin_id` | TEXT |
| `target` | TEXT |
| `action_type` | TEXT |
| `result` | TEXT |

---

## Menu structure

### Non-admin user opens bot (`/start`)

```
No pending request:
  "This bot is for authorized personnel only."
  [📨 Request admin access]

Pending request exists:
  "Your request is already pending (name: Alice)."
  [✏ Update name]  [❌ Cancel request]

  → Click "Request admin access" or "Update name":
      "Enter your name for the admin access request:"
      [❌ Cancel]
      → Type name → "✅ Request submitted as 'Alice'."
```

---

### Admin opens bot (`/start` or after registration)

The main menu shown depends on the admin's role:

```
global_admin or group_admin:
  [⚡ Actions]  [📋 Manage]
  [👑 Group Admins]          ← global_admin only

global_executor or group_executor:
  [⚡ Actions]

global_manager or group_manager:
  [📋 Manage]
```

---

### ⚡ Actions menu

> Available to: executors and admins (not managers)

```
[⚡ Actions] →
  "Select scope:"
  [👤 Specific user]  [👥 Group]
  [⬅ Back]

  → Specific user:
      List of registered users (filtered by accessible groups)
      → Select user → [🔌 Terminate sessions] [🧹 Wipe data] [💥 Full wipe] [⬅ Back]

      → Terminate sessions:
          [All sessions]  [Specific session]  [⬅ Back]
          → Specific session: list of active sessions with device/app info
          → Confirm: "⚠️ Are you sure?" [✅ Confirm] [❌ Cancel]
          → Result with per-phone status

      → Wipe data / Full wipe:
          → Confirm: "⚠️ Are you sure?" [✅ Confirm] [❌ Cancel]
          → Result with per-phone status

  → Group:
      Toggle picker (multi-select) of groups
      [⬜ Group A (3 members)]
      [⬜ Group B (1 member)]
      [💾 Confirm (0 selected)]  [⬅ Back]
      → Select groups → Confirm → same action picker as above
```

**Actions:**

| Action | What it does |
|---|---|
| 🔌 Terminate sessions | Revokes one or all active Telegram sessions via MTProto |
| 🧹 Wipe data | Deletes dialogs, contacts, profile, photos, stories |
| 💥 Full wipe | Terminate all sessions, then wipe data |

---

### 📋 Manage menu

> Available to: admins and managers (both global and group-scoped)

```
[📋 Manage] →
  "Manage — choose a section:"
  [👤 Users]  [🗂 Groups]
  [👑 Group Admins]   ← global_admin only
  [⬅ Back]
```

#### 👤 Users submenu

```
[👤 Users] →
  [➕ Add user]  [➖ Delete user]
  [📋 List users]
  [⬅ Back]

  → Add user: starts phone registration flow (admin-initiated)
      Phone → Display name → Group picker
      → Verification code → (2FA password if enabled) → ✅ Registered

  → Delete user:
      List of users → Select → Confirm
      → Terminates session file → removes from DB

  → List users:
      Formatted list: "Name (+phone) — GroupName"
```

#### 🗂 Groups submenu

```
[🗂 Groups] →
  Shows current groups with member counts
  [📋 List groups]  [➕ Add group]
  [➖ Delete group]  [🔀 Move users]
  [⬅ Back]

  → List groups: shows all groups with member counts

  → Add group: type name → created (global_admin / global_manager only)

  → Delete group:
      Select group →
        If empty: [✅ Confirm delete] [❌ Cancel]
        If has members:
          [❌ Cancel]
          [🗑 Delete all N users in group]
          [🔀 Move users to another group] → pick target group → move + delete

  → Move users: select user → select target group → moved
```

---

### 👑 Group Admins menu

> Available to: `global_admin` only

```
[👑 Group Admins] →
  Lists all group role assignments:
    • Alice (id:123) → payments [group_admin]
    • Bob (id:456)   → hr [group_executor]

  [📨 Pending requests (N)]   ← shown only when requests exist
  [➕ Add assignment]  [➖ Remove assignment]
  [⬅ Back]

  → Pending requests:
      Lists all pending requests with Approve / Reject per row
      [✅ Alice]  [❌]
      [✅ Bob]    [❌]
      [⬅ Back]

      → Approve:
          Pre-fills name + Telegram ID from request
          → Role picker → group toggle picker → saved + request deleted

      → Reject:
          "Reject request from Alice (id:123)?"
          [✅ Confirm reject]  [⬅ Back]
          → Request deleted

  → Add assignment:
      Shows existing admins (by name) + "Add manually"
      [Alice]  [Bob]  [➕ Add manually]  [⬅ Back]

      → Existing admin selected:
          → Role picker → group toggle picker → saved

      → Add manually:
          "Enter the Telegram user ID (numeric):"  [❌ Cancel]
          → If ID already exists: skip name, show role picker
          → If new: "Enter a display name:"  [❌ Cancel]
          → Role picker:
              [👑 Group Admin]  [⚡ Group Executor]  [📋 Group Manager]  [❌ Cancel]

          → Group toggle picker (multi-select):
              [⬜ Group A]
              [⬜ Group B]
              [💾 Confirm (0 selected)]
              [➕ New Group]   ← creates a group inline
              [❌ Cancel]

          → Confirm → ✅ Assigned role to person for N group(s)

  → Remove assignment:
      Toggle picker of all assignments
      [⬜ Alice (id:123) → payments [group_admin]]
      [⬜ Bob (id:456) → hr [group_executor]]
      [🗑 Remove]
      [⬅ Back]
      → Toggle items → Remove → ✅ Removed N assignment(s)
```

---

## File structure

```
tgwipe/
├── bot.py                              # Entry point, handler registration
├── core/
│   ├── config.py                       # Env vars, constants
│   ├── database.py                     # SQLite CRUD functions
│   ├── operations.py                   # Telethon operations (terminate, wipe)
│   └── logger.py                       # Logging + audit_log()
├── bot_handlers/
│   ├── common.py                       # Role checks, keyboard builder, admin menu
│   ├── employee.py                     # /start, self-registration, admin request flow
│   ├── admin_users.py                  # ⚡ Actions: terminate/wipe flows
│   ├── admin_groups.py                 # 📋 Manage: users + groups submenu
│   └── admin_role_assignments.py       # 👑 Group Admins: assign/remove/requests
├── sessions/                           # Telethon .session files
├── tgwipe.db                           # SQLite database
├── tgwipe.log                          # Application log
└── docs/
    └── PROJECT.md                      # This file
```

---

## Telethon operations (`core/operations.py`)

All operations open a fresh `TelegramClient` from a stored `.session` file, do their work,
then disconnect.

| Function | Description |
|---|---|
| `get_active_sessions(phone)` | Lists active sessions via `GetAuthorizationsRequest` |
| `terminate_sessions(phone, hash)` | Terminates one or all sessions; `hash=None` = all |
| `terminate_sessions_parallel(phones)` | Parallel terminate for a list of phones |
| `terminate_own_session(phone)` | Logs out only the stored session (`LogOutRequest`) |
| `wipe_data(phone)` | Deletes: dialogs, saved messages, contacts, profile, photos, stories |
| `wipe_data_sequential(phones)` | Sequential wipe with 0.5s pause between accounts |
| `full_wipe(phones)` | Parallel terminate → sequential wipe |

`_retry_flood()` wraps any call with up to 5 retries on `FloodWaitError`.

---

## Registration flow (admin-initiated)

Admins can add employee accounts via **Manage → Users → Add user**:

```
Admin clicks "Add user"
  → Enter phone (international format)
  → Enter display name
  → Select group
  → Telegram sends verification code to the phone
  → Admin enters the code
  → (2FA password if the account has it)
  → Session saved to sessions/ → user record created in DB
```

---

## Admin access request flow

Non-admins cannot register themselves. Instead, they can request admin access:

```
Non-admin clicks "Request admin access"
  → Enters their name
  → Request saved with their Telegram user ID (captured automatically)

Global admin sees "📨 Pending requests (N)" in the Group Admins menu
  → Views the list
  → Approves → assigns role + groups → request deleted
  → Or rejects → request deleted
```

Approving a request auto-fills the person's name and Telegram ID into the role assignment flow —
no manual lookup needed.
