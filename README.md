# Tank Stars — Multiplayer Artillery Game with a Reinforcement-Learning AI

A 2D turn-based tank artillery game built in **Unity 6**, with an **AI opponent trained using reinforcement learning (Unity ML-Agents, PPO)** and **real-time online multiplayer** on a containerised **Node.js** backend.

> DAM final project (2025–26) · Institut Pedralbes · by **Pariskar Rijal**

---

## Highlights

- 🤖 **Reinforcement-learning opponent** — trained with Unity ML-Agents (PPO) for ~2M steps; ships as an `.onnx` model with a heuristic fallback when no model is loaded.
- 🌐 **Online multiplayer** — room codes, WebSocket game server with **server-authoritative shot simulation** so both players always see the same result.
- 🏔️ **Procedural, destructible terrain** — 5 biomes generated with Perlin FBM noise; craters reshape the mesh and collider on impact.
- 🔐 **REST API with auth** — Express 5, JWT, bcrypt, MySQL, player stats and match results.
- 🐳 **One-command stack** — Docker Compose runs MySQL, API, game server, nginx and the ML trainer.

---

## Game modes

| Mode | How it works |
|---|---|
| **Vs AI** | Play against the trained ML-Agents tank on a procedurally generated map. |
| **Online multiplayer** | Create or join a room with a code; the game server runs the match and syncs every shot. |

**Rules:** turn-based, 15 seconds per turn, barrel angle 0–90°, power 10–100. A direct hit deals 35 HP, a near miss 15 HP. First tank to 0 HP loses.

---

## The AI opponent (Unity ML-Agents)

| | |
|---|---|
| **Algorithm** | PPO (Proximal Policy Optimization) |
| **Network** | 2 hidden layers × 128 units, normalised inputs |
| **Training** | 2,000,000 steps, checkpoints every 100k |
| **Observations (14)** | Both tanks' positions, relative distance and angle to enemy, both HP values, HP difference, current barrel angle, terrain height under each tank |
| **Actions** | 2 continuous outputs → barrel angle (0–90°) and shot power (0–100) |
| **Rewards** | +1.0 for a hit, a penalty proportional to miss distance, a small per-step cost to encourage decisive shots |
| **Fallback** | Built-in heuristic aiming if no trained model is assigned |

Training config: [`Tank Stars/ml-agents/simple.yaml`](Tank%20Stars/ml-agents/simple.yaml) · Trained models: [`Tank Stars/ml-agents/results/`](Tank%20Stars/ml-agents/results)

---

## Architecture

```
                  ┌──────────────────────┐
                  │  Unity 6 client (C#) │
                  │  URP · UI Toolkit    │
                  │  ML-Agents inference │
                  └─────────┬────────────┘
                            │ HTTP + WebSocket
                    ┌───────▼────────┐
                    │     nginx      │  reverse proxy (/api, /game)
                    └───┬────────┬───┘
            REST /api   │        │  WebSocket /game
          ┌─────────────▼─┐   ┌──▼──────────────────┐
          │  API service  │◄──┤  Game service       │
          │  Express 5    │   │  ws · rooms · turns │
          │  JWT · bcrypt │   │  server-side shot   │
          └──────┬────────┘   │  physics (120 Hz)   │
                 │            └─────────────────────┘
          ┌──────▼──────┐
          │  MySQL 8    │  users · games · results
          └─────────────┘

  ML trainer container (mlagents-learn, PPO) ◄── Unity Editor (port 5004)
```

**Backend design notes**

- **Server authority:** the game service simulates each projectile itself (gravity, step size 1/120 s) and resolves hits and craters, so clients can't disagree or cheat.
- **Repository pattern:** API data access goes through `UserRepository` / `GameRepository` interfaces with **MySQL** and **in-memory** implementations, making the API easy to test.
- **Health-checked startup:** the API waits for MySQL to be healthy before starting.

---

## Tech stack

| Layer | Technology |
|---|---|
| Game client | Unity 6, C#, URP, UI Toolkit, Unity 2D Physics |
| AI | Unity ML-Agents, PPO, ONNX |
| Game server | Node.js, `ws` (WebSocket) |
| API | Node.js, Express 5, JWT, bcrypt |
| Database | MySQL 8 |
| Infrastructure | Docker Compose, nginx |

---

## Project structure

```
Tank Stars/
├── client/UnityTankStar/   # Unity game (Assets/Scripts: TankAgent, TerrainGenerator, CombatManager…)
├── server/
│   ├── api/                # REST API: auth, games, results, repositories, schema.sql
│   └── game/               # WebSocket game server
├── ml-agents/              # PPO config, Dockerfile, trained models
├── docker-compose.yml
└── nginx.conf
specs/                      # foundations, technical spec and implementation plan
```

---

## Run it

**Play:** download the build for your platform from this repo — `TANKSTARS_WINDOWS.zip` or `TankStart_Mac.zip`.

**Backend (multiplayer + API):**

```bash
cd "Tank Stars"
docker compose up --build mysql api game nginx
```

**Train the AI yourself:**

```bash
cd "Tank Stars"
docker compose run --rm --service-ports trainer \
  mlagents-learn /workspace/simple.yaml --run-id=tankAgent1
# then press Play in the Unity Editor to connect on port 5004
```

---

## Documentation

- [Foundations](specs/foundations.md) — overview, stack, key systems, design principles
- [Technical spec](specs/spec.md) — terrain, tanks, projectiles, game states
- [Implementation plan](specs/plan.md) — phased build plan, from terrain to AI and multiplayer

---

## Author

**Pariskar Rijal** — Software Developer, Barcelona
[LinkedIn](https://www.linkedin.com/in/pariskar-rijal-2b25a0257/) · [GitHub](https://github.com/pariskarrijal)
