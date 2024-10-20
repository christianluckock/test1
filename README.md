# Team & Player Management System Implementation Guide - FastAPI Version

## 1. Project Setup

```python
from fastapi import FastAPI, HTTPException
from typing import Optional
import sqlite3
from contextlib import contextmanager

app = FastAPI(title="Team Management API")

@contextmanager
def get_db_connection():
    conn = sqlite3.connect('teams.db')
    conn.row_factory = sqlite3.Row
    conn.execute('PRAGMA foreign_keys = ON')
    try:
        yield conn
    finally:
        conn.close()

def create_tables():
    with get_db_connection() as conn:
        conn.execute('''
            CREATE TABLE IF NOT EXISTS teams (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL UNIQUE,
                mascot TEXT NOT NULL,
                latitude REAL NOT NULL,
                longitude REAL NOT NULL
            )
        ''')
        
        conn.execute('''
            CREATE TABLE IF NOT EXISTS players (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                number INTEGER NOT NULL,
                position TEXT NOT NULL,
                team_id INTEGER,
                FOREIGN KEY (team_id) REFERENCES teams(id) ON DELETE RESTRICT,
                UNIQUE(team_id, number)
            )
        ''')
        conn.commit()

@app.on_event("startup")
async def startup_event():
    create_tables()
```

## 2. Team Endpoints

```python
@app.get("/api/teams")
async def get_teams():
    with get_db_connection() as conn:
        teams = conn.execute('SELECT * FROM teams').fetchall()
        return [dict(team) for team in teams]

@app.get("/api/teams/{team_id}")
async def get_team(team_id: int):
    with get_db_connection() as conn:
        team = conn.execute('SELECT * FROM teams WHERE id = ?', 
                          (team_id,)).fetchone()
        if team is None:
            raise HTTPException(status_code=404, detail="Team not found")
        return dict(team)

@app.post("/api/teams", status_code=201)
async def create_team(team: dict):
    required_fields = {'name', 'mascot', 'latitude', 'longitude'}
    if not all(field in team for field in required_fields):
        raise HTTPException(
            status_code=400,
            detail="Missing required fields"
        )

    with get_db_connection() as conn:
        try:
            cursor = conn.execute('''
                INSERT INTO teams (name, mascot, latitude, longitude)
                VALUES (?, ?, ?, ?)
            ''', (team['name'], team['mascot'], 
                 team['latitude'], team['longitude']))
            conn.commit()
            
            new_team = conn.execute(
                'SELECT * FROM teams WHERE id = ?', 
                (cursor.lastrowid,)
            ).fetchone()
            return dict(new_team)
        except sqlite3.IntegrityError:
            raise HTTPException(
                status_code=400,
                detail="Team name must be unique"
            )

@app.put("/api/teams/{team_id}")
async def update_team(team_id: int, team: dict):
    required_fields = {'name', 'mascot', 'latitude', 'longitude'}
    if not all(field in team for field in required_fields):
        raise HTTPException(
            status_code=400,
            detail="Missing required fields"
        )

    with get_db_connection() as conn:
        existing = conn.execute(
            'SELECT * FROM teams WHERE id = ?', 
            (team_id,)
        ).fetchone()
        
        if existing is None:
            raise HTTPException(status_code=404, detail="Team not found")
            
        try:
            conn.execute('''
                UPDATE teams 
                SET name = ?, mascot = ?, latitude = ?, longitude = ?
                WHERE id = ?
            ''', (team['name'], team['mascot'], 
                 team['latitude'], team['longitude'], team_id))
            conn.commit()
            
            updated = conn.execute(
                'SELECT * FROM teams WHERE id = ?', 
                (team_id,)
            ).fetchone()
            return dict(updated)
        except sqlite3.IntegrityError:
            raise HTTPException(
                status_code=400,
                detail="Team name must be unique"
            )

@app.delete("/api/teams/{team_id}", status_code=204)
async def delete_team(team_id: int):
    with get_db_connection() as conn:
        team = conn.execute(
            'SELECT * FROM teams WHERE id = ?', 
            (team_id,)
        ).fetchone()
        
        if team is None:
            raise HTTPException(status_code=404, detail="Team not found")
            
        players = conn.execute(
            'SELECT COUNT(*) as count FROM players WHERE team_id = ?', 
            (team_id,)
        ).fetchone()
        
        if players['count'] > 0:
            raise HTTPException(
                status_code=400,
                detail="Cannot delete team with existing players"
            )
        
        conn.execute('DELETE FROM teams WHERE id = ?', (team_id,))
        conn.commit()
```

## 3. Player Endpoints

```python
@app.get("/api/players")
async def get_players(team_id: Optional[int] = None):
    with get_db_connection() as conn:
        if team_id:
            players = conn.execute(
                'SELECT * FROM players WHERE team_id = ?', 
                (team_id,)
            ).fetchall()
        else:
            players = conn.execute('SELECT * FROM players').fetchall()
        return [dict(player) for player in players]

@app.get("/api/players/{player_id}")
async def get_player(player_id: int):
    with get_db_connection() as conn:
        player = conn.execute(
            'SELECT * FROM players WHERE id = ?', 
            (player_id,)
        ).fetchone()
        if player is None:
            raise HTTPException(status_code=404, detail="Player not found")
        return dict(player)

@app.post("/api/players", status_code=201)
async def create_player(player: dict):
    required_fields = {'name', 'number', 'position', 'team_id'}
    if not all(field in player for field in required_fields):
        raise HTTPException(
            status_code=400,
            detail="Missing required fields"
        )

    with get_db_connection() as conn:
        # Verify team exists
        team = conn.execute(
            'SELECT id FROM teams WHERE id = ?', 
            (player['team_id'],)
        ).fetchone()
        if team is None:
            raise HTTPException(status_code=404, detail="Team not found")
        
        try:
            cursor = conn.execute('''
                INSERT INTO players (name, number, position, team_id)
                VALUES (?, ?, ?, ?)
            ''', (player['name'], player['number'], 
                 player['position'], player['team_id']))
            conn.commit()
            
            new_player = conn.execute(
                'SELECT * FROM players WHERE id = ?', 
                (cursor.lastrowid,)
            ).fetchone()
            return dict(new_player)
        except sqlite3.IntegrityError:
            raise HTTPException(
                status_code=400,
                detail="Player number already exists for this team"
            )

@app.put("/api/players/{player_id}")
async def update_player(player_id: int, player: dict):
    required_fields = {'name', 'number', 'position', 'team_id'}
    if not all(field in player for field in required_fields):
        raise HTTPException(
            status_code=400,
            detail="Missing required fields"
        )

    with get_db_connection() as conn:
        existing = conn.execute(
            'SELECT * FROM players WHERE id = ?', 
            (player_id,)
        ).fetchone()
        
        if existing is None:
            raise HTTPException(status_code=404, detail="Player not found")
            
        # Verify team exists if team_id is changing
        if player['team_id'] != existing['team_id']:
            team = conn.execute(
                'SELECT id FROM teams WHERE id = ?', 
                (player['team_id'],)
            ).fetchone()
            if team is None:
                raise HTTPException(status_code=404, detail="Team not found")
        
        try:
            conn.execute('''
                UPDATE players 
                SET name = ?, number = ?, position = ?, team_id = ?
                WHERE id = ?
            ''', (player['name'], player['number'], 
                 player['position'], player['team_id'], player_id))
            conn.commit()
            
            updated = conn.execute(
                'SELECT * FROM players WHERE id = ?', 
                (player_id,)
            ).fetchone()
            return dict(updated)
        except sqlite3.IntegrityError:
            raise HTTPException(
                status_code=400,
                detail="Player number already exists for this team"
            )

@app.delete("/api/players/{player_id}", status_code=204)
async def delete_player(player_id: int):
    with get_db_connection() as conn:
        player = conn.execute(
            'SELECT * FROM players WHERE id = ?', 
            (player_id,)
        ).fetchone()
        
        if player is None:
            raise HTTPException(status_code=404, detail="Player not found")
        
        conn.execute('DELETE FROM players WHERE id = ?', (player_id,))
        conn.commit()
```

## 4. Running the Application

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Example API Requests

### Create a Team
```bash
curl -X POST "http://localhost:8000/api/teams" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "Dragons",
           "mascot": "Dragon",
           "latitude": 40.7128,
           "longitude": -74.0060
         }'
```

### Create a Player
```bash
curl -X POST "http://localhost:8000/api/players" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "John Doe",
           "number": 23,
           "position": "Forward",
           "team_id": 1
         }'
```

## Installation and Setup

1. Install dependencies:
```bash
pip install fastapi uvicorn
```

2. Run the application:
```bash
uvicorn main:app --reload
```

3. Access the API documentation:
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

## Key Features:

1. ✓ Simple dictionary-based request/response handling
2. ✓ Full CRUD operations for teams and players
3. ✓ SQLite database with foreign key constraints
4. ✓ Input validation for required fields
5. ✓ Error handling for common cases
6. ✓ Automatic API documentation
7. ✓ Query parameter support for filtering players by team
