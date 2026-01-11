# AGENTS.md - Developer Guidelines for Antigravity Tools

## Project Overview

Antigravity Tools is a Tauri desktop application (Rust backend + React/TypeScript frontend) that manages AI account credentials and provides API proxy services for multiple protocols (OpenAI, Anthropic, Gemini).

## Build, Lint, and Test Commands

### Frontend (TypeScript/React)
```bash
# Development
npm run dev              # Start Vite dev server (port 1420)

# Build
npm run build            # Run TypeScript compiler then Vite build
                        # Equivalent to: tsc && vite build

# Preview
npm run preview          # Preview production build

# Tauri commands
npm run tauri           # Access Tauri CLI (e.g., npm run tauri build)
```

### Backend (Rust)
```bash
# Build
cargo build --release          # Production build
cargo build                   # Debug build

# Run tests
cargo test                     # Run all tests
cargo test --package antigravity_tools --lib  # Run library tests only

# Run specific test module
cargo test --test comprehensive   # Run specific test file

# Linting (clippy)
cargo clippy                   # Check for common issues
cargo clippy -- -D warnings     # Treat warnings as errors
```

### Running a Single Test
```bash
# Run tests in a specific file
cargo test --test comprehensive

# Run a specific test function
cargo test test_first_thinking_request_permissive_mode

# Run tests with output
cargo test -- --nocapture

# Run tests in release mode
cargo test --release
```

## Code Style Guidelines

### Frontend (TypeScript/React)

#### Import Organization
```typescript
// Order: 1. External libraries, 2. Local modules
import { useState, useEffect } from 'react';
import { useTranslation } from 'react-i18next';
import { invoke } from '@tauri-apps/api/core';
import { Search, RefreshCw } from 'lucide-react';

// Local imports (services, components, stores, utils, types)
import * as accountService from '../services/accountService';
import { useAccountStore } from '../stores/useAccountStore';
import AccountTable from '../components/accounts/AccountTable';
import { Account } from '../types/account';
import { cn } from '../utils/cn';
```

#### Component Structure
- Use functional components with hooks (no class components)
- Extract components into dedicated files under `src/components/`
- Use TypeScript interfaces for props

```typescript
function Accounts() {
    const { t } = useTranslation();
    const { accounts, fetchAccounts, loading } = useAccountStore();

    const [searchQuery, setSearchQuery] = useState('');
    const [currentPage, setCurrentPage] = useState(1);

    useEffect(() => {
        fetchAccounts();
    }, []);

    return (
        <div className="container">
            {/* JSX content */}
        </div>
    );
}
```

#### State Management (Zustand)
- All state stores in `src/stores/` using Zustand
- Define interface for state and actions
- Use `create()` pattern with async actions

```typescript
interface AccountState {
    accounts: Account[];
    currentAccount: Account | null;
    loading: boolean;
    error: string | null;

    fetchAccounts: () => Promise<void>;
    addAccount: (email: string, refreshToken: string) => Promise<void>;
}

export const useAccountStore = create<AccountState>((set, get) => ({
    accounts: [],
    currentAccount: null,
    loading: false,
    error: null,

    fetchAccounts: async () => {
        set({ loading: true, error: null });
        try {
            const accounts = await accountService.listAccounts();
            set({ accounts, loading: false });
        } catch (error) {
            console.error('[Store] Fetch failed:', error);
            set({ error: String(error), loading: false });
        }
    },

    addAccount: async (email, refreshToken) => {
        set({ loading: true, error: null });
        try {
            await accountService.addAccount(email, refreshToken);
            await get().fetchAccounts(); // Refresh list
            set({ loading: false });
        } catch (error) {
            set({ error: String(error), loading: false });
            throw error;
        }
    },
}));
```

#### API/Service Call Patterns
- Service layer wraps Tauri `invoke()` calls
- All services in `src/services/`
- Use type-safe function signatures

```typescript
// services/accountService.ts
import { request as invoke } from '../utils/request';

export async function listAccounts(): Promise<Account[]> {
    return await invoke('list_accounts');
}

export async function addAccount(email: string, refreshToken: string): Promise<Account> {
    return await invoke('add_account', { email, refreshToken });
}

// Usage in component
const accounts = await accountService.listAccounts();
```

#### Styling (Tailwind + DaisyUI)
- Use Tailwind utility classes directly
- Use `cn()` utility for conditional class merging
- DaisyUI components for UI elements
- Dark mode: use `dark:` prefix (controlled by class on html/body)

```typescript
import { cn } from '../utils/cn';

<div className={cn(
    "base-200 rounded-lg p-4",
    isActive && "ring-2 ring-primary",
    className // Allow override
)}>
    <button className="btn btn-primary">Click</button>
</div>

// Responsive
<div className="grid grid-cols-1 lg:grid-cols-2 gap-4">

// Dark mode
<div className="bg-white dark:bg-gray-800 text-black dark:text-white">
```

#### Error Handling
- Try-catch blocks in async functions
- Store errors in state
- Log errors with console.error
- Show user-friendly error messages via toast

```typescript
try {
    await accountService.addAccount(email, token);
    showToast({ message: t('accounts.added'), type: 'success' });
} catch (error) {
    console.error('Add account failed:', error);
    showToast({ message: String(error), type: 'error' });
    throw error; // Re-throw for store handling
}
```

#### i18n (Internationalization)
- Use `useTranslation()` hook from `react-i18next`
- Translation keys in `src/locales/en.json` and `src/locales/zh.json`
- Access with `t('key.path')`

```typescript
import { useTranslation } from 'react-i18next';

function Component() {
    const { t } = useTranslation();

    return (
        <h1>{t('accounts.title')}</h1>
        <p>{t('common.loading')}</p>
    );
}

// Translation file (en.json)
{
  "accounts": { "title": "Accounts" },
  "common": { "loading": "Loading..." }
}
```

#### TypeScript Configuration
- Strict mode enabled (`"strict": true`)
- No unused locals/parameters allowed
- No implicit any

```typescript
// From tsconfig.json
{
  "strict": true,
  "noUnusedLocals": true,
  "noUnusedParameters": true,
  "noFallthroughCasesInSwitch": true
}
```

### Backend (Rust)

#### Import Organization
```rust
// Order: std library → third-party crates → local modules
use std::sync::{Arc, Mutex};
use tokio::sync::RwLock;
use serde::{Serialize, Deserialize};
use tauri::{State, Manager};

// Local modules with crate:: prefix
use crate::models::{Account, TokenData};
use crate::modules::account;
use crate::proxy::handlers;
```

#### Error Handling
- Use `thiserror` for error types (NOT `anywhere`)
- Define `AppError` enum in `src-tauri/src/error.rs`
- Implement `Serialize` for Tauri compatibility
- Use `Result<T, String>` for Tauri commands

```rust
// error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] rusqlite::Error),

    #[error("Network error: {0}")]
    Network(#[from] reqwest::Error),

    #[error("OAuth error: {0}")]
    OAuth(String),
}

impl Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        serializer.serialize_str(self.to_string().as_str())
    }
}

// Tauri command return type
#[tauri::command]
pub async fn list_accounts() -> Result<Vec<Account>, String> {
    modules::list_accounts().map_err(|e| e.to_string())
}
```

#### Struct/Module Organization
```rust
// Data models with serde derives
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Account {
    pub id: String,
    pub email: String,
    #[serde(default)]
    pub disabled: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub disabled_reason: Option<String>,
}

// Implement methods with impl blocks
impl Account {
    pub fn new(id: String, email: String) -> Self {
        Self { id, email, ..Default::default() }
    }

    pub fn update_last_used(&mut self) {
        self.last_used = chrono::Utc::now().timestamp();
    }
}

// Module structure
// models/mod.rs
pub mod account;
pub mod token;
pub mod quota;

pub use account::{Account, AccountIndex};
pub use token::TokenData;
```

#### Async/Await Patterns
```rust
use tokio::sync::Semaphore;
use futures::future::join_all;

// Async functions
pub async fn start_service() -> Result<(), String> {
    let state = instance.read().await;
    // ... async operations
}

// Concurrency with semaphore
const MAX_CONCURRENT: usize = 5;
let semaphore = Arc::new(Semaphore::new(MAX_CONCURRENT));

let tasks: Vec<_> = accounts.into_iter().map(|mut account| {
    let permit = semaphore.clone();
    async move {
        let _guard = permit.acquire().await.unwrap();
        process_account(&mut account).await
    }
}).collect();

let results = join_all(tasks).await;

// Delays
tokio::time::sleep(Duration::from_millis(100)).await;
```

#### Tauri Command Patterns
```rust
#[tauri::command]
pub async fn list_accounts() -> Result<Vec<Account>, String> {
    modules::list_accounts()
}

#[tauri::command]
pub async fn start_proxy_service(
    config: ProxyConfig,
    state: State<'_, ProxyServiceState>,
    app_handle: tauri::AppHandle,
) -> Result<ProxyStatus, String> {
    // Access shared state with state.inner()
    let mut instance = state.instance.write().await;
    // Use app_handle for Tauri API
    Ok(ProxyStatus::Running)
}

// Register in lib.rs
.invoke_handler(tauri::generate_handler![
    list_accounts,
    start_proxy_service,
    // ... more commands
])
```

#### Database/File Operations
- JSON files for account storage in `~/.antigravity/accounts/`
- SQLite for V1 database import
- Use `serde_json` for serialization

```rust
use serde_json;
use std::fs;

pub fn save_account(account: &Account) -> Result<(), String> {
    let content = serde_json::to_string_pretty(account)
        .map_err(|e| format!("Serialization failed: {}", e))?;

    fs::write(path, content)
        .map_err(|e| format!("Write failed: {}", e))?;

    Ok(())
}

pub fn load_account(id: &str) -> Result<Account, String> {
    let content = fs::read_to_string(path)
        .map_err(|e| format!("Read failed: {}", e))?;

    serde_json::from_str(&content)
        .map_err(|e| format!("Parse failed: {}", e))?
}
```

#### Logging (tracing)
```rust
use tracing::{info, error, warn, debug, trace};

// Log levels:
info!("Proxy service started");
warn!("Rate limit exceeded for account {}", email);
error!("Failed to fetch quota: {}", e);
debug!("Request payload: {:?}", body);
trace!("Streaming chunk: {}", chunk);

// Custom logger wrapper
modules::logger::log_info("Account added successfully");
```

### Naming Conventions

#### TypeScript/React
- **Components**: PascalCase (`AccountTable.tsx`, `AddAccountDialog.tsx`)
- **Hooks**: camelCase starting with `use` (`useAccountStore.ts`, `useProxyModels.tsx`)
- **Services**: camelCase (`accountService.ts`, `configService.ts`)
- **Types/Interfaces**: PascalCase (`Account.ts`, `Config.ts`)
- **Functions**: camelCase (`fetchAccounts`, `addAccount`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_CONCURRENT`, `DEFAULT_PORT`)

#### Rust
- **Structs/Enums**: PascalCase (`Account`, `ProxyConfig`, `AppError`)
- **Functions/Methods**: snake_case (`list_accounts`, `fetch_quota`)
- **Modules**: snake_case (`mod account`, `mod proxy_handlers`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_CONCURRENT_REQUESTS`, `DEFAULT_TIMEOUT`)
- **Type Parameters**: Single uppercase letter (`T`, `E`)

### File Organization

```
src/
├── components/          # React components
│   ├── layout/        # Layout components (Navbar, Layout)
│   ├── accounts/       # Account-related components
│   ├── dashboard/      # Dashboard components
│   ├── proxy/         # Proxy-related components
│   └── common/        # Shared components (Toast, Modal, Pagination)
├── pages/             # Page components (Dashboard, Accounts, Settings)
├── stores/            # Zustand state stores
├── services/          # Tauri API wrappers
├── types/             # TypeScript interfaces
├── utils/             # Utility functions (cn.ts, request.ts)
├── i18n.ts           # i18next configuration
├── App.tsx            # Root component with router
└── main.tsx           # Entry point

src-tauri/src/
├── lib.rs             # Tauri setup, command registration
├── main.rs            # Minimal entry point
├── error.rs           # Global error types
├── models/            # Data structures (serde models)
├── commands/          # Tauri command handlers
├── modules/           # Business logic (OAuth, account, DB)
└── proxy/             # HTTP proxy server (Axum)
    ├── handlers/       # Request handlers
    ├── mappers/        # Protocol transformation
    ├── middleware/     # Axum middleware
    └── tests/         # Unit tests
```

### Important Notes

1. **No TypeScript linter configured** - The codebase does not use ESLint. Follow `tsconfig.json` strict mode rules.

2. **No Rust linter configured** - No `.clippy.toml` found. Run `cargo clippy` manually for linting.

3. **Tests** - Only Rust tests exist in `src-tauri/src/proxy/tests/`. No frontend test framework configured.

4. **No .env file** - Configuration is managed through the app settings UI and Tauri commands.

5. **Commit guidelines** - Never use `@ts-ignore` or `@ts-expect-error` without justification. Fix type errors properly.

6. **Platform-specific code** - Use `#[cfg(target_os = "macos")]` or similar attributes for platform-specific Rust code.

7. **State updates in Rust** - Use `Arc<RwLock<T>>` for shared mutable state in async contexts.
