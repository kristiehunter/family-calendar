# Family Calendar App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a wall-mounted touchscreen React web app for families to manage shared calendar events, to-do lists, recurring tasks, and reward points with automatic task-to-reward automation.

**Architecture:** Frontend-only SPA with tab-based navigation. React hooks for state management, localStorage/IndexedDB for persistence. Google Calendar OAuth 2.0 for read-only sync every 5-10 minutes. No backend. Local events and task/reward data stored on device only.

**Tech Stack:** React 19 LTS with TypeScript, TailwindCSS for styling, React Router v8 for tab navigation, Vite for build tool, Google Calendar API SDK, localStorage/IndexedDB for persistence.

**Spec:** `docs/superpowers/specs/2026-10-02-family-calendar-app-design.md`

## Global Constraints

- React 19 LTS with TypeScript (strict mode)
- TailwindCSS for all styling (no inline styles except dynamic values)
- Touch-friendly UI: minimum 48px tap targets, no hover-dependent interactions
- Google Calendar API OAuth 2.0 (read + write for user's own calendars)
- localStorage primary storage, migrate to IndexedDB if data grows beyond 5MB
- Tablet deployment target: Chrome on Android or Safari on iPad
- No backend services; all data persisted on device
- Offline functionality for local data (tasks, to-dos, rewards)
- Tab bar fixed at bottom or top; always visible
- Single app instance (one browser tab, kiosk mode on device)

## Review Focus

1. **Google Calendar sync failures** — If OAuth token expires or API is unavailable, app should display cached events and prompt to re-authenticate when user tries to create/edit Google events, not crash.
2. **Multi-daily task completion** — Tasks must allow unlimited daily completions; complete same task 3x in one day and verify points accumulate correctly.
3. **Offline reward updates** — Completing tasks offline should immediately update the rewards counter and persist to localStorage; verify on reconnection data is not lost.
4. **Concurrent family member actions** — If two family members tap the same task simultaneously, one completion should be recorded; test with localStorage transaction isolation.
5. **Data migration from localStorage to IndexedDB** — If app grows and switches storage backends, existing localStorage data must migrate without loss or manual intervention.

---

## File Structure

```
src/
├── App.tsx                       # Main app component, routing setup
├── index.css                     # Global TailwindCSS imports
├── components/
│   ├── NavBar.tsx               # Fixed tab navigation bar
│   ├── CalendarTab/
│   │   ├── CalendarTab.tsx       # Calendar tab container
│   │   ├── EventList.tsx         # Display Google + local events
│   │   └── EventDetail.tsx       # Edit/delete event modal
│   ├── ToDoTab/
│   │   ├── ToDoTab.tsx           # To-Do tab container
│   │   ├── ToDoList.tsx          # Display to-do items
│   │   └── ToDoDetail.tsx        # Edit to-do modal
│   ├── TasksTab/
│   │   ├── TasksTab.tsx          # Tasks tab container
│   │   ├── TaskList.tsx          # Display tasks
│   │   └── TaskDetail.tsx        # Edit task modal
│   ├── RewardsTab/
│   │   ├── RewardsTab.tsx        # Rewards tab container
│   │   ├── PointsDisplay.tsx     # Show current/target points
│   │   ├── ProgressBar.tsx       # Visual progress indicator
│   │   └── TransactionHistory.tsx # Show point history
│   └── common/
│       ├── Modal.tsx             # Reusable modal wrapper
│       └── ConfirmDialog.tsx     # Confirmation dialog
├── hooks/
│   ├── useStorage.ts             # localStorage/IndexedDB abstraction
│   ├── useGoogleAuth.ts          # Google Calendar OAuth flow
│   ├── useCalendarSync.ts        # Periodic Google Calendar sync
│   └── useRewardAutomation.ts    # Task completion → points flow
├── types/
│   ├── index.ts                  # Shared TypeScript types
│   ├── google.ts                 # Google Calendar types
│   └── storage.ts                # Storage layer types
├── services/
│   ├── googleCalendarApi.ts      # Google Calendar API wrapper
│   ├── storageService.ts         # localStorage/IndexedDB manager
│   └── rewardService.ts          # Reward calculation logic
├── utils/
│   ├── dateUtils.ts              # Date formatting, comparisons
│   ├── constants.ts              # App-wide constants
│   └── validators.ts             # Input validation helpers
└── __tests__/
    ├── unit/                     # Unit tests
    ├── integration/              # Integration tests
    └── e2e/                      # End-to-end tests (tablet manual)
```

---

## Task Breakdown

### Task 1: Project Setup & React Scaffolding

**Acceptance Criteria:**
- React 19 LTS app scaffolded with Vite
- TypeScript strict mode enabled
- TailwindCSS configured with PostCSS
- Dev server runs without errors
- App shell component renders

**Files to Create/Modify:**
- Create: `package.json`, `.eslintrc.json`, `tsconfig.json`, `vite.config.ts`
- Create: `src/index.tsx`, `src/App.tsx`, `src/index.css`
- Create: `public/index.html`

**Testing:**
- Dev server starts: `npm run dev`
- App renders in browser at `localhost:5173`
- No console errors or TypeScript errors

---

### Task 2: App Shell, Routing & Tab Navigation

**Acceptance Criteria:**
- React Router v8 installed and configured
- 4 tab routes working: `/calendar`, `/todos`, `/tasks`, `/rewards`
- NavBar component visible and clickable
- Tab navigation changes URL and highlights current tab
- Shared types defined for Calendar, Todo, Task, Reward

**Files to Create/Modify:**
- Modify: `src/App.tsx`
- Create: `src/components/NavBar.tsx`
- Create: `src/components/CalendarTab/CalendarTab.tsx`
- Create: `src/components/ToDoTab/ToDoTab.tsx`
- Create: `src/components/TasksTab/TasksTab.tsx`
- Create: `src/components/RewardsTab/RewardsTab.tsx`
- Create: `src/types/index.ts`

**Testing:**
- Click each tab, verify URL changes and correct tab renders
- Verify NavBar highlights current tab
- Verify TypeScript types compile without errors

---

### Task 3: Storage Layer (localStorage/IndexedDB Abstraction)

**Acceptance Criteria:**
- Storage service wraps localStorage with prefixed keys
- `useStorage()` hook provides reactive read/write
- All stored values persist across page reloads
- Unit tests cover get, set, remove, clear, keys operations
- Storage handles JSON serialization/deserialization

**Files to Create/Modify:**
- Create: `src/services/storageService.ts`
- Create: `src/hooks/useStorage.ts`
- Create: `src/types/storage.ts`
- Create: `src/__tests__/unit/storageService.test.ts`

**Testing:**
- Set and retrieve a value, verify persistence across page reload
- Test all CRUD operations: create, read, update, delete
- Run unit tests: `npm run test`
- Verify no data loss on concurrent writes

---

### Task 4: Google Calendar OAuth Setup & API Client

**Acceptance Criteria:**
- Google Cloud Console app registered with OAuth credentials
- `.env.local` populated with `VITE_GOOGLE_CLIENT_ID` and `VITE_GOOGLE_API_KEY`
- `useGoogleAuth()` hook manages auth state and token storage
- Google Calendar API wrapper supports listEvents, createEvent, deleteEvent
- API client handles errors gracefully
- Unit tests mock API responses

**Files to Create/Modify:**
- Create: `src/services/googleCalendarApi.ts`
- Create: `src/hooks/useGoogleAuth.ts`
- Create: `src/types/google.ts`
- Create: `.env.example`
- Create: `src/__tests__/unit/googleCalendarApi.test.ts`

**Testing:**
- Run unit tests: `npm run test`
- Verify auth state loads from storage on mount
- Test API methods with mocked fetch

---

### Task 5: Google Calendar Event Sync & Caching

**Acceptance Criteria:**
- Calendar sync fetches events from 3 months ago to 3 months ahead
- Sync runs automatically every 5-10 minutes
- Events cached in localStorage when sync succeeds
- Cached events displayed if API call fails
- Last sync timestamp stored for reference
- Integration tests verify cache behavior

**Files to Create/Modify:**
- Create: `src/hooks/useCalendarSync.ts`
- Create: `src/services/calendarSyncService.ts`
- Create: `src/__tests__/integration/calendarSync.test.ts`

**Testing:**
- Run integration tests: `npm run test`
- Verify sync happens on auth and periodically
- Disconnect network and verify cached events still display
- Reconnect and verify new events sync

---

### Task 6: Calendar Tab — Display Google & Local Events

**Acceptance Criteria:**
- EventList component displays both Google (blue) and local (green) events
- Events sorted by start time
- Event details modal shows when event clicked
- Modal displays title, start/end times, location, description
- Local events distinguished from Google events (source badge)
- Modal dismisses when close button clicked
- Loading state visible during sync

**Files to Create/Modify:**
- Modify: `src/components/CalendarTab/CalendarTab.tsx`
- Create: `src/components/CalendarTab/EventList.tsx`
- Create: `src/components/common/Modal.tsx`
- Create: `src/__tests__/components/EventList.test.tsx`

**Testing:**
- Render event list, verify events appear sorted
- Click event, verify modal opens with details
- Verify Google events have blue styling, local events have green
- Run component tests: `npm run test`

---

### Task 7: Calendar Tab — Local Event Management (Create/Edit/Delete)

**Acceptance Criteria:**
- EventDetail modal for creating/editing/deleting events
- Create: "Add Event" button opens empty form
- Edit: Click event, modify fields, save updates
- Delete: Only available for local events, shows confirmation
- Form validation requires title and valid date range
- Events persisted to localStorage
- Unsaved changes discarded on cancel

**Files to Create/Modify:**
- Create: `src/components/CalendarTab/EventDetail.tsx`
- Modify: `src/components/CalendarTab/CalendarTab.tsx`
- Create: `src/__tests__/components/EventDetail.test.tsx`

**Testing:**
- Create event, verify it appears in list
- Edit event, verify changes persisted
- Delete event, verify confirmation dialog
- Verify validation prevents empty titles

---

### Task 8: To-Do Tab Implementation (CRUD)

**Acceptance Criteria:**
- ToDoList displays all items with checkbox for completion
- Completed items show strikethrough text
- ToDoDetail modal for create/edit/delete
- Optional fields: dueDate, priority, assignedTo
- Checkbox toggle marks item complete/incomplete
- Items persisted to localStorage
- All CRUD operations working

**Files to Create/Modify:**
- Modify: `src/components/ToDoTab/ToDoTab.tsx`
- Create: `src/components/ToDoTab/ToDoList.tsx`
- Create: `src/components/ToDoTab/ToDoDetail.tsx`
- Create: `src/__tests__/components/ToDoTab.test.tsx`

**Testing:**
- Create to-do, verify appears in list
- Toggle checkbox, verify strikethrough applied
- Edit to-do, verify changes saved
- Delete to-do, verify removed from list
- Verify optional fields handled correctly

---

### Task 9: Tasks Tab Implementation (Display & Completion Tracking)

**Acceptance Criteria:**
- TaskList displays recurring tasks with point values
- Complete button shows point reward (+5, +10, etc)
- Completion tracked with timestamp
- Tasks allow unlimited daily completions
- "X completions today" counter visible
- Last completion date/time shown
- Task categories visible (cleaning, household, personal, other)
- TaskDetail modal for create/edit/delete
- Completion history stored for each task

**Files to Create/Modify:**
- Modify: `src/components/TasksTab/TasksTab.tsx`
- Create: `src/components/TasksTab/TaskList.tsx`
- Create: `src/components/TasksTab/TaskDetail.tsx`
- Create: `src/__tests__/components/TasksTab.test.tsx`
- Modify: `src/types/index.ts` (add completionHistory)

**Testing:**
- Complete task, verify completion recorded
- Complete same task 3 times, verify count shows "3x today"
- Edit task, verify changes saved
- Delete task, verify removed from list
- Verify categories display correctly

---

### Task 10: Reward Points Automation (Task Completion → Points)

**Acceptance Criteria:**
- When task completed, points added automatically to rewards
- Reward history logs each point transaction with timestamp and task name
- Points persist across page reloads
- Multiple completions accumulate points (3 completions = 3x points)
- Points never go negative
- Offline completion adds points when connection restored
- Unit tests verify points calculation

**Files to Create/Modify:**
- Create: `src/services/rewardService.ts`
- Create: `src/hooks/useRewardAutomation.ts`
- Create: `src/__tests__/unit/rewardService.test.ts`

**Testing:**
- Complete task, verify points added immediately
- Complete same task 3x, verify total points = 3 × reward value
- Reload page, verify points persisted
- Test offline: complete task offline, reload, verify points added
- Run unit tests: `npm run test`

---

### Task 11: Rewards Tab UI (Display, Progress, Claiming)

**Acceptance Criteria:**
- PointsDisplay shows current points / target (e.g., "47 / 100")
- ProgressBar fills proportionally to progress
- When points ≥ target, "Claim Reward" button enabled
- Claim button shows confirmation dialog
- Claiming resets points to 0
- User prompted to set new target after claiming
- TransactionHistory lists all point transactions with timestamps
- "Set New Target" button allows changing goal

**Files to Create/Modify:**
- Modify: `src/components/RewardsTab/RewardsTab.tsx`
- Create: `src/components/RewardsTab/PointsDisplay.tsx`
- Create: `src/components/RewardsTab/ProgressBar.tsx`
- Create: `src/components/RewardsTab/TransactionHistory.tsx`
- Modify: `src/components/common/ConfirmDialog.tsx`
- Create: `src/__tests__/components/RewardsTab.test.tsx`

**Testing:**
- Add points until reaching target
- Click claim button, verify confirmation dialog
- Confirm claim, verify points reset to 0
- Verify target setting dialog works
- Verify history shows all transactions

---

### Task 12: Touch Styling & Responsive Layout

**Acceptance Criteria:**
- All buttons/inputs 48px+ minimum tap target
- No hover-only interactions (use active/focus states instead)
- Layout responsive at common tablet sizes (768px iPad, 1024px iPad Pro)
- Nav bar fixed, doesn't overlap content
- Text readable at 768px width without zooming
- No horizontal scroll on tablet widths
- Touch events work on actual tablet device
- TailwindCSS responsive classes used throughout

**Files to Create/Modify:**
- Modify: `src/index.css` (add touch-friendly sizing utilities)
- Modify: All component files (apply responsive classes)

**Testing:**
- Test in browser DevTools at 768px and 1024px widths
- Verify all buttons/inputs are 48px+ tall
- Test on actual tablet device (iPad or Android)
- Verify no horizontal scroll
- Test landscape orientation

---

### Task 13: Testing Setup & Comprehensive Test Suite

**Acceptance Criteria:**
- Vitest configured with jsdom environment
- React Testing Library set up for component testing
- All 14 tasks have unit/integration/component tests
- Test coverage reports generated
- CI-ready test configuration
- Test helpers and setup file created

**Files to Create/Modify:**
- Create: `vitest.config.ts`
- Create: `src/__tests__/setup.ts`
- Modify: `package.json` (add test scripts)

**Testing:**
- Run full test suite: `npm run test`
- Generate coverage report: `npm run test:coverage`
- Verify all tests pass

---

### Task 14: Deployment Setup & Tablet Kiosk Configuration

**Acceptance Criteria:**
- Build process creates optimized bundle
- Deployment guide documents both local and cloud options
- PWA manifest created for installability
- Environment example file shows required variables
- Kiosk mode setup instructions for iPad and Android
- README includes OAuth setup steps

**Files to Create/Modify:**
- Create: `docs/DEPLOYMENT.md` (with setup and kiosk instructions)
- Create: `public/manifest.json` (PWA manifest)
- Modify: `public/index.html` (add PWA meta tags)
- Create: `.env.example` (environment variables)

**Testing:**
- Build: `npm run build`
- Verify dist/ folder contains index.html, JS, CSS
- Test build on local server
- Deploy to staging environment
- Test on actual tablet device in kiosk mode

---

## Self-Review Checklist

✓ All 14 tasks specified  
✓ Each task has acceptance criteria  
✓ Files to create/modify listed  
✓ Testing strategy defined per task  
✓ Spec requirements mapped to tasks  
✓ No placeholders or TBD sections  
✓ React 19 LTS specified  
✓ Touch-optimized design included  
✓ Offline functionality considered  
✓ Review focus items testable
