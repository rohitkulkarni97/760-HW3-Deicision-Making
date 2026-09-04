# A Generic, Type-Safe State Machine for Unity Agents

**IGME 760.01 · AI for Gameplay · Rochester Institute of Technology · Homework 3**

Decision-making for two agent types that share **one state machine implementation**. The framework is
generic over the state enum, the agent type, and the machine itself — so `Enemy` and `Player` get
compile-time-checked states with no casting, no `switch` statements, and no duplicated transition code.

The player is **not** player-controlled. Both sides are AI: the player agent decides for itself whether to
pursue its objective, fight, or run, based on how much health it has left.

<!--
  SHOT LIST — replace this block with real media.
  1. states.gif    — THE hero shot. The state label floats over each agent's head at runtime, so a single
                     GIF shows the whole system: watch the player flip GoToTarget -> Attack -> Flee as an
                     enemy closes in and its health drops. Nothing else demos this project as well.
  2. flee.gif      — player at low health running to a safe point while enemies converge.
  3. gizmos.png    — Scene view: perception radius spheres (black -> red when in range) plus the black
                     A* path line from the movement component.
  Put files in docs/ and reference as ![States](docs/states.gif)
  HERO IMAGE — uncomment this line once the media exists:
  > **Demo GIF goes here** — live state labels above each agent as they transition.
-->

---

## The framework

Three layers, each adding one thing.

### Layer 1 — a state machine that knows nothing about agents

```csharp
public abstract class StateMachine<TState> : MonoBehaviour where TState : Enum
public abstract class BaseState<TState> where TState : Enum
```

The update loop is four lines and contains the whole idea — **states decide their own transitions**:

```csharp
var nextStateKey = CurrentState.GetNextState();
if (nextStateKey.Equals(CurrentState.StateKey))
    CurrentState.UpdateState();   // stay put, keep working
else
    CurrentState = States[nextStateKey];   // property setter fires ExitState/EnterState
```

There is no central transition table to keep in sync. Each state answers one question — *given what I can
see right now, who should be running?* — and the machine does the bookkeeping. `CurrentState` is a property
whose setter calls `ExitState()` on the outgoing state and `EnterState()` on the incoming one, guarded by an
`_isSwitchingStates` flag so a transition can never re-enter itself mid-swap.

The machine also **forwards Unity's physics callbacks into the active state**. `OnTriggerEnter2D`,
`OnCollisionEnter2D` and friends are declared `virtual` on `BaseState`, so a state can react to a collision
directly instead of the MonoBehaviour having to ask "which state am I in?" first. States can start and stop
coroutines too, proxied through the owning machine.

### Layer 2 — agents, tied together with recursive generics

```csharp
public abstract class Agent<TState, TStateMachine, TAgent> : MonoBehaviour
    where TStateMachine : AgentStateMachine<TState, TAgent, TStateMachine>
    where TState        : Enum
    where TAgent        : Agent<TState, TStateMachine, TAgent>
```

That third constraint — `TAgent : Agent<..., TAgent>` — is the curiously recurring template pattern. It is
what makes `Agent` property inside a state return the **concrete** agent type. Inside `EnemyAttackState`,
`Agent` is an `Enemy`, not an `Agent` needing a cast. Get the generics right once and every state written
afterwards is type-safe for free.

Each agent composes three serialized components: `AgentMovement`, `AgentHealth`, and its state machine.

### Layer 3 — the concrete agents

`AgentStateMachine` adds one small thing that makes the whole project demoable: an optional `TMP_Text` that
is set to the current state key every frame. The state of every agent is legible on screen while the game
runs, which is the difference between debugging AI and guessing at it.

---

## The behaviours

**Enemy — two states.** Wander picks a random walkable point and re-picks whenever it arrives; Attack paths
to the player's live position. The transition both ways is `HasEnemyInRange()`.

**Player — four states, and the interesting one is Flee.**

| State | What it does | Transitions out |
|---|---|---|
| **Idle** | Nothing — the resting state | Target exists → GoToTarget · Enemy in range → `health < 30% ? Flee : Attack` |
| **GoToTarget** | Paths to the objective | No target → Idle · Enemy in range → `health < 30% ? Flee : Attack` |
| **Attack** | Picks the **nearest living enemy** by linear scan and re-paths to it whenever it crosses into a new grid node | `health <= 30%` → Flee · No enemy in range → GoToTarget |
| **Flee** | Runs to a point no enemy can reach | No enemy in range → GoToTarget · `health > 30%` → Attack |

Three things make this read as a decision rather than a reflex.

**Fleeing is a search, not a direction.** Running directly away from a threat walks you into walls and into
other enemies. Instead the agent samples random walkable points and **rejects any point within
`EnemySearchRadius` of any living enemy**, then paths there through A*:

```csharp
var fleePoint = GameManager.Instance.GroundSystem.GetRandomPoint();
while (GameManager.Instance.Enemies.Where(enemy => enemy)
       .Any(enemy => Vector2.Distance(enemy.transform.position, fleePoint) <= GameManager.Instance.EnemySearchRadius))
{
    fleePoint = GameManager.Instance.GroundSystem.GetRandomPoint();   // not safe, try another
}
```

**The 30% health threshold is a two-way door, because health regenerates.** `PlayerHealth` overrides
`AgentHealth` to add `Hp += rejuvenationPerSecond * Time.deltaTime` — a slow 0.1/sec trickle. That single
line is what turns the threshold into a decision instead of a death spiral: the agent breaks off at 30%,
survives long enough to tick back above it, and re-engages. The same number gates entry from `GoToTarget`
and `Idle`, exit from `Attack`, and the return out of `Flee`.

**Attack commits to a specific enemy.** It does not just move toward "danger" — `GetNearestEnemy()` scans
the live enemy array, skips destroyed entries, and picks the closest. That target is then re-evaluated each
frame, but the agent only re-paths when the enemy actually crosses into a different grid node, so pursuit
does not thrash the pathfinder.

`GoToTarget` also uses the physics forwarding from Layer 1 — it overrides `OnCollisionEnter2D`, checks for
the `Target` tag, destroys the objective and raises `TargetAchieved()` on the agent. The collision handling
lives in the one state where it is meaningful.

---

## Movement, reused

`GroundSystem` is carried over from [HW2](https://github.com/rohitkulkarni97/760-HW2-Pathfinding)
unchanged — same grid generation, same `OrderBy(F).ThenBy(H)` A\* — with two additions: `Init()` is now
public so `GameManager` controls when the grid builds, and `GetRandomPoint()` returns a point from a cached
`WalkableNodes` array, which is what Wander and Flee sample against.

`AgentMovement` is the new layer on top, built for agents whose goals move:

- Paths are consumed a node at a time and lerped over `timeBetweenNodes`
- A `_targetChanged` flag defers the re-path to the moment the agent finishes its current segment, so it
  never rubber-bands mid-step
- The live path is mirrored into a `Node[]` under `#if UNITY_EDITOR` purely so gizmos can draw it without
  disturbing the `Stack` the traversal is popping from

That editor-only mirror is the kind of thing you only write after debugging the alternative.

---

## Code map

| File | Responsibility |
|---|---|
| [`StateMachine.cs`](Assets/Scripts/State%20Machine/StateMachine.cs) · [`BaseState.cs`](Assets/Scripts/State%20Machine/BaseState.cs) | The generic framework. Agent-agnostic. |
| [`Agent.cs`](Assets/Scripts/Characters/Base/Agent.cs) | Recursive-generic agent base. Composes movement, health, state machine. |
| [`AgentStateMachine.cs`](Assets/Scripts/Characters/Base/State%20Machine/AgentStateMachine.cs) | Binds a machine to its agent; renders the live state label. |
| [`AgentMovement.cs`](Assets/Scripts/Characters/Base/AgentMovement.cs) | A* traversal with moving-target re-pathing. |
| [`AgentHealth.cs`](Assets/Scripts/Characters/Base/AgentHealth.cs) | Clamped HP, collision damage, normalized value the states branch on. |
| [`Characters/Enemy/`](Assets/Scripts/Characters/Enemy) | Wander, Attack. |
| [`Characters/Player/`](Assets/Scripts/Characters/Player) | Idle, GoToTarget, Attack, Flee. |
| [`GameManager.cs`](Assets/Scripts/GameManager.cs) | Spawns agents and objective at random walkable points; win/lose; async scene reset. |

---

## Running it

- **Unity 2022.3.7f1** (2D)
- Open `Assets/Scenes/Main Scene.unity` and press Play
- Nothing to control — watch the labels above each agent and the round resets after a win or loss
- Adjust `numberOfEnemies` and `enemySearchRadius` on the `GameManager` to change the pressure. More
  enemies makes `Flee` struggle to find a safe point, which is the fun failure mode.

---

## Part of a four-project series

| | Project | Focus |
|---|---|---|
| HW1 | [Arrive Steering](https://github.com/rohitkulkarni97/760-HW1-Movement) | Dynamic movement against 2D physics |
| HW2 | [Pathfinding](https://github.com/rohitkulkarni97/760-HW2-Pathfinding) | A* written from scratch on a generated grid |
| **HW3** | **Decision Making** — you are here | A generic, type-safe finite state machine |
| Project | [Top-Down Shooter](https://github.com/rohitkulkarni97/Top-Down-Shooter-760-AIG) | All three, in a playable game |

Built by [Rohit Kulkarni](https://github.com/rohitkulkarni97) · MS Game Design & Development, RIT
