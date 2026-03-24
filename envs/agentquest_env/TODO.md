# AgentQuest Mini-Game RL Environment - Implementation Plan

## Overview
Transform AgentQuest from a simple echo environment into a sophisticated mini-game RL environment with:
- Clearly defined tasks with progressive difficulty
- Automated graders for task verification
- Structured reward logic with bonuses/penalties
- OpenEnv packaging for evaluation

---

## Phase 1: Game Design & Architecture

### 1.1 Define the Game Concept
- [ ] Choose game type (e.g., puzzle-solving, resource management, navigation, strategy)
  - **Options to consider:**
    - **Treasure Hunt**: Agent solves riddles/puzzles in sequence to find treasures
    - **Quest Builder**: Agent completes quests with specific requirements
    - **Tower Defense Mini**: Agent places defenses optimally against waves
    - **Trading Simulator**: Agent buys/sells resources, learns market dynamics
    - **Dungeon Crawler**: Agent navigates rooms, solves puzzles, defeats enemies
  - [ ] Document game rules, objectives, and mechanics
  - [ ] Create a game design document (GDD)

### 1.2 Define Task Hierarchy
- [ ] Create 3-5 task difficulty levels:
  - [ ] Level 1 (Tutorial): Trivial task, 100% success baseline
  - [ ] Level 2 (Easy): Basic strategy, ~70% success target
  - [ ] Level 3 (Medium): Multi-step planning, ~40% success target
  - [ ] Level 4 (Hard): Complex optimization, ~15% success target
  - [ ] Level 5 (Expert): Edge cases, luck-based elements, <5% success target
- [ ] Define success criteria for each level
- [ ] Document expected agent strategies per level

### 1.3 Design Observation/Action Space
- [ ] Define observation structure (what agent sees):
  - [ ] Game state representation (JSON-serializable)
  - [ ] Difficulty indicator
  - [ ] Progress/history tracking
  - [ ] Hints or guidance (optional)
- [ ] Define action space:
  - [ ] Type of actions (discrete enum vs. continuous vs. structured)
  - [ ] Action constraints per difficulty level
  - [ ] Invalid action handling

---

## Phase 2: Core Game Logic Implementation

### 2.1 Update `models.py`
- [ ] Replace `AgentquestAction` with game-specific action:
  - [ ] Define action enum or structured action class
  - [ ] Add validation for action constraints
  - [ ] Example: `class QuestAction(Action): choice: int; strategy: Optional[str]`
- [ ] Replace `AgentquestObservation` with game observation:
  - [ ] Game state JSON structure
  - [ ] Current difficulty level
  - [ ] Available actions list
  - [ ] Status message for agent feedback
  - [ ] Metadata (turn count, progress, etc.)
  - [ ] Example: `class QuestObservation(Observation): state: dict; difficulty: int; available_actions: List[str]`

### 2.2 Implement Game Engine (`AgentQuest_environment.py`)
- [ ] Create `GameState` dataclass to track:
  - [ ] Current difficulty level
  - [ ] Task progress (0-100%)
  - [ ] Agent inventory/resources (if applicable)
  - [ ] Game-specific state variables
  - [ ] Episode history (list of actions taken)
  - [ ] Random seed for reproducibility
- [ ] Implement `__init__()`:
  - [ ] Initialize game parameters
  - [ ] Set up random number generator with seed support
  - [ ] Create OpenEnv `State` object
- [ ] Implement `reset()`:
  - [ ] Generate new task based on difficulty level
  - [ ] Return initial `GameObservation`
  - [ ] Reset all game state variables
  - [ ] Record episode_id
- [ ] Implement `step(action)`:
  - [ ] Validate action
  - [ ] Apply action to game state
  - [ ] Update progress metrics
  - [ ] Return updated observation
  - [ ] Track step count

### 2.3 Implement Task Generator
- [ ] Create `TaskGenerator` class:
  - [ ] `generate_task(difficulty: int) -> Task` method
  - [ ] Parametrized task generation (difficulty affects complexity)
  - [ ] Deterministic generation (same seed = same task)
  - [ ] Task includes: objectives, constraints, initial state
- [ ] Create 5 task templates (one per difficulty level):
  - [ ] Serialize tasks to JSON for observation transmission

### 2.4 Implement Game State Progression
- [ ] Add logic to track task completion percentage
- [ ] Implement difficulty progression:
  - [ ] Start at level 1 by default
  - [ ] Allow reset to specific difficulty level
  - [ ] Option: Auto-increase difficulty on episode success
- [ ] Handle edge cases:
  - [ ] Invalid action handling (penalize or reject)
  - [ ] Partial task completion
  - [ ] Task timeout logic (if applicable)

---

## Phase 3: Grading System Implementation

### 3.1 Create `Grader` Base Class
- [ ] Define abstract grader interface:
  - [ ] `grade(task: Task, agent_history: List[Action]) -> GradeResult` method
  - [ ] Return: (success: bool, score: float, feedback: str)
  - [ ] Support composable graders

### 3.2 Implement Task-Specific Graders
- [ ] Create grader for each difficulty level:
  - [ ] `Level1Grader`: Check task completion
  - [ ] `Level2Grader`: Check completion + efficiency metric
  - [ ] `Level3Grader`: Check + multi-step strategy validation
  - [ ] `Level4Grader`: Check + optimal path scoring
  - [ ] `Level5Grader`: Check + near-optimal scoring with randomness
- [ ] Grader evaluation criteria:
  - [ ] Task objective completion (pass/fail)
  - [ ] Efficiency score (steps taken vs. optimal)
  - [ ] Strategy quality (does agent use smart tactics?)
  - [ ] Edge case handling (agent robust to variations?)

### 3.3 Implement Auto-Grading Pipeline
- [ ] Create `AutoGrader` class that:
  - [ ] Compiles action history into structured format
  - [ ] Validates task completion
  - [ ] Computes intermediate scores
  - [ ] Generates detailed feedback
  - [ ] Returns final `GradeResult(success, score, feedback, details)`
- [ ] Add grading hooks to `step()` method:
  - [ ] Check task progress after each action
  - [ ] Implement early termination if task fails
  - [ ] Collect grading data during episode

### 3.4 Validation & Testing
- [ ] Unit tests for each grader
- [ ] Test graders with:
  - [ ] Perfect solution (should score 100%)
  - [ ] Terrible solution (should score 0%)
  - [ ] Partial solutions
  - [ ] Edge cases

---

## Phase 4: Reward Logic Implementation

### 4.1 Design Multi-Component Reward Function
- [ ] Define reward components:
  - [ ] **Completion Reward**: +100 for task success (binary)
  - [ ] **Efficiency Reward**: 0 to +50 based on steps taken vs optimal
  - [ ] **Strategy Reward**: 0 to +30 for using correct strategies
  - [ ] **Exploration Penalty**: -0.5 per invalid action (exploration cost)
  - [ ] **Time Penalty**: -0.1 per step taken (living cost)
  - [ ] **Difficulty Bonus**: 2x reward multiplier for harder tasks (scaling)
- [ ] **Total Reward = Completion + Efficiency + Strategy - Exploration - Time × DifficultyMultiplier**

### 4.2 Implement Reward Calculator
- [ ] Create `RewardCalculator` class:
  - [ ] `calculate(task_result: TaskResult, agent_history: List[Action], difficulty: int) -> float` method
  - [ ] Normalize rewards (-100 to +300 range)
  - [ ] Configurable weights for each component
- [ ] Create variant reward functions:
  - [ ] Dense rewards (reward every step toward solution)
  - [ ] Sparse rewards (only reward on completion)
  - [ ] Shaped rewards (with potential-based shaping)

### 4.3 Integrate Rewards into `step()` Method
- [ ] Calculate and return reward for every step
- [ ] Track cumulative episode reward
- [ ] Bonus reward logic:
  - [ ] Speed bonus: Extra reward if task completed quickly
  - [ ] Perfect run bonus: Extra reward with zero invalid actions
  - [ ] Difficulty multiplier: Scale rewards by difficulty level

### 4.4 Reward Testing
- [ ] Verify reward ranges (sanity checks)
- [ ] Test reward progression:
  - [ ] Early steps: Low rewards
  - [ ] Progress steps: Increasing reward signals
  - [ ] Completion: High reward
- [ ] Test difficulty scaling (harder tasks yield higher rewards)

---

## Phase 5: State Management & Persistence

### 5.1 Enhanced State Tracking
- [ ] Extend `GameState` dataclass to track:
  - [ ] Episode timestamp
  - [ ] Difficulty level history
  - [ ] Action history (list of actions with timestamps)
  - [ ] Intermediate task progress snapshots
  - [ ] Grading results (if available)
- [ ] Implement state serialization:
  - [ ] Convert `GameState` to JSON for logging
  - [ ] Support state reconstruction from JSON

### 5.2 Episode Logging & Metrics
- [ ] Collect episode statistics:
  - [ ] Total steps taken
  - [ ] Total reward accumulated
  - [ ] Task completion status
  - [ ] Agent efficiency score
  - [ ] Invalid actions count
- [ ] Optional: Store episode logs:
  - [ ] Save episode replay to file (JSON format)
  - [ ] Include all observations and actions
  - [ ] Include grading results

### 5.3 Difficulty Mode Persistence
- [ ] Implement difficulty switching:
  - [ ] Allow agent to request specific difficulty
  - [ ] Support progressive difficulty (auto-increase on success)
  - [ ] Reset difficulty on explicit request
- [ ] Track difficulty progression statistics

---

## Phase 6: Client-Side Implementation

### 6.1 Update `client.py`
- [ ] Update `AgentquestEnv` class to `AgentQuestEnv`:
  - [ ] Update generics: `EnvClient[QuestAction, QuestObservation, State]`
  - [ ] Implement `_step_payload()` (convert action to dict)
  - [ ] Implement `_parse_result()` (parse server response)
  - [ ] Implement `_parse_state()` (parse state from server)
- [ ] Add convenience methods:
  - [ ] `set_difficulty(level: int) -> QuestObservation`
  - [ ] `get_task_description() -> str`
  - [ ] `get_episode_stats() -> dict`

### 6.2 Implement Example Agent
- [ ] Create `example_agent.py`:
  - [ ] Implement random baseline agent
  - [ ] Implement greedy heuristic agent
  - [ ] Document expected performance per difficulty
  - [ ] Add to examples in README

### 6.3 Update `__init__.py`
- [ ] Export new classes:
  - [ ] `QuestAction`, `QuestObservation`
  - [ ] `GameState`
  - [ ] `AgentQuestEnv`

---

## Phase 7: Testing & Validation

### 7.1 Unit Tests
- [ ] Test game logic:
  - [ ] `test_task_generation_determinism()` (same seed = same task)
  - [ ] `test_action_validation()`
  - [ ] `test_state_transitions()`
  - [ ] `test_reward_calculation()`
- [ ] Test graders:
  - [ ] `test_grader_perfect_solution()`
  - [ ] `test_grader_invalid_solution()`
  - [ ] `test_grader_edge_cases()`
- [ ] Test client:
  - [ ] `test_client_reset()`
  - [ ] `test_client_step()`
  - [ ] `test_client_difficulty_switching()`

### 7.2 Integration Tests
- [ ] Full episode tests:
  - [ ] `test_episode_reset_to_completion(difficulty=1)`
  - [ ] `test_episode_reset_to_completion(difficulty=5)`
  - [ ] `test_episode_invalid_actions()`
  - [ ] `test_episode_reward_accumulation()`
- [ ] Multi-episode tests:
  - [ ] Progressive difficulty increase
  - [ ] Difficulty reset functionality

### 7.3 Performance Tests
- [ ] Benchmark task generation speed
- [ ] Benchmark grading speed
- [ ] Benchmark environment step latency
- [ ] Set performance targets (e.g., step < 100ms)

### 7.4 Validation Against Design
- [ ] Verify task difficulty scaling:
  - [ ] Level 1: >90% success rate expected
  - [ ] Level 3: ~40% success rate expected
  - [ ] Level 5: <5% success rate expected
- [ ] Verify reward ranges per level
- [ ] Verify grader accuracy on known solutions

---

## Phase 8: Documentation & Deployment

### 8.1 Update Documentation
- [ ] Rewrite `README.md`:
  - [ ] New game description
  - [ ] Game rules & objectives
  - [ ] Difficulty levels explanation
  - [ ] Observation/action space description
  - [ ] Reward function explanation
  - [ ] Example usage with client
  - [ ] Links to example agents
- [ ] Create `GAME_DESIGN.md`:
  - [ ] Detailed game mechanics
  - [ ] Task specifications per level
  - [ ] Grading criteria
  - [ ] Expected agent strategies
- [ ] Create `API.md`:
  - [ ] `QuestObservation` fields
  - [ ] `QuestAction` format
  - [ ] Reward breakdown
  - [ ] Error handling

### 8.2 Docker Configuration
- [ ] Verify `Dockerfile` works:
  - [ ] All dependencies in `requirements.txt`
  - [ ] Correct PYTHONPATH setup
  - [ ] Health checks functional
  - [ ] Web interface enabled (optional)
- [ ] Test Docker build locally:
  - [ ] `docker build -t agentquest-env:latest .`
  - [ ] `docker run -p 8000:8000 agentquest-env:latest`

### 8.3 Deployment
- [ ] Update `openenv.yaml` if needed
- [ ] Update `pyproject.toml`:
  - [ ] Add new dependencies
  - [ ] Update version number
  - [ ] Update description
- [ ] Prepare for Hugging Face Spaces:
  - [ ] Create `.dockerignore` (already exists)
  - [ ] Test `openenv push` command
  - [ ] Deploy to HF Spaces: `openenv push --repo-id pranayy/AgentQuest`

### 8.4 Create Example Notebooks
- [ ] Create Jupyter notebook showing:
  - [ ] Basic environment usage
  - [ ] Running full episodes
  - [ ] Analyzing rewards per difficulty
  - [ ] Example agent performance
  - [ ] Batch evaluation across difficulties
- [ ] Save to `/examples/agentquest_example.ipynb`

---

## Phase 9: Optional Enhancements

### 9.1 Advanced Grading
- [ ] Implement machine learning-based grader:
  - [ ] Train classifier on "good" vs "bad" strategies
  - [ ] Use for Level 4-5 grading
- [ ] Implement custom grading rules DSL
- [ ] Support multi-objective grading (Pareto frontiers)

### 9.2 Curriculum Learning Support
- [ ] Implement agent progress tracking
- [ ] Auto-progression on success
- [ ] Plateau detection (stuck on difficulty)
- [ ] Curriculum restart option

### 9.3 Web Interface Enhancements
- [ ] Add visualization:
  - [ ] Real-time game state display
  - [ ] Agent action visualization
  - [ ] Reward timeline chart
  - [ ] Difficulty progression tracker
- [ ] Add interactive player mode (human can play)
- [ ] Add leaderboard (compare agents across difficulties)

### 9.4 Evaluation Dashboard
- [ ] Create eval dashboard that shows:
  - [ ] Success rates per difficulty
  - [ ] Average rewards per level
  - [ ] Agent strategy analysis
  - [ ] Top agent solutions
  - [ ] Comparison charts

### 9.5 Randomization & Robustness
- [ ] Implement task variation:
  - [ ] Same difficulty, different task instances
  - [ ] Adversarial task generation
  - [ ] Stress-test agent generalization
- [ ] Add environment noise (optional):
  - [ ] Stochasticity in state transitions
  - [ ] Observation noise
  - [ ] Action failure probability

---

## Implementation Timeline Estimate

| Phase | Complexity | Est. Hours | Dependencies |
|-------|-----------|-----------|--------------|
| 1. Design | Medium | 4-6 | None |
| 2. Core Logic | High | 8-12 | Phase 1 |
| 3. Grading | Medium-High | 6-10 | Phase 2 |
| 4. Rewards | Medium | 4-8 | Phase 2 |
| 5. State Mgmt | Low-Medium | 3-5 | Phase 2 |
| 6. Client | Low | 2-3 | Phase 2 |
| 7. Testing | Medium | 6-10 | Phases 2-5 |
| 8. Documentation | Low-Medium | 3-5 | All phases |
| 9. Enhancements | Variable | 5-20+ | All phases |
| **Total (Core)** | | **32-59** | |

---

## Key Metrics to Track

- [ ] **Task Success Rate**: % of random agents that complete task at each difficulty
- [ ] **Reward Statistics**: Mean/std of rewards per difficulty level
- [ ] **Grading Accuracy**: % agreement between auto-grader and human review
- [ ] **Agent Generalization**: Performance variance across task instances
- [ ] **Evaluation Time**: Average ms per episode
- [ ] **Environment Stability**: Error rate during long runs

---

## Success Criteria

✅ Game is playable by RL agents  
✅ Automatic graders verify task completion with >95% accuracy  
✅ Reward function scales appropriately with difficulty (harder tasks = higher max reward)  
✅ 5 difficulty levels with clear progression in agent success rates  
✅ Full test coverage (>80% code coverage)  
✅ Docker image builds and runs locally  
✅ Example agent demonstrates playable solution  
✅ Comprehensive documentation with API reference  
✅ Ready for deployment to Hugging Face Spaces  

---

## Next Steps

1. **Review & refine game concept** (Phase 1)
2. **Get stakeholder approval** on game design
3. **Start implementation** with Phase 2 (Core Logic)
4. **Test incrementally** after each phase
5. **Deploy when Phase 8 complete**
