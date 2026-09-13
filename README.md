# FMS Attendant POS Terminal — Wails Implementation Guide

**A comprehensive guide to building a sophisticated fuel station POS terminal with Wails, Go, React, and SQLite.**

**Version:** 1.0  
**Date:** September 2026  
**Author:** Technical Documentation  
**Framework:** Wails 2.x + React 18 + Go 1.21+

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Design](#architecture--design)
3. [Prerequisites & Setup](#prerequisites--setup)
4. [Project Structure](#project-structure)
5. [Getting Started](#getting-started)
6. [Go Backend Reference](#go-backend-reference)
7. [React Frontend Components](#react-frontend-components)
8. [State Management](#state-management)
9. [Database Schema](#database-schema)
10. [Wails Patterns & Limitations](#wails-patterns--limitations)
11. [Feature Implementation Guides](#feature-implementation-guides)
12. [Testing & Debugging](#testing--debugging)
13. [Performance Optimization](#performance-optimization)
14. [Deployment](#deployment)
15. [Troubleshooting](#troubleshooting)
16. [Future Enhancements](#future-enhancements)

---

## Project Overview

### What Is This?

**FMS Attendant POS Terminal** is a sophisticated point-of-sale application designed for fuel station attendants. It runs on Wails (Electron-like, but Go-powered) and allows attendants to:

- Scan vehicle license plates
- Authorize pump dispensers
- Monitor live fueling progress
- Collect payments (cash, M-Pesa, card, credit account)
- Generate receipts
- Track running balance and transaction history
- Queue transactions offline and sync when reconnected

### Why Wails?

| Aspect | Why Wails |
|---|---|
| **Performance** | Go backend handles heavy lifting (PDF generation, SQLite, encryption) |
| **Native Feel** | Desktop app, not a browser tab — direct file system & hardware access |
| **Offline-First** | SQLite local persistence + sync queue |
| **Single Binary** | No node_modules on the machine, single executable |
| **Real-Time Updates** | Go goroutines stream live pump status to React at 10+ Hz |
| **Type Safety** | Go + TypeScript across the stack |

### Why This Project Tests Wails Well

1. **High-frequency updates** (pump progress 10x/sec) — exposes webview performance ceiling
2. **Complex state machine** (15+ screens with interdependent flows) — tests React Context + Wails bindings
3. **Offline persistence** (local SQLite queue) — exposes sync pattern limitations
4. **File generation** (PDF receipts) — shows Go strength in backend tasks
5. **Hardware simulation** (camera, printer, card reader) — tests hardware abstraction patterns

---

## Architecture & Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    WAILS APP WINDOW                         │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────┐   ┌──────────────────────┐│
│  │   REACT FRONTEND (TypeScript)│   │  Tauri/WebView2/WRY ││
│  │  ┌────────────────────────┐ │   └──────────────────────┘│
│  │  │ Pages (15 screens)     │ │   ┌──────────────────────┐│
│  │  │ Components (reusable)  │ │   │  Bridge/IPC Message  ││
│  │  │ Context (global state) │ │   └──────────────────────┘│
│  │  │ Hooks (logic)          │ │                           │
│  │  │ CSS (Material Design)  │ │                           │
│  │  └────────────────────────┘ │                           │
│  └─────────────────────────────┘                           │
│                  │ IPC Messages                            │
│                  │ (JSON serialized)                       │
├──────────────────┼────────────────────────────────────────┤
│                  ▼                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │      GO BACKEND (Main Process)                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │ Handlers (transaction, payment, auth, etc)   │   │   │
│  │  │ Models (Transaction, Shift, Payment, etc)    │   │   │
│  │  │ Database (SQLite CRUD operations)            │   │   │
│  │  │ Services (PDF generation, sync logic)        │   │   │
│  │  │ Goroutines (pump streaming, sync ticker)     │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                  │                                          │
│                  ▼                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │   SQLite Database (Local Persistence)              │   │
│  │   - transactions (current shift)                   │   │
│  │   - pending_sync (offline queue)                   │   │
│  │   - cache (recent vehicles, products)              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
        │
        │ (Simulation only — no real backend)
        │
        ▼
   DUMMY DATA
   (Fixtures in Go memory)
```

### Data Flow: Transaction Lifecycle

```
User Flow                Go Handler                 SQLite              Frontend Event
─────────────────────────────────────────────────────────────────────────────────────

START TX
  │
  ├─→ Scan Plate ──→ FetchVehicle() ──→ Query cache  ──→ emit:vehicle ──→ Update UI
  │
  ├─→ Confirm      ──→ CreateTransaction() ─→ INSERT  ──→ emit:txn_created
  │
  ├─→ Auth Pump    ──→ AuthorizePump() ─→ (no DB yet)  ──→ emit:pump_auth ──→ Status = AUTHORIZED
  │
  ├─→ Monitor      ──→ StreamPumpStatus() ─→ (goroutine)  ──→ emit:pump:progress ──→ Animate meter
  │   (live updates)    ticker every 100ms
  │
  ├─→ Nozzle Stop  ──→ CompleteFueling() ──→ UPDATE ──→ emit:fueling_complete
  │
  ├─→ Select Pay   ──→ (no action)
  │
  ├─→ Process Pay  ──→ PaymentCash/Mpesa/Card() ──→ UPDATE  ──→ emit:payment_posted
  │
  ├─→ Gen Receipt  ──→ GenerateReceipt() ──→ PDF file  ──→ emit:receipt_ready ──→ Show print UI
  │   (PDF)
  │
  └─→ Complete     ──→ PostTransaction() ──→ INSERT to shift_txns ──→ emit:txn_complete ──→ Balance updated
       (back to Home)
```

### State Management Flow

```
┌─────────────────────────────────────────┐
│         AppContext (Global)             │
├─────────────────────────────────────────┤
│ - auth (attendant ID, shift ID)         │
│ - shiftBalance (KES collected)          │
│ - connectedStatus (online/offline)      │
│ - notificationStack (errors/success)    │
└─────────────────────────────────────────┘
           │
           ├─→ useTransaction() Hook
           │   (active transaction state)
           │   - customerData
           │   - productSelected
           │   - amountKES
           │   - paymentMethod
           │   - validation errors
           │
           ├─→ usePumpStream() Hook
           │   (live updates from Go)
           │   - pumpStatus (idle/fueling/payment/error)
           │   - litres (live from PTS)
           │   - amount (live calculated)
           │   - progress %
           │
           ├─→ usePayment() Hook
           │   - paymentInProgress
           │   - paymentError
           │   - receipt
           │   - confirmationMessage
           │
           └─→ useHistory() Hook
               (transaction list + filtering)
               - transactions[]
               - filteredBy (pump, method)
               - pagination
```

---

## Prerequisites & Setup

### System Requirements

| Component | Version | Notes |
|---|---|---|
| **Go** | 1.21+ | `go version` |
| **Node.js** | 16.x+ | `npm --version` |
| **Wails CLI** | 2.5.0+ | `wails --version` |
| **Git** | Latest | Version control |
| **OS** | Windows/Mac/Linux | Tested on all three |

### Install Wails

```bash
# macOS / Linux
curl https://raw.githubusercontent.com/wailsapp/wails/master/scripts/install.sh -o - | bash

# Windows (PowerShell, Admin)
iwr https://raw.githubusercontent.com/wailsapp/wails/master/scripts/install.ps1 -usebasicparsing | iex

# Verify
wails --version
```

### Create New Project

```bash
# Create project scaffolding
wails create -n fms-attendant-pos -t react

cd fms-attendant-pos

# Install dependencies
npm install          # Frontend
go mod tidy          # Go

# Verify project structure
ls -la
# Expected: main.go, frontend/, app.go, wails.json, go.mod, etc.
```

### Install Required Go Packages

```bash
# Add to go.mod and download
go get github.com/google/uuid           # UUIDs
go get github.com/mattn/go-sqlite3      # SQLite
go get github.com/jung-kurt/gofpdf      # PDF generation
go get github.com/joho/godotenv         # .env config (optional)

go mod tidy
```

### Install React/TypeScript Packages

```bash
cd frontend

npm install --save \
  axios                    # HTTP client (simulation) \
  react-router-dom         # Routing (15 screens) \
  zustand                  # State management (optional, we use Context) \
  date-fns                 # Date formatting \
  zustand                  # Lightweight alternative to Context

# TypeScript types
npm install --save-dev \
  @types/react             \
  @types/react-dom         \
  @types/node

cd ..
```

---

## Project Structure

### Complete Directory Tree

```
fms-attendant-pos/
│
├── main.go                                 # Wails entry point + App struct
├── app.go                                  # App methods (called from React)
├── wails.json                              # Wails config
├── go.mod                                  # Go dependencies
├── go.sum
│
├── handlers/                               # Go HTTP-like handlers
│   ├── auth.go                             #   - AttendantLogin, GetShift
│   ├── transaction.go                      #   - CreateTx, AuthorizePump, CompleteFueling
│   ├── payment.go                          #   - PaymentCash, PaymentMpesa, PaymentCard, PostCredit
│   ├── receipt.go                          #   - GenerateReceipt (PDF)
│   ├── pump_stream.go                      #   - StreamPumpStatus (goroutine)
│   ├── sync.go                             #   - SyncPendingTxns, QueueOffline
│   └── vehicle.go                          #   - FetchVehicle, GetVehicleHistory
│
├── models/                                 # Data structures
│   ├── transaction.go                      #   - Transaction, FuelingUpdate, PaymentPost
│   ├── shift.go                            #   - Shift, AttendantShift
│   ├── payment.go                          #   - Payment, MpesaReceipt, CardPayment
│   ├── vehicle.go                          #   - Vehicle, CustomerProfile, CreditAccount
│   ├── product.go                          #   - Product, PricingInfo
│   └── errors.go                           #   - Custom error types
│
├── db/                                     # Database layer
│   ├── db.go                               #   - DB connection, init
│   ├── schema.go                           #   - CREATE TABLE statements
│   ├── transaction_repo.go                 #   - Insert, Update, Query txns
│   ├── vehicle_repo.go                     #   - Cache vehicle profiles
│   ├── sync_queue_repo.go                  #   - Offline queue CRUD
│   └── fixtures.go                         #   - Dummy data (vehicles, products, etc)
│
├── services/                               # Business logic
│   ├── pdf_service.go                      #   - Receipt PDF generation
│   ├── sync_service.go                     #   - Retry logic, deduplication
│   ├── payment_processor.go                #   - Payment validation, amounts
│   └── notification_service.go             #   - Event emission (pump updates, etc)
│
├── frontend/                               # React app
│   ├── index.html                          #   - Main HTML entry
│   ├── public/                             #   - Static assets (icons, fonts)
│   │   └── wails.js                        #   - Wails runtime (auto-generated)
│   │
│   ├── src/
│   │   ├── index.css                       #   - Global styles
│   │   ├── main.tsx                        #   - Entry point
│   │   │
│   │   ├── App.tsx                         #   - Root component + routing
│   │   │
│   │   ├── context/
│   │   │   └── AppContext.tsx              #   - Global state (auth, balance, online status)
│   │   │
│   │   ├── hooks/
│   │   │   ├── useTransaction.ts           #   - Transaction state + validation
│   │   │   ├── usePumpStream.ts            #   - Listen to pump updates
│   │   │   ├── usePayment.ts               #   - Payment flow state
│   │   │   ├── useHistory.ts               #   - Transaction history + filtering
│   │   │   ├── useWailsEvent.ts            #   - Wrapper for event listening
│   │   │   └── useApi.ts                   #   - Call Go handlers
│   │   │
│   │   ├── pages/
│   │   │   ├── Login.tsx                   #   - PIN entry
│   │   │   ├── Home.tsx                    #   - Pump grid (main screen)
│   │   │   ├── StartTransaction.tsx        #   - Plate scan / manual entry
│   │   │   ├── ProductAmount.tsx           #   - Product + amount selection
│   │   │   ├── FuelingProgress.tsx         #   - Live meter (animated)
│   │   │   ├── PaymentSelection.tsx        #   - Cash / MPesa / Card / Credit
│   │   │   ├── CashPayment.tsx             #   - Cash collection flow
│   │   │   ├── MpesaPayment.tsx            #   - STK push or match recent
│   │   │   ├── CardPayment.tsx             #   - Card reader simulation
│   │   │   ├── SplitPayment.tsx            #   - Multi-tender split
│   │   │   ├── Receipt.tsx                 #   - Show receipt + print
│   │   │   ├── Balance.tsx                 #   - Running total per shift
│   │   │   ├── History.tsx                 #   - Transaction list + filter
│   │   │   ├── OfflineQueue.tsx            #   - Debug: show pending syncs
│   │   │   └── Settings.tsx                #   - Attendant settings (future)
│   │   │
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   │   ├── Button.tsx              #   - Primary, secondary, outline, error
│   │   │   │   ├── Card.tsx                #   - Standard card component
│   │   │   │   ├── Input.tsx               #   - Text field
│   │   │   │   ├── Chip.tsx                #   - Status badges
│   │   │   │   ├── Modal.tsx               #   - Dialog
│   │   │   │   ├── Snackbar.tsx            #   - Toast notifications
│   │   │   │   └── Divider.tsx             #   - Visual separator
│   │   │   │
│   │   │   ├── pump/
│   │   │   │   ├── PumpCard.tsx            #   - Single pump status card
│   │   │   │   ├── PumpGrid.tsx            #   - 4-pump grid
│   │   │   │   └── StatusLegend.tsx        #   - Idle / Fueling / Payment / Error
│   │   │   │
│   │   │   ├── payment/
│   │   │   │   ├── PaymentCard.tsx         #   - Cash / MPesa / Card selector
│   │   │   │   ├── MpesaReceiptList.tsx    #   - Recent receipt matcher
│   │   │   │   ├── CashEntry.tsx           #   - Amount + change calculator
│   │   │   │   └── SplitPaymentRow.tsx     #   - Individual payment method row
│   │   │   │
│   │   │   ├── receipt/
│   │   │   │   ├── ReceiptDisplay.tsx      #   - Formatted receipt
│   │   │   │   ├── ReceiptPrinter.tsx      #   - Print button
│   │   │   │   └── ReceiptSMS.tsx          #   - SMS option
│   │   │   │
│   │   │   ├── transaction/
│   │   │   │   ├── CustomerCard.tsx        #   - Vehicle matched display
│   │   │   │   ├── ProductOption.tsx       #   - Selectable product
│   │   │   │   ├── AmountToggle.tsx        #   - KES / Litres / Fill toggle
│   │   │   │   └── QuickAmounts.tsx        #   - Preset amount buttons
│   │   │   │
│   │   │   ├── navigation/
│   │   │   │   ├── BottomNav.tsx           #   - Tab bar (Home, Balance, History)
│   │   │   │   ├── AppBar.tsx              #   - Top bar with back button
│   │   │   │   └── StatusBar.tsx           #   - Signal, battery, time
│   │   │   │
│   │   │   ├── list/
│   │   │   │   ├── TransactionList.tsx     #   - Scrollable txn list (100+ items)
│   │   │   │   └── TransactionRow.tsx      #   - Single row with tap target
│   │   │   │
│   │   │   └── common/
│   │   │       ├── LoadingSpinner.tsx
│   │   │       ├── ErrorBoundary.tsx
│   │   │       └── OfflineIndicator.tsx
│   │   │
│   │   ├── types/
│   │   │   ├── index.ts                    #   - All TypeScript interfaces
│   │   │   └── api.ts                      #   - Go handler return types
│   │   │
│   │   ├── utils/
│   │   │   ├── formatting.ts               #   - Format currency, date, phone
│   │   │   ├── validation.ts               #   - Form validation logic
│   │   │   ├── device.ts                   #   - Device detection (mobile vs desktop)
│   │   │   └── constants.ts                #   - Colors, endpoints, etc
│   │   │
│   │   ├── styles/
│   │   │   ├── colors.css                  #   - CSS variables (from Figma)
│   │   │   ├── typography.css              #   - Font sizes, weights
│   │   │   ├── spacing.css                 #   - Margins, padding scale
│   │   │   └── animations.css              #   - Transitions, keyframes
│   │   │
│   │   └── wailsjs/
│   │       ├── go/                         #   - Auto-generated Go bindings
│   │       │   └── handlers/               #   - Types mirrored from Go
│   │       └── runtime/                    #   - Wails runtime helpers
│   │
│   ├── package.json
│   ├── tsconfig.json
│   └── vite.config.ts
│
├── assets/                                 # Images, fonts
│   ├── logo.png
│   └── fonts/
│       └── segoe-ui.ttf
│
└── README.md
```

---

## Getting Started

### 1. Initialize Project

```bash
# Create Wails project
wails create -n fms-attendant-pos -t react

cd fms-attendant-pos

# Install dependencies
npm install
go mod tidy
```

### 2. Run in Development

```bash
# From root directory, start dev server
wails dev

# This will:
# - Start Vite dev server (frontend, http://localhost:5173)
# - Start Wails app window (connects to Vite)
# - Enable hot reload for React + Go
# - Open DevTools (F12)
```

**Expected output:**
```
✓ Frontend dev server started on http://localhost:5173
✓ Backend built successfully
✓ Wails app running...
```

### 3. Verify Wails Binding

The Go handlers become callable from React:

```typescript
// In React component
import { EventsEmit, EventsOn } from 'wailsjs/runtime';
import { handlers } from 'wailsjs/go/handlers/Transaction';

// Call Go function
const result = await handlers.AuthorizePump(ctx, "TXN-123", 5000);

// Listen to Go events
EventsOn('pump:progress', (update) => {
  console.log('Progress:', update.litres, update.amount);
});
```

### 4. Build for Production

```bash
# Build native executable
wails build

# Output: build/bin/fms-attendant-pos (executable)
# Can run standalone without Node.js or npm
```

---

## Go Backend Reference

### Main Application Structure

#### `main.go` — Entry Point

```go
package main

import (
    "github.com/wailsapp/wails/v2/pkg/options"
    "github.com/wailsapp/wails/v2/pkg/options/assetserver"
)

func main() {
    app := NewApp()

    err := wails.Run(&options.App{
        Title:  "FMS Attendant POS",
        Width:  390,  // Mobile width
        Height: 844,  // Mobile height
        AssetServer: &assetserver.Options{
            Assets: assets,  // Embed frontend
        },
        BackgroundColour: &options.RGBA{R: 255, G: 255, B: 255, A: 255},
        OnStartup:        app.startup,
        OnShutdown:       app.shutdown,
        Bind: []interface{}{
            app,  // Bind App methods to frontend
        },
    })

    if err != nil {
        println("Error:", err)
    }
}
```

#### `app.go` — App Structure & Lifecycle

```go
package main

import (
    "context"
    "database/sql"
    "github.com/wailsapp/wails/v2/pkg/runtime"
)

type App struct {
    ctx context.Context
    db  *sql.DB
    
    // Services
    notificationService *NotificationService
    syncService         *SyncService
}

func NewApp() *App {
    return &App{
        notificationService: NewNotificationService(),
        syncService:         NewSyncService(),
    }
}

// Startup hook (called when Wails window opens)
func (a *App) startup(ctx context.Context) {
    a.ctx = ctx
    
    // Initialize database
    var err error
    a.db, err = initDB()
    if err != nil {
        runtime.LogError(a.ctx, "DB init failed: "+err.Error())
        return
    }
    
    // Start background goroutines
    go a.startSyncTicker()    // Sync pending txns every 5 sec
    go a.streamPumpUpdates()  // Mock pump status updates
}

// Shutdown hook (cleanup)
func (a *App) shutdown(ctx context.Context) {
    if a.db != nil {
        a.db.Close()
    }
}
```

### Handler Functions (Called from React)

#### `handlers/auth.go` — Authentication

```go
package handlers

import (
    "context"
    "errors"
)

type AuthHandler struct {
    db *sql.DB
}

// AttendantLogin: Validate PIN and return shift info
func (h *AuthHandler) AttendantLogin(ctx context.Context, employeeID, pin string) (*AttendantShift, error) {
    // Dummy validation (in real app, hash PIN)
    if pin != "1234" {
        return nil, errors.New("invalid PIN")
    }
    
    // Return dummy shift data
    shift := &AttendantShift{
        ShiftID:       "SHIFT-20260901-001",
        EmployeeID:    employeeID,
        EmployeeName:  "Ali Hassan",
        ShiftType:     "MORNING",
        AssignedPumps: []string{"P1", "P2"},
        StationName:   "Main Station",
        OpenedAt:      time.Now().Add(-2 * time.Hour),
    }
    
    // Log to database
    _, err := h.db.ExecContext(ctx, `
        INSERT INTO attendant_sessions (session_id, employee_id, shift_id, started_at)
        VALUES (?, ?, ?, ?)
    `, sessionID, employeeID, shift.ShiftID, time.Now())
    
    return shift, err
}
```

#### `handlers/transaction.go` — Transaction Lifecycle

```go
package handlers

// CreateTransaction: Start new transaction
func (h *TransactionHandler) CreateTransaction(ctx context.Context, req CreateTxnRequest) (*Transaction, error) {
    txn := &Transaction{
        ID:        uuid.New().String(),
        ShiftID:   req.ShiftID,
        PumpID:    req.PumpID,
        Plate:     req.Plate,
        CustomerID: req.CustomerID,
        Status:    "CREATED",
        CreatedAt: time.Now(),
    }
    
    _, err := h.db.ExecContext(ctx, `
        INSERT INTO transactions (id, shift_id, pump_id, plate, status, created_at)
        VALUES (?, ?, ?, ?, ?, ?)
    `, txn.ID, txn.ShiftID, txn.PumpID, txn.Plate, txn.Status, txn.CreatedAt)
    
    return txn, err
}

// AuthorizePump: Send pump authorization + start streaming
func (h *TransactionHandler) AuthorizePump(ctx context.Context, txnID string, maxKES float64) error {
    // Update transaction status
    _, err := h.db.ExecContext(ctx, `
        UPDATE transactions SET status = 'AUTHORIZED', max_kES = ? WHERE id = ?
    `, maxKES, txnID)
    
    // Emit "authorized" event
    runtime.EventsEmit(ctx, "pump:authorized", map[string]interface{}{
        "txnID": txnID,
        "status": "AUTHORIZED",
    })
    
    // Start fueling simulation (goroutine)
    go h.simulateFueling(ctx, txnID, maxKES)
    
    return err
}

// simulateFueling: Stream live updates every 100ms
func (h *TransactionHandler) simulateFueling(ctx context.Context, txnID string, maxKES float64) {
    started := time.Now()
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    flowRate := 6.5 // litres per second
    pricePerLitre := 182.0
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            elapsed := time.Since(started).Seconds()
            litres := elapsed * flowRate
            amount := litres * pricePerLitre
            
            if amount >= maxKES {
                // Complete fueling
                h.completeFueling(ctx, txnID, litres, amount)
                return
            }
            
            // Emit progress
            runtime.EventsEmit(ctx, "pump:progress", FuelingUpdate{
                TxnID:    txnID,
                Litres:   litres,
                Amount:   amount,
                Progress: amount / maxKES,
                Elapsed:  elapsed,
            })
        }
    }
}

// CompleteFueling: Finalize pump transaction
func (h *TransactionHandler) completeFueling(ctx context.Context, txnID string, litres, amount float64) error {
    ptsRef := fmt.Sprintf("TXN-%d", time.Now().Unix())
    
    _, err := h.db.ExecContext(ctx, `
        UPDATE transactions 
        SET status = 'COMPLETED', volume = ?, final_amount = ?, pts_ref = ?
        WHERE id = ?
    `, litres, amount, ptsRef, txnID)
    
    runtime.EventsEmit(ctx, "pump:complete", FuelingComplete{
        TxnID:     txnID,
        Volume:    litres,
        Amount:    amount,
        PTSRef:    ptsRef,
        Timestamp: time.Now(),
    })
    
    return err
}
```

#### `handlers/payment.go` — Payment Processing

```go
package handlers

// PaymentCash: Record cash payment
func (h *PaymentHandler) PaymentCash(ctx context.Context, txnID, amountTendered, changeAmount float64) error {
    payment := &Payment{
        ID:       uuid.New().String(),
        TxnID:    txnID,
        Method:   "CASH",
        Amount:   amountTendered,
        Change:   changeAmount,
        PostedAt: time.Now(),
    }
    
    _, err := h.db.ExecContext(ctx, `
        INSERT INTO payments (id, txn_id, method, amount, change, posted_at)
        VALUES (?, ?, ?, ?, ?, ?)
    `, payment.ID, txnID, payment.Method, amountTendered, changeAmount, payment.PostedAt)
    
    if err == nil {
        runtime.EventsEmit(ctx, "payment:posted", payment)
    }
    return err
}

// PaymentMpesa: STK Push or manual receipt
func (h *PaymentHandler) PaymentMpesa(ctx context.Context, req MpesaPaymentRequest) (*MpesaResponse, error) {
    // Simulate STK push
    if req.Mode == "STK_PUSH" {
        // In real app, call Daraja STK API
        // For now, simulate with random delay
        
        time.Sleep(time.Duration(rand.Intn(2000)+500) * time.Millisecond)
        
        // Random success (70% chance)
        if rand.Float32() < 0.7 {
            receipt := &MpesaReceipt{
                ReceiptNumber: fmt.Sprintf("QAB%d", time.Now().UnixNano()%1000000),
                Amount:        req.Amount,
                Sender:        "JOHN KAMAU",
                Timestamp:     time.Now(),
            }
            
            // Record in DB
            h.recordMpesaPayment(ctx, req.TxnID, receipt)
            
            runtime.EventsEmit(ctx, "mpesa:confirmed", receipt)
            return &MpesaResponse{Status: "SUCCESS", Receipt: receipt}, nil
        } else {
            runtime.EventsEmit(ctx, "mpesa:timeout", nil)
            return nil, errors.New("STK timeout")
        }
    }
    
    // Mode: "MATCH_RECENT" — return recent receipts
    if req.Mode == "MATCH_RECENT" {
        recents := h.getRecentMpesaReceipts(ctx, req.Amount, 30)
        return &MpesaResponse{Status: "PENDING", Receipts: recents}, nil
    }
    
    return nil, errors.New("invalid mode")
}

// PaymentCard: Simulate card reader
func (h *PaymentHandler) PaymentCard(ctx context.Context, txnID string, amount float64) error {
    // Simulate card processing delay
    time.Sleep(time.Duration(rand.Intn(3000)+1000) * time.Millisecond)
    
    // 95% approval rate
    if rand.Float32() < 0.95 {
        payment := &Payment{
            ID:         uuid.New().String(),
            TxnID:      txnID,
            Method:     "CARD",
            Amount:     amount,
            AuthCode:   fmt.Sprintf("%06d", rand.Intn(1000000)),
            CardLast4:  "4821",
            PostedAt:   time.Now(),
        }
        
        _, err := h.db.ExecContext(ctx, `
            INSERT INTO payments (id, txn_id, method, amount, auth_code, card_last4, posted_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        `, payment.ID, txnID, payment.Method, amount, payment.AuthCode, payment.CardLast4, payment.PostedAt)
        
        runtime.EventsEmit(ctx, "payment:card_approved", payment)
        return err
    } else {
        runtime.EventsEmit(ctx, "payment:card_declined", "Insufficient funds")
        return errors.New("card declined")
    }
}
```

#### `handlers/receipt.go` — PDF Generation (Go Strength)

```go
package handlers

import "github.com/jung-kurt/gofpdf"

type ReceiptHandler struct {
    db *sql.DB
}

// GenerateReceipt: Create PDF receipt
func (h *ReceiptHandler) GenerateReceipt(ctx context.Context, txnID string) (string, error) {
    // Fetch transaction + payment details
    txn, payment, err := h.fetchTxnDetails(ctx, txnID)
    if err != nil {
        return "", err
    }
    
    // Create PDF
    pdf := gofpdf.New("P", "mm", "A4", "")
    pdf.AddPage()
    
    // Header
    pdf.SetFont("Arial", "B", 16)
    pdf.Cell(0, 10, "⛽ FMS STATION", 0, 1, "C")
    
    pdf.SetFont("Arial", "", 10)
    pdf.Cell(0, 5, "Nairobi Station", 0, 1, "C")
    pdf.Ln(3)
    
    // Transaction details
    pdf.SetFont("Arial", "B", 12)
    pdf.Cell(0, 6, "RECEIPT", 0, 1, "C")
    pdf.SetFont("Arial", "", 10)
    
    pdf.Cell(50, 5, "Txn #:")
    pdf.Cell(0, 5, txn.ID, 0, 1)
    
    pdf.Cell(50, 5, "Vehicle:")
    pdf.Cell(0, 5, txn.Plate, 0, 1)
    
    pdf.Cell(50, 5, "Product:")
    pdf.Cell(0, 5, txn.Product, 0, 1)
    
    pdf.Cell(50, 5, "Volume:")
    pdf.Cell(0, 5, fmt.Sprintf("%.2f L", txn.Volume), 0, 1)
    
    // Amount (bold & large)
    pdf.Ln(2)
    pdf.SetFont("Arial", "B", 14)
    pdf.Cell(50, 8, "AMOUNT:")
    pdf.Cell(0, 8, fmt.Sprintf("KES %.2f", txn.Amount), 0, 1)
    
    // Payment method
    pdf.SetFont("Arial", "", 10)
    pdf.Ln(2)
    pdf.Cell(50, 5, "Payment:")
    pdf.Cell(0, 5, payment.Method, 0, 1)
    
    if payment.Method == "MPESA" {
        pdf.Cell(50, 5, "Receipt:")
        pdf.Cell(0, 5, payment.MpesaReceipt, 0, 1)
    }
    
    pdf.Cell(50, 5, "Time:")
    pdf.Cell(0, 5, txn.CreatedAt.Format("15:04:05"), 0, 1)
    
    pdf.Cell(50, 5, "Attendant:")
    pdf.Cell(0, 5, txn.AttendantName, 0, 1)
    
    // Footer
    pdf.Ln(3)
    pdf.SetFont("Arial", "", 8)
    pdf.Cell(0, 4, "Thank you for your business!", 0, 1, "C")
    
    // Save to temp file
    filename := filepath.Join(os.TempDir(), fmt.Sprintf("receipt_%d.pdf", time.Now().Unix()))
    err = pdf.OutputFileAndClose(filename)
    
    if err == nil {
        runtime.LogInfof(ctx, "Receipt generated: %s", filename)
    }
    
    return filename, err
}
```

### Models (Data Structures)

#### `models/transaction.go`

```go
package models

import "time"

type Transaction struct {
    ID            string    `json:"id"`
    ShiftID       string    `json:"shift_id"`
    PumpID        string    `json:"pump_id"`
    AttendantID   string    `json:"attendant_id"`
    Plate         string    `json:"plate"`           // Vehicle plate
    CustomerID    string    `json:"customer_id"`     // May be null for walk-in
    Product       string    `json:"product"`         // Diesel / Petrol / Premium
    Volume        float64   `json:"volume"`          // Litres
    Amount        float64   `json:"amount"`          // KES
    Status        string    `json:"status"`          // CREATED, AUTHORIZED, COMPLETED, PAYMENT_PENDING
    PTSRef        string    `json:"pts_ref"`         // Pump timestamp reference
    CreatedAt     time.Time `json:"created_at"`
    CompletedAt   *time.Time `json:"completed_at"`
}

type FuelingUpdate struct {
    TxnID    string  `json:"txn_id"`
    Litres   float64 `json:"litres"`
    Amount   float64 `json:"amount"`
    Progress float64 `json:"progress"`   // 0.0 - 1.0
    Elapsed  float64 `json:"elapsed"`    // seconds
}

type FuelingComplete struct {
    TxnID     string    `json:"txn_id"`
    Volume    float64   `json:"volume"`
    Amount    float64   `json:"amount"`
    PTSRef    string    `json:"pts_ref"`
    Timestamp time.Time `json:"timestamp"`
}
```

#### `models/payment.go`

```go
package models

type Payment struct {
    ID           string    `json:"id"`
    TxnID        string    `json:"txn_id"`
    Method       string    `json:"method"`    // CASH, MPESA, CARD, CREDIT
    Amount       float64   `json:"amount"`
    Change       *float64  `json:"change"`    // For cash
    MpesaReceipt string    `json:"mpesa_receipt"`  // For MPesa
    AuthCode     string    `json:"auth_code"`      // For card
    CardLast4    string    `json:"card_last4"`     // For card
    PostedAt     time.Time `json:"posted_at"`
}

type MpesaReceipt struct {
    ReceiptNumber string    `json:"receipt_number"`
    Sender        string    `json:"sender"`
    Amount        float64   `json:"amount"`
    Timestamp     time.Time `json:"timestamp"`
}

type SplitPayment struct {
    CashAmount   float64 `json:"cash_amount"`
    MpesaAmount  float64 `json:"mpesa_amount"`
    CardAmount   float64 `json:"card_amount"`
    CreditAmount float64 `json:"credit_amount"`
}
```

### Database Layer

#### `db/schema.go` — Table Definitions

```go
package db

const schema = `
CREATE TABLE IF NOT EXISTS transactions (
    id TEXT PRIMARY KEY,
    shift_id TEXT NOT NULL,
    pump_id TEXT NOT NULL,
    attendant_id TEXT NOT NULL,
    plate TEXT,
    customer_id TEXT,
    product TEXT,
    volume REAL,
    amount REAL,
    status TEXT,
    pts_ref TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME,
    FOREIGN KEY (shift_id) REFERENCES shifts(id)
);

CREATE TABLE IF NOT EXISTS payments (
    id TEXT PRIMARY KEY,
    txn_id TEXT NOT NULL,
    method TEXT,      -- CASH, MPESA, CARD, CREDIT
    amount REAL,
    change REAL,
    mpesa_receipt TEXT,
    auth_code TEXT,
    card_last4 TEXT,
    posted_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (txn_id) REFERENCES transactions(id)
);

CREATE TABLE IF NOT EXISTS pending_sync (
    id TEXT PRIMARY KEY,
    txn_id TEXT,
    payload TEXT,      -- JSON
    retries INT DEFAULT 0,
    last_retry DATETIME,
    synced BOOLEAN DEFAULT 0,
    FOREIGN KEY (txn_id) REFERENCES transactions(id)
);

CREATE TABLE IF NOT EXISTS vehicle_cache (
    plate TEXT PRIMARY KEY,
    customer_id TEXT,
    customer_name TEXT,
    account_type TEXT,
    credit_limit REAL,
    credit_used REAL,
    cached_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS attendant_sessions (
    session_id TEXT PRIMARY KEY,
    employee_id TEXT,
    shift_id TEXT,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME
);

CREATE TABLE IF NOT EXISTS shifts (
    id TEXT PRIMARY KEY,
    station_id TEXT,
    shift_type TEXT,  -- MORNING, AFTERNOON, NIGHT
    opened_at DATETIME,
    closed_at DATETIME
);
`

func InitDB(dbPath string) (*sql.DB, error) {
    db, err := sql.Open("sqlite3", dbPath)
    if err != nil {
        return nil, err
    }
    
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    if _, err := db.Exec(schema); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

#### `db/transaction_repo.go` — CRUD Operations

```go
package db

import (
    "context"
    "database/sql"
    "models"
)

type TransactionRepo struct {
    db *sql.DB
}

// Create
func (r *TransactionRepo) Insert(ctx context.Context, txn *models.Transaction) error {
    query := `
        INSERT INTO transactions 
        (id, shift_id, pump_id, attendant_id, plate, customer_id, product, status, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
    `
    _, err := r.db.ExecContext(ctx, query,
        txn.ID, txn.ShiftID, txn.PumpID, txn.AttendantID, txn.Plate, 
        txn.CustomerID, txn.Product, txn.Status, txn.CreatedAt)
    return err
}

// Read
func (r *TransactionRepo) GetByID(ctx context.Context, txnID string) (*models.Transaction, error) {
    query := `SELECT id, shift_id, pump_id, plate, product, volume, amount, status, created_at FROM transactions WHERE id = ?`
    
    txn := &models.Transaction{}
    err := r.db.QueryRowContext(ctx, query, txnID).Scan(
        &txn.ID, &txn.ShiftID, &txn.PumpID, &txn.Plate, &txn.Product,
        &txn.Volume, &txn.Amount, &txn.Status, &txn.CreatedAt)
    
    return txn, err
}

// Update
func (r *TransactionRepo) Update(ctx context.Context, txn *models.Transaction) error {
    query := `UPDATE transactions SET volume = ?, amount = ?, status = ?, completed_at = ? WHERE id = ?`
    _, err := r.db.ExecContext(ctx, query, txn.Volume, txn.Amount, txn.Status, txn.CompletedAt, txn.ID)
    return err
}

// List by shift
func (r *TransactionRepo) GetByShift(ctx context.Context, shiftID string, limit int) ([]models.Transaction, error) {
    query := `SELECT id, shift_id, pump_id, plate, product, volume, amount, status, created_at 
              FROM transactions WHERE shift_id = ? ORDER BY created_at DESC LIMIT ?`
    
    rows, err := r.db.QueryContext(ctx, query, shiftID, limit)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var txns []models.Transaction
    for rows.Next() {
        txn := models.Transaction{}
        err := rows.Scan(&txn.ID, &txn.ShiftID, &txn.PumpID, &txn.Plate, &txn.Product,
            &txn.Volume, &txn.Amount, &txn.Status, &txn.CreatedAt)
        if err != nil {
            return nil, err
        }
        txns = append(txns, txn)
    }
    
    return txns, rows.Err()
}
```

---

## React Frontend Components

### Page Components (Screens 01–19)

#### `pages/Login.tsx` — Attendant PIN Entry

```typescript
import React, { useState, useContext } from 'react';
import { AppContext } from '../context/AppContext';
import { AttendantLogin } from 'wailsjs/go/handlers/Auth';

export const Login: React.FC = () => {
    const { setAuth } = useContext(AppContext);
    const [pin, setPin] = useState('');
    const [error, setError] = useState('');
    const [loading, setLoading] = useState(false);

    const handlePINAdd = (digit: string) => {
        if (pin.length < 6) {
            setPin(pin + digit);
        }
    };

    const handlePINClear = () => {
        setPin(pin.slice(0, -1));
    };

    const handleLogin = async () => {
        setLoading(true);
        setError('');
        
        try {
            const result = await AttendantLogin('EMP-1042', pin);
            setAuth({
                attendantID: result.employee_id,
                attendantName: result.employee_name,
                shiftID: result.shift_id,
                shiftType: result.shift_type,
                assignedPumps: result.assigned_pumps,
            });
        } catch (err) {
            setError('Invalid PIN');
        } finally {
            setLoading(false);
        }
    };

    return (
        <div className="login-screen">
            <div className="login-header">
                <div className="pump-emoji">⛽</div>
                <h1>FMS Attendant</h1>
                <p>Fuel Management System</p>
            </div>

            <Card>
                <div className="shift-info">
                    <Chip label="MORNING SHIFT" icon="🕐" />
                    <Chip label="MAIN Station" icon="📍" />
                </div>

                <Input 
                    label="Employee ID" 
                    value="EMP-1042" 
                    readOnly 
                />

                <label className="input-label">PIN</label>
                <div className="pin-dots">
                    {[1, 2, 3, 4].map((i) => (
                        <div 
                            key={i} 
                            className={`pin-dot ${i <= pin.length ? 'filled' : ''}`}
                        />
                    ))}
                </div>

                <div className="pin-pad">
                    {[1, 2, 3, 4, 5, 6, 7, 8, 9, 0].map((num) => (
                        <Button 
                            key={num}
                            onClick={() => handlePINAdd(String(num))}
                            variant="secondary"
                        >
                            {num}
                        </Button>
                    ))}
                    <Button onClick={handlePINClear} variant="error">⌫</Button>
                </div>
            </Card>

            {error && <Snackbar message={error} type="error" />}

            <Button 
                onClick={handleLogin} 
                variant="primary" 
                loading={loading}
            >
                SIGN IN
            </Button>
        </div>
    );
};
```

#### `pages/Home.tsx` — Pump Grid + Overview

```typescript
import React, { useContext, useEffect, useState } from 'react';
import { AppContext } from '../context/AppContext';
import { PumpCard } from '../components/pump/PumpCard';
import { EventsOn } from 'wailsjs/runtime';

interface Pump {
    id: string;
    status: 'idle' | 'fueling' | 'payment' | 'error';
    currentPlate?: string;
    currentAmount?: number;
}

export const Home: React.FC = () => {
    const { auth, shiftBalance } = useContext(AppContext);
    const [pumps, setPumps] = useState<Pump[]>([
        { id: 'P1', status: 'idle' },
        { id: 'P2', status: 'idle' },
        { id: 'P3', status: 'idle' },
        { id: 'P4', status: 'idle' },
    ]);

    useEffect(() => {
        // Listen to pump status updates from Go
        const unsubscribe = EventsOn('pump:update', (update: any) => {
            setPumps((prev) =>
                prev.map((p) =>
                    p.id === update.pump_id ? { ...p, ...update } : p
                )
            );
        });

        return unsubscribe;
    }, []);

    return (
        <div className="home-screen">
            <AppBar>
                <div className="app-bar-title">{auth?.attendantName}</div>
                <div className="app-bar-subtitle">{auth?.shiftType} Shift — Pumps {auth?.assignedPumps.join(', ')}</div>
                <div className="app-bar-balance">
                    <Chip label={`KES ${shiftBalance.toLocaleString()}`} color="primary" />
                    <span className="txn-count">0 transactions</span>
                </div>
            </AppBar>

            <div className="content">
                <h3 className="section-title">My Pumps</h3>
                
                <div className="pump-grid">
                    {pumps.map((pump) => (
                        <PumpCard
                            key={pump.id}
                            pump={pump}
                            onStartTransaction={() => navigateTo(`/start-transaction/${pump.id}`)}
                        />
                    ))}
                </div>

                <h3 className="section-title">Status Legend</h3>
                <div className="status-legend">
                    <Chip icon="⚪" label="Idle" />
                    <Chip icon="🟢" label="Fueling" />
                    <Chip icon="🟡" label="Payment" />
                    <Chip icon="🔴" label="Error" />
                </div>
            </div>

            <BottomNav />
        </div>
    );
};
```

#### `pages/FuelingProgress.tsx` — Live Meter Animation

```typescript
import React, { useContext, useEffect, useState } from 'react';
import { usePumpStream } from '../hooks/usePumpStream';
import { EventsOn } from 'wailsjs/runtime';

export const FuelingProgress: React.FC<{ txnID: string }> = ({ txnID }) => {
    const [litres, setLitres] = useState(0);
    const [amount, setAmount] = useState(0);
    const [progress, setProgress] = useState(0);
    const [elapsed, setElapsed] = useState(0);

    useEffect(() => {
        const unsubscribe = EventsOn('pump:progress', (update: any) => {
            if (update.txn_id === txnID) {
                setLitres(update.litres);
                setAmount(update.amount);
                setProgress(update.progress);
                setElapsed(update.elapsed);
            }
        });

        return unsubscribe;
    }, [txnID]);

    return (
        <div className="fueling-screen">
            <div className="fueling-header">Pump 1 — FUELING</div>

            <div className="fueling-display">
                <div className="litres pulse">{litres.toFixed(2)}</div>
                <div className="litres-label">litres</div>

                <div className="amount">KES {amount.toLocaleString()}</div>

                <div className="progress-bar">
                    <div 
                        className="progress-fill" 
                        style={{ width: `${progress * 100}%` }}
                    />
                </div>

                <div className="progress-labels">
                    <span>KES 0</span>
                    <span>{Math.round(progress * 100)}%</span>
                    <span>KES 5,000</span>
                </div>

                <div className="elapsed">⏱ Elapsed: {elapsed.toFixed(2)}s</div>
            </div>

            <Button onClick={handleEmergencyStop} variant="error">
                STOP PUMP (EMERGENCY)
            </Button>
        </div>
    );
};
```

---

### Reusable Components

#### `components/ui/Button.tsx`

```typescript
import React from 'react';
import './Button.css';

interface ButtonProps {
    onClick?: () => void;
    children: React.ReactNode;
    variant?: 'primary' | 'secondary' | 'outline' | 'error' | 'success';
    size?: 'sm' | 'md' | 'lg';
    disabled?: boolean;
    loading?: boolean;
    className?: string;
}

export const Button: React.FC<ButtonProps> = ({
    onClick,
    children,
    variant = 'primary',
    size = 'md',
    disabled = false,
    loading = false,
    className = '',
}) => {
    return (
        <button
            onClick={onClick}
            className={`btn btn-${variant} btn-${size} ${className}`}
            disabled={disabled || loading}
        >
            {loading ? <span className="spinner" /> : children}
        </button>
    );
};
```

#### `components/ui/Card.tsx`

```typescript
import React from 'react';
import './Card.css';

interface CardProps {
    children: React.ReactNode;
    elevated?: boolean;
    className?: string;
}

export const Card: React.FC<CardProps> = ({ 
    children, 
    elevated = false, 
    className = '' 
}) => {
    return (
        <div className={`card ${elevated ? 'card-elevated' : ''} ${className}`}>
            {children}
        </div>
    );
};
```

#### `components/pump/PumpCard.tsx`

```typescript
import React from 'react';
import { Card } from '../ui/Card';
import { Button } from '../ui/Button';

interface Pump {
    id: string;
    status: 'idle' | 'fueling' | 'payment' | 'error';
    currentPlate?: string;
    currentAmount?: number;
}

interface PumpCardProps {
    pump: Pump;
    onStartTransaction: () => void;
}

export const PumpCard: React.FC<PumpCardProps> = ({ pump, onStartTransaction }) => {
    const statusColor = {
        idle: '#9E9E9E',
        fueling: '#4CAF50',
        payment: '#FFC107',
        error: '#F44336',
    };

    return (
        <Card className={`pump-card status-${pump.status}`}>
            <div className="pump-num">{pump.id}</div>
            
            <div className="pump-status">
                <span 
                    className="status-dot" 
                    style={{ background: statusColor[pump.status] }}
                />
                {pump.status.toUpperCase()}
            </div>

            {pump.status === 'idle' && (
                <Button onClick={onStartTransaction} variant="primary" size="sm">
                    START TX
                </Button>
            )}

            {pump.status === 'fueling' && (
                <>
                    <div className="pump-detail">{pump.currentPlate}</div>
                    <div className="pump-amount">KES {pump.currentAmount?.toLocaleString()}</div>
                </>
            )}

            {pump.status === 'error' && (
                <div className="pump-error">Nozzle fault</div>
            )}
        </Card>
    );
};
```

---

## State Management

### Context & Hooks Pattern

#### `context/AppContext.tsx` — Global State

```typescript
import React, { createContext, useState, ReactNode } from 'react';

interface AuthState {
    attendantID: string;
    attendantName: string;
    shiftID: string;
    shiftType: string;
    assignedPumps: string[];
}

interface AppContextType {
    auth: AuthState | null;
    setAuth: (auth: AuthState) => void;
    shiftBalance: number;
    setShiftBalance: (balance: number) => void;
    isOnline: boolean;
    setIsOnline: (online: boolean) => void;
    notifications: Array<{ id: string; message: string; type: 'success' | 'error' | 'info' }>;
    addNotification: (message: string, type: 'success' | 'error' | 'info') => void;
}

export const AppContext = createContext<AppContextType | undefined>(undefined);

export const AppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [auth, setAuth] = useState<AuthState | null>(null);
    const [shiftBalance, setShiftBalance] = useState(0);
    const [isOnline, setIsOnline] = useState(true);
    const [notifications, setNotifications] = useState<any[]>([]);

    const addNotification = (message: string, type: 'success' | 'error' | 'info') => {
        const id = Date.now().toString();
        setNotifications((prev) => [...prev, { id, message, type }]);

        // Auto-remove after 4s
        setTimeout(() => {
            setNotifications((prev) => prev.filter((n) => n.id !== id));
        }, 4000);
    };

    return (
        <AppContext.Provider
            value={{
                auth,
                setAuth,
                shiftBalance,
                setShiftBalance,
                isOnline,
                setIsOnline,
                notifications,
                addNotification,
            }}
        >
            {children}
        </AppContext.Provider>
    );
};
```

#### `hooks/useTransaction.ts` — Transaction State

```typescript
import { useState, useCallback } from 'react';

interface TransactionState {
    txnID?: string;
    plate?: string;
    customer?: {
        name: string;
        creditLimit: number;
        creditUsed: number;
    };
    product?: string;
    amountKES?: number;
    estimatedLitres?: number;
    paymentMethod?: string;
    status?: 'CREATED' | 'AUTHORIZED' | 'FUELING' | 'PAYMENT_PENDING' | 'COMPLETED';
    errors?: Record<string, string>;
}

export const useTransaction = () => {
    const [state, setState] = useState<TransactionState>({});

    const setPlate = useCallback((plate: string) => {
        setState((prev) => ({ ...prev, plate }));
    }, []);

    const setCustomer = useCallback((customer: any) => {
        setState((prev) => ({ ...prev, customer }));
    }, []);

    const setProduct = useCallback((product: string) => {
        setState((prev) => ({ ...prev, product }));
    }, []);

    const setAmount = useCallback((amountKES: number) => {
        const pricePerLitre = 182; // Diesel
        setState((prev) => ({
            ...prev,
            amountKES,
            estimatedLitres: amountKES / pricePerLitre,
        }));
    }, []);

    const validate = useCallback(() => {
        const errors: Record<string, string> = {};

        if (!state.plate) errors.plate = 'Plate required';
        if (!state.product) errors.product = 'Select product';
        if (!state.amountKES || state.amountKES <= 0) errors.amount = 'Amount must be > 0';
        if (!state.paymentMethod) errors.payment = 'Select payment method';

        if (Object.keys(errors).length > 0) {
            setState((prev) => ({ ...prev, errors }));
            return false;
        }

        return true;
    }, [state]);

    const reset = useCallback(() => {
        setState({});
    }, []);

    return {
        state,
        setPlate,
        setCustomer,
        setProduct,
        setAmount,
        validate,
        reset,
    };
};
```

#### `hooks/usePumpStream.ts` — Real-Time Pump Updates

```typescript
import { useEffect, useState, useCallback } from 'react';
import { EventsOn, EventsOff } from 'wailsjs/runtime';

interface PumpUpdate {
    litres: number;
    amount: number;
    progress: number;
    elapsed: number;
}

export const usePumpStream = (txnID?: string) => {
    const [update, setUpdate] = useState<PumpUpdate | null>(null);
    const [status, setStatus] = useState<'idle' | 'fueling' | 'complete' | 'error'>('idle');

    useEffect(() => {
        if (!txnID) return;

        // Listen to progress updates
        const unsubProgress = EventsOn('pump:progress', (data: any) => {
            if (data.txn_id === txnID) {
                setUpdate({
                    litres: data.litres,
                    amount: data.amount,
                    progress: data.progress,
                    elapsed: data.elapsed,
                });
                setStatus('fueling');
            }
        });

        // Listen to completion
        const unsubComplete = EventsOn('pump:complete', (data: any) => {
            if (data.txn_id === txnID) {
                setStatus('complete');
            }
        });

        return () => {
            unsubProgress();
            unsubComplete();
        };
    }, [txnID]);

    return { update, status };
};
```

---

## Database Schema

### Table Structure

```sql
-- Transactions
CREATE TABLE transactions (
    id TEXT PRIMARY KEY,
    shift_id TEXT NOT NULL,
    pump_id TEXT NOT NULL,
    attendant_id TEXT NOT NULL,
    plate TEXT,
    customer_id TEXT,
    product TEXT,
    volume REAL,
    amount REAL,
    status TEXT,
    pts_ref TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME
);

-- Payments
CREATE TABLE payments (
    id TEXT PRIMARY KEY,
    txn_id TEXT NOT NULL,
    method TEXT,
    amount REAL,
    change REAL,
    mpesa_receipt TEXT,
    auth_code TEXT,
    card_last4 TEXT,
    posted_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (txn_id) REFERENCES transactions(id)
);

-- Offline Queue
CREATE TABLE pending_sync (
    id TEXT PRIMARY KEY,
    txn_id TEXT NOT NULL,
    payload TEXT,
    retries INT DEFAULT 0,
    last_retry DATETIME,
    synced BOOLEAN DEFAULT 0,
    FOREIGN KEY (txn_id) REFERENCES transactions(id)
);

-- Vehicle Cache
CREATE TABLE vehicle_cache (
    plate TEXT PRIMARY KEY,
    customer_id TEXT,
    customer_name TEXT,
    account_type TEXT,
    credit_limit REAL,
    credit_used REAL,
    cached_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_txn_shift ON transactions(shift_id);
CREATE INDEX idx_txn_pump ON transactions(pump_id);
CREATE INDEX idx_payment_txn ON payments(txn_id);
CREATE INDEX idx_pending_synced ON pending_sync(synced);
```

---

## Wails Patterns & Limitations

### Pattern 1: Go → React Event Streaming

**Use Case:** Pump progress updates at 10 Hz

```go
// Go: Stream updates
go func() {
    ticker := time.NewTicker(100 * time.Millisecond)
    for range ticker.C {
        runtime.EventsEmit(ctx, "pump:progress", update)
    }
}()
```

```typescript
// React: Listen
useEffect(() => {
    const unsub = EventsOn('pump:progress', (update) => {
        setLitres(update.litres);
    });
    return unsub;
}, []);
```

**Limitation:** Message ordering not guaranteed. Add sequence numbers:

```go
runtime.EventsEmit(ctx, "pump:progress", map[string]interface{}{
    "seq": atomicSeq.AddUint64(1),
    "litres": litres,
})
```

### Pattern 2: Bidirectional RPC-Style Calls

**Go Handler Called from React:**

```typescript
// React calls Go
const result = await AuthorizePump(txnID, maxKES);
```

```go
// Go handler
func (h *Handler) AuthorizePump(ctx context.Context, txnID string, maxKES float64) error {
    // ... do work ...
    return nil
}
```

**Limitation:** No streaming responses. For long operations, use events + background goroutines.

### Pattern 3: Local Persistence + Sync Queue

**SQLite Queue Pattern:**

```go
// Store locally when offline
func (s *SyncService) EnqueueTransaction(txn *Transaction) error {
    _, err := s.db.Exec(`
        INSERT INTO pending_sync (id, txn_id, payload, synced)
        VALUES (?, ?, ?, 0)
    `, uuid.New().String(), txn.ID, jsonMarshal(txn))
    return err
}

// Retry every 5 seconds
go func() {
    ticker := time.NewTicker(5 * time.Second)
    for range ticker.C {
        s.syncPendingTransactions()
    }
}()
```

**Limitation:** No conflict resolution. Dummy data = no actual backend, so "sync" is simulated.

### Known Limitations

| Limitation | Workaround |
|---|---|
| **No message ordering guarantee** | Add sequence numbers to events |
| **No WebSocket streaming** | Use polling (tick every 100ms) or event-based updates |
| **Binary data requires disk write** | Save PDF to temp, then serve via file:// |
| **Camera access limited** | Use file picker for plate scan, or embed camera.js library |
| **No service worker** | Manual sync + retry queue in Go + SQLite |
| **Single Go process** | No horizontal scaling; single machine only |
| **Window size fixed** | Mobile size (390x844) hardcoded in wails.json |

---

## Feature Implementation Guides

### Feature 1: Live Fueling Progress (Animated Meter)

**Architecture:**

```
User taps "AUTHORIZE PUMP"
    ↓
React calls Go: AuthorizePump(txnID, maxKES)
    ↓
Go goroutine starts: simulateFueling()
    ↓
Every 100ms: EventsEmit("pump:progress", update)
    ↓
React listener updates state:
    - setLitres(update.litres)
    - setAmount(update.amount)
    - setProgress(update.progress)
    ↓
Component re-renders with CSS animation:
    - width: progress * 100% (progress bar)
    - fontSize grows with litres (mega number effect)
```

**Code (Go Side):**

```go
func (h *Handler) AuthorizePump(ctx context.Context, txnID string, maxKES float64) error {
    go h.simulateFueling(ctx, txnID, maxKES)
    return nil
}

func (h *Handler) simulateFueling(ctx context.Context, txnID string, maxKES float64) {
    started := time.Now()
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            elapsed := time.Since(started).Seconds()
            litres := elapsed * 6.5
            amount := litres * 182
            
            if amount >= maxKES {
                h.CompleteFueling(ctx, txnID, litres, amount)
                return
            }
            
            runtime.EventsEmit(ctx, "pump:progress", FuelingUpdate{
                TxnID:    txnID,
                Litres:   litres,
                Amount:   amount,
                Progress: amount / maxKES,
            })
        }
    }
}
```

**Code (React Side):**

```typescript
export const FuelingProgress: React.FC<{ txnID: string }> = ({ txnID }) => {
    const [litres, setLitres] = useState(0);
    const [progress, setProgress] = useState(0);

    useEffect(() => {
        const unsub = EventsOn('pump:progress', (update: any) => {
            if (update.txn_id === txnID) {
                setLitres(update.litres);
                setProgress(update.progress);
            }
        });
        return unsub;
    }, [txnID]);

    return (
        <div className="fueling-display">
            <div className="litres" style={{ fontSize: 40 + litres }}>
                {litres.toFixed(2)}
            </div>
            <div 
                className="progress-bar"
                style={{ width: `${progress * 100}%` }}
            />
        </div>
    );
};
```

---

### Feature 2: Offline-First Transaction Queue

**Architecture:**

```
User completes transaction (online/offline doesn't matter)
    ↓
PostTransaction(txn) called in React
    ↓
React calls Go: PostTransaction(txn)
    ↓
Go inserts into LOCAL SQLite (not backend)
    ↓
Return success to React immediately
    ↓
Background goroutine (every 5s):
    - Query pending_sync table
    - Simulate POST to backend (random delay)
    - If success: mark synced = 1
    - If fail: retry count++
    ↓
React listens to "sync:updated" event
    ↓
Show sync status in UI (badge: pending/synced)
```

**Code (Go):**

```go
func (s *SyncService) PostTransaction(ctx context.Context, txn *Transaction) error {
    // Always insert locally (offline-first)
    err := s.txnRepo.Insert(ctx, txn)
    if err != nil {
        return err
    }
    
    // Queue for sync
    _, err = s.db.ExecContext(ctx, `
        INSERT INTO pending_sync (id, txn_id, payload, synced)
        VALUES (?, ?, ?, 0)
    `, uuid.New().String(), txn.ID, jsonMarshal(txn))
    
    return err
}

func (s *SyncService) StartSyncTicker(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        rows, err := s.db.QueryContext(ctx, `
            SELECT id, txn_id, payload FROM pending_sync 
            WHERE synced = 0 AND retries < 3
        `)
        if err != nil {
            continue
        }
        
        for rows.Next() {
            var id, txnID, payload string
            rows.Scan(&id, &txnID, &payload)
            
            // Simulate HTTP POST with random delay + success rate
            time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
            
            if rand.Float32() < 0.9 { // 90% success
                // Mark synced
                s.db.ExecContext(ctx, `
                    UPDATE pending_sync SET synced = 1 WHERE id = ?
                `, id)
                
                runtime.EventsEmit(ctx, "sync:success", txnID)
            } else {
                // Increment retry count
                s.db.ExecContext(ctx, `
                    UPDATE pending_sync 
                    SET retries = retries + 1, last_retry = NOW()
                    WHERE id = ?
                `, id)
                
                runtime.EventsEmit(ctx, "sync:retry", txnID)
            }
        }
    }
}
```

**Code (React):**

```typescript
export const OfflineQueue: React.FC = () => {
    const [pending, setPending] = useState(0);
    const [synced, setSynced] = useState(0);

    useEffect(() => {
        const unsubSuccess = EventsOn('sync:success', () => {
            setSynced((p) => p + 1);
            setPending((p) => Math.max(0, p - 1));
        });

        const unsubRetry = EventsOn('sync:retry', () => {
            // Keep pending count same
        });

        return () => {
            unsubSuccess();
            unsubRetry();
        };
    }, []);

    return (
        <div className="offline-queue">
            <Chip label={`Pending: ${pending}`} />
            <Chip label={`Synced: ${synced}`} color="success" />
        </div>
    );
};
```

---

## Testing & Debugging

### Unit Tests (Go)

```go
// handlers/transaction_test.go
package handlers

import (
    "context"
    "testing"
)

func TestAuthorizePump(t *testing.T) {
    h := NewTransactionHandler(mockDB)
    
    err := h.AuthorizePump(context.Background(), "TXN-123", 5000)
    if err != nil {
        t.Fatalf("Expected no error, got %v", err)
    }
}

func TestSimulateFueling(t *testing.T) {
    // Test that fueling completes in reasonable time
    h := NewTransactionHandler(mockDB)
    start := time.Now()
    
    h.simulateFueling(context.Background(), "TXN-123", 5000)
    
    elapsed := time.Since(start)
    if elapsed < 10*time.Millisecond {
        t.Fatal("Fueling completed too fast")
    }
}
```

### Component Tests (React)

```typescript
// pages/FuelingProgress.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { FuelingProgress } from './FuelingProgress';
import { EventsEmit } from 'wailsjs/runtime';

describe('FuelingProgress', () => {
    it('should display live litres', async () => {
        render(<FuelingProgress txnID="TXN-123" />);
        
        // Simulate pump update
        EventsEmit('pump:progress', {
            txn_id: 'TXN-123',
            litres: 10.5,
            amount: 1911,
            progress: 0.38,
        });
        
        await waitFor(() => {
            expect(screen.getByText(/10.50/)).toBeInTheDocument();
        });
    });
});
```

### Debugging with Wails DevTools

```bash
# DevTools automatically available in dev mode
# Press F12 to open

# View console logs from Go (in browser console):
runtime.LogInfo("message")
runtime.LogError("error")
```

---

## Performance Optimization

### 1. Virtualize Long Lists

```typescript
// HistoryList: 500+ transactions
import { FixedSizeList } from 'react-window';

<FixedSizeList
    height={600}
    itemCount={transactions.length}
    itemSize={60}
    width="100%"
>
    {({ index, style }) => (
        <TransactionRow
            key={transactions[index].id}
            txn={transactions[index]}
            style={style}
        />
    )}
</FixedSizeList>
```

### 2. Debounce Input Handlers

```typescript
// ProductAmount page: input every keystroke
const [amount, setAmount] = useState('');
const [estimatedLitres, setEstimatedLitres] = useState(0);

const handleAmountChange = useCallback(
    debounce((value: string) => {
        const kES = parseFloat(value) || 0;
        setEstimatedLitres(kES / 182); // Diesel
    }, 300),
    []
);
```

### 3. Memoize Components

```typescript
// Pump card rendered 4 times, expensive calculation
const PumpCard = memo(({ pump, onSelect }: Props) => {
    return <div>...</div>;
});
```

### 4. SQLite Indexes

```sql
-- Add indexes for common queries
CREATE INDEX idx_txn_shift ON transactions(shift_id);
CREATE INDEX idx_txn_created ON transactions(created_at DESC);
CREATE INDEX idx_pending_synced ON pending_sync(synced, retries);
```

---

## Deployment

### Build Production Executable

```bash
# From root directory
wails build

# Output:
# - build/bin/fms-attendant-pos        (Linux)
# - build/bin/fms-attendant-pos.exe    (Windows)
# - build/bin/fms-attendant-pos.app    (macOS)
```

### Create Installer (Windows)

```bash
# Install WiX toolset first, then:
wails build -nsis
```

### Distribute

1. **Windows:** Distribute .exe or .msi
2. **Linux:** Create .AppImage or .deb
3. **macOS:** Code sign + distribute .app

---

## Troubleshooting

### Issue: "Module not found" / Go bindings missing

**Solution:**
```bash
# Regenerate bindings
wails build -clean

# Check wailsjs/ directory was created
ls frontend/wailsjs
```

### Issue: High CPU usage during pump update streaming

**Solution:** Reduce update frequency
```go
// Instead of 100ms
ticker := time.NewTicker(200 * time.Millisecond) // 5 Hz instead of 10 Hz
```

### Issue: PDF not rendering / blank file

**Solution:** Ensure gofpdf is generating valid PDF
```go
// Test PDF generation
pdf := gofpdf.New("P", "mm", "A4", "")
pdf.AddPage()
pdf.SetFont("Arial", "", 12)
pdf.Cell(0, 10, "Test", 0, 1)
filename := filepath.Join(os.TempDir(), "test.pdf")
err := pdf.OutputFileAndClose(filename)
```

### Issue: SQLite "database is locked"

**Solution:** Close transactions properly
```go
defer rows.Close()
defer stmt.Close()
```

---

## Future Enhancements

### Phase 2 Features

1. **Real Backend Integration**
   - Replace dummy data with actual Odoo API calls
   - Daraja M-Pesa integration (real STK push)
   - Real pump controller integration (PTS-2/3)

2. **Supervisor Dashboard**
   - Shift open/close
   - Tank dip entry
   - Attendant variance tracking

3. **Receipt Customization**
   - Custom header (logo, station name)
   - Thermal printer optimization
   - SMS receipt delivery

4. **Analytics**
   - Pump utilization charts
   - Revenue by attendant
   - Hourly sales trends

5. **Hardware Integration**
   - Native camera + OCR for plate scanning
   - Bluetooth printer pairing
   - Card reader over serial/TCP

### Phase 3: Scale

- Multi-station support
- Separate app for supervisor
- Real-time supervisor alerts
- Dispute resolution flow

---

## Appendix: Quick Reference

### Common Go Functions (App Context)

```go
// Get context from frontend call
func (a *App) Handler(ctx context.Context, ...) error {
    // ctx automatically passed
}

// Emit event to React
runtime.EventsEmit(ctx, "event_name", data)

// Log
runtime.LogInfo(ctx, "message")
runtime.LogError(ctx, "error")
```

### Common React Imports

```typescript
import { EventsOn, EventsEmit } from 'wailsjs/runtime';
import { handlers } from 'wailsjs/go/handlers/[HandlerName]';
```

### wails.json Configuration

```json
{
  "name": "fms-attendant-pos",
  "outputfilename": "fms-attendant-pos",
  "frontend:build": "npm run build",
  "frontend:install": "npm install",
  "wailsjs": "./frontend/wailsjs",
  "devServer": "http://localhost:5173",
  "bindings": "models"
}
```

---

## Conclusion

This guide provides a complete blueprint for building a sophisticated desktop POS application using Wails. The FMS Attendant project demonstrates:

- **Go backend strength** in PDF generation, SQLite, and goroutine-based streaming
- **React frontend** for complex multi-screen flows
- **Wails IPC model** for Go ↔ JS communication
- **Offline-first patterns** with local SQLite queuing
- **Real-time updates** via event streaming
- **Edge cases** in payment reconciliation and error handling

Use this as a reference for your own Wails projects. The patterns outlined here scale from simple single-screen apps to complex multi-window applications with real-time data sync.

---

**Last Updated:** September 13, 2026  
**Maintainer:** Technical Documentation Team
