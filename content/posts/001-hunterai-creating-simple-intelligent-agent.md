---
title: "HunterAI #1 : Creating Simple Intelligent Agent"
author: "Adam Jordan"
date: 2019-09-16
description: "Today, I try to make an intelligent agent that can hunt monster. I name this agent: **Hunter**.
    In this devlog, I will write the understanding and design process of a very simple intelligent agent."
tags: [ai, devlog, hunter-ai]
draft: false
---


> **About HunterAI**

> This is a series of my project devlog to create _a certain complex_ AI.
> In each devlog, I will write about my progress in developing and upgrading the complexity of the AI
> program from the previous version.


## Today's Goal

In this article, I will try to make an intelligent agent that can hunt monster.
I name this agent: **Hunter**.
(_In the future, today's version will be referenced as **Alexander**_)


## Defining "Agent"

How can we build something if we do not what it is?

***What is Agent?***
Agent is anything that can be viewed as: **perceiving** environment through **sensors**; and **acting** upon environment through **actuators**.
It will run in **cycles** of perceiving, thinking, and acting.


***What is an Intelligent Agent?***
An agent that acts in a manner that causes it to be as _successful_ as it can.
This agent acts in a way that is expected to **maximise** to its **performance measure**, given the _evidence_ provided by what it perceived and whatever _built-in knowledge_ it has.
The **performance measure** defines the criterion of success for an agent.


## Designing an Intelligent Agent

Okay, we already understand what an intelligent agent is. Now, how do we start designing the agent?

When designing an intelligent agent, we can describe the agent by defining _(PEAS Descriptors)s_:

* **P**erformance -- how we measure the agent's achievements
* **E**nvironment -- what is the agent is interacting with
* **A**ctuator -- what produces the outputs of the agent
* **S**ensor -- what provides the inputs to the agent


Alternatively, we can desribe the agent by defining _(PAGE Descriptor)_:

* **P**ercepts -- the inputs to our system
* **A**ctions -- the outputs of our system
* **G**oals -- what the agent is expected to achieve
* **E**nvironment -- what the agent is interacting with


Formally, we can define structure of an Agent as:

$$Agent = Architecture + Program$$

**Architecture**: a device with sensors and actuators.

**Agent Program**: a program implementing *Agent Function* $f$ on top an architecture, where **Agent Function**: a function that maps a **sequence of perceptions** into **action**.

$$f(P) = A$$


## Agent Interaction Concept

I propose a concept idea of agent interaction as follow:

* Agent is represented as $\langle Program, Architecture \rangle$.

* Architecture is responsible for **perceiving** environment through **sensors** and **acting** upon environment through **actuators**.

* *Percept* perceived by architecture is then passed to Agent Program.
According to agent function, Agent Program then map the *percept* to an *action*.
*Action* is then passed to Architecture.

* Environment is a representation that will be perceived and acted upon by architecture.

* Action is represented as $\langle id \rangle$, where $id$ is used to identify the type of the action.

* Percept is represented as $\langle state \rangle$, where $state$ is a key-value map containing the information perceived by sensor.


Let's start by making a simple abstract code implementing the above concepts.

```python
# concepts.py

class Environment:
    pass


class Action:
    def __init__(self, id):
        self.id = id


class Percept:
    def __init__(self, state):
        self.state = state


class Architecture:
    def perceive(self, environment):
        raise NotImplementedError()

    def act(self, environment, action):
        raise NotImplementedError()


class Program:
    def process(self, percept):
        raise NotImplementedError()


class Agent:
    def __init__(self, program, architecture):
        self.program = program
        self.architecture = architecture

    def step(self, environment):
        percept = self.architecture.perceive(environment)
        action = self.program.process(percept)
        if action:
            self.architecture.act(environment, action)

```

## Designing Hunter AI

I will try to build **Hunter** as an intelligent agent that will shoot an arrow when it find a monster.

Let's describe **Hunter** with _PEAS Descriptor_:

* **P**erformance: Able to kill all monsters.
* **E**nvironment: A battefield filled with monsters.
* **A**ctuator: Hunter's bow
* **S**ensor: Hunter's vision


## Let's start simple

Let's try implementing our **Hunter** as a simple reflex agent.
Because this is our first attempt, let's simplify things:

* There is one monster in battlefield.
* We only care about the monster visibility. We don't care about its position.
* When a monster is visible, hunter will react by shooting it.
* When a monster got hit, it will disappear.

I know that this looks oversimplified and seems very easy.
But this is important as our building foundation.


First, let's define our environment.
According to our simplification, there is only one state attribute: `monster_visible`.

```python
# environment.py

from .concepts import Environment


class HunterEnvironment(Environment):
    def __init__(self):
        self.monster_visible = False
```


Next, let's define our Hunter's Agent program.
According to our simplification, when the agent perceives that a monster is visible, it will do shooting action.

```python
# program.py

from .concepts import Action, Program


class HunterProgram(Program):
    def process(self, percept):
        if percept.state['monster_visible'] is True:
            return Action('shoot')
        else:
            return None
```

Next, let's define our Architecture.
Perceiving is simple, it will check the `monster_visible` attribute of the environment.
If shooting action is acted upon this environment, the monster will disappear, thus setting `monster_visible` to `False`.

```python
# architecture.py

from .concepts import Architecture, Percept
import logging


class HunterArchitecture(Architecture):
    def perceive(self, environment):
        return Percept({'monster_visible': environment.monster_visible})

    def act(self, environment, action):
        if action.id == 'shoot':
            logging.debug('Pew! Shooting monster')
            environment.monster_visible = False
```

So far, we have already finished implementing our agent program and architecure.
And this is sufficient to say that we already finished implementing our agent.
Our agent can be instantiated with the following code:

```python
program = HunterProgram()
architecture = HunterArchitecture()
agent = Agent(program, architecture)
```


However, currently we don't have any way to run and test our agent.
Let's make a simulator to run this agent.

```python
# simulator.py

from .concepts import Agent
from .architecture import HunterArchitecture
from .program import HunterProgram
from .environment import HunterEnvironment
import logging
import os
import time


class Simulator:
    def __init__(self, environment, agents):
        self.environment = environment
        self.agents = agents
        self.time = 0

    def step(self):
        for agent in self.agents:
            agent.step(self.environment)
        self.time += 1

    def debug(self):
        logging.debug('monster_visible is %s', self.environment.monster_visible)

    @staticmethod
    def instantiate():
        environment = HunterEnvironment()

        program = HunterProgram()
        architecture = HunterArchitecture()
        agent = Agent(program, architecture)

        return Simulator(environment, [agent])
```

Our simulator class represents a _world_ where our agent is running.
As you may realize, We have not yet defined what a _world_ is.
Let's define **world** as a triplet $\langle time, environment, agents \rangle$, where $time$ is an increasing number that represents a moment in the world.
A *world* contains an $environment$ and a list of agent ($agents$) that will interact with the environment.

A moment in the world can be moved forward by calling `step()` function.
When stepping, the agents will start processing the environment, perceiving and acting upon it; and the time will increase.

## Running our agent with simulator

Now that we finished programming our intelligent agent and simulator, let's run and play around with them.

Spawn a python interpreter, and import our `simulator.py` programs.

```python
$ python3

>>> from simulator import *

>>> logging.basicConfig(level='DEBUG')

>>> simulator = Simulator.instantiate()

>>> simulator.debug()
DEBUG:root:[time:0] monster_visible is False
```

We step one time unit period in our world simulator by calling `simulator.step()`.
```python
>>> simulator.step()

>>> debug(world)
DEBUG:root:[time:1] monster_visible is False
```

Let's try updating the environment.
We want to see if our agent react accordingly when a monster appears.

```python
>>> debug(world)
DEBUG:root:[time:1] monster_visible is False

>>> simulator.environment.monster_visible = True

>>> debug(world)
DEBUG:root:[time:1] monster_visible is True
```

After running step, we will see that our **Hunter** should shoot the monster.

```python
>>> debug(world)
DEBUG:root:[time:1] monster_visible is False

>>> simulator.step()
DEBUG:root:Pew! Shooting monster

>>> debug(world)
DEBUG:root:[time:2] monster_visible is False
```


## Conclusion

Today, we learned about Intelligent agent and we have built a very simple intelligent agent.
In the future, we will try to extend this agent to have more advanced features.
I will also try to create a visualization for this agent.
While this seems very simple, this is my first step towards creating _a certain complex_ AI program.


## Source Code

https://github.com/adamyordan/hunterAI/tree/master/alexander


## References

* https://hackernoon.com/rational-agents-for-artificial-intelligence-caf94af2cec5
* https://www.geeksforgeeks.org/agents-artificial-intelligence/
* http://www.cs.bham.ac.uk/~jxb/IAI/w4.pdf
