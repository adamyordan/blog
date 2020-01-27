---
title: "HunterAI #3: Top-down Shooter AI"
author: "Adam Jordan"
date: 2020-01-27
description: "In this post, I will try to create a **reflex agent with internal state** in an imperfect information environment.
The agent is a top-down shooter with a limited vision range, therefore it needs to store information about the map it already visited.
Furthermore, it needs to make a rational decision to find the enemies and make necessary actions shoot them.
"
tags: [ai, devlog, hunter-ai, unity]
draft: false
---


> **About HunterAI**

> This is a series of my project devlog to create _a certain complex_ AI.
> In each devlog, I will write about my progress in developing and upgrading the complexity of the AI
> program from the previous version.


## Today's Goal

In this post, I will try to create a **reflex agent with internal state** in an imperfect information environment.
The agent is a top-down shooter with a limited vision range, therefore it needs to store information about the map it already visited.
Furthermore, it needs to make a rational decision to find enemies and take necessary actions to shoot them.

_For future reference, today's version will be referenced as **Caleb**._

![Visualization](/post_assets/005/preview.png)


## The Scenario

We are going to expand from the [previous scenario]({{< ref "002-hunterai-connecting-agent-with-unity" >}}).
In today's scenario, we are going to put our hunter in a grid of `M x N`.

 - There will be some monsters placed at random coordinates.
 - Hunter's vision range is limited. Thus making hunter to not have perfect information.
 - Projectile range is limited. Thus, hunter needs to position itself accordingly to shoot monsters.
 - In a moment, the action that hunter can take is: $MoveForward$, $RotateLeft$, $RotateRight$, or $Shoot$.


## Demo

{{< youtube XRnqPfFGsps >}}


## Finding Shortest Path with BFS

In this scenario, we are going to face a problem where we need to find the shortest path to multiple cells.
For example, there are multiple monsters in the grid, and hunter needs to approach the nearest monster.

To solve this problem, we can use Breadth-first-Search (BFS);
with Hunter's coordinate as starting point and monsters coordinates as goals.
With BFS, since we are going to do level order traversal, the first goal that we reach is the nearest goal.
We can then follow the path formed, which is the shortest path to the nearest goal.


{{< figure src="/post_assets/005/drawing-bfs.png" alt="BFS to find nearest target" position="center" style="max-height: 300px;" caption="BFS to find nearest target" captionPosition="center" captionStyle="" >}}


```python
# caleb/algorithm/bfs.py

def neighbors(current, grid):
    size_x = len(grid[0])
    size_y = len(grid)

    res = []
    if current[0] - 1 >= 0:
        res.append((current[0] - 1, current[1]))
    if current[0] + 1 < size_x:
        res.append((current[0] + 1, current[1]))
    if current[1] - 1 >= 0:
        res.append((current[0], current[1] - 1))
    if current[1] + 1 < size_y:
        res.append((current[0], current[1] + 1))
    return res


def bfs(start, goals, grid):
    """
    Using BFS to get path to nearest goal.
    """
    size_x = len(grid[0])
    size_y = len(grid)

    visited = [[False for _ in range(size_x)] for _ in range(size_y)]
    parent = [[None for _ in range(size_x)] for _ in range(size_y)]
    queue = [start]
    visited[start[1]][start[0]] = True

    while queue:
        current = queue.pop(0)
        if current in goals:
            path = []
            while parent[current[1]][current[0]]:
                path.append(current)
                current = parent[current[1]][current[0]]
            return path[::-1]
        for neighbor in neighbors(current, grid):
            if not visited[neighbor[1]][neighbor[0]]:
                queue.append(neighbor)
                parent[neighbor[1]][neighbor[0]] = current
                visited[neighbor[1]][neighbor[0]] = True

    raise ValueError('No Path Found')
```


Let's create a test script to check the correctness of our implementation.

```python
# caleb/test.py

def test_bfs():
    grid_str = [
        '-M------',
        '------M-',
        '--M-----',
        '--------',
        '--------',
        '--------',
        '----H---',
        '--------',
    ]

    print('\nTesting: BFS to nearest monster\n')

    grid, hunter, monsters = parse_grid(grid_str)
    debug_grid(grid)

    while True:
        if hunter in monsters:
            break
        path = bfs.bfs(hunter, monsters, grid)
        next_hunter = path[0]
        set_content(grid, hunter, GridContent.EMPTY)
        set_content(grid, next_hunter, GridContent.HUNTER)
        hunter = next_hunter
        debug_grid(grid, refresh=False)


if __name__ == '__main__':
    test_bfs()
```

By running the test script, we can see that the hunter is going toward the monster on the left bottom, which is the nearest monster.

{{< vimeox 387264527 >}}

## The Agent Implementation in Python

I will reuse the concepts explained from previous posts:

* [Creating simple Intelligent Agent & the fundamental concepts]({{< ref "001-hunterai-creating-simple-intelligent-agent" >}})
* [Connecting agent program with Unity frontend visualization]({{< ref "002-hunterai-connecting-agent-with-unity" >}})


### Generalized Remote Architecture and Remote Environment Framework

Based on the idea in [previous version]({{< ref "002-hunterai-connecting-agent-with-unity" >}}),
let's make it more generalized for remote architecture and remote environment.
So in the future we might be able to reuse the general piece of code.

For remote architecture, there are two functions that we must implement in an `Architecture` class.

* In `perceive` function, we just return a Percept instance containing the environment state.
* In `act` function, we just pass the action to remote environment through `set_remote_action`.

    ```python
    # caleb/architecture.py

    from alexander.concepts import Percept, Architecture


    class GeneralRemoteArchitecture(Architecture):
        def perceive(self, environment):
            return Percept(environment.state)

        def act(self, environment, action):
            environment.set_remote_action(action)
    ```

For remote environment:

* We store a state in a Key-Value data structure.
* We provide a function to update this state data: `update_state`.

    ```python
    # caleb/environment.py

    from barton.concepts import RemoteEnvironment


    class GeneralRemoteEnvironment(RemoteEnvironment):
        def __init__(self):
            super().__init__()
            self.state = {}

        def update_state(self, state):
            self.state = state
    ```


### Implementing the Hunter Agent Program

Next, we are going to write the Hunter agent program.
First, let's define the actions that the hunter agent can possibly do.
Also define the possible content of a cell in the grid.

```python
# caleb/program.py (partial)

class Actions(Enum):
    MOVE_FORWARD = Action('move_forward')
    SHOOT = Action('shoot')
    ROTATE_LEFT = Action('rotate_left')
    ROTATE_RIGHT = Action('rotate_right')

class GridContent(Enum):
    UNKNOWN = -1
    EMPTY = 0
    MONSTER = 1
    HUNTER = 2
```

Our agent is a **reflex agent with internal state**.
The state information is initiated at `__init__()`.
The `process()`, which implements the agent function, will perform three tasks when invoked:

 - Update the agent's state to reflect new percept;
 - Choose the best action;
 - Update the agent's state according to action.

The code:

```python
# caleb/program.py (partial)

class HunterProgram(Program):
    def __init__(self):
        pass # todo: initiate default state here

    def process(self, percept):
        self.update_state_with_percept(percept)
        action = self.choose_action()
        self.update_state_with_action(action)
        return action

    def update_state_with_percept(self, percept):
        pass # todo

    def choose_action(self):
        return None # todo

    def update_state_with_action(self, action):
        pass # todo
```

Then, we are going to write the main file.
In main, we define our `init_function` that will instantiate the hunter environment and agent.
Then we pass the it to Remote module (Read [previous post]({{< ref "002-hunterai-connecting-agent-with-unity" >}})), and run the python remote server.


```python
from alexander.concepts import Agent
from barton.remote import Remote
from .architecture import GeneralRemoteArchitecture
from .environment import GeneralRemoteEnvironment
from .program import HunterProgram


def init_function():
    environment = GeneralRemoteEnvironment()
    program = HunterProgram()
    architecture = GeneralRemoteArchitecture()
    agent = Agent(program, architecture)
    return environment, [agent]


if __name__ == '__main__':
    remote = Remote(init_function)
    remote.app().run(debug=True)
```


We can now run the hunter agent by running `main.py`.

```bash
$ python3 -m caleb.main
 * Serving Flask app "barton.remote" (lazy loading)
 * Environment: production
 * Debug mode: on
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 ```


### The CLI World Environment

Before building the world environment with Unity, we can first build an environment with CLI.

The `cli_world.py` contains an implementation of hunter world environment.
When running, it will communicate with the Hunter agent via HTTP connection and update itself in an eternal loop.
In each loop step, it will update itself, send the environment state to agent, and actuate the action received from agent to the environment.

```python
# caleb/test/cli_world.py

class CliWorld:

    def __init__(self, grid_str, hunter_face):
        self.grid, self.hunter, self.monsters = parse_grid(grid_str)
        self.hunter_face = hunter_face

    def run(self):
        self.debug(sleep=1)
        self.init()
        while True:
            self.update()
            state = self.get_state()
            step_response = self.step(state)
            self.actuate(step_response['action'])
            self.debug(refresh=False, sleep=1)

    def init(self):
        requests.put('http://localhost:5000/init')

    def step(self, state):
        r = requests.put('http://localhost:5000/step', data=json.dumps(state))
        return r.json()

    ...


if __name__ == '__main__':
    grid_str = [
        '-M-------',
        '------M--',
        '--M------',
        '---------',
        '---------',
        '---------',
        '----H----',
        '---------',
        '---------',
    ]

    print('\nTesting: Hunter Agent\n')
    world = CliWorld(grid_str, hunter_face=0)
    world.run()
```

The State passed from this CLI World is as follows:

```python
state = {
    'Vision': visionContent,
    'MonsterCount': len(self.monsters),
    'MapSize': {'x': map_size_x, 'y': map_size_y},
    'HunterPosition': self.hunter,
    'HunterRotation': self.hunter_face,
    'HunterProjectileDistance': 3
}
```

We can run the script and this CLI World will connect to our Hunter Agent program through HTTP connection.
However, there is no functionality as of now.

```bash
$ python3 -m caleb.test.cli_world

Testing: Hunter Agent

  - M - - - - - - -
  - - - - - - M - -
  - - M - - - - - -
  - - - - - - - - -
  - - - - - - - - -
  - - - - - - - - -
  - - - - H - - - -
  - - - - - - - - -
  - - - - - - - - -
```


## Implementing the Agent Logic

### Scanning Grid with Limited Vision

We are going to implement the algorithm of how our Hunter Agent can explore the grid.
Because our agent has limited vision, it does not have perfect information about the contents of every cells.
Thus, it need to store the content of the cells already seen.
We store it in `self.grid` variable.

In `update_state_with_percept`, we update the internal state with the percept received from environment.
We also update the `grid` variable with the `Vision` percept.

```python
def update_state_with_percept(self, percept):
    if not self.initiated:
        self.grid_size = percept.state['MapSize']
        self.grid = [[GridContent.UNKNOWN for _ in range(self.grid_size['x'])] for _ in range(self.grid_size['y'])]
        self.initiated = True

    self.hunter_position = percept.state['HunterPosition']
    self.hunter_rotation = percept.state['HunterRotation']
    self.monster_count = percept.state['MonsterCount']
    self.hunter_projectile_range = percept.state['HunterProjectileDistance']
    for vision in percept.state['Vision']:
        x, y = vision['Position']
        self.grid[y][x] = GridContent(vision['Content'])
```

Then, in `choose_action`, we will return an action to move towards the nearest unknown cell.
Using the BFS algorithm we discussed earlier, we can know the shortest path to the nearest unknown cell.
Then, the hunter is heading to the first step in the shortest path, while considering rotation action if necessary.


```python
def get_unknown_cells(self):
    unknown_cells = []
    for y, row in enumerate(self.grid):
        for x, col in enumerate(row):
            if col == GridContent.UNKNOWN:
                unknown_cells.append((x, y))
    return unknown_cells

def action_move_towards(self, cells):
    path_to_target = bfs.bfs(self.hunter_position, cells, self.grid)
    if path_to_target:
        next_cell = path_to_target[0]
        rotate_action = self.calc_rotate_action(next_cell) # check if rotation is necessary
        return rotate_action or Actions.MOVE_FORWARD.value

def choose_action(self):
    if self.monster_count > 0:
        unknown_cells = self.get_unknown_cells()
        return self.action_move_towards(unknown_cells)
```

As of now, our Hunter Agent can already do basic grid scanning function.
With this function, the agent can gain more information inside the imperfect information environment.
 
{{< vimeox 387417991 >}}


### Shooting Monsters

Now we are going to implement the functionality for our Hunter agent to shoot monsters.

The idea is as follows:

- List the coordinates of monsters already sighted.
- If no monsters are sighted yet, move towards the nearest unknown cell.
- Else, going to shoot the nearest monster:
    - Calculate the possible shooting position cells.
      The shooting position is a coordinate in which the Hunter can shoot a monster in a direction.
    - If hunter already in shooting position, shoot the target. Do rotation if necessary.
    - If hunter is not yet in shooting position, use BFS to move towards the nearest shooting position.


The related code:

```python
# caleb/program.py

def get_monster_cells(self):
    monsters_sighted = []
    for y, row in enumerate(self.grid):
        for x, col in enumerate(row):
            if col == GridContent.MONSTER:
                monsters_sighted.append((x, y))
    return monsters_sighted

def get_shooting_position_cells(self, monsters_sighted):
    target_cells = []
    target_cells_monster = {}
    for x, y in monsters_sighted:
        for i in range(1, self.hunter_projectile_range + 1):
            target_cells_curr = [(x + i, y), (x - i, y), (x, y + i), (x, y - i)]
            for target_cell in target_cells_curr:
                target_cells.append(target_cell)
                key = '%d,%d' % (target_cell[0], target_cell[1])
                target_cells_monster[key] = (x, y)
    return target_cells, target_cells_monster

def action_shoot_monster(self, target_cells_monster):
    key = '%d,%d' % (self.hunter_position[0], self.hunter_position[1])
    monster_cell = target_cells_monster[key]
    rotate_action = self.calc_rotate_action(monster_cell)
    return rotate_action or Actions.SHOOT.value

def choose_action(self):
    if self.monster_count > 0:
        unknown_cells = self.get_unknown_cells()
        monsters_sighted = self.get_monster_cells()

        if len(monsters_sighted) == 0:
            return self.action_move_towards(unknown_cells)
        else:
            target_cells, target_cells_monster = self.get_shooting_position_cells(monsters_sighted)
            if self.hunter_position in target_cells:
                return self.action_shoot_monster(target_cells_monster)
            else:
                return self.action_move_towards(target_cells)
```

As of now, our agent is already fully working.
If we connect our agent with the previous `cli_world` environment, we can see that our agent is working as expected.

{{< vimeox 387427033 >}}


## The Unity World Environment

Now that we already have a fully working agent in CLI environment,
let's make a 3D World environment visualized using Unity.
Please take a loot at "[Connecting agent program with Unity frontend visualization]({{< ref "002-hunterai-connecting-agent-with-unity" >}})" to understand the mechanic to connect Unity visualization with our python agent program.

![The Unity Editor](/post_assets/005/unity.png)

As shown in the image above, I already made a Unity scene containing a grid of `M x N` cells.
The next thing interesting to discuss is the `simulation.cs` which contains the implementation of our simulation program logic.

- In `Initiate()`, we are going to initiate the connection with our agent program via `HunterSDK.Initiate()`.
  Then, we will invoke `Step()` to be run every `SdkStepInterval` seconds (default: 0.1).

- In `Step()`:
    - Get state from Unity environment,
    - Send the state via `HunterSDK.Step()`
    - Receive the action from the agent response, then actuate the action to Unity environment.

The code:

```csharp
// simulator.cs

public class Simulator : MonoBehaviour
{
    public void Initiate()
    {
        ...
        if (SdkAiEnabled)
        {
            hunterSDK = new HunterSDK<State, StepResponse>(this, SdkHost);
            hunterSDK.Initiate();
            InvokeRepeating("Step", 2.0f, SdkStepInterval);
        }
        ...
    }

    void Step()
    {
        if (SdkAiEnabled)
        {
            TheHunter.ShowThinking(true);
            State state = GetState();
            hunterSDK.Step(state, (stepResponse) =>
            {
                AgentAction action = stepResponse.action;
                ActuateAction(action);
                Helper.RunLater(this, () => TheHunter.ShowThinking(false), 0.1f);
            });
        }
    }

    public State GetState()
    {
        List<GridWithContent> vision = getVision();

        State state = new State();
        state.Vision = vision;
        state.MonsterCount = monsterObjs.Count;
        state.MapSize = TheGridManager.mapSize;
        state.HunterPosition = TheHunter.targetPosition;
        state.HunterRotation = Mathf.RoundToInt(TheHunter.targetRotation.eulerAngles.y);
        state.HunterProjectileDistance = HunterProjectileDistance;

        return state;
    }

    ...
}
```

I also created a simulation option panel that allows user to configure the simulation option.
For example, user can configure the grid size, the simulation step speed, or the hunter vision range.
This allows us to experiment further, such as knowing the optimal vision range (in case having long vision range costs something else).

{{< figure src="/post_assets/005/option-panel.png" alt="Hello Friend" position="center" style="max-height: 400px;" caption="Simulation Option Panel" captionPosition="center" captionStyle="" >}}


### The Challenges with Unity: Different Position Indexing

In Unity 3D, there are three axis in coordinates, i.e. `x`, `y`, `z`, which consecutively represents horizontal position, vertical position, and depth position.
Because we are interested in 2D grid, we can ignore `y` coordinate, thus focusing in `x` for horizontal axis in grid, and `y` for vertical axis in grid.

But there is another problem related in axis positional value.
In our python agent program, the grid is represented vertically in _X_ axis from index `0` to `M`, and horizontally in _Y_ axis from index `0` to `N`.
Meanwhile, in Unity, the position grid is represented vertically in _X_ axis from index `-M/2` to `M/2`, and horizontally in _Y_ axis from index `N/2` to `-N/2`.
This inconsistency proves to become a problematic hassle during implementation, making implementation prone to indexing errors.

{{< figure src="/post_assets/005/drawing-indexing.png" alt="" position="center" style="max-height: 300px;" caption="Different indexing in Unity and our python program" captionPosition="center" captionStyle="" >}}

One way to solve this problem is by adding a position translation in python side:

```python
def translate_pos(self, x, y):
    return self.grid_size['x'] // 2 + x, self.grid_size['y'] // 2 + y
```

## Demo & Conclusion

{{< youtube XRnqPfFGsps >}}

Today, we learned how to create a **reflex agent with internal state** in an imperfect information environment.
Specifically, we created a top-down shooter AI with limited vision range to its surrounding.
Using common pathfinding algorithm (in this case, BFS), our AI agent is able to determine an efficient way to discover monsters and position itself in the nearest shooting position.

Combined with the configurability of our visualization, I believe this is a good learning material in learning how to implement and playing around basic AI.

## Source code

https://github.com/adamyordan/hunterAI/tree/master/caleb


## Reference

* https://people.eecs.berkeley.edu/~russell/aima1e/chapter02.pdf
  (Reference when implementing reflex agent function)
