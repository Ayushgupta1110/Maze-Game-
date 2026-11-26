# Maze Explorer

Maze Explorer is a Python desktop game made with Pygame.

This guide explains how to start the game and how its main pieces fit together. Source locations below refer to line numbers in the files as they are currently checked in; edits to those files can move the lines.

## What happens when you play

1. `main.py` reads any command-line options and creates the application.
2. The application opens a game window and shows the mode-selection screen.
3. Choose **User Only** or **User vs Agent**, then choose **Easy**, **Medium**, **Difficult**, or **Hard**.
4. The game builds a maze, places collectibles, and creates the player. In Race mode it also plans the agent's route.
5. In Race mode, the maze is visible for a short preview. When the countdown ends, the renderer covers most of the maze with fog; the player can see around their character.
6. Move through the corridors, collect items, and reach the exit. On difficulties that require a key, collect the key before trying to leave.
7. Reaching the exit clears the level and keeps the score for the next level. In Race mode, the first one to finish wins. A timer can end the level on timed difficulties.

The program repeatedly handles input, updates game state, and draws the current screen. This is the game loop in `game/app.py`, in `MazeExplorerApp.run()` (line 588).

## Requirements and running the game

- Python 3.11 or newer is recommended.
- The only third-party package listed by the project is Pygame (`requirements.txt`).
- A graphical desktop session is needed to play normally.

### Windows PowerShell

Open PowerShell in the project folder:

```powershell
cd "C:\Maze Game"
python --version
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python main.py
```

If PowerShell blocks activating the virtual environment, you can run the game without activating it:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe main.py
```

On Windows, `run_game.bat` is a convenience launcher. Double-click it, or run it from PowerShell:

```powershell
.\run_game.bat
```

The launcher uses the Python launcher (`py -3`) when it is installed; if it is not found, it tries `python`. If the selected launch command fails, it displays an error and pauses so the message remains visible.

### Command-line options

These options are parsed in `main.py` (`parse_arguments()`, line 10):

| Option | What it does | Example |
|---|---|---|
| `--difficulty` | Sets and saves the starting difficulty preference. It does not skip the menus. Valid values: `Easy`, `Medium`, `Difficult`, `Hard`. | `python main.py --difficulty Difficult` |
| `--seed` | Makes maze generation repeatable for the same starting seed and level sequence. | `python main.py --seed 42` |
| `--headless` | Creates an off-screen drawing surface instead of opening a display window. Useful for basic automated runs, not normal play. | `python main.py --headless` |
| `--frames N` | Stops after drawing N frames. Most useful together with `--headless`. | `python main.py --headless --frames 120` |

For example, to run a short headless check:

```powershell
python main.py --headless --frames 120
```

## Modes and difficulty

Every difficulty currently uses a 25-by-25 grid. The grid dimensions are set directly in `game/app.py` (`_init_level()`, line 261); `config.py` also contains grid constants.

| Difficulty | Time limit | Hints per level | Key required | Coins / gems | Preview / agent step interval |
|---|---:|---:|---|---:|---|
| Easy | None | 5 | No | 8 / 4 | 4 s / 0.44 s |
| Medium | 120 s | 3 | Yes | 12 / 6 | 4 s / 0.32 s |
| Difficult | 80 s | 2 | Yes | 16 / 8 | 3 s / 0.22 s |
| Hard | 50 s | 1 | Yes | 20 / 10 | 2 s / 0.16 s |

These settings are defined in `config.py` around lines 89-154. The actual grid size is fixed separately in `game/app.py`. Race mode uses the preview and agent; Solo mode does not create an agent. When a level is cleared, later levels increase collectible counts up to limits and increase the level-clear bonus.

## Controls

| Action | Keys |
|---|---|
| Move | Arrow keys or `W`, `A`, `S`, `D` |
| Request a route hint | `H` |
| Pause or resume | `P` or `Escape` |
| Restart the current level | `R` |
| Return to the mode menu | `M` |
| Choose a menu option | Mouse click or the number/letter shortcuts shown on screen |
| Quit from the mode menu | `Q` or `Escape` |

The event/state handling is in `game/app.py` (`handle_events()`, line 365, and `_process_movement_key()`, line 448). The movement attempt itself is checked by `Player.move()` in `game/player.py` (line 48), so a wall or out-of-bounds step is rejected.

## How the algorithms work

### Maze generation: randomized backtracking

`Maze` is defined in `game/maze_generator.py` (line 18); generation is in `_generate_maze()` (line 71).

1. The generator fills a 2D grid with walls.
2. Starting at cell `(1, 1)`, it considers cells two steps away in the four cardinal directions. The in-between cell is the wall to carve open.
3. It chooses one unvisited neighbor at random, opens the connecting wall and neighbor, and moves there.
4. If there are no unvisited neighbors, it goes back to the previous cell and tries another branch. This is called backtracking.
5. Once the first pass has visited all maze cells, `_add_loops()` (line 110) can open a small number of extra walls to create alternate routes.
6. `_place_collectibles()` (line 131) puts coins, gems, and, when the difficulty requires it, a key on distinct walkable cells.

The traversal is implemented with an explicit stack (a Python list), rather than a recursive function call. The `visited` set prevents revisiting maze cells during the carving pass. If a seed is supplied, the generator's random choices can be repeated.

### Route finding: A*, breadth-first search, and depth-first search

The algorithms are in `game/pathfinding.py`:

- `find_path_astar()` (line 115) is used for the rival agent and player hints. It tracks the distance already traveled (`g`) and estimates remaining distance with Manhattan distance (`h`): the row difference plus the column difference. A min-heap prioritizes the cell with the lowest `g + h`. When the goal is reached, parent pointers are followed backwards to build the route. Each maze step has equal cost, and this heuristic does not overestimate, so A* finds a shortest route.
- `find_path_bfs()` (line 42) explores equally distant cells in order using a FIFO queue. It finds a shortest route when every move costs the same. It is implemented as an available helper; the game does not call it during normal play.
- `find_path_dfs()` (line 76) follows one branch deeply using a stack before backtracking. It can find a route, but it does not promise the shortest one. It is also an available helper, not the normal gameplay route finder.
- `get_hint_route()` (line 162) plans the player's hint. If an uncollected key is present, it joins a route to the key with a route from the key to the exit; otherwise it routes directly to the exit.
- `get_neighbors()` (line 14) supplies the adjacent walkable cells, and `reconstruct_path()` (line 25) turns recorded parent links into an ordered path.

The agent calculates its route through `Agent.calculate_path()` in `game/agent.py` (line 58), which calls the hint-route planner. It then advances along that planned path in `Agent.update()` (line 80). This agent follows a route already calculated for the maze; it is not a machine-learning model.

### Movement, scoring, and hints

`Player.move()` in `game/player.py` checks whether a requested neighboring cell is walkable before changing the player's grid position. `Player.check_collectibles()` (line 76) removes an item from the maze and updates the player's inventory and score.

`MazeExplorerApp` in `game/app.py` coordinates those pieces:

- `_init_level()` (line 261) creates the maze, player, and (in Race mode) agent.
- `trigger_hint()` (line 334) checks the remaining hint count and cooldown before requesting a route.
- `_on_player_moved()` (line 465) handles pickups and checks whether the player reached the exit.
- `_complete_level()` (line 490) applies the clear and time bonuses and changes the screen to the level-clear state.
- `update()` (line 512) advances timers and the agent, handles continuous key movement, and checks for timeout.
- `render()` (line 544) asks the renderer to draw the screen appropriate to the current game state.

Coins are worth 50 points, gems 150, and a key 200 (`config.py`, around lines 76-86). A level-clear bonus is based on difficulty and level, with an additional 1,500 points for clearing a race; the time bonus depends on remaining time for timed levels. The exact calculation is in `_complete_level()`.

### What the Python data structures are doing

These examples explain the structures used by this code, rather than suggesting extra features:

| Structure | Plain-language meaning | Where and why this project uses it |
|---|---|---|
| **List** (`list`) | An ordered, changeable collection. | The maze grid is a list of row lists; paths, buttons, and particles are lists. A list is also used as a **stack** where the newest item is added with `append()` and removed with `pop()`, in maze generation and DFS. |
| **Tuple** (`tuple`) | A small, fixed group of values. | A grid position is `(row, column)`, such as `(1, 1)`. It is convenient to pass around and use as a dictionary or set key. |
| **Dictionary / map** (`dict`) | Looks up a value using a key. | `maze.collectibles` maps a position tuple to an item type. A* uses dictionaries for parent links and distances; settings and difficulty configuration are also key/value dictionaries. Python's built-in `map()` function is not used here. |
| **Set** (`set`) | A collection of unique values, with quick membership checks. | Maze generation and pathfinding use sets such as `visited` and `closed_set` to avoid processing a position repeatedly. |
| **Queue** | First in, first out: the oldest waiting item is handled first. | BFS uses `collections.deque`; `popleft()` takes the next position to explore. |
| **Heap / priority queue** | Quickly takes the item with the best priority. | A* uses Python's `heapq` to take the position with the lowest estimated total route cost. |
| **Lambda** (`lambda`) | A short function written inline. | `game/app.py` attaches callbacks to buttons. A callback runs later, when the button is clicked. In the difficulty-button loop, `lambda d=d: ...` captures that button's own difficulty instead of accidentally using the loop's final value. |

## What does "hosting" mean for this project?

This repository is a **desktop game**, not a website or server. Running `python main.py` starts the game on the computer where the command is run; there is no web server, hosting configuration, or browser version in the project.

To share the current version, another person needs the project files, Python, and Pygame, then follows the setup steps above. The repository does not currently include a packaged installer or executable. Hosting it as a browser game would require a separate web-based implementation or a suitable Python-to-browser approach; it is not enabled by these files.

## Project files: what each one does

| File or folder | Purpose | If it were missing |
|---|---|---|
| `main.py` | Program entry point, command-line options, startup, and top-level error display. | There would be no standard command to launch the game. |
| `config.py` | Shared colors, grid constants, item values, difficulty settings, paths, and window settings. | The other modules that import these values would fail to load. |
| `requirements.txt` | Lists Pygame, the external runtime dependency. | Setup would no longer have a one-command dependency list; Pygame might not be installed. |
| `run_game.bat` | Windows convenience launcher with Python detection and an error pause. | You could still launch with `python main.py`; only the shortcut would be lost. |
| `README.md` | This setup and code explanation. | The game would still run, but this documentation would be gone. |
| `.gitignore` | Keeps common Python caches, virtual environments, logs, and local files out of Git. | The game could still run, but generated or local files could more easily be added to version control by accident. |
| `game/__init__.py` | Marks `game` as the project's Python package. | Modern Python can sometimes import it as a namespace package, but keeping this file makes the package explicit and supports predictable imports. |
| `game/app.py` | Main controller: screens/states, input, game loop, level setup, scoring, and coordination of game objects. | There would be no code to coordinate a playable session. |
| `game/maze_generator.py` | Builds the grid maze and places/looks up collectibles. | The game would have no `Maze` implementation for levels to use. |
| `game/pathfinding.py` | Neighbor lookup, path reconstruction, BFS, DFS, A*, and hint-route planning. | Hints and the agent's route planning would not work; the app imports these functions. |
| `game/player.py` | Player position, legal movement, inventory, score, and visual movement state. | There would be no player movement or pickup logic. |
| `game/agent.py` | Race opponent: route execution, step timing, progress, and visual trail data. | The application imports this module, so it would fail to start even in Solo mode. |
| `game/renderer.py` | Draws menus, maze, characters, collectibles, HUD, fog, hints, particles, and result screens. | The game would have no visual interface. |
| `game/storage.py` | Reads and writes the saved difficulty preference in JSON. | The game would lose its settings persistence and the app's settings imports would fail. |
| `data/settings.json` | Current saved difficulty preference; this checked-in copy says `Easy`. | Not fatal: storage uses the default difficulty (`Medium`) if no settings file exists, and can create the data directory when saving. |
| `data/.gitkeep` | Keeps `data/` represented in Git if its generated or settings files are absent. | Not fatal: `game/storage.py` creates the directory when it saves settings. |
| `assets/README.md` | Notes that the game draws its visuals in Pygame and describes the assets folder. | No gameplay code would be lost; the current renderer does not require external image or sound files. |

There are no Python test source files tracked in this checkout. A local `tests` folder may contain old compiled `.pyc` cache files, but those are not a runnable test suite. Also, although the assets note mentions synthesized audio, this source tree has no sound manager or sound module, and the gameplay described above does not depend on audio.

## Main source locations

Line references here are starting lines for the relevant class or function:

| Topic | Source |
|---|---|
| Command-line parsing and app startup | `main.py:10`, `main.py:41` |
| Application setup, game states, and UI callbacks | `game/app.py:51`, `game/app.py:132` |
| Starting and building a level | `game/app.py:254`, `game/app.py:261` |
| Hints, input, movement, and pickups | `game/app.py:334`, `game/app.py:365`, `game/app.py:448`, `game/app.py:465` |
| Scoring, updates, drawing, and main loop | `game/app.py:490`, `game/app.py:512`, `game/app.py:544`, `game/app.py:588` |
| Maze generation and item placement | `game/maze_generator.py:18`, `game/maze_generator.py:71`, `game/maze_generator.py:110`, `game/maze_generator.py:131` |
| BFS, DFS, A*, and key-aware hint route | `game/pathfinding.py:42`, `game/pathfinding.py:76`, `game/pathfinding.py:115`, `game/pathfinding.py:162` |
| Player movement and item collection | `game/player.py:48`, `game/player.py:76` |
| Agent route and timed movement | `game/agent.py:58`, `game/agent.py:80` |
| Screen sizing, gameplay rendering, and fog | `game/renderer.py:228`, `game/renderer.py:244`, `game/renderer.py:365` |
| Saved settings | `game/storage.py:24`, `game/storage.py:46` |
| Difficulty configuration | `config.py:89` |
