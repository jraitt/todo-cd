# Todo App Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a local Kanban todo web app (React + Express + SQLite) for managing 15+ personal side projects.

**Architecture:** Express serves a REST API backed by SQLite (`better-sqlite3`); a Vite React SPA consumes that API via a dev proxy. Both run together via `concurrently` with a single `npm run dev`.

**Tech Stack:** Node.js, Express, better-sqlite3, React 18, Vite, TailwindCSS, react-beautiful-dnd, concurrently

---

### Task 1: Project Scaffold

**Files:**
- Create: `package.json`
- Create: `vite.config.js`
- Create: `tailwind.config.js`
- Create: `postcss.config.js`
- Create: `index.html`
- Create: `src/main.jsx`
- Create: `src/index.css`
- Create: `server/index.js`
- Create: `.gitignore`

**Step 1: Initialize npm and install dependencies**

```bash
cd /c/Users/jrait/Documents/AI/Projects/Todo

npm init -y

# Backend deps
npm install express better-sqlite3 cors

# Frontend deps
npm install react react-dom react-beautiful-dnd

# Dev deps
npm install -D vite @vitejs/plugin-react tailwindcss postcss autoprefixer concurrently

# Init tailwind
npx tailwindcss init -p
```

**Step 2: Create `package.json` scripts section**

Edit `package.json` — replace the `"scripts"` block with:

```json
"scripts": {
  "dev": "concurrently \"node server/index.js\" \"vite\"",
  "build": "vite build",
  "preview": "vite preview"
}
```

**Step 3: Create `vite.config.js`**

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:3001'
    }
  }
})
```

**Step 4: Create `tailwind.config.js`**

```js
export default {
  content: ['./index.html', './src/**/*.{js,jsx}'],
  theme: { extend: {} },
  plugins: [],
}
```

**Step 5: Create `index.html`**

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Todo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

**Step 6: Create `src/index.css`**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**Step 7: Create `src/main.jsx`**

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

**Step 8: Create `src/App.jsx` (placeholder)**

```jsx
export default function App() {
  return <div className="p-4 text-xl">Todo App</div>
}
```

**Step 9: Create `.gitignore`**

```
node_modules/
dist/
todo.db
```

**Step 10: Create `server/index.js` (placeholder)**

```js
const express = require('express')
const app = express()
app.use(express.json())
app.get('/api/health', (req, res) => res.json({ ok: true }))
app.listen(3001, () => console.log('Server running on :3001'))
```

**Step 11: Verify it runs**

```bash
npm run dev
```

Expected: Vite starts on `http://localhost:5173`, Express on `:3001`. Open browser — see "Todo App".

**Step 12: Commit**

```bash
git add -A
git commit -m "feat: project scaffold (React + Vite + Express + Tailwind)"
```

---

### Task 2: Database Schema

**Files:**
- Create: `server/db.js`
- Modify: `server/index.js`

**Step 1: Create `server/db.js`**

```js
const Database = require('better-sqlite3')
const path = require('path')

const db = new Database(path.join(__dirname, '..', 'todo.db'))

db.exec(`
  CREATE TABLE IF NOT EXISTS projects (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT    NOT NULL,
    description TEXT    DEFAULT '',
    color       TEXT    DEFAULT '#6366f1',
    status      TEXT    DEFAULT 'active',
    created_at  TEXT    DEFAULT (datetime('now')),
    updated_at  TEXT    DEFAULT (datetime('now'))
  );

  CREATE TABLE IF NOT EXISTS tasks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id  INTEGER NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title       TEXT    NOT NULL,
    notes       TEXT    DEFAULT '',
    status      TEXT    DEFAULT 'backlog',
    position    REAL    DEFAULT 0,
    created_at  TEXT    DEFAULT (datetime('now')),
    updated_at  TEXT    DEFAULT (datetime('now'))
  );

  CREATE INDEX IF NOT EXISTS idx_tasks_project ON tasks(project_id);
  CREATE INDEX IF NOT EXISTS idx_tasks_status  ON tasks(status);
`)

module.exports = db
```

**Step 2: Modify `server/index.js` to import db**

Add at top:
```js
const db = require('./db')
```

**Step 3: Verify db initializes**

Restart the server (`npm run dev`). Verify `todo.db` appears in project root with no errors in console.

**Step 4: Commit**

```bash
git add server/db.js server/index.js
git commit -m "feat: SQLite schema (projects + tasks)"
```

---

### Task 3: Projects API

**Files:**
- Create: `server/routes/projects.js`
- Modify: `server/index.js`

**Step 1: Create `server/routes/projects.js`**

```js
const express = require('express')
const router = express.Router()
const db = require('../db')

// List all active projects
router.get('/', (req, res) => {
  const projects = db.prepare(`
    SELECT p.*, COUNT(t.id) as task_count
    FROM projects p
    LEFT JOIN tasks t ON t.project_id = p.id
    WHERE p.status = 'active'
    GROUP BY p.id
    ORDER BY p.created_at DESC
  `).all()
  res.json(projects)
})

// List archived projects
router.get('/archived', (req, res) => {
  const projects = db.prepare(`
    SELECT p.*, COUNT(t.id) as task_count
    FROM projects p
    LEFT JOIN tasks t ON t.project_id = p.id
    WHERE p.status = 'archived'
    GROUP BY p.id
    ORDER BY p.updated_at DESC
  `).all()
  res.json(projects)
})

// Create project
router.post('/', (req, res) => {
  const { name, description = '', color = '#6366f1' } = req.body
  if (!name?.trim()) return res.status(400).json({ error: 'name required' })
  const result = db.prepare(
    `INSERT INTO projects (name, description, color) VALUES (?, ?, ?)`
  ).run(name.trim(), description, color)
  const project = db.prepare('SELECT * FROM projects WHERE id = ?').get(result.lastInsertRowid)
  res.status(201).json(project)
})

// Update project
router.patch('/:id', (req, res) => {
  const { name, description, color, status } = req.body
  const project = db.prepare('SELECT * FROM projects WHERE id = ?').get(req.params.id)
  if (!project) return res.status(404).json({ error: 'not found' })

  db.prepare(`
    UPDATE projects SET
      name        = COALESCE(?, name),
      description = COALESCE(?, description),
      color       = COALESCE(?, color),
      status      = COALESCE(?, status),
      updated_at  = datetime('now')
    WHERE id = ?
  `).run(name ?? null, description ?? null, color ?? null, status ?? null, req.params.id)

  res.json(db.prepare('SELECT * FROM projects WHERE id = ?').get(req.params.id))
})

// Delete project (cascades to tasks via FK)
router.delete('/:id', (req, res) => {
  db.prepare('DELETE FROM projects WHERE id = ?').run(req.params.id)
  res.status(204).end()
})

module.exports = router
```

**Step 2: Modify `server/index.js` to mount routes**

Add after `app.use(express.json())`:

```js
const cors = require('cors')
app.use(cors())
app.use('/api/projects', require('./routes/projects'))
```

**Step 3: Manually test the API**

```bash
# Create a project
curl -s -X POST http://localhost:3001/api/projects \
  -H 'Content-Type: application/json' \
  -d '{"name":"My First Project","color":"#f59e0b"}' | jq

# List projects
curl -s http://localhost:3001/api/projects | jq
```

Expected: project appears with `id`, timestamps, `task_count: 0`.

**Step 4: Commit**

```bash
git add server/routes/projects.js server/index.js
git commit -m "feat: projects REST API (CRUD + archive)"
```

---

### Task 4: Tasks API

**Files:**
- Create: `server/routes/tasks.js`
- Modify: `server/index.js`

**Step 1: Create `server/routes/tasks.js`**

```js
const express = require('express')
const router = express.Router({ mergeParams: true })
const db = require('../db')

// List tasks for a project
router.get('/', (req, res) => {
  const tasks = db.prepare(`
    SELECT * FROM tasks WHERE project_id = ? ORDER BY status, position ASC
  `).all(req.params.projectId)
  res.json(tasks)
})

// Create task
router.post('/', (req, res) => {
  const { title, notes = '', status = 'backlog' } = req.body
  if (!title?.trim()) return res.status(400).json({ error: 'title required' })

  // Position: put at end of its column
  const maxPos = db.prepare(
    `SELECT MAX(position) as m FROM tasks WHERE project_id = ? AND status = ?`
  ).get(req.params.projectId, status)
  const position = (maxPos.m ?? 0) + 1000

  const result = db.prepare(
    `INSERT INTO tasks (project_id, title, notes, status, position) VALUES (?, ?, ?, ?, ?)`
  ).run(req.params.projectId, title.trim(), notes, status, position)

  res.status(201).json(db.prepare('SELECT * FROM tasks WHERE id = ?').get(result.lastInsertRowid))
})

module.exports = router
```

**Step 2: Create `server/routes/taskActions.js`** (for PATCH/DELETE on `/api/tasks/:id`)

```js
const express = require('express')
const router = express.Router()
const db = require('../db')

// Update task (title, notes, status, position)
router.patch('/:id', (req, res) => {
  const { title, notes, status, position } = req.body
  const task = db.prepare('SELECT * FROM tasks WHERE id = ?').get(req.params.id)
  if (!task) return res.status(404).json({ error: 'not found' })

  db.prepare(`
    UPDATE tasks SET
      title      = COALESCE(?, title),
      notes      = COALESCE(?, notes),
      status     = COALESCE(?, status),
      position   = COALESCE(?, position),
      updated_at = datetime('now')
    WHERE id = ?
  `).run(title ?? null, notes ?? null, status ?? null, position ?? null, req.params.id)

  res.json(db.prepare('SELECT * FROM tasks WHERE id = ?').get(req.params.id))
})

// Delete task
router.delete('/:id', (req, res) => {
  db.prepare('DELETE FROM tasks WHERE id = ?').run(req.params.id)
  res.status(204).end()
})

module.exports = router
```

**Step 3: Mount task routes in `server/index.js`**

```js
app.use('/api/projects/:projectId/tasks', require('./routes/tasks'))
app.use('/api/tasks', require('./routes/taskActions'))
```

**Step 4: Test**

```bash
PROJECT_ID=1  # use id from task 3

# Create tasks
curl -s -X POST http://localhost:3001/api/projects/$PROJECT_ID/tasks \
  -H 'Content-Type: application/json' \
  -d '{"title":"Write README"}' | jq

# List tasks
curl -s http://localhost:3001/api/projects/$PROJECT_ID/tasks | jq
```

**Step 5: Commit**

```bash
git add server/routes/tasks.js server/routes/taskActions.js server/index.js
git commit -m "feat: tasks REST API (CRUD + reorder)"
```

---

### Task 5: Search API

**Files:**
- Create: `server/routes/search.js`
- Modify: `server/index.js`

**Step 1: Create `server/routes/search.js`**

```js
const express = require('express')
const router = express.Router()
const db = require('../db')

router.get('/', (req, res) => {
  const q = (req.query.q || '').trim()
  if (!q) return res.json([])

  const like = `%${q}%`

  const projectHits = db.prepare(`
    SELECT 'project' as type, id, name as title, description as subtitle, color, NULL as project_id
    FROM projects
    WHERE (name LIKE ? OR description LIKE ?) AND status = 'active'
    LIMIT 10
  `).all(like, like)

  const taskHits = db.prepare(`
    SELECT 'task' as type, t.id, t.title, t.notes as subtitle, p.color, t.project_id,
           p.name as project_name, t.status as task_status
    FROM tasks t
    JOIN projects p ON p.id = t.project_id
    WHERE (t.title LIKE ? OR t.notes LIKE ?) AND p.status = 'active'
    LIMIT 20
  `).all(like, like)

  res.json([...projectHits, ...taskHits])
})

module.exports = router
```

**Step 2: Mount in `server/index.js`**

```js
app.use('/api/search', require('./routes/search'))
```

**Step 3: Test**

```bash
curl -s "http://localhost:3001/api/search?q=readme" | jq
```

**Step 4: Commit**

```bash
git add server/routes/search.js server/index.js
git commit -m "feat: global search API"
```

---

### Task 6: Frontend API Client

**Files:**
- Create: `src/api.js`

**Step 1: Create `src/api.js`**

```js
const BASE = '/api'

async function req(method, path, body) {
  const res = await fetch(BASE + path, {
    method,
    headers: body ? { 'Content-Type': 'application/json' } : {},
    body: body ? JSON.stringify(body) : undefined,
  })
  if (!res.ok) throw new Error(await res.text())
  if (res.status === 204) return null
  return res.json()
}

export const api = {
  // Projects
  getProjects:         ()         => req('GET',    '/projects'),
  getArchivedProjects: ()         => req('GET',    '/projects/archived'),
  createProject:       (data)     => req('POST',   '/projects', data),
  updateProject:       (id, data) => req('PATCH',  `/projects/${id}`, data),
  deleteProject:       (id)       => req('DELETE', `/projects/${id}`),

  // Tasks
  getTasks:   (projectId)     => req('GET',    `/projects/${projectId}/tasks`),
  createTask: (projectId, data) => req('POST', `/projects/${projectId}/tasks`, data),
  updateTask: (id, data)      => req('PATCH',  `/tasks/${id}`, data),
  deleteTask: (id)            => req('DELETE', `/tasks/${id}`),

  // Search
  search: (q) => req('GET', `/search?q=${encodeURIComponent(q)}`),
}
```

**Step 2: Commit**

```bash
git add src/api.js
git commit -m "feat: frontend API client"
```

---

### Task 7: App State + Sidebar

**Files:**
- Create: `src/App.jsx` (replace placeholder)
- Create: `src/components/Sidebar.jsx`

**Step 1: Create `src/components/Sidebar.jsx`**

```jsx
import { useState } from 'react'
import { api } from '../api'

const COLORS = ['#6366f1','#f59e0b','#10b981','#ef4444','#3b82f6','#8b5cf6','#ec4899','#14b8a6']

export default function Sidebar({ projects, archived, selectedId, onSelect, onProjectsChange }) {
  const [showArchived, setShowArchived] = useState(false)
  const [creating, setCreating] = useState(false)
  const [newName, setNewName] = useState('')
  const [newColor, setNewColor] = useState(COLORS[0])

  async function createProject(e) {
    e.preventDefault()
    if (!newName.trim()) return
    await api.createProject({ name: newName.trim(), color: newColor })
    setNewName('')
    setCreating(false)
    onProjectsChange()
  }

  return (
    <aside className="w-64 min-h-screen bg-gray-900 text-gray-100 flex flex-col p-3 gap-2 shrink-0">
      <div className="flex items-center justify-between mb-2">
        <span className="font-bold text-lg tracking-tight">Projects</span>
        <button
          onClick={() => setCreating(c => !c)}
          className="text-gray-400 hover:text-white text-xl leading-none"
          title="New project"
        >+</button>
      </div>

      {creating && (
        <form onSubmit={createProject} className="flex flex-col gap-2 bg-gray-800 rounded p-2">
          <input
            autoFocus
            value={newName}
            onChange={e => setNewName(e.target.value)}
            placeholder="Project name"
            className="bg-gray-700 text-white rounded px-2 py-1 text-sm outline-none"
          />
          <div className="flex gap-1 flex-wrap">
            {COLORS.map(c => (
              <button
                key={c}
                type="button"
                onClick={() => setNewColor(c)}
                className={`w-5 h-5 rounded-full border-2 ${newColor === c ? 'border-white' : 'border-transparent'}`}
                style={{ backgroundColor: c }}
              />
            ))}
          </div>
          <div className="flex gap-2">
            <button type="submit" className="bg-indigo-600 hover:bg-indigo-500 text-white text-xs px-2 py-1 rounded">Create</button>
            <button type="button" onClick={() => setCreating(false)} className="text-gray-400 text-xs">Cancel</button>
          </div>
        </form>
      )}

      <nav className="flex flex-col gap-1 flex-1 overflow-y-auto">
        {projects.map(p => (
          <button
            key={p.id}
            onClick={() => onSelect(p.id)}
            className={`flex items-center gap-2 px-2 py-1.5 rounded text-sm text-left truncate transition-colors
              ${selectedId === p.id ? 'bg-gray-700 text-white' : 'text-gray-300 hover:bg-gray-800'}`}
          >
            <span className="w-2.5 h-2.5 rounded-full shrink-0" style={{ backgroundColor: p.color }} />
            <span className="truncate flex-1">{p.name}</span>
            <span className="text-gray-500 text-xs">{p.task_count}</span>
          </button>
        ))}
      </nav>

      {archived.length > 0 && (
        <div>
          <button
            onClick={() => setShowArchived(s => !s)}
            className="text-gray-500 text-xs hover:text-gray-300 w-full text-left px-2 py-1"
          >
            {showArchived ? '▾' : '▸'} Archived ({archived.length})
          </button>
          {showArchived && archived.map(p => (
            <button
              key={p.id}
              onClick={() => onSelect(p.id)}
              className="flex items-center gap-2 px-2 py-1 rounded text-sm text-left text-gray-500 hover:text-gray-300 w-full truncate"
            >
              <span className="w-2 h-2 rounded-full shrink-0" style={{ backgroundColor: p.color }} />
              <span className="truncate">{p.name}</span>
            </button>
          ))}
        </div>
      )}
    </aside>
  )
}
```

**Step 2: Rewrite `src/App.jsx`**

```jsx
import { useState, useEffect, useCallback } from 'react'
import { api } from './api'
import Sidebar from './components/Sidebar'
import KanbanBoard from './components/KanbanBoard'
import SearchBar from './components/SearchBar'

export default function App() {
  const [projects, setProjects] = useState([])
  const [archived, setArchived] = useState([])
  const [selectedId, setSelectedId] = useState(null)
  const [toast, setToast] = useState(null)

  const loadProjects = useCallback(async () => {
    try {
      const [active, arch] = await Promise.all([api.getProjects(), api.getArchivedProjects()])
      setProjects(active)
      setArchived(arch)
      if (!selectedId && active.length > 0) setSelectedId(active[0].id)
    } catch (e) {
      showToast(e.message)
    }
  }, [selectedId])

  useEffect(() => { loadProjects() }, [])

  function showToast(msg) {
    setToast(msg)
    setTimeout(() => setToast(null), 3000)
  }

  const selectedProject = [...projects, ...archived].find(p => p.id === selectedId)

  return (
    <div className="flex min-h-screen bg-gray-950 text-white">
      <Sidebar
        projects={projects}
        archived={archived}
        selectedId={selectedId}
        onSelect={setSelectedId}
        onProjectsChange={loadProjects}
      />
      <main className="flex-1 flex flex-col overflow-hidden">
        <SearchBar onSelect={setSelectedId} showToast={showToast} />
        {selectedProject
          ? <KanbanBoard project={selectedProject} onProjectChange={loadProjects} showToast={showToast} />
          : <div className="flex-1 flex items-center justify-center text-gray-600">Select or create a project</div>
        }
      </main>

      {toast && (
        <div className="fixed bottom-4 right-4 bg-red-600 text-white px-4 py-2 rounded shadow-lg text-sm">
          {toast}
        </div>
      )}
    </div>
  )
}
```

**Step 3: Commit**

```bash
git add src/App.jsx src/components/Sidebar.jsx
git commit -m "feat: app shell + sidebar with project create/list/archive"
```

---

### Task 8: Search Bar Component

**Files:**
- Create: `src/components/SearchBar.jsx`

**Step 1: Create `src/components/SearchBar.jsx`**

```jsx
import { useState, useRef, useEffect } from 'react'
import { api } from '../api'

export default function SearchBar({ onSelect, showToast }) {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])
  const [open, setOpen] = useState(false)
  const ref = useRef()

  useEffect(() => {
    if (!query.trim()) { setResults([]); return }
    const t = setTimeout(async () => {
      try {
        const data = await api.search(query)
        setResults(data)
        setOpen(true)
      } catch (e) { showToast(e.message) }
    }, 200)
    return () => clearTimeout(t)
  }, [query])

  useEffect(() => {
    function handleClick(e) { if (ref.current && !ref.current.contains(e.target)) setOpen(false) }
    document.addEventListener('mousedown', handleClick)
    return () => document.removeEventListener('mousedown', handleClick)
  }, [])

  function pick(r) {
    if (r.project_id) onSelect(r.project_id)
    else onSelect(r.id)
    setQuery('')
    setOpen(false)
  }

  return (
    <div ref={ref} className="relative px-4 py-2 border-b border-gray-800">
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search projects and tasks..."
        className="w-full max-w-md bg-gray-800 text-white rounded px-3 py-1.5 text-sm outline-none placeholder-gray-500 focus:ring-1 focus:ring-indigo-500"
      />
      {open && results.length > 0 && (
        <div className="absolute top-full left-4 mt-1 w-96 bg-gray-800 rounded shadow-xl z-50 overflow-hidden">
          {results.map(r => (
            <button
              key={`${r.type}-${r.id}`}
              onClick={() => pick(r)}
              className="flex items-start gap-2 w-full px-3 py-2 text-left hover:bg-gray-700 text-sm"
            >
              <span className="w-2 h-2 rounded-full mt-1.5 shrink-0" style={{ backgroundColor: r.color }} />
              <div>
                <div className="text-white">{r.title}</div>
                {r.type === 'task' && (
                  <div className="text-gray-400 text-xs">{r.project_name} · {r.task_status}</div>
                )}
              </div>
            </button>
          ))}
        </div>
      )}
    </div>
  )
}
```

**Step 2: Commit**

```bash
git add src/components/SearchBar.jsx
git commit -m "feat: global search bar with debounce"
```

---

### Task 9: Kanban Board

**Files:**
- Create: `src/components/KanbanBoard.jsx`
- Create: `src/components/KanbanColumn.jsx`
- Create: `src/components/TaskCard.jsx`
- Create: `src/components/TaskModal.jsx`

**Step 1: Create `src/components/TaskModal.jsx`**

```jsx
import { useState, useEffect } from 'react'
import { api } from '../api'

export default function TaskModal({ task, onClose, onSave, onDelete }) {
  const [title, setTitle] = useState(task.title)
  const [notes, setNotes] = useState(task.notes || '')
  const [status, setStatus] = useState(task.status)

  async function save() {
    await onSave(task.id, { title, notes, status })
    onClose()
  }

  async function del() {
    if (!confirm('Delete this task?')) return
    await onDelete(task.id)
    onClose()
  }

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onClose}>
      <div className="bg-gray-800 rounded-lg p-5 w-full max-w-md shadow-xl" onClick={e => e.stopPropagation()}>
        <input
          value={title}
          onChange={e => setTitle(e.target.value)}
          onBlur={save}
          className="w-full bg-transparent text-white text-lg font-semibold outline-none border-b border-gray-600 pb-1 mb-3"
        />
        <textarea
          value={notes}
          onChange={e => setNotes(e.target.value)}
          onBlur={save}
          placeholder="Notes..."
          rows={4}
          className="w-full bg-gray-700 text-gray-200 rounded px-3 py-2 text-sm outline-none resize-none mb-3"
        />
        <div className="flex items-center justify-between">
          <select
            value={status}
            onChange={e => { setStatus(e.target.value); onSave(task.id, { status: e.target.value }) }}
            className="bg-gray-700 text-white text-sm rounded px-2 py-1"
          >
            <option value="backlog">Backlog</option>
            <option value="in_progress">In Progress</option>
            <option value="done">Done</option>
          </select>
          <div className="flex gap-2">
            <button onClick={del} className="text-red-400 hover:text-red-300 text-sm">Delete</button>
            <button onClick={onClose} className="text-gray-400 hover:text-white text-sm">Close</button>
          </div>
        </div>
      </div>
    </div>
  )
}
```

**Step 2: Create `src/components/TaskCard.jsx`**

```jsx
import { useState } from 'react'
import TaskModal from './TaskModal'

export default function TaskCard({ task, provided, onSave, onDelete }) {
  const [open, setOpen] = useState(false)

  return (
    <>
      <div
        ref={provided.innerRef}
        {...provided.draggableProps}
        {...provided.dragHandleProps}
        onClick={() => setOpen(true)}
        className="bg-gray-700 hover:bg-gray-600 rounded p-2.5 cursor-pointer select-none text-sm text-white shadow-sm"
      >
        <div className="font-medium leading-snug">{task.title}</div>
        {task.notes && <div className="text-gray-400 text-xs mt-1 line-clamp-2">{task.notes}</div>}
      </div>
      {open && (
        <TaskModal
          task={task}
          onClose={() => setOpen(false)}
          onSave={onSave}
          onDelete={onDelete}
        />
      )}
    </>
  )
}
```

**Step 3: Create `src/components/KanbanColumn.jsx`**

```jsx
import { useState } from 'react'
import { Droppable, Draggable } from 'react-beautiful-dnd'
import TaskCard from './TaskCard'

const LABELS = { backlog: 'Backlog', in_progress: 'In Progress', done: 'Done' }
const COLORS = { backlog: 'text-gray-400', in_progress: 'text-yellow-400', done: 'text-green-400' }

export default function KanbanColumn({ status, tasks, projectId, onTaskCreate, onTaskSave, onTaskDelete }) {
  const [adding, setAdding] = useState(false)
  const [newTitle, setNewTitle] = useState('')

  async function addTask(e) {
    e.preventDefault()
    if (!newTitle.trim()) return
    await onTaskCreate({ title: newTitle.trim(), status })
    setNewTitle('')
    setAdding(false)
  }

  return (
    <div className="flex flex-col bg-gray-900 rounded-lg p-3 min-w-64 w-72 shrink-0">
      <div className={`text-xs font-bold uppercase tracking-wider mb-3 ${COLORS[status]}`}>
        {LABELS[status]} <span className="text-gray-600 font-normal">({tasks.length})</span>
      </div>

      <Droppable droppableId={status}>
        {(provided) => (
          <div
            ref={provided.innerRef}
            {...provided.droppableProps}
            className="flex flex-col gap-2 flex-1 min-h-8"
          >
            {tasks.map((task, index) => (
              <Draggable key={task.id} draggableId={String(task.id)} index={index}>
                {(provided) => (
                  <TaskCard
                    task={task}
                    provided={provided}
                    onSave={onTaskSave}
                    onDelete={onTaskDelete}
                  />
                )}
              </Draggable>
            ))}
            {provided.placeholder}
          </div>
        )}
      </Droppable>

      {adding ? (
        <form onSubmit={addTask} className="mt-2">
          <input
            autoFocus
            value={newTitle}
            onChange={e => setNewTitle(e.target.value)}
            onKeyDown={e => e.key === 'Escape' && setAdding(false)}
            placeholder="Task title"
            className="w-full bg-gray-700 text-white rounded px-2 py-1.5 text-sm outline-none"
          />
          <div className="flex gap-2 mt-1">
            <button type="submit" className="bg-indigo-600 hover:bg-indigo-500 text-white text-xs px-2 py-1 rounded">Add</button>
            <button type="button" onClick={() => setAdding(false)} className="text-gray-500 text-xs">Cancel</button>
          </div>
        </form>
      ) : (
        <button
          onClick={() => setAdding(true)}
          className="mt-2 text-gray-500 hover:text-gray-300 text-sm text-left px-1"
        >
          + Add task
        </button>
      )}
    </div>
  )
}
```

**Step 4: Create `src/components/KanbanBoard.jsx`**

```jsx
import { useState, useEffect, useCallback } from 'react'
import { DragDropContext } from 'react-beautiful-dnd'
import { api } from '../api'
import KanbanColumn from './KanbanColumn'

const STATUSES = ['backlog', 'in_progress', 'done']

function computePosition(tasks, destinationIndex) {
  const before = tasks[destinationIndex - 1]?.position ?? 0
  const after  = tasks[destinationIndex]?.position ?? (before + 2000)
  return (before + after) / 2
}

export default function KanbanBoard({ project, onProjectChange, showToast }) {
  const [tasks, setTasks] = useState([])

  const load = useCallback(async () => {
    try {
      const data = await api.getTasks(project.id)
      setTasks(data)
    } catch (e) { showToast(e.message) }
  }, [project.id])

  useEffect(() => { load() }, [load])

  function tasksFor(status) {
    return tasks.filter(t => t.status === status).sort((a, b) => a.position - b.position)
  }

  async function onDragEnd({ source, destination, draggableId }) {
    if (!destination) return

    const taskId = parseInt(draggableId)
    const newStatus = destination.droppableId
    const colTasks = tasksFor(newStatus).filter(t => t.id !== taskId)
    const position = computePosition(colTasks, destination.index)

    // Optimistic update
    setTasks(prev => prev.map(t => t.id === taskId ? { ...t, status: newStatus, position } : t))

    try {
      await api.updateTask(taskId, { status: newStatus, position })
      onProjectChange()
    } catch (e) {
      showToast(e.message)
      load()
    }
  }

  async function createTask(data) {
    try {
      const task = await api.createTask(project.id, data)
      setTasks(prev => [...prev, task])
      onProjectChange()
    } catch (e) { showToast(e.message) }
  }

  async function saveTask(id, data) {
    try {
      const updated = await api.updateTask(id, data)
      setTasks(prev => prev.map(t => t.id === id ? updated : t))
    } catch (e) { showToast(e.message) }
  }

  async function deleteTask(id) {
    try {
      await api.deleteTask(id)
      setTasks(prev => prev.filter(t => t.id !== id))
      onProjectChange()
    } catch (e) { showToast(e.message) }
  }

  return (
    <div className="flex-1 overflow-auto p-6">
      <div className="flex items-center gap-3 mb-6">
        <span className="w-3 h-3 rounded-full" style={{ backgroundColor: project.color }} />
        <h1 className="text-xl font-bold">{project.name}</h1>
        {project.description && <span className="text-gray-500 text-sm">{project.description}</span>}
        <ProjectMenu project={project} onProjectChange={onProjectChange} showToast={showToast} />
      </div>

      <DragDropContext onDragEnd={onDragEnd}>
        <div className="flex gap-4 items-start">
          {STATUSES.map(status => (
            <KanbanColumn
              key={status}
              status={status}
              tasks={tasksFor(status)}
              projectId={project.id}
              onTaskCreate={createTask}
              onTaskSave={saveTask}
              onTaskDelete={deleteTask}
            />
          ))}
        </div>
      </DragDropContext>
    </div>
  )
}

function ProjectMenu({ project, onProjectChange, showToast }) {
  const [open, setOpen] = useState(false)

  async function archive() {
    try {
      await api.updateProject(project.id, { status: project.status === 'active' ? 'archived' : 'active' })
      onProjectChange()
    } catch (e) { showToast(e.message) }
    setOpen(false)
  }

  async function del() {
    if (!confirm(`Delete "${project.name}" and all its tasks?`)) return
    try {
      await api.deleteProject(project.id)
      onProjectChange()
    } catch (e) { showToast(e.message) }
    setOpen(false)
  }

  return (
    <div className="relative ml-auto">
      <button onClick={() => setOpen(o => !o)} className="text-gray-500 hover:text-white px-2">···</button>
      {open && (
        <div className="absolute right-0 top-full mt-1 bg-gray-800 rounded shadow-xl z-10 min-w-36 overflow-hidden">
          <button onClick={archive} className="block w-full text-left px-4 py-2 text-sm hover:bg-gray-700">
            {project.status === 'active' ? 'Archive' : 'Unarchive'}
          </button>
          <button onClick={del} className="block w-full text-left px-4 py-2 text-sm text-red-400 hover:bg-gray-700">
            Delete project
          </button>
        </div>
      )}
    </div>
  )
}
```

**Step 5: Verify the full app works**

- `npm run dev` → open `http://localhost:5173`
- Create 2-3 projects
- Add tasks in each column
- Drag tasks between columns — verify order persists on refresh
- Open a task, edit title/notes — verify saves on blur
- Archive a project — verify it moves to archived section

**Step 6: Commit**

```bash
git add src/components/
git commit -m "feat: Kanban board with drag-and-drop, task CRUD, project menu"
```

---

### Task 10: Polish & QoL

**Files:**
- Modify: `src/App.jsx`
- Modify: `src/components/KanbanColumn.jsx`

**Step 1: Add `N` keyboard shortcut to add task in backlog**

In `KanbanBoard.jsx`, add a `useEffect`:

```jsx
useEffect(() => {
  function handler(e) {
    if (e.key === 'n' && e.target.tagName === 'BODY') {
      // Signal the backlog column to open its add-task form
      document.getElementById('add-backlog')?.click()
    }
  }
  window.addEventListener('keydown', handler)
  return () => window.removeEventListener('keydown', handler)
}, [])
```

In `KanbanColumn.jsx`, give the "+ Add task" button `id={status === 'backlog' ? 'add-backlog' : undefined}`.

**Step 2: Add empty state when no projects**

In `App.jsx`, replace the fallback div:

```jsx
<div className="flex-1 flex flex-col items-center justify-center text-gray-600 gap-3">
  <div className="text-5xl">📋</div>
  <div className="text-lg">No project selected</div>
  <div className="text-sm">Create a project in the sidebar to get started</div>
</div>
```

**Step 3: Final smoke test**

- Search across multiple projects
- Archive / unarchive
- Delete project (confirm dialog)
- Keyboard `N` shortcut works
- Refresh page — all data persists

**Step 4: Final commit**

```bash
git add -A
git commit -m "feat: keyboard shortcut + empty state polish"
```

---

## Done

The app is fully functional. Run with:

```bash
npm run dev
```

Open `http://localhost:5173`.
