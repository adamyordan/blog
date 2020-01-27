---
title: "HunterAI #2: Connecting AI Agent with Unity"
author: "Adam Jordan"
date: 2019-09-19
description: "Today, I will try to create a 3d visualization using Unity and connect it to our agent written in python. In this devlog, I will write the remote communication module and also the updated AI concepts to support remote functionality."
tags: [ai, devlog, hunter-ai, unity]
draft: false
---


> **About HunterAI**

> This is a series of my project devlog to create _a certain complex_ AI.
> In each devlog, I will write about my progress in developing and upgrading the complexity of the AI
> program from the previous version.


## Today's Goal

In this post, I will try to create a 3D visualization using Unity and connect it to our agent written in python.
You may want to read the [previous post]({{< ref "001-hunterai-creating-simple-intelligent-agent" >}}) about **Alexander** AI concept because we will reuse a lot of the concepts.
(_In the future, today's version will be referenced as **Barton**_).


![Visualization](/post_assets/002/visualization.png)

## Idea


I will still continue using the concepts introduced in [previous post]({{< ref "001-hunterai-creating-simple-intelligent-agent" >}}).
However, there is an update that should be done to accommodate remote functionality.

Updated remote agent concepts in python side:

* **Remote environment**: remote environment is similar with common environment, a representation that will be perceived and acted upon by architecture.
  The differences are:
  * The architecture is not allowed to modify environment state directly.
    Action from architecture will be stored at `remote_action`.
  * Environment state is updated at every moment / step.
    This constant update is required because the Remote environment is actually a representation of the corresponding environment objects in the Unity side.
    
* **Remote architecture**: implementation-wise there is no difference with common architecture.
  However, there is an important concept that must be followed by remote architecture during actuating actions.
  Remote architecture should never modify environment state directly, instead it should use `set_remote_action(action)` function to pass the action (`remote_action`) to the Unity side.
  The action will be then realized by the corresponding agent object in the Unity side.


Next, We have to define a **remote communication module** that allows communication between the python and Unity side.
One of the simplest approach is by serving a **python HTTP server**, and let Unity side send requests and receive responds from the python side.

![Visualization](/post_assets/002/diagram.png)

The HTTP server behavior is defined as follows:

* Endpoint `/init` will initiate the time, environment and agents.
  The initiation procedure is passed as `init_function` parameter when constructing `Remote` module.

* Endpoint `/step` will trigger the *world* step, moving the time forward.
  This endpoint accepts environment state in JSON format.
  These actions will be executed:
  * Retrieve the environment state from Unity (from the HTTP body), then update the the python representation of remote environment will update its state accordingly to the passed state.
  * The agents will start processing the environment, perceiving the environment and producing `remote_action`.
  * The `remote_action` will be returned as HTTP Response. The Unity side will read this action and actuate it accordingly.
  * Increase time.


## The Unity

![Unity Editor](/post_assets/002/unity-editor.png)

For illustration, I show the completed today's Unity project above.
Quick explanation of what is happening in the Unity scene:

* The blue cube represents the `Hunter` (player) object.
* The red cube represents the `Monster` object.
* The blue cube can spawn `Projectile` object that will hit the red cube out, making it fall off the ground (then destroyed).

The main problem here is how to make `Hunter` object communicate with our Hunter agent in python?
As discussed in the *Idea* section, we are going to make the Unity communicate with our python with an HTTP client.

First, put an empty object (`Simulator`) into our Unity scene.
Then, add `HunterSDK` and `SimulatorScript` as components to `Simulator` object.

The `HunterSDK` is essentially the code to communicate with our server protocol.
It provides the `Initiate()` and `Step()` method.
The `initiate()` method will send a request to `/init` endpoint, initiating agents and environment representation in python side.
The `Step()` method accepts the current environment state in Unity, then send it in JSON format alongside a request to `/step` endpoint, and then pass the respond to the callback function.
The respond should be containing the action to be actuated.


```csharp
// HunterSDK.cs (partial)

public class HunterSDK : MonoBehaviour
{
    public string Host = "http://127.0.0.1:5000/";

    public void Initiate()
    {
        StartCoroutine(PutRequest(Host + "init"));
    }

    public void Step(State state, Action<StepResponse> callback)
    {
        string stateJson = JsonUtility.ToJson(state);
        StartCoroutine(PutRequest(Host + "step", stateJson, (res) =>
        {
            StepResponse stepResponse = JsonUtility.FromJson<StepResponse>(res);
            callback(stepResponse);
        }));
    }

    IEnumerator PutRequest(string uri, string data = "{}", Action<string> callback = null)
    {
        using (UnityWebRequest webRequest = UnityWebRequest.Put(uri, data))
        {
            webRequest.SetRequestHeader("Content-Type", "application/json");
            yield return webRequest.SendWebRequest();
            callback?.Invoke(webRequest.downloadHandler.text);
        }
    }
}
```

Next, the `SimulatorScript` is a behavioural code that should define the behaviour of our agents and environment in Unity side.
Initially in `Start()`, it will run `HunterSDK.Initiate()`.
Then, after a fixed interval, it will periodically call the `SimulatorScript.Step()` funtion.
In this step function, you need to determine the current environment state with Unity functionality (e.g. Calculate the visibility of monster), then pass it to `HunterSDK.Step(state)`.
Finally, you also need to define what should be done to actuate action accordingly (e.g. Spawn a projectile).


```csharp
// SimulatorScript.cs

public class SimulatorScript : MonoBehaviour
{
    public float stepInterval;
    HunterSDK hunterSDK;

    void Start()
    {
		hunterSDK = gameObject.GetComponent<HunterSDK>();
        hunterSDK.Initiate();
        InvokeRepeating("Step", stepInterval, stepInterval);
    }

    void Step()
    {
        State state = new State();

        // todo: Sensor code block; fill in state values

        hunterSDK.Step(state, (stepResponse) =>
        {
            AgentAction agentAction = stepResponse.action;

            // todo: Actuator code block; actuate action accordingly

        });
    }
}
```


## Python Code

### Remote Agent Concept

As discussed above, let's define the updated concepts in `barton/concept.py`.
Note that we will still be reusing the old concepts from `alexander/concept.py`.

```python
# barton/concepts.py

from alexander.concepts import Architecture, Environment


class RemoteArchitecture(Architecture):
    def perceive(self, environment):
        raise NotImplementedError()

    def act(self, environment, action):
        raise NotImplementedError()


class RemoteEnvironment(Environment):
    def __init__(self):
        self.remote_action = None

    def update(self, state):
        self.remote_action = None
        self.update_state(state)

    def set_remote_action(self, action):
        self.remote_action = action

    def get_remote_action(self):
        return self.remote_action

    def update_state(self, state):
        raise NotImplementedError()
```

### Remote Communication Module

As discussed above, we will be serving a python HTTP server, and let Unity side send requests and receive responds from the python side.
I use `flask` for its simplicity for running HTTP server.
The HTTP server behavior is defined as follows:

* Endpoint `/init` will initiate the time, environment and agents.
  The initiation procedure is passed as `init_function` parameter when constructing `Remote` module.

* Endpoint `/step` will trigger the *world* step, moving the time forward.
  This endpoint accepts environment state in JSON format.
  These actions will be executed:
  * Retrieve the environment state from Unity (from the HTTP body), then update the the python representation of remote environment will update its state accordingly to the passed state.
  * The agents will start processing the environment, perceiving the environment and producing `remote_action`.
  * `remote_action` will be returned as HTTP Response. The Unity side will read this action and actuate its accordingly.
  * Increase time.


The implementation is as follows:

```python
# barton/remote.py

from flask import Flask, jsonify, request
from flask_cors import CORS


class Remote:
    class Controller:
        @staticmethod
        def ping():
            return 'pong'

        @staticmethod
        def init(remote):
            remote.init()
            return jsonify({'ok': True})

        @staticmethod
        def step(remote):
            state = request.get_json()
            response = remote.step(state)
            return jsonify(response)

    def __init__(self, init_function):
        self.init_function = init_function
        self.agents = None
        self.environment = None
        self.time = 0

    def init(self):
        self.environment, self.agents = self.init_function()
        self.time = 0

    def step(self, state):
        self.environment.update(state)
        for agent in self.agents:
            agent.step(self.environment)
        action = self.environment.get_remote_action()
        action_serializable = action.__dict__ if action is not None else None
        self.time += 1
        return {'action': action_serializable}

    def app(self):
        app = Flask(__name__)
        CORS(app)
        app.add_url_rule('/', 'ping', lambda: Remote.Controller.ping())
        app.add_url_rule('/init', 'init', lambda: Remote.Controller.init(self), methods=['PUT'])
        app.add_url_rule('/step', 'step', lambda: Remote.Controller.step(self), methods=['PUT'])
        return app
```

## Writing Hunter Agent with the Remote Function in Python

First, let's define our hunter architecture using the updated remote architecture concept.
If you notice, the logic is still similar with the previous `alexander/architecture.py`.
The difference is that instead of changing state `environment.monster_visible` directly, we will delegate the action to the Unity side through `remote_action`.

```python
# barton/architecture.py

from alexander.concepts import Architecture, Percept
import logging


class HunterRemoteArchitecture(Architecture):
    def perceive(self, environment):
        return Percept({'monster_visible': environment.monster_visible})

    def act(self, environment, action):
        if action.id == 'shoot':
            logging.debug('Pew! Shooting monster')
        environment.set_remote_action(action)
```

Next, let's define our hunter environment using the updated remote environment concept.

```python
# barton/environment.py

from .concepts import RemoteEnvironment


class HunterRemoteEnvironment(RemoteEnvironment):
    def __init__(self):
        super().__init__()
        self.monster_visible = False

    def update_state(self, state):
        if 'monster_visible' in state:
            self.monster_visible = state['monster_visible']
```

Finally, let's write our `main` file.
In main, we will define our `init_function` that will instantiate the hunter environment and agent.
Then we will pass the it to `Remote` module, and run the python remote server.

```python
# barton/main.py

from .remote import Remote
from .architecture import HunterRemoteArchitecture
from .environment import HunterRemoteEnvironment
from alexander.program import HunterProgram
from alexander.concepts import Agent

if __name__ == '__main__':
    def init_function():
        environment = HunterRemoteEnvironment()
        program = HunterProgram()
        architecture = HunterRemoteArchitecture()
        agent = Agent(program, architecture)
        return environment, [agent]

    remote = Remote(init_function)
    remote.app().run()
```

We can start the python remove server by running the following shell command:

```bash
$ python3 -m barton.main 
```

## Hunter Agent in Unity Side

Remember that we need to write the code to fill in environment state values and actuating agent action in Unity scene.

According to our current Hunter scenario, there is only one state information that we need to attain: monster visibility (`monster_visible`). Let's just implement that `monster_visible` is true if the monster object exists and the distance between player object and the monster object is less than 4 unit.

Next, in actuator code block, we check that if the action id is `"shoot"`, then we call the method `playerScript.Shoot()`. This method is essentially spawn a projectile and add a big force toward the monster object direction. By the physics of Unity engine, the monster object will be pushed backward by the projectile and it will fall off the ground.

I also add a method `SetAIEnabled()` to toggle the activation of the AI.
This method is bound to the Toggle UI "Enable AI".
There is also a `SpawnMonster()` method that will be executed when clicking the button "Spawn Monster".

The complete Unity Simulator script is as follows:

```csharp
// SimulatorScript.cs

public class SimulatorScript : MonoBehaviour
{

    public GameObject monster;
    public GameObject player;
    public float stepInterval;

    bool aiEnabled = true;
    HunterSDK hunterSDK;
    PlayerScript playerScript;


    void Start()
    {
		hunterSDK = gameObject.GetComponent<HunterSDK>();
        playerScript = player.GetComponent<PlayerScript>();
        hunterSDK.Initiate();
        InvokeRepeating("Step", stepInterval, stepInterval);
    }

    void Step()
    {
        if (aiEnabled)
        {
            playerScript.ShowThinking(true);
            GameObject monsterGameObject = GameObject.FindGameObjectWithTag("Monster");

            State state = new State();
            state.monster_visible = monsterGameObject != null
                && Vector3.Distance(player.transform.position, monsterGameObject.transform.position) <= 4.0f;

            hunterSDK.Step(state, (stepResponse) =>
            {
                AgentAction agentAction = stepResponse.action;
                if (agentAction.id == "shoot")
                {
                    playerScript.Shoot();
                }
                Helper.RunLater(this, () => playerScript.ShowThinking(false), 0.1f);
            });
        }
    }

    public void SpawnMonster()
    {
        GameObject monsterGameObject = GameObject.FindGameObjectWithTag("Monster");
        if (monsterGameObject == null)
        {
            Instantiate(monster, new Vector3(2, 6, 0), transform.rotation);
        }
    }

    public void SetAIEnabled(bool isEnabled)
    {
        aiEnabled = isEnabled;
    }
}
```

## Demo & Conclusion

{{< youtube id="P3zoo9JVg3w" autoplay="true" >}}

Today, we learned how to connect an Intelligent agent with a Unity scene.
The Unity scene acts as a visualization or a frontend for our agent.
In the future, I will keep using visualization in order to better showcase the progress of our agent.
And the agent feels more like a real robot, right?

## Source Code

https://github.com/adamyordan/hunterAI/tree/master/barton
