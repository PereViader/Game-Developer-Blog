---
layout: post
title: "Clean data driven systems and content"
date: 2021-04-04 17:40:00 +1
---

The following things must be taken into account when reading the post:
- The examples are in [C#](https://en.wikipedia.org/wiki/C_Sharp_(programming_language)) but won't have any [game engine](https://en.wikipedia.org/wiki/Game_engine) specific code
- The examples will be centered around the data and how the systems ingest. Actual system implementation is omitted or simplified to ease the reading.
- [Strategy pattern is known](https://en.wikipedia.org/wiki/Strategy_pattern)
- [The concept of refactoring is known](https://en.wikipedia.org/wiki/Code_refactoring)
- [Further clean code cleaning and other techniques could be applied to the examples](http://principles-wiki.net/resources:clean_code)

For the interests of this post, we are going to generalize and say a game is a piece of software that can be split in two parts: systems and data (content). The systems are pieces of software that given some data will behave in some way or another. For example, a text dialogue system might be given a piece of text it needs to play or an NPC might be given an AI behaviour to execute. Whatever the complexity and concrete use case details these two parts work together to provide the experience desired to the player.


# Use Cases

Lets tackle the same use case of a system and its data, working together to create an experience, growing in complexity one at a time.


## Single

- There is an NPC
- The NPC has a locomotion system
- The locomotion system can only move the NPC in a single way: following some waypoints
- We have a piece of data with the waypoints.
- The data is provided to the locomotion system for it to move

We only have one system and it can only do one thing. In order to do this thing, the system is given the data it needs to do it. 

```
public class LocomotionSystemData
{
    public IReadOnlyList<Position> Positions { get; }
}


public class LocomotionSystem
{
    private LocomotionSystemData locomotionSystemData;

    public void Move(LocomotionSystemData locomotionSystemData)
    {
        this.locomotionSystemData = locomotionSystemData;
    }

    //Method called once per frame
    public void Tick()
    {
        //Movement code using data here...
    }
}
```

On this piece of code we can observe:
- There is a class that holds the data given to the system
- There is a class that implements the system
- The class that holds the data is given to the system so it can use it


## Multiple static

- There is an NPC
- The NPC has a locomotion system
- There are many locomotion system implementations: following some waypoints and following a target at a distance
- We have multiple pieces of data each one with the data for each implementation
- The data is provided to the locomotion system for it to move

We have many systems and each one can only do one thing. In order to do this thing, each system is given the data it needs to do it.


```
public class WaypointLocomotionSystemData
{
    public IReadOnlyList<Position> Positions { get; }
}

public class TargetAtDistanceLocomotionSystemData
{
    public Position TargetPosition { get; }
    public float Distance { get; }
}


public WaypointLocomotionSystem
{
    private WaypointLocomotionSystemData waypointData;

    public void Move(WaypointLocomotionSystemData waypointData)
    {
        this.waypointData = waypointData;
    }

    public void Tick()
    {
        //Movement code for waypoint...
    }
}

public TargetAtDistanceLocomotionSystem
{
    private TargetAtDistanceLocomotionSystemData targetAtDistanceData;

    public void Move(TargetAtDistanceLocomotionSystemData targetAtDistanceData)
    {
        this.targetAtDistanceData = targetAtDistanceData; 
    }

    public void Tick()
    {
        //Movement code for target at distance...
    }
}
```

On this piece of code we can observe:
- There is a data class for each movement strategy
- There is a system class for each movement strategy
- Each system strategy has access to the corresponding data
- When the class is move is requested the actual strategy class is built


## Multiple dinamic

- There is an NPC
- The NPC has a locomotion system
- The locomotion system can move in many ways: following some waypoints and following a target at a distance
- We have multiple pieces of data each one with the data for each behaviour
- The data is provided to the locomotion system for it to move

A single system can run all the strategies. In order to do all of them it uses the type of the data to choose the strategy instance.


```
public class ILocomotionSystemData
{
}

public class WaypointLocomotionSystemData : ILocomotionSystemData
{
    public IReadOnlyList<Position> Positions { get; }
}

public class TargetAtDistanceLocomotionSystemData : ILocomotionSystemData
{
    public Position TargetPosition { get; }
    public float Distance { get; }
}


public ILocomotionSystemBehaviour
{
    void Tick();
}

public WaypointLocomotionSystemBehaviour : ILocomotionSystemBehaviour
{
    private readonly WaypointLocomotionSystemData waypointData;

    public WaypointLocomotionSystemBehaviour(WaypointLocomotionSystemData waypointData)
    {
        this.waypointData = waypointData; 
    }

    public void Tick()
    {
        //Movement code for waypoint...
    }
}

public TargetAtDistanceLocomotionSystemBehaviour : ILocomotionSystemBehaviour
{
    private readonly TargetAtDistanceLocomotionSystemData targetAtDistanceData;

    public TargetAtDistanceLocomotionSystemBehaviour(TargetAtDistanceLocomotionSystemData targetAtDistanceData)
    {
        this.targetAtDistanceData = targetAtDistanceData; 
    }

    public void Tick()
    {
        //Movement code for target at distance...
    }
}


public class LocomotionSystem
{
    private ILocomotionSystemBehaviour locomotionSystemBehaviour;

    public void Move(ILocomotionSystemData locomotionSystemData)
    {
        switch(locomotionSystemData)
        {
            case WaypointLocomotionSystemBehaviour waypointData:
                locomotionSystemBehaviour = new WaypointLocomotionSystemBehaviour(locomotionSystemData);
                break;

            case TargetAtDistanceLocomotionSystemBehaviour targetAtDistanceData:
                locomotionSystemBehaviour = new TargetAtDistanceLocomotionSystemBehaviour(targetAtDistanceData);
                break;
            default:
                throw new ArgumentOutOfRange();
        }
    }

    //Method called once per frame
    public void Tick()
    {
        locomotionSystemBehaviour.Tick();
    }
}

```

On this piece of code we can observe:
- There is an empty interface for the data classes
- There is a data class for each movement strategy
- There is an interface for the system movement behaviour
- There is a moment class for each movement strategy
- Each movement strategy has access to the corresponding data
- There is a class that implements the system
- When the system is requested to Move the concrete strategy is selected and used


# Conclusions 

**Single** should be used when a single strategy is needed.

**Multiple static** should be used when there are multiple strategies and it is selected when first creating it.

**Multiple dynamic** should be used when there are multiple strategies and they should all be runnable from the same place.

Further notes and recomendations:
- **Single** and **Multiple static** are the same implementation wise, but a design desision has been made when selecting **Multiple single** over **Multiple dynamic**. 
- If you find yourself implementing some data driven functionality and currently have a single strategy to implement, implementing it using **Single** and when the requirements change the functionality can be easily refactored to **Multiple dynamic**.