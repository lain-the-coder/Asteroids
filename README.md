# Asteroids

A faithful re-creation of the classic 1979 Atari arcade game *Asteroids*, built from scratch in Python using [pygame](https://www.pygame.org/). The project is structured around clean object-oriented design, sprite groups, and a delta-time game loop, making it both playable and a useful reference for game-loop fundamentals.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Controls](#controls)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Running the Game](#running-the-game)
- [Configuration](#configuration)
- [Logging and Telemetry](#logging-and-telemetry)
- [Development](#development)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgements](#acknowledgements)

---

## Overview

In *Asteroids*, the player pilots a small spaceship in a 2D field of drifting asteroids. The objective is simple: survive as long as possible by shooting asteroids before they collide with your ship. Each asteroid struck splits into two smaller fragments, until they shatter completely.

This implementation focuses on:

- A frame-rate-independent game loop using delta time
- Sprite group management via pygame's built-in `pygame.sprite.Group`
- Inheritance-based game objects built on a shared `CircleShape` base class
- Event and state logging to JSONL for debugging and analytics

---

## Features

- Smooth, rotation-based player movement with forward and reverse thrust
- Continuously spawning asteroids that enter from random screen edges
- Asteroid splitting — large asteroids break into two smaller ones when shot
- Projectile system with configurable shoot cooldown
- Circle-based collision detection between player, asteroids, and shots
- Structured JSONL logging for game state snapshots and discrete events
- All gameplay values centralised in a single `constants.py` for easy tuning

---

## Controls

| Key          | Action          |
|--------------|-----------------|
| `W`          | Thrust forward  |
| `S`          | Thrust backward |
| `A`          | Rotate left     |
| `D`          | Rotate right    |
| `Space`      | Fire projectile |
| Window close | Quit game       |

---

## Architecture

The codebase follows a lightweight component model built on top of pygame's sprite system.

### Class Hierarchy

```
pygame.sprite.Sprite
        |
        +-- CircleShape (base)
                |
                +-- Player
                +-- Asteroid
                +-- Shot
```

`CircleShape` provides the shared interface — `position`, `velocity`, `radius`, and a `collides_with()` method — that all gameplay objects build on.

### Sprite Groups

Four `pygame.sprite.Group` instances coordinate updates, rendering, and collisions:

| Group       | Purpose                                                    |
|-------------|------------------------------------------------------------|
| `updatable` | Every object whose `update(dt)` is called per frame        |
| `drawable`  | Every object whose `draw(screen)` is called per frame      |
| `asteroids` | Asteroids only — used for player and shot collision checks |
| `shots`     | Player projectiles — used for asteroid collision checks    |

Each subclass declares its membership through a class-level `containers` attribute, and `CircleShape.__init__` automatically registers new instances with the relevant groups.

### Game Loop

The main loop in `main.py` follows the canonical pattern:

1. Tick the clock and compute `dt` (seconds since last frame, capped at 60 FPS).
2. Process input by polling the pygame event queue.
3. Update every object in the `updatable` group.
4. Detect collisions between player, asteroids, and shots.
5. Render every object in the `drawable` group.
6. Flip the display buffer.

---

## Project Structure

```
asteroids/
├── main.py              # Entry point: game loop, group setup, collision logic
├── player.py            # Player ship: input handling, movement, shooting
├── asteroid.py          # Asteroid: drift, splitting behaviour
├── asteroidfield.py     # Spawner: emits asteroids from random screen edges
├── shot.py              # Projectile fired by the player
├── circleshape.py       # Shared base class for all circular game objects
├── constants.py         # Centralised gameplay tuning values
├── logger.py            # JSONL state and event logging utilities
├── pyproject.toml       # Project metadata and dependency declarations
└── README.md            # You are here
```

---

## Installation

### Prerequisites

- Python 3.13 or later
- A package manager — [`uv`](https://docs.astral.sh/uv/) is recommended for speed and reproducibility, but `pip` works fine

### Clone the Repository

```bash
git clone https://github.com/<your-username>/asteroids.git
cd asteroids
```

### Install Dependencies

Using uv (recommended):

```bash
uv sync
```

Using pip:

```bash
python -m venv .venv
source .venv/bin/activate          # On Windows: .venv\Scripts\activate
pip install pygame==2.6.1
```

---

## Running the Game

With uv:

```bash
uv run main.py
```

With pip / activated virtualenv:

```bash
python main.py
```

On launch, the console prints the pygame version and configured screen dimensions. A 1280x720 window opens and the game begins immediately.

---

## Configuration

All gameplay parameters live in [`constants.py`](constants.py). Adjusting them is the fastest way to tweak difficulty or feel.

### Display

| Constant        | Default | Description                  |
|-----------------|---------|------------------------------|
| `SCREEN_WIDTH`  | 1280    | Window width in pixels       |
| `SCREEN_HEIGHT` | 720     | Window height in pixels      |
| `LINE_WIDTH`    | 2       | Outline thickness for shapes |

### Player

| Constant                        | Default | Description                       |
|---------------------------------|---------|-----------------------------------|
| `PLAYER_RADIUS`                 | 20      | Collision radius of the ship      |
| `PLAYER_SPEED`                  | 200     | Thrust speed in pixels/second     |
| `PLAYER_TURN_SPEED`             | 300     | Rotation speed in degrees/second  |
| `PLAYER_SHOOT_SPEED`            | 500     | Projectile speed in pixels/second |
| `PLAYER_SHOOT_COOLDOWN_SECONDS` | 0.3     | Seconds between consecutive shots |

### Asteroids

| Constant                      | Default                        | Description                       |
|-------------------------------|--------------------------------|-----------------------------------|
| `ASTEROID_MIN_RADIUS`         | 20                             | Smallest asteroid radius          |
| `ASTEROID_KINDS`              | 3                              | Number of asteroid size tiers     |
| `ASTEROID_MAX_RADIUS`         | `ASTEROID_MIN_RADIUS * 3` = 60 | Largest asteroid radius (derived) |
| `ASTEROID_SPAWN_RATE_SECONDS` | 0.8                            | Seconds between asteroid spawns   |

### Shots

| Constant      | Default | Description       |
|---------------|---------|-------------------|
| `SHOT_RADIUS` | 5       | Projectile radius |

---

## Logging and Telemetry

The `logger.py` module emits two JSONL files at the project root on each run. Both files are overwritten at the start of every session.

### `game_state.jsonl`

A periodic snapshot of all sprite groups, captured roughly once per second for the first 16 seconds of the run. Each line contains:

- `timestamp` — wall-clock time (`HH:MM:SS.mmm`)
- `elapsed_s` — seconds since process start
- `frame` — frame counter
- `screen_size` — current window dimensions
- A summary of each sprite group: `count` plus a sample of up to 10 sprites, each with their type, position, velocity, radius, and rotation (when applicable)

### `game_events.jsonl`

Discrete gameplay events, emitted as they occur. Currently logged events:

| Event Type       | Trigger                                   |
|------------------|-------------------------------------------|
| `player_hit`     | Asteroid collides with the player         |
| `asteroid_shot`  | A shot connects with an asteroid          |
| `asteroid_split` | An asteroid splits into smaller fragments |

These logs are useful for replay analysis, debugging difficulty curves, or training simple gameplay agents.

---

## Development

### Code Style

The project keeps to a small, readable footprint:

- Each game object lives in its own module
- Behaviour is split between `update(dt)` (logic) and `draw(screen)` (rendering)
- Shared maths and collision logic lives on `CircleShape`

### Adding a New Entity

1. Subclass `CircleShape` in a new module.
2. Implement `update(dt)` and `draw(screen)`.
3. In `main.py`, declare a `containers` tuple before instantiating, e.g.:
   ```python
   MyEntity.containers = (updatable, drawable)
   ```
4. Instantiate where appropriate; the base class auto-registers it with the right groups.

### Tuning Difficulty

The most impactful knobs are `ASTEROID_SPAWN_RATE_SECONDS` (lower = harder), `PLAYER_SHOOT_COOLDOWN_SECONDS` (higher = harder), and `PLAYER_SPEED` (lower = harder).

---

## Roadmap

Potential improvements for future iterations:

- Score tracking and high-score persistence
- Lives system instead of instant game-over
- Screen-wrap behaviour for the player and asteroids (classic Asteroids feel)
- Sound effects and background music
- Particle effects for explosions and thrust
- Main menu, pause menu, and game-over screen
- Power-ups (rapid fire, shield, multi-shot)
- UFO enemies that fire back
- Configurable difficulty levels
- Unit tests around collision and splitting logic

---

## Contributing

Contributions are welcome. To propose a change:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes with a clear message
4. Push to your fork and open a Pull Request

For larger changes, please open an issue first to discuss the proposed direction.

---

## License

This project is released under the MIT License. See [`LICENSE`](LICENSE) for details.

---

## Acknowledgements

- Inspired by Atari's original *Asteroids* (1979)
- Built with [pygame](https://www.pygame.org/), the long-standing Python game development library
- Project scaffolding inspired by the [Boot.dev](https://www.boot.dev/) curriculum
