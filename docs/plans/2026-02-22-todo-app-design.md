# Todo App Design — 2026-02-22

## Overview

A local Kanban-based todo web app for managing 15+ personal side projects. Runs via a local dev server, data persists in SQLite.

## Stack

- **Frontend:** React (Vite), TailwindCSS, react-beautiful-dnd
- **Backend:** Express.js REST API
- **Database:** SQLite via `better-sqlite3`
- **Dev runner:** `concurrently` (single `npm run dev` starts both)

## Project Structure

```
Todo/
├── server/
│   ├── db.js           # SQLite init + schema
│   ├── index.js        # Express entry point
│   └── routes/
│       ├── projects.js
│       └── tasks.js
├── src/
│   ├── components/
│   │   ├── Sidebar.jsx
│   │   ├── KanbanBoard.jsx
│   │   ├── KanbanColumn.jsx
│   │   ├── TaskCard.jsx
│   │   ├── TaskModal.jsx
│   │   └── SearchResults.jsx
│   ├── hooks/
│   │   ├── useProjects.js
│   │   └── useTasks.js
│   ├── api.js
│   └── main.jsx
├── todo.db             # SQLite database file (gitignored)
├── package.json
└── vite.config.js
```

## Data Model

### projects

| Column      | Type    | Notes                        |
|-------------|---------|------------------------------|
| id          | INTEGER | Primary key, autoincrement   |
| name        | TEXT    | Project name                 |
| description | TEXT    | Optional                     |
| color       | TEXT    | Hex color for visual identity|
| status      | TEXT    | `active` or `archived`       |
| created_at  | TEXT    | ISO timestamp                |
| updated_at  | TEXT    | ISO timestamp                |

### tasks

| Column     | Type    | Notes                                    |
|------------|---------|------------------------------------------|
| id         | INTEGER | Primary key, autoincrement               |
| project_id | INTEGER | Foreign key → projects.id                |
| title      | TEXT    | Task title                               |
| notes      | TEXT    | Optional extended notes                  |
| status     | TEXT    | `backlog`, `in_progress`, or `done`      |
| position   | REAL    | Float for drag-and-drop ordering         |
| created_at | TEXT    | ISO timestamp                            |
| updated_at | TEXT    | ISO timestamp                            |

## API Routes

```
GET    /api/projects           # list all active projects
POST   /api/projects           # create project
PATCH  /api/projects/:id       # update (name, color, description, status)
DELETE /api/projects/:id       # delete project + its tasks

GET    /api/projects/:id/tasks # list tasks for a project
POST   /api/projects/:id/tasks # create task
PATCH  /api/tasks/:id          # update task (title, notes, status, position)
DELETE /api/tasks/:id          # delete task

GET    /api/search?q=...       # full-text search across projects + tasks
```

## UI Layout

### Sidebar (left, fixed)
- Search box (global, searches projects + tasks)
- "New Project" button
- Scrollable list of active projects — name, color dot, task count
- Collapsible "Archived" section at bottom

### Main Area
- Selected project header (name, color, description, action menu)
- Kanban board: 3 columns — **Backlog / In Progress / Done**
- Each column has an "Add task" button at bottom
- Drag tasks within and between columns

### Task Card
- Title, optional notes preview, drag handle
- "..." menu: Edit, Delete
- Click opens slide-over modal: edit title, notes, change status

### Project Actions (via "..." menu)
- Rename, change color, archive, delete (with confirmation dialog)

## Key Behaviors

- **Drag-and-drop:** uses `react-beautiful-dnd`; position stored as float (midpoint between neighbors) to avoid renumbering
- **Auto-save:** task edits save on blur — no explicit save button
- **Keyboard shortcut:** `N` adds a new task to the focused column
- **Toast notifications:** shown on API errors
- **Archiving:** hides project from main list, preserves all task data; visible in collapsed sidebar section
- **Search:** queries project names, descriptions, and task titles/notes; results grouped by project

## Out of Scope (for now)

- User authentication / multi-user
- Due dates / reminders
- Tags or labels
- File attachments
- Mobile app
