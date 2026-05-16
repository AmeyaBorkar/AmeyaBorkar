# BankSystem 1.7

> A 15,000+ line console banking application in C for Windows — deposits, transfers, two-factor auth, multi-session management, fraud detection, and a real-time stock market simulator.

**Repository:** [`AmeyaBorkar/BankSystem1.7`](https://github.com/AmeyaBorkar/BankSystem1.7)  
**Category:** Systems / Application  
**Visibility:** Public  
**Primary language:** C  
**Default branch:** `master`  
**License:** —  
**Created:** 2024-08-27  
**Last pushed:** 2026-03-02  
**Metadata updated:** 2026-03-02  
**Size (GitHub reported):** 73 KB  

---

## What it is (one-paragraph version)

A 15,000+ line console banking application in C for Windows — deposits, transfers, two-factor auth, multi-session management, fraud detection, and a real-time stock market simulator.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| C | 340,842 | 100.0% |

## File tree

- Total entries indexed: **36** (36 files, 0 directories)

```
.gitignore  (421 B)
BankSystem1.7[FileIO].sln  (1 KB)
BankSystem1.7[FileIO].vcxproj  (7 KB)
BankSystem1.7[FileIO].vcxproj.filters  (3 KB)
README.md  (15 KB)
adminPanel.c  (32 KB)
bank.h  (9 KB)
chat.c  (25 KB)
commented_out_code_archive.c  (58 KB)
companies  (3 KB)
complaint_data  (2 B)
coreFunctionalities.c  (18 KB)
features.c  (18 KB)
fileHandling.c  (30 KB)
fraudDetection.c  (23 KB)
groups.txt  (0 B)
help.c  (17 KB)
instanceHandling.c  (24 KB)
interfaceSupport.c  (8 KB)
investmentManager.c  (43 KB)
investment_data.txt  (204 B)
investments  (10 B)
login.c  (4 KB)
main.c  (1,019 B)
menu.c  (8 KB)
notification.c  (3 KB)
personalMessages  (306 B)
scheduledTransactions.c  (7 KB)
suspLogs.txt  (338 B)
threshold_requests  (20 B)
timestamps.txt  (137 B)
transactionProcess.c  (5 KB)
transactionRequests.txt  (108 B)
transactions  (389 B)
userLog.txt  (3 KB)
users1  (359 B)
```

## README (verbatim)

# 🏦 BankSystem 1.7 — File I/O Edition

A feature-rich **console-based banking system** built entirely in **C** for Windows, featuring multi-instance session management, real-time investment simulation, fraud detection, in-app messaging, and more — all persisted through file I/O.

---

## ✨ Key Highlights

- **Multi-instance support** — Multiple banking sessions can run simultaneously with file-based locking and session management
- **Real-time stock market simulation** — Background thread continuously updates stock prices using sinusoidal trends, random noise, and company-specific factors
- **Fraud detection engine** — Statistical anomaly detection using mean, standard deviation, and pattern analysis to automatically flag and revert suspicious transactions
- **Two-Factor Authentication (2FA)** — PIN + OTP verification for secure login
- **File-based persistence** — All user data, transactions, investments, sessions, and chat messages are stored in flat files

---

## 📋 Features

### 🔐 Authentication & Security
- **UID + PIN login** with masked input (asterisk display)
- **Two-Factor Authentication** — 6-digit OTP generated after PIN verification
- **Account lockout** after 3 failed login attempts
- **Session logging** — Every login/logout event is recorded with timestamps
- **Console close handler** — Graceful cleanup on abnormal window closure (Ctrl+C, window close)

### 💰 Core Banking
- **Deposit & Withdraw** with balance validation
- **Money transfers** between users by UID
- **Bank statements** — Full transaction history with formatted display
- **Transaction threshold system** — Large transactions require admin approval
- **Transaction filtering** — Search by exact amount or amount range
- **Admin-level transaction management** — Delete individual, all, or other users' transactions

### 📈 Investment Platform
The investment module is a comprehensive stock market simulation with its own registration, login, and menu system:

- **Investments Dashboard** — Separate login/registration gateway with a $1000 registration fee deducted from bank balance
- **Live Market View** — Browse all listed companies with current price, P/E ratio, dividend yield, and market cap
- **Buy Stocks** — Purchase by dollar amount or number of shares across multiple listed companies
- **Sell Stocks** — Sell all shares or partial positions with real-time profit/loss calculation
- **Investment Analysis** — Portfolio summary with:
  - Total invested vs. current value
  - Average return percentage
  - Most and least profitable holdings (color-coded green/red/blue)
- **AI-Powered Investment Advice** — Per-company recommendations based on:
  - Price volatility (standard deviation of historical prices)
  - Short-term (7-day), mid-term (14-day), and long-term (30-day) trends
  - P/E ratio and dividend yield thresholds
  - Market capitalization
  - Growth rate analysis
  - Risk level assessment (Low/Medium/High)
  - Detailed written review with actionable recommendations (Strong Buy / Buy / Hold / Value Investment / Income Investment / Sell)
- **Mutual Funds** — Create mutual fund positions spread across multiple companies, view performance, and sell
- **Market History** — ASCII graph visualization of 30-day historical price data per company
- **Background Price Updates** — A dedicated thread (`background_task`) updates all stock prices every 20 seconds using a realistic model with:
  - Sinusoidal periodic components (market sentiment)
  - Random noise
  - Monthly event simulation (earnings reports, product launches)
  - Company-specific P/E and dividend yield influences
- **Persistent storage** — All company data, investments, and mutual fund positions saved to files

### 🛡️ Fraud Detection
- **Statistical anomaly detection** — Calculates average and standard deviation across recent transactions; flags amounts exceeding 2σ above the mean
- **Transaction frequency monitoring** — Detects 3+ transactions within 60 seconds → automatic account lock + revert
- **Near-threshold detection** — Flags transactions at 85–100% of the user's threshold limit
- **Pattern recognition** — Detects 4 consecutive transactions with similar amounts (within $3 range)
- **Suspicious transfer detection** — Flags accounts with 4+ transfers in the last 6 transactions
- **Auto-revert** — All flagged transactions are reversed, including corresponding entries on recipient accounts
- **Auto-lock** — Accounts are locked and the user is force-logged-out with a warning animation
- **Self-healing** — `unflagTransactions()` removes false-positive flags when followed by 4+ normal transactions
- **Suspicious activity logs** — Viewable from the admin panel

### 💬 Chat System
- **Direct messaging** between users by UID
- **Chat history** — Previous conversations are loaded and displayed
- **Group chats** — Create groups, add/remove members, send group messages
- **Group management** — View group details, delete groups
- **Message persistence** — All messages stored in files

### 🔔 Notifications
- **In-app notification system** — Notifications appear on login
- **Read/unread tracking** with color-coded display and fade effects
- **Positioned display** using `gotoxy()` for clean UI placement
- **Auto-generated for**: transaction approvals/rejections, threshold changes, admin actions

### 👤 Profile & Account Management
- **Update profile** — Name, address, phone, email
- **Change PIN** with verification
- **User registration** — New account creation with unique UID generation

### ⏰ Scheduled Transactions
- **Schedule deposits, withdrawals, and transfers** for future execution
- **Recurring transactions** with configurable interval (in seconds)
- **Auto-execution** — Scheduled transactions run automatically at their scheduled time
- **Transfer validation** — Checks recipient existence and lock status before scheduling

### 🔧 Admin Panel
- **Manage transaction requests** — Approve/reject high-value transactions pending admin review
- **Threshold change requests** — Users can request higher transaction limits; admin approves/rejects with reply
- **User overview** — Generate reports with all user balances and account statuses
- **Account lock/unlock** — Manage locked accounts
- **Session management** — View active instances, session details, and manage multi-instance state
- **Suspicious activity logs** — Review flagged transactions
- **Request status tracking** — Both admin and users can view request history and responses

### 💎 Premium Features (Paid)
Accessible through an in-app purchase system that deducts from the user's balance:
- **Currency Converter** — 50+ currencies with pre-loaded exchange rates (INR base)
- **Finance Manager** — Expense tracking extracted from transaction history, budget reports, and financial health scoring
- **Loan Calculator** — EMI calculation with principal, rate, and tenure inputs
- **Savings Tracker** — Goal-based savings tracking with progress visualization
- **Custom Balance Alerts** — Set low-balance threshold alerts
- **Transaction Tagging** — Tag transactions with custom labels
- **Bill Payments** — Pay electricity, water, internet, or phone bills directly from the app

### 🎨 UI & Experience
- **Typewriter text effect** — Character-by-character text rendering for immersive feel
- **ANSI color codes** — Yellow, cyan, green, red, blue text for visual hierarchy
- **Sound effects** — Beep audio for login, logout, exit, errors, and confirmations
- **Musical tunes** — Welcome, logout, and exit melodies using Windows multimedia timers
- **Loading bar animation** — Animated progress bar for session lock waits
- **In-app advertisements** — Randomly displayed feature promotions
- **Formatted separators** — Consistent visual structure across all menus

### 📂 Help & Support
- **FAQ system** — Answers to common banking questions
- **Complaint system** — Submit and view complaints (admin can review)
- **Contact information** — Branch directions with step-by-step navigation
- **Feature guide** — Detailed descriptions of each banking function

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                    main.c                           │
│         (Entry point, ConsoleHandler, threads)      │
├─────────────────────────────────────────────────────┤
│  menu.c          │  login.c                         │
│  (Navigation)    │  (Auth, PIN, 2FA)                │
├──────────────────┼──────────────────────────────────┤
│  coreFunctionalities.c    │  transactionProcess.c   │
│  (Deposit, Withdraw,      │  (Add, Delete, Filter   │
│   Transfer, Statement)    │   transactions)         │
├───────────────────────────┼─────────────────────────┤
│  investmentManager.c      │  fraudDetection.c       │
│  (Stocks, Mutual Funds,   │  (Anomaly detection,    │
│   Analysis, Advice,       │   Auto-lock, Revert)    │
│   Market simulation)      │                         │
├───────────────────────────┼─────────────────────────┤
│  adminPanel.c             │  chat.c                 │
│  (Requests, Reports,      │  (DMs, Groups,          │
│   Session management)     │   Message history)      │
├───────────────────────────┼─────────────────────────┤
│  features.c               │  notification.c         │
│  (Currency, Finance,      │  (Alerts, Read/Unread)  │
│   Loans, Savings, Bills)  │                         │
├───────────────────────────┼─────────────────────────┤
│  scheduledTransactions.c  │  instanceHandling.c     │
│  (Scheduling, Recurring)  │  (Multi-session, Locks) │
├───────────────────────────┼─────────────────────────┤
│  fileHandling.c           │  interfaceSupport.c     │
│  (Read/Write all data)    │  (UI, Sound, Animation) │
├───────────────────────────┴─────────────────────────┤
│                     bank.h                          │
│        (Structs, Macros, Globals, Prototypes)       │
└─────────────────────────────────────────────────────┘
```

### Data Files
| File | Contents |
|------|----------|
| `users1` | User profiles, PINs, balances, transactions, notifications |
| `transactions` | Transaction history |
| `companies` | Stock company data (prices, P/E, dividends, history) |
| `investments` | User investment portfolios |
| `investment_data.txt` | Investment registration list and mutual fund data |
| `personalMessages` | Direct and group chat messages |
| `groups.txt` | Chat group definitions |
| `transactionRequests.txt` | Pending high-value transaction requests |
| `threshold_requests` | Pending threshold change requests |
| `complaint_data` | User complaints |
| `userLog.txt` | Session login/logout audit log |
| `activeInstances.txt` | Currently active banking sessions |
| `suspLogs.txt` | Suspicious activity logs |
| `timestamps.txt` | Transaction timestamps |

---

## 🛠️ Build & Run

### Prerequisites
- **Windows OS** (uses `windows.h`, `conio.h`, `winmm.lib`)
- **Visual Studio** (recommended) or any C compiler supporting Windows APIs
- **MSVC** (Microsoft Visual C++ Compiler)

### Build with Visual Studio
1. Open `BankSystem1.7[FileIO].sln` in Visual Studio
2. Set configuration to **x64 Debug** or **x64 Release**
3. Build → Build Solution (`Ctrl+Shift+B`)
4. Run → Start Debugging (`F5`)

### Build with MSVC Command Line
```bash
cl /W3 /Fe:BankSystem.exe main.c menu.c login.c coreFunctionalities.c transactionProcess.c fileHandling.c adminPanel.c chat.c fraudDetection.c instanceHandling.c investmentManager.c features.c notification.c help.c interfaceSupport.c scheduledTransactions.c winmm.lib
```

### Default Login
After first launch, use the pre-configured admin account:
- **UID**: Check `users1` data file  
- **PIN**: Check `users1` data file  
- **2FA**: The code is displayed on screen for demo purposes

---

## 📁 Project Structure

```
BankSystem1.7[FileIO]/
├── bank.h                          # Header — structs, macros, globals, prototypes
├── main.c                          # Entry point, console handler, background thread
├── menu.c                          # Main menu navigation (arrow-key driven)
├── login.c                         # Authentication (PIN + 2FA)
├── coreFunctionalities.c           # Deposit, withdraw, transfer, bank statement
├── transactionProcess.c            # Transaction CRUD and filtering
├── fileHandling.c                  # All file read/write operations
├── adminPanel.c                    # Admin tools, request management, reports
├── chat.c                          # Direct messaging and group chat
├── fraudDetection.c                # Statistical fraud detection engine
├── instanceHandling.c              # Multi-instance session management
├── investmentManager.c             # Full investment platform
├── features.c                      # Premium features (currency, loans, etc.)
├── notification.c                  # Notification display system
├── help.c                          # FAQ, complaints, contact info
├── interfaceSupport.c              # UI utilities, sound effects, animations
├── scheduledTransactions.c         # Scheduled and recurring transactions
├── commented_out_code_archive.c    # Archive of superseded legacy code
├── .gitignore                      # Git exclusions
├── users1                          # User data file
├── transactions                    # Transaction data file
├── companies                       # Stock market company data
└── ...                             # Other data files
```

---

## ⚠️ Notes

- This is a **Windows-only** application due to usage of `windows.h`, `conio.h`, `Beep()`, and `winmm.lib`
- The 2FA code is displayed on screen for demonstration — in production, this would be sent via SMS/email
- Stock prices are simulated and update every 20 seconds via a background thread
- The `commented_out_code_archive.c` file contains earlier versions of functions kept for reference

---

## 📄 License

This project is for educational purposes.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/BankSystem1.7*
