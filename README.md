# Task Management API Tutorial

Welcome to the Task Management API tutorial! 👋 We'll build an API (Application Programming Interface) that helps manage tasks and task lists. Think of it like building the backend for a todo list application!

## Project Name: task_management_api

## What You'll Learn

- Setting up a FastAPI project (a modern framework for building APIs)
- Working with SQLite databases (a simple, file-based database)
- Implementing CRUD operations (Create, Read, Update, Delete)
- Managing relationships between database tables
- RESTful API design principles
- Testing API endpoints

## Prerequisites

- Basic Python knowledge
- Understanding of HTTP methods (GET, POST, PUT, DELETE)
- Python 3.8+ installed
- A code editor (VS Code recommended)
- Terminal/Command Prompt familiarity

## Project Structure

```
task_management_api/
│
├── database/
│   ├── __init__.py
│   ├── db_setup.py      # Database initialization and test data
│   └── task.db          # SQLite database file
│
├── routers/
│   ├── __init__.py
│   ├── task_lists.py    # Task List endpoints
│   └── tasks.py         # Task endpoints
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_task_lists.py
│   └── test_tasks.py
│
├── main.py              # FastAPI application entry point
└── requirements.txt     # Project dependencies
```

## Getting Started

1. First, create a new project directory:

```bash
mkdir task_management_api
cd task_management_api
```

2. Install required packages:

```bash
pip install fastapi uvicorn sqlite3 pytest pytest-asyncio httpx
pip freeze > requirements.txt
```

## Database Setup

Create `database/db_setup.py`:

```python
import sqlite3
import random

def init_db():
    """
    Initialize the database by creating necessary tables.
    This function will:
    1. Create a connection to the SQLite database
    2. Create the task_list table if it doesn't exist
    3. Create the task table if it doesn't exist
    """
    # Connect to SQLite database (creates file if it doesn't exist)
    conn = sqlite3.connect('database/task.db')
    cur = conn.cursor()
    
    # Create TaskList table
    # - id: unique identifier for each task list
    # - name: name of the task list
    # - description: optional description of the task list
    cur.execute('''
        CREATE TABLE IF NOT EXISTS task_list (
            id INTEGER PRIMARY KEY AUTOINCREMENT,  -- Auto-incrementing ID
            name TEXT NOT NULL,                    -- Required name field
            description TEXT                       -- Optional description
        )
    ''')
    
    # Create Task table with foreign key to TaskList
    # - id: unique identifier for each task
    # - name: name of the task
    # - description: optional description
    # - priority: number from 1-5
    # - task_list_id: references the task list this task belongs to
    cur.execute('''
        CREATE TABLE IF NOT EXISTS task (
            id INTEGER PRIMARY KEY AUTOINCREMENT,  -- Auto-incrementing ID
            name TEXT NOT NULL,                    -- Required name field
            description TEXT,                      -- Optional description
            priority INTEGER CHECK(priority BETWEEN 1 AND 5),  -- Priority must be 1-5
            task_list_id INTEGER,                 -- Reference to task_list table
            FOREIGN KEY (task_list_id) REFERENCES task_list(id)
                ON DELETE CASCADE                  -- If task list is deleted, delete its tasks
        )
    ''')
    
    conn.commit()  # Save changes to database
    return conn

def load_test_data():
    """
    Populate the database with sample data for testing.
    Creates 5 task lists, each with 0-10 random tasks.
    """
    # Initialize database and get connection
    conn = init_db()
    cur = conn.cursor()
    
    # Sample task lists data
    task_lists = [
        ("Work Projects", "Professional tasks and deadlines"),
        ("Home Chores", "Household maintenance tasks"),
        ("Shopping List", "Items to purchase"),
        ("Learning Goals", "Educational objectives"),
        ("Fitness Goals", "Exercise and health tasks")
    ]
    
    # Insert sample task lists into database
    cur.executemany(
        "INSERT INTO task_list (name, description) VALUES (?, ?)",
        task_lists
    )
    
    # Create sample tasks for each task list
    for list_id in range(1, len(task_lists) + 1):
        # Randomly decide how many tasks to create (0-10)
        num_tasks = random.randint(0, 10)
        
        # Create list of task tuples (name, description, priority, list_id)
        tasks = [
            (f"Task {i} for list {list_id}", 
             f"Description for task {i}", 
             random.randint(1, 5),  # Random priority 1-5
             list_id)
            for i in range(1, num_tasks + 1)
        ]
        
        # Insert tasks into database
        cur.executemany(
            """INSERT INTO task (name, description, priority, task_list_id)
               VALUES (?, ?, ?, ?)""",
            tasks
        )
    
    conn.commit()  # Save all changes
    conn.close()   # Close database connection

# If this file is run directly (not imported), load test data
if __name__ == "__main__":
    load_test_data()
```

## Main Application

Create `main.py`:

```python
from fastapi import FastAPI, HTTPException
from database.db_setup import init_db, load_test_data
import sqlite3
from typing import List, Optional

# Create FastAPI application instance
app = FastAPI(
    title="Task Management API",
    description="A simple API for managing tasks and task lists",
    version="1.0.0"
)

def get_db():
    """
    Helper function to create a database connection.
    Returns a connection object with row_factory set to sqlite3.Row
    for easier dictionary access.
    """
    conn = sqlite3.connect('database/task.db')
    # This allows us to access rows as dictionaries instead of tuples
    conn.row_factory = sqlite3.Row
    return conn

# This function runs when the API starts up
@app.on_event("startup")
async def startup():
    """
    Initialize the database when the application starts.
    This ensures our tables exist before we try to use them.
    """
    init_db()
```

## Creating the Routers

Create `routers/task_lists.py`:

```python
from fastapi import APIRouter, HTTPException
from typing import List, Optional
import sqlite3

router = APIRouter(prefix="/task-lists", tags=["Task Lists"])

@router.get("/")
async def get_all_task_lists():
    """Get all task lists"""
    conn = get_db()
    cur = conn.cursor()
    task_lists = cur.execute("SELECT * FROM task_list").fetchall()
    conn.close()
    return [dict(row) for row in task_lists]

@router.post("/")
async def create_task_list(name: str, description: Optional[str] = None):
    """Create a new task list"""
    conn = get_db()
    cur = conn.cursor()
    
    cur.execute(
        "INSERT INTO task_list (name, description) VALUES (?, ?)",
        (name, description)
    )
    list_id = cur.lastrowid
    conn.commit()
    conn.close()
    
    return {"id": list_id, "name": name, "description": description}

@router.put("/{list_id}")
async def update_task_list(list_id: int, name: Optional[str] = None, description: Optional[str] = None):
    """Update a task list"""
    conn = get_db()
    cur = conn.cursor()
    
    if name is None and description is None:
        raise HTTPException(status_code=400, detail="No update parameters provided")
    
    updates = []
    values = []
    if name is not None:
        updates.append("name = ?")
        values.append(name)
    if description is not None:
        updates.append("description = ?")
        values.append(description)
    
    values.append(list_id)
    query = f"UPDATE task_list SET {', '.join(updates)} WHERE id = ?"
    
    cur.execute(query, values)
    if cur.rowcount == 0:
        raise HTTPException(status_code=404, detail="Task list not found")
    
    conn.commit()
    conn.close()
    return {"message": "Task list updated successfully"}

@router.delete("/{list_id}")
async def delete_task_list(list_id: int):
    """Delete a task list"""
    conn = get_db()
    cur = conn.cursor()
    
    cur.execute("DELETE FROM task_list WHERE id = ?", (list_id,))
    if cur.rowcount == 0:
        raise HTTPException(status_code=404, detail="Task list not found")
    
    conn.commit()
    conn.close()
    return {"message": "Task list deleted successfully"}
```

Create `routers/tasks.py`:

```python
from fastapi import APIRouter, HTTPException
from typing import List, Optional
import sqlite3

router = APIRouter(prefix="/tasks", tags=["Tasks"])

@router.get("/")
async def get_all_tasks():
    """Get all tasks"""
    conn = get_db()
    cur = conn.cursor()
    tasks = cur.execute("SELECT * FROM task").fetchall()
    conn.close()
    return [dict(row) for row in tasks]

@router.get("/list/{list_id}")
async def get_tasks_by_list(list_id: int):
    """Get all tasks in a specific list"""
    conn = get_db()
    cur = conn.cursor()
    tasks = cur.execute(
        "SELECT * FROM task WHERE task_list_id = ?",
        (list_id,)
    ).fetchall()
    conn.close()
    return [dict(row) for row in tasks]

@router.post("/")
async def create_task(name: str, task_list_id: int, description: Optional[str] = None, priority: int = 1):
    """Create a new task"""
    if not 1 <= priority <= 5:
        raise HTTPException(status_code=400, detail="Priority must be between 1 and 5")
    
    conn = get_db()
    cur = conn.cursor()
    
    # Verify task list exists
    cur.execute("SELECT id FROM task_list WHERE id = ?", (task_list_id,))
    if not cur.fetchone():
        raise HTTPException(status_code=404, detail="Task list not found")
    
    cur.execute(
        """INSERT INTO task (name, description, priority, task_list_id)
           VALUES (?, ?, ?, ?)""",
        (name, description, priority, task_list_id)
    )
    
    task_id = cur.lastrowid
    conn.commit()
    conn.close()
    
    return {
        "id": task_id,
        "name": name,
        "description": description,
        "priority": priority,
        "task_list_id": task_list_id
    }

@router.put("/{task_id}")
async def update_task(
    task_id: int,
    name: Optional[str] = None,
    description: Optional[str] = None,
    priority: Optional[int] = None,
    task_list_id: Optional[int] = None
):
    """Update a task"""
    if priority is not None and not 1 <= priority <= 5:
        raise HTTPException(status_code=400, detail="Priority must be between 1 and 5")
    
    conn = get_db()
    cur = conn.cursor()
    
    updates = []
    values = []
    if name is not None:
        updates.append("name = ?")
        values.append(name)
    if description is not None:
        updates.append("description = ?")
        values.append(description)
    if priority is not None:
        updates.append("priority = ?")
        values.append(priority)
    if task_list_id is not None:
        # Verify new task list exists
        cur.execute("SELECT id FROM task_list WHERE id = ?", (task_list_id,))
        if not cur.fetchone():
            raise HTTPException(status_code=404, detail="Task list not found")
        updates.append("task_list_id = ?")
        values.append(task_list_id)
    
    if not updates:
        raise HTTPException(status_code=400, detail="No update parameters provided")
    
    values.append(task_id)
    query = f"UPDATE task SET {', '.join(updates)} WHERE id = ?"
    
    cur.execute(query, values)
    if cur.rowcount == 0:
        raise HTTPException(status_code=404, detail="Task not found")
    
    conn.commit()
    conn.close()
    return {"message": "Task updated successfully"}

@router.delete("/{task_id}")
async def delete_task(task_id: int):
    """Delete a task"""
    conn = get_db()
    cur = conn.cursor()
    
    cur.execute("DELETE FROM task WHERE id = ?", (task_id,))
    if cur.rowcount == 0:
        raise HTTPException(status_code=404, detail="Task not found")
    
    conn.commit()
    conn.close()
    return {"message": "Task deleted successfully"}
```

## Testing

Create `tests/conftest.py`:

```python
import pytest
from fastapi.testclient import TestClient
import sqlite3
import os
from main import app
from database.db_setup import init_db

@pytest.fixture
def test_db():
    """Create a test database"""
    test_db_path = "database/test_task.db"
    conn = sqlite3.connect(test_db_path)
    conn.row_factory = sqlite3.Row
    init_db()
    yield conn
    conn.close()
    os.remove(test_db_path)

@pytest.fixture
def test_client(test_db):
    """Create a FastAPI test client"""
    return TestClient(app)

@pytest.fixture
def sample_task_list(test_client):
    """Create a sample task list for testing"""
    response = test_client.post(
        "/task-lists/",
        params={"name": "Test List", "description": "A test task list"}
    )
    return response.json()

@pytest.fixture
def sample_task(test_client, sample_task_list):
    """Create a sample task for testing"""
    response = test_client.post(
        "/tasks/",
        params={
            "name": "Test Task",
            "description": "A test task",
            "priority": 3,
            "task_list_id": sample_task_list["id"]
        }
    )
    return response.json()
```

Create `tests/test_task_lists.py`:

```python
def test_create_task_list(test_client):
    """Test creating a new task list"""
    response = test_client.post(
        "/task-lists/",
        params={"name": "New List", "description": "Test description"}
    )
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "New List"
    assert data["description"] == "Test description"
    assert "id" in data

def test_get_nonexistent_task_list(test_client):
    """Test getting a task list that doesn't exist"""
    response = test_client.get("/task-lists/999")
    assert response.status
