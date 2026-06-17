# BankSystem 1.7

> A Windows-only, console-based banking simulator written in C across 17 `.c`/`.h` modules. It models accounts (UID + 4-digit PIN + on-screen "2FA"), deposits/withdrawals/transfers with an admin-approval threshold, a statistical fraud-detection engine, a stock-market simulation whose prices drift on a background thread, in-app direct + group chat, scheduled transactions, paid "premium" features, and a file-lock-based multi-instance/session manager. All state is persisted to flat text files in the working directory — there is no database and no real network/GUI; the "GUI" is an ANSI-colored text UI driven by the Win32 console API.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/BankSystem1.7 |
| **Visibility** | Public |
| **Category** | apps |
| **Primary language(s)** | C (100%) — MSVC, Win32 / `windows.h`, `conio.h`, `winmm` |
| **Local path** | `C:\Users\ameya\.repo-cache\BankSystem1.7` |
| **Default branch** | `master` |
| **Lines of code (computed)** | **10,104** total across `.c` + `.h`; **8,466** excluding the 1,638-line `commented_out_code_archive.c`. (README/task claim of "~15,000 lines" is **not** accurate.) |
| **Source files (computed)** | 16 `.c` + 1 `.h` = **17** source files (plus `commented_out_code_archive.c`, which is not part of the build) |
| **Key components** | `login.c` (auth/2FA), `coreFunctionalities.c` (deposit/withdraw/transfer), `fraudDetection.c`, `investmentManager.c` (stock sim + background thread), `instanceHandling.c` (file-lock sessions), `fileHandling.c` (persistence), `adminPanel.c`, `chat.c`, `features.c` |
| **License** | No `LICENSE` file. README states only "This project is for educational purposes." |
| **Last commit** | `9eec212` — "Initial commit: BankSystem 1.7 with investments, fraud detection, chat, and more" (2026-03-02) — single-commit repo |

---

## What it actually is

A single-process (with one helper thread) **console application** for Windows. On launch (`main.c`) it:

1. Registers a `ConsoleHandler` for Ctrl+C / window-close cleanup.
2. Spawns one Win32 thread running `background_task` (the stock-price ticker).
3. Calls `initialize()` to load all flat files into global arrays.
4. Generates a unique 6-digit "instance code", logs the active instance, and enters `mainMenu()`.

There is **no GUI in the windowed sense** and **no networking**. Everything is `printf`/`scanf`/`_getch()` over the console, dressed up with ANSI escape sequences, Win32 console cursor positioning (`SetConsoleCursorPosition`), and `Beep`/`winmm` timer "tunes". State lives entirely in process-global arrays declared in `bank.h` (`User users[100]`, `Company companies[20]`, etc.) and is round-tripped to text files on virtually every action via `write()` (load) and `quickSave()` (save).

The notion of "multiple instances" is real but unusual: the app is meant to be launched several times as separate console processes, which coordinate **through files** (`lockfile.txt`, `lockfilelog.txt`, `lockfilelogA.txt`, `activeInstances.txt`, `userLog.txt`) using busy-wait spin-locks, not through OS mutexes or shared memory.

---

## Architecture & how it's structured

Flat repo — all sources sit in the root next to their data files. Annotated tree:

```
BankSystem1.7/
├── bank.h                       # All structs, ~70 macros, ~30 globals, all prototypes. Single shared header.
├── main.c                       # 38 lines. Entry point: console handler + CreateThread(background_task) + initialize() + mainMenu().
├── menu.c                       # Main menu loop; routes to login/register; per-role (admin vs user) numbered menu.
├── login.c                      # UID lookup, masked PIN entry, lock checks, generate2FACode()/validate2FACode().
├── coreFunctionalities.c        # registerUser, changePin, updateProfile, updateBalance, transferMoney, bankStatement.
├── transactionProcess.c         # addTransaction (the central write path), delete*/filter* transactions.
├── fraudDetection.c             # Statistical anomaly detection, frequency/pattern checks, auto-lock + revert.
├── investmentManager.c          # Stock platform: dashboard, market, buy/sell, analysis, advice, mutual funds, graphs + background_task + updateStockPrices.
├── adminPanel.c                 # Admin menu, reports, transaction/threshold request management, session/instance management.
├── chat.c                       # 1:1 messaging + group chat (create/delete/remove member/send).
├── features.c                   # Paid features: currency converter, finance manager, loan calc, savings, alerts, tags, bill pay, schedule.
├── notification.c               # addNotification / displayNotifications (positioned, read/unread, color-coded).
├── scheduledTransactions.c      # scheduleTransaction / executeScheduledTransactions / scheduleTransactionPrompt.
├── instanceHandling.c           # File-lock session machinery, instance codes, session lock/expiry, ConsoleHandler, cleanup.
├── interfaceSupport.c           # typeWriterEffect, ANSI separators, gotoxy/setCursorPosition, loading-bar, Beep tunes (winmm).
├── help.c                       # FAQ, complaint add/view, contact "directions".
├── fileHandling.c               # ALL load*/save* functions + initialize() + quickSave() + write() (the lock-wrap loader).
├── commented_out_code_archive.c # 1,638 lines of dead/legacy code. NOT in the .vcxproj; not compiled.
├── BankSystem1.7[FileIO].sln    # VS2022 solution (Format 12.00 / VS17).
├── BankSystem1.7[FileIO].vcxproj# MSBuild project — see "README vs. code" for the broken file list.
└── data files: users1, transactions, companies, investments, investment_data.txt,
    personalMessages, groups.txt, transactionRequests.txt, threshold_requests,
    complaint_data, userLog.txt, suspLogs.txt, timestamps.txt
```

**Module coupling:** every module `#include "bank.h"` and reads/writes the same global arrays. There is no encapsulation; "passing" state between menus is done by mutating globals (`currentUserIndex`, `loggedIn`, `userCount`, …) and re-reading files. A typical action follows the pattern: `write()` (acquire file lock + reload everything) → mutate global → `quickSave()` (persist everything + release lock).

---

## Code walkthrough (the important parts)

### `main.c` — bootstrap
Sets the console control handler, creates exactly **one** background thread (`background_task`), then `initialize()` → `generateUniqueInstanceCode()` → `logActiveInstance()` → `mainMenu()`. The thread handle is created but `CloseHandle` is only reached after `mainMenu()` returns; there is no join/cancel, so the ticker runs for the whole program lifetime.

### `menu.c` — navigation & RBAC
`mainMenu()` is a nested loop. Outer loop: Login / Register / Exit. After a successful `login()`, an inner loop renders a numbered menu that differs by role: **`currentUserIndex == 0` is the admin** (hard-coded — user index 0 in `users1`), everyone else is a normal user. The same number maps to different actions depending on role (e.g. option 8 is "Send Limit Change request" for users but "Delete transaction" for admin). Navigation is **`scanf("%d")` numeric input**, not arrow keys (contradicting the README's "arrow-key driven" claim). `goto mainMenuLable` is used to bail back to the top when an account/session lock trips.

### `login.c` — auth + "2FA"
- Looks up the user by UID, refuses locked accounts (and bumps a suspicious counter + session lock for repeated attempts on a locked account).
- Reads a 4-digit PIN with `getch()`, echoing `*`, supporting backspace. PINs are stored as **plaintext** `int pin[4]` — no hashing.
- On correct PIN, `generate2FACode()` returns `rand()%900000+100000` and **prints it to the screen**, then `validate2FACode()` asks the user to type back the number it just displayed. This is "2FA" only nominally — it is a demo/no-op OTP.
- `MAX_FAILED_ATTEMPTS` (3) wrong PINs → `users[i].isLocked = 1` and persisted.

### `coreFunctionalities.c` — accounts & money
- `registerUser()` validates name/PIN(4 digits)/phone(10 digits)/email(must contain `@` and `.com`)/balance(`<= MAX_BALANCE` = 20,000). New `uid = userCount + 1`. Default `transactionThreshold = 40000`.
- `updateBalance(isDeposit)` and `transferMoney()` are the money paths. Both check the per-user `transactionThreshold`: if an amount exceeds it (and the actor is not admin), instead of executing they call `addTransactionRequest(...)` to queue admin approval (encoded with sentinel recipient UIDs: `-1` deposit, `-2` withdrawal, else a real UID for transfer). Otherwise they mutate balances directly and call `addTransaction()`.
- `bankStatement()` prints profile + the transaction string array.

### `transactionProcess.c` — the central write path
`addTransaction(user, desc, amount)` formats `"<desc> : $<amount>"` into `user->transactions[]`, stamps `transactionTimestamps[]` with `time(NULL)`, increments the count, then **immediately calls `detectFraud(user)`** and `checkAlertThreshold()`. So fraud analysis runs synchronously on every single transaction. Also holds delete (single/all/other-user) and the exact-amount / range filters, which parse amounts back out of the formatted strings with `sscanf(..., "%*s : $%f", ...)`.

### `fraudDetection.c` — the anomaly engine
`detectFraud()` orchestrates several heuristics, each parsing amounts out of the transaction strings:
- **2σ rule:** `calculateAverageTransaction()` + `calculateStandardDeviation()` over recent non-admin, non-investment transactions; flags the latest if `amount > mean + 2σ` by appending `*` to the string and logging it.
- **Frequency:** `checkTransactionFrequency()` — 3+ transactions within 60 s ⇒ revert them (including the recipient side of transfers) and return fraud.
- **Near-threshold:** `checkTransactionsNearThreshold()` — amounts in `[0.85·threshold, threshold]`, 3+ such ⇒ flag.
- **Pattern:** `areLastFourTransactionsSimilar()` — last 4 amounts within a `$3` range ⇒ fraud.
- **Transfer burst:** `checkRecentTransfers()` / `checkAndHandleSuspiciousTransfers()` — 4+ transfers in last 6 transactions ⇒ revert.
- **Self-healing:** `unflagTransactions()` clears `*` flags when followed by ≥4 normal transactions.

If any trigger fires (or `*`-count ≥ 3), the account is locked, suspicious transactions reverted (`revertSuspiciousTransactions`, which also undoes the recipient side), the user is force-logged-out, and a centered warning + spinner animation plays before clearing the screen. Flags live **inside the transaction text** (a trailing `*`), which is fragile — amount parsing and string mutation are interleaved.

### `adminPanel.c` — admin tooling
Admin-only menu: list users, lock/unlock by UID, `generateReport()` (dumps every user's name/UID/balance/**PIN in cleartext**/lock status + total bank balance), manage threshold-change requests and high-value transaction requests (approve/reject with notifications), view suspicious logs and complaints, and **Instance Management** (live `activeInstances.txt` table + `userLog.txt` session-history table + lock/unlock other sessions).

### `features.c` — paid extras
`accessPaidFeatures()` is an in-app store: each item calls `advertiseFeature(name,cost)` then `confirmPurchase(cost)` (which deducts from balance) before running the feature. Items: Finance Manager (expense extraction from transaction history + budget + a "financial score"), Currency Converter (a static `Currency currencies[]` table with **53** hard-coded entries, INR-relative rates), Loan Calculator (EMI), Savings Tracker, balance alerts, transaction tagging, bill pay, and "Schedule Transactions" (cost 50). Note costs 4/5/6/7 are 0.0 — those are effectively free.

### `chat.c`, `notification.c`, `help.c`
Chat is 1:1 (by listed user) + groups (create/delete/remove member/send), all stored in `personalMessages`/`groups.txt`. Notifications are per-user, shown on the right half of the login screen via `gotoxy`, with read/unread color coding (auto-marked read on display). Help provides FAQ text, complaint submit/view, and a hard-coded "directions to branch" string.

---

## The stock-market simulation — how prices/orders work in code

All in `investmentManager.c`. A `Company` has `name, currentPrice, details, historicalPrices[30], peRatio, dividendYield, marketCap`. There are **10 companies** (`saveCompanyData`/`loadCompanyData` hard-code `companyCount = 10`), seeded from the `companies` file (e.g. `CompanyA`, current price `123.20`, with a 30-day history).

### Price movement — `updateStockPrices()`
Prices are **not** a random walk on the previous price; each tick recomputes an absolute price from a model:

```
phase  += 0.1                                   // advances each call (smoothing)
basePrice        = 100.0 + trend                // trend is a slow random drift
periodicComponent= 20.0 * sin(phase + i)        // per-company sine wave (market sentiment)
randomComponent  = (rand/RAND_MAX - 0.5) * 10   // noise
eventImpact      = (day%30==0) ? (rand-0.5)*5 : 0   // "monthly" earnings/event shock
peInfluence      = (peRatio < 20) ? 1.05 : 0.95
dividendInfluence= (dividendYield > 3) ? 1.02 : 0.98

currentPrice = basePrice*peInfluence*dividendInfluence + periodicComponent + randomComponent + eventImpact
currentPrice = max(currentPrice, 1.0)           // floor at 1.0
```

Then it shifts `historicalPrices[]` (index 0 = newest), and nudges `peRatio` (clamped 5–40) and `dividendYield` (clamped 0–10) by small random deltas. Finally `trend += (rand-0.5)*2; day++`. Because `basePrice` is ~100 for everyone, all 10 stocks track each other closely — they are differentiated mainly by the `sin(phase + i)` phase offset and their P/E and dividend influence multipliers.

`srand(time(NULL))` is re-seeded **inside** `updateStockPrices()` on every call; with a 20 s tick this is harmless but means determinism is purely time-based.

### "Orders" — there is no order book
Buying/selling is instantaneous at `currentPrice`; there are no bids/asks, no liquidity, no slippage.
- **Buy** (`addInvestment`): choose a company index, then invest by **dollar amount** (`shares = amount / currentPrice`) or by **share count** (`amount = shares * currentPrice`). Validated against balance; creates an `Investment{type:"Stock", ...}`, deducts balance, records a transaction.
- **Sell** (`sellInvestments`): sell-all credits `shares * currentPrice` and removes the holding; partial sell reduces `shares` and `amountInvested`.
- **Mutual funds** (`addMutualfund`/`processMutualFund`): picks the **3 lowest-priced** companies and splits the user's mutual-fund amount 50/30/20 across them, creating three `Investment{type:"Mutual Funds"}` rows. `sellMutualFunds` liquidates them all, charging a **10% commission** (on profit if up, on the principal if down). A large `else` block for auto-rebalancing on >20% profit is **commented out**.

### Analytics
- `analyzeInvestments()` — total invested vs current value, average return %, best/worst holding (color-coded).
- `provideInvestmentAdvice()` — per-company "Strong Buy/Buy/Hold/Value/Income/Sell" decision from `calculateVolatility()` (stddev of history), `calculateTrend()` (% change between two history indices, used for 7/14/30-day windows), `calculateGrowthRate()`, plus P/E, dividend, and market-cap thresholds. This is rule-based, not "AI" despite the README heading.
- `drawGraph()` / `viewHistoricalPrices()` — ASCII chart of the 30-day history with green/red `*` for up/down days.

---

## Concurrency — what the threads do and how they're synchronized

There is **one** real thread plus the main thread, and a separate **multi-process** coordination layer.

### The background thread — `background_task` (investmentManager.c)
```c
DWORD WINAPI background_task(LPVOID lpParam) {
    while (1) { Sleep(20000); updateStockPrices(); saveCompanyData("companies", companies); }
    return 0;
}
```
Every 20 s it mutates the global `companies[]` and overwrites the `companies` file. **It does not touch any lock file** and is not synchronized with the main thread. The main thread also reads/writes `companies` (via `initialize`/`quickSave`/`write`) on its own. So there is a genuine, unguarded **data race** on the `companies[]` array and the `companies` file: the ticker can overwrite the file while the main thread is mid-load, or stomp a price the user just traded against. This works in practice only because writes are infrequent and the file is small — it is not actually thread-safe.

### Multi-instance / session coordination — file spin-locks (instanceHandling.c, fileHandling.c)
The "multithreading/multi-session" the README advertises is mostly **inter-process** coordination via files, using busy-wait locking rather than OS primitives:
- `write()` and `logSession()`/`logUserLogout()`/`loadSessionsFromFile()` each open a lock file (`lockfile.txt`, `lockfilelog.txt`, `lockfilelogA.txt`), spin in a `while(checkLockFile()==1) printLoadingBar();` loop until it reads `0`, then write `1`, do the work, and write `0` to release. The animated loading bar **is** the wait loop.
- `activeInstances.txt` is a flat list of `Session{sessionID, currentUserID, logoutCount, sessionLock, suspiciousCount}` records (5 lines each). `loadSessionsFromFile`/`saveSessionsToFile` read/write the whole list under `lockfilelogA.txt`.
- `isUserIDInSession()` blocks login while the same UID is active in another instance.
- `checkSessionLock()` lets an admin (or the suspicious-count logic) flag a session's `sessionLock`; the affected instance detects it on its next menu pass and `exit(0)`s with a "Malicious activity suspected" animation. `logoutCount >= 3` similarly force-exits.

This is a clever-but-brittle design: the locks are advisory, depend on cooperating processes, and a crashed instance can leave a lock file stuck at `1`, hanging every other instance on the spinner. The three lock files and `activeInstances.txt` are `.gitignore`d (treated as runtime scratch).

---

## Win32 GUI layer

There is **no Win32 windowing/GDI GUI**. The "GUI layer" is a text UI built on the Win32 **console** API plus ANSI escapes, concentrated in `interfaceSupport.c` (and reused everywhere):

- **ANSI color** via raw escape sequences (`\033[1;33m`, plus the `RED`/`GREEN`/`ANSI_*` macros in `bank.h`). Relies on the Windows 10+ console honoring VT sequences.
- **Cursor positioning** — `setCursorPosition`/`gotoxy` wrap `SetConsoleCursorPosition`; `getScreenDimensions`/`getConsoleDimensions` use `GetConsoleScreenBufferInfo` to center messages, draw the right-side notification panel, and place the loading bar.
- **Masked/raw input** — `_getch()` from `conio.h` for PIN entry (echoing `*`) and "press any key" prompts.
- **Animations** — `printLoadingBar()` cycles 6 stop-motion frames (`[|||   ]` …) with `Sleep(175)`; used as the busy-wait UI during lock contention and as lock/expiry "closing" animations.
- **Audio** — `Beep()` for confirmations/errors, and `playWelcomeTune`/`playLogoutTune`/`playExitTune` schedule periodic `Beep`s via `timeSetEvent` (`winmm.lib`, pulled in with `#pragma comment(lib, "winmm.lib")`) to play 3-note C-major arpeggios.
- **Typewriter effect** — `typeWriterEffect()` prints char-by-char with `Sleep(50)`. (It accepts varargs but ignores them — it does not actually format.)

`SubSystem` in the project is `Console` for all four configs, confirming this is a console app.

---

## Persistence — data format & files

Pure **flat text files**, line-oriented, no headers/JSON/CSV — each value on its own line, parsed positionally. There is no schema versioning; field order is implicit in the matching `load*`/`save*` pair in `fileHandling.c`. `initialize()` loads everything; `quickSave()` writes everything; `write()` is `initialize` wrapped in a lock acquire.

| File | Written by | Format (per record) |
|---|---|---|
| `users1` | `saveUserData` | 14 lines/user: name, uid, balance(`%.2f`), pin(4 digits as one line e.g. `1234`), transactionCount, isLocked, address, phone, email, threshold, messageCount, alertThreshold, investmentCount, mutualFund. **Note:** notifications are *not* persisted here. |
| `transactions` | `saveUserTransactions` | per user: count line, then that many `"<desc> : $<amt>"` lines (a trailing `*` marks suspicious). |
| `timestamps.txt` | `saveUserTimestamps` | per user: count, then that many epoch seconds (`time_t` as `%ld`). |
| `companies` | `saveCompanyData` | count(=10), then per company: name, currentPrice, details, 30 history floats, peRatio, dividendYield, marketCap. |
| `investments` | `saveUsersWithInvestments` | per user: investmentCount, then per holding: type, name, amountInvested, shares, companyIndex. |
| `investment_data.txt` | `saveInvestmentData` | mutualFundCount, investmentRegisterCount, then 100 ints of `investmentList[]`. |
| `personalMessages` | `saveUsersWithMessages` | per user: messageCount, then per msg: sender, recipient, message, timestamp. |
| `groups.txt` | `saveGroups` | per group: name, admin, memberCount, members, messageCount, then per msg: sender, message, timestamp. (empty in checkout) |
| `transactionRequests.txt` | `saveTransactionRequests` | count, then per req: amount, userIndex, recipientUid (`-1`/`-2`/uid), description, isApproved. |
| `threshold_requests` | `saveThresholdRequests` | count, then per req: requestId, userId, requestedThreshold, adminRemark, isApproved, userReply. |
| `complaint_data` | `saveComplaintData` | count, then complaint lines. |
| `suspLogs.txt` | `saveSuspLogs` | count, then `"User: <name>, Transaction: <txn>"` lines. |
| `userLog.txt` | `logSession`/`logUserLogout` | append-only audit: sessionID, userIndex, uid, name, login time (`ctime`), then `"active"` later replaced in-place by the logout time. |
| `activeInstances.txt` | `saveSessionsToFile` | 5 lines/session: sessionID, currentUserID, logoutCount, sessionLock, suspiciousCount. (runtime, gitignored) |
| `lockfile*.txt` | `write`/session fns | single char `0`/`1` advisory lock. (runtime, gitignored) |

Robustness caveats observed in the data: `users1` contains `@.com` placeholder emails that nonetheless "pass" the `@`+`.com` validation; the checked-in `userLog.txt` has a malformed record (`a` where a session ID should be). Parsing is by `fgets`/`atoi`/`atof`/`sscanf` with little error recovery, so a misaligned line cascades.

---

## Tech stack & build system

- **Language:** C (MSVC dialect; uses `strcpy`/`sprintf`/`scanf` freely — the x64 configs define `_CRT_SECURE_NO_WARNINGS`).
- **Platform APIs:** `windows.h` (threads, console, `Beep`, process control), `conio.h` (`_getch`), `winmm` (`timeSetEvent` tunes), `process.h`, `math.h`, `time.h`, `stdbool.h`.
- **Build:** Visual Studio 2022 (`BankSystem1.7[FileIO].sln`, toolset **v143**, `WindowsTargetPlatformVersion 10.0`). Four configs: Debug/Release × Win32/x64; all `SubSystem=Console`, `CharacterSet=Unicode`.
- **No** CMake, Makefile, package manager, tests, or CI. `winmm.lib` is linked via `#pragma comment` in `interfaceSupport.c`.

---

## Build / run (actual commands; Win32 specifics)

**Important:** the checked-in `.vcxproj` will **not build as-is** — its `<ClCompile>` list references files that don't exist (`savingsAccount.c`, `sheduledTransactions.c`, `variableSupport.c`) and a misspelled `sheduledTransactions.c`, while the real `scheduledTransactions.c` is missing from the list (see "README vs. code"). You must fix the project's source list first, or compile from the command line.

**Visual Studio:** open the `.sln`, correct the source file list in the project, pick `x64 Debug`/`x64 Release`, Build (`Ctrl+Shift+B`), Run (`F5`). Run from the directory containing the data files (the program opens them by relative path).

**MSVC command line** (from a *Developer Command Prompt*, in the repo root) — this is the corrected file list:
```bat
cl /W3 /Fe:BankSystem.exe main.c menu.c login.c coreFunctionalities.c ^
   transactionProcess.c fileHandling.c adminPanel.c chat.c fraudDetection.c ^
   instanceHandling.c investmentManager.c features.c notification.c help.c ^
   interfaceSupport.c scheduledTransactions.c winmm.lib
```
(The README's one-liner is essentially this and is correct — but it does not match the `.vcxproj`.) Do **not** add `commented_out_code_archive.c`.

**Runtime notes:** Windows-only. Needs a VT-capable console for the colors. Admin = the user at index 0 in `users1`; the 2FA code is printed on screen. If a prior run crashed, delete leftover `lockfile*.txt` / `activeInstances.txt` to avoid the lock spinner hanging.

---

## Status, completeness & notable gaps

- **Works as a single-instance demo**, with broad but uneven feature coverage. It is clearly a student/learning project, not production software.
- **Build is broken out of the box** via the `.vcxproj` source-list mismatch (above).
- **Thread safety:** the background ticker races the main thread on `companies[]`/the `companies` file with no synchronization.
- **Security:** PINs and the "2FA" code are plaintext; `generateReport()` prints all PINs; the OTP is shown on screen. There is no real authentication beyond UID+PIN.
- **Memory/format fragility:** amounts are parsed back out of formatted strings; fraud flags are stored as a `*` inside the transaction text; many fixed-size buffers with `strcpy`/`sprintf` and `scanf("%s")` (unbounded in places like `updateProfile`'s phone/email reads) — buffer-overflow-prone.
- **Dead/half-wired features:** `executeScheduledTransactions()` is **never called** (commented out in `menu.c` and the archive), so scheduled/recurring transactions are created but **never auto-run**; mutual-fund auto-rebalance is commented out; `procterLaunch`/`launchMonitoringProgram` reference a hard-coded external `BankSystemMoniter.exe` path that won't exist on other machines.
- **Hard-coded constants:** `companyCount` forced to 10 in load/save regardless of file; admin is fixed at index 0.
- **`commented_out_code_archive.c`** (1,638 lines) is dead weight kept in-tree (not compiled).

---

## README vs. code

| README / task claim | Reality in code |
|---|---|
| "~15,000-line banking system" | **10,104** total `.c`+`.h` lines; **8,466** excluding the dead archive. Overstated by ~50%. |
| "Win32 GUI" (task framing) | No windowed GUI. It's a **console** app using the Win32 *console* API + ANSI escapes. `SubSystem=Console`. |
| `menu.c` "(arrow-key driven)" (Project Structure section) | Menus are **numeric `scanf` input**, not arrow keys. |
| "Multithreading … multiple banking sessions run simultaneously" | One real worker thread (the ticker). Multi-session is **inter-process** coordination via file spin-locks, not threads. |
| "Two-Factor Authentication … secure login" | The OTP is `rand()` **printed to the screen** and typed back — a no-op demo, not real 2FA. |
| "Background thread … saved to files" (thread-safe implied) | True it saves every 20 s, but **unsynchronized** with the main thread — a real data race. |
| "AI-Powered Investment Advice" | Rule-based threshold logic in `provideInvestmentAdvice()`. No ML/AI. |
| "Currency Converter — 50+ currencies" | Accurate — **53** hard-coded entries in `features.c`. |
| Build instructions ("Open the .sln … Build") | The `.vcxproj` lists **nonexistent** files (`savingsAccount.c`, `sheduledTransactions.c`, `variableSupport.c`) and omits the real `scheduledTransactions.c` — it won't build unedited. The README's *command-line* `cl` list, however, is correct. |
| "Scheduled transactions … Auto-execution at their scheduled time" | `executeScheduledTransactions()` exists but is **never called** (all call sites commented out). Scheduling works; auto-execution does not. |
| "License: educational purposes" | No `LICENSE` file; only the README sentence. |
| Data-file table (README) | Matches the code's `load*/save*` set. `users1` is described as holding "notifications" — but notifications are **not** persisted by `saveUserData`. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\.repo-cache\BankSystem1.7. Source of truth: https://github.com/AmeyaBorkar/BankSystem1.7*
