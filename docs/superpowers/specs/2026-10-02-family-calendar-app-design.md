# Family Calendar App - Architecture Design

## Overview

A wall-mounted touchscreen app for families to manage their shared calendar, to-do list, recurring tasks, and reward points system. The app displays Google Calendar events alongside locally-created events, allows multiple family members to add tasks and track reward points toward a manually-set target.

### Success Criteria
- Always-visible, frequently-interactive display
- Multiple family members can add/update items without login
- Google Calendar events sync automatically
- Task completion triggers automatic reward points
- Touch-friendly interface optimized for tablet use
- No data sync to Google Calendar (local events stay local)

### Key Constraints
- Frontend-only architecture (no backend initially)
- Single wall-mounted tablet as primary deployment target
- Google Calendar read + local event management
- Simple point counters for rewards (no complex gamification)

---

## Architecture

### Tech Stack
- **Frontend Framework:** React with TypeScript
- **Styling:** TailwindCSS (responsive, touch-friendly)
- **Routing:** React Router (tab-based navigation)
- **State Management:** React hooks (useState, useContext for shared state)
- **Google Integration:** Google Calendar API SDK
- **Storage:** localStorage/IndexedDB for persistent local data
- **Build Tool:** Vite or Create React App

### High-Level App Structure

```
┌─────────────────────────────────────┐
│      React Single Page App          │
│  (Always-on wall-mounted display)   │
├─────────────────────────────────────┤
│  Navigation Tabs (bottom/top):      │
│  • Calendar                         │
│  • To-Do                            │
│  • Tasks                            │
│  • Rewards                          │
├─────────────────────────────────────┤
│  Shared State Layer                 │
│  (points, tasks, to-dos, events)    │
├─────────────────────────────────────┤
│  Storage Layer:                     │
│  • Google Calendar API (read sync)  │
│  • localStorage/IndexedDB (local)   │
└─────────────────────────────────────┘
```

### Design Principles
- **Single Page App:** No page reloads. All tabs live in one React app. State is shared across tabs so completing a task immediately updates rewards.
- **Offline First (Partial):** Calendar syncs when possible; local data (tasks, to-dos, rewards) works offline
- **Frontend-Only:** No backend service. All data stored on the device. Enables rapid deployment and eliminates server management.
- **Touch-Optimized:** Large tap targets, minimal small interactions, responsive layout for tablet screens

---

## Key Features

### 1. Calendar Tab
**Responsibilities:**
- Display Google Calendar events (fetched via API)
- Display locally-created calendar events (stored in localStorage)
- Allow users to create, edit, and delete local calendar events
- Allow users to edit/delete Google Calendar events (sync back to Google)

**Implementation Details:**
- Month or week view with visual distinction between Google and local events (colors, badges)
- "Add Event" button to create new events
- Tap event to open detail view (edit/delete options)
- Background sync every 5-10 minutes to pull new Google Calendar updates
- Cached Google events shown if API is unavailable

### 2. To-Do Tab
**Responsibilities:**
- Display list of to-do items
- Allow creation, editing, deletion of to-dos
- Support optional metadata: due dates, priorities, assigned family member

**Implementation Details:**
- Simple checklist-style UI with checkbox for completion
- "Add To-Do" button at top
- Each item displays: text, optional due date badge, optional priority indicator, optional assigned-to label
- Tap to open detail view for editing
- Completed items can be archived or hidden
- No rewards associated with to-dos (unlike Tasks)

### 3. Tasks Tab
**Responsibilities:**
- Display recurring work items (chores, household, personal tasks)
- Track task completions with timestamps and who did it
- Trigger reward point updates when tasks are completed
- Support unlimited completions per day (not limited to once-daily)

**Implementation Details:**
- List of task items, each with a "Complete" button
- Categories visible: cleaning, household, personal, other
- Shows last completion date and who completed it
- "Add Task" button for creating new recurring items
- Tap task to edit details (title, description, points value, category, assigned person)
- Task can be completed multiple times in a day (e.g., "put something away" 3x = 15 points)
- Completion history logged for audit/tracking

### 4. Rewards Tab
**Responsibilities:**
- Display current points total vs. manually-set target
- Show progress bar toward target
- Record and display point-earning history
- Allow claiming reward (reset points and set new target)

**Implementation Details:**
- Large, prominent points display (e.g., "47 / 100")
- Progress bar showing visual progress
- "Claim Reward" button (appears when target is reached)
- History list showing point transactions with timestamps and task names
- User manually sets new target after claiming (e.g., "let's aim for 80 this week")

---

## Data Model

### Stored Locally (localStorage/IndexedDB)

**Calendar Events (Local)**
```
{
  id: string
  title: string
  description?: string
  startTime: ISO datetime
  endTime: ISO datetime
  color?: string
}
```

**To-Do Items**
```
{
  id: string
  text: string
  completed: boolean
  dueDate?: ISO date
  priority?: 'low' | 'medium' | 'high'
  assignedTo?: string
  createdAt: ISO datetime
}
```

**Tasks**
```
{
  id: string
  title: string
  description?: string
  category: 'cleaning' | 'household' | 'personal' | 'other'
  assignedTo?: string
  pointsReward: number
  completionHistory: [
    { date: ISO datetime, completedBy?: string }
  ]
}
```

**Rewards State**
```
{
  currentPoints: number
  target: number
  history: [
    { date: ISO datetime, pointsEarned: number, description: string }
  ]
}
```

**Google Calendar Events (Cached)**
```
{
  id: string
  googleEventId: string
  title: string
  description?: string
  startTime: ISO datetime
  endTime: ISO datetime
  source: 'google'
}
```

### Storage Strategy
- Use localStorage for small datasets (to-dos, tasks, rewards, local calendar events)
- Migrate to IndexedDB if data grows large
- Google Calendar events cached; refreshed on app load and every 5-10 minutes
- No sync back to Google for local calendar events (they stay on device only)

---

## Google Calendar Integration

### Authentication & Setup
1. User taps "Sign in with Google" on first load
2. OAuth 2.0 flow via Google Calendar API
3. Access token stored in localStorage (expires in ~1 hour, auto-refresh)
4. Refresh token stored securely for long-term access

### Sync Strategy
- **On App Load:** Fetch calendar events from last 3 months to next 3 months
- **Background Refresh:** Every 5-10 minutes, fetch updates (catches changes made on other devices)
- **Caching:** Display cached events immediately if API is unavailable
- **Error Handling:** Show cached data if authentication fails; prompt to re-authenticate when needed

### In-App Event Management
- **Create:** New events created locally (don't push to Google)
- **Edit:** Edits to Google Calendar events sync back to Google
- **Delete:** Deletions of Google events sync back to Google
- **Distinction:** Visual differentiation (colors, badges) between local and Google events

---

## Task → Reward Automation

### Completion Flow
1. User taps "Complete" on a task (can happen multiple times per day)
2. System records:
   - Task ID
   - Completion timestamp
   - Who completed it (optional prompt if multiple family members)
3. **Automatic Reward Update:**
   - Add task's `pointsReward` value to `currentPoints`
   - Record transaction in rewards history
   - Update rewards tab display in real-time
4. **Task History:**
   - Log completion in task's `completionHistory`
   - Task remains available for re-completion (no daily reset)

### Reward Claiming
- When user taps "Claim Reward" (available when `currentPoints >= target`):
  - Reset `currentPoints` to 0
  - Show celebration or confirmation
  - Prompt to set new target manually

### Example Workflow
- Target: 50 points
- 9am: Complete "clean kitchen" (5 pts) → 5/50
- 10am: Complete "put away clutter" twice (5 pts each) → 15/50
- Noon: Complete "laundry" (10 pts) → 25/50
- 3pm: Complete "put away clutter" again (5 pts) → 30/50
- 6pm: Complete "clean kitchen" (5 pts) → 35/50
- 8pm: Complete "dishes" (10 pts) → 45/50
- 9pm: Complete "put away clutter" (5 pts) → 50/50 → User claims reward, resets, sets new target

---

## Components & Navigation

### Component Hierarchy
```
App
├── NavBar (tab navigation)
├── CalendarTab
│   ├── EventList/Calendar View
│   └── EventDetail (edit/delete)
├── ToDoTab
│   ├── ToDoList
│   └── ToDoDetail (edit/delete)
├── TasksTab
│   ├── TaskList
│   ├── TaskDetail (edit/delete)
│   └── CompletionHistory
└── RewardsTab
    ├── PointsDisplay
    ├── ProgressBar
    ├── ClaimRewardButton
    └── TransactionHistory
```

### Navigation Structure
- Fixed tab bar (bottom or top) with 4 tabs: Calendar, To-Do, Tasks, Rewards
- Tab icons + labels for clarity
- Current tab highlighted
- Tap to switch tabs instantly (no loading delays)

---

## Deployment

### Development Environment
- Node.js + npm/yarn
- React dev server (runs on `localhost:3000`)
- Hot reload for rapid iteration

### Deployment to Wall-Mounted Tablet

**Option A: Local Server (Recommended for Initial Launch)**
- Run React dev server on a Raspberry Pi or local machine
- Tablet accesses app via `http://[IP]:3000`
- Device stays on that URL (kiosk mode)
- Simple, no external dependencies

**Option B: Cloud Deployment (Vercel/Netlify)**
- Deploy React app to Vercel or Netlify
- Tablet accesses app via fixed URL
- Automatic updates when you redeploy
- Requires internet connectivity

### Tablet Kiosk Setup
- Most tablets support kiosk mode to lock browser on one URL
- Disable accidental navigation (back button, address bar)
- Auto-wake if needed (power/sleep management)

---

## Testing Strategy

### Unit Tests
- Reward calculation logic (points earned, reset, target tracking)
- Task completion and history
- Event filtering and sorting

### Component Tests
- Calendar event display (Google vs. local distinction)
- To-do CRUD operations
- Task completion flow
- Rewards tab state updates
- Tab navigation

### Manual Testing
- Test on actual tablet/touchscreen device
- Verify touch interactions (tap targets, responsiveness)
- Test Google Calendar sync (add events on desktop Google Calendar, verify they appear on tablet)
- Test offline fallback (disable internet, verify local data still works)
- Test multi-person workflow (multiple family members tapping/adding items simultaneously)

### Browser Compatibility
- Safari (iPad)
- Chrome (Android tablet)
- Firefox (if needed)

---

## Success Metrics
- App loads and displays all tabs on tablet
- Google Calendar events sync within 5-10 minutes
- Task completion immediately updates rewards counter
- Touch interactions responsive (no lag)
- No data loss if internet drops (offline fallback works)
- Multiple family members can use simultaneously without conflicts

---

## Open Questions & Notes
- Visual design / color scheme (TBD — can iterate after core functionality works)
- Tablet device specification (brand, OS, screen size)
- Reward "claiming" UX (should it prompt for new target or just reset?)
- If family grows or more features requested, consider backend for cross-device sync

---

## Next Steps
1. Scaffold React app with TypeScript and Tailwind
2. Set up folder structure and component organization
3. Implement Google Calendar authentication and sync
4. Build each tab component (Calendar, To-Do, Tasks, Rewards)
5. Implement task completion → reward automation
6. Polish touch UX and test on actual tablet
7. Deploy to wall-mounted display
