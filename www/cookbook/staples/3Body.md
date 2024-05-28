---
title: 3-Body Problem
outline: [2, 3]
---

# 3-Body Problem

The [three-body problem](https://en.wikipedia.org/wiki/Three-body_problem) is a classic problem in physics and mathematics that involves predicting the motion of three bodies (such as stars or planets) under their mutual gravitational influence. It's a complex problem because the bodies continuously affect each other's trajectories, leading to chaotic and often unpredictable behavior.

> *Trisolarians, Trishmolarians ...* 

`1:N` and `N:M` relations where Entities continuously influence each other in complex ways have always been a challenge in Entity Component Systems. Usually, these require expensive additional or reverse lookups that do not scale well with the rest of the ECS design.

This includes typical gameplay scenarios such as many troops following a few leaders, flocking, gravitational simulations, threat assessments, and more.

**fenn**ecs's relation features allow us you to model such relations with ease - reaping the full benefits of ECS iteration. In this recipe, we'll show you how to simulate a simple 3-Body stellar system, where each body exerts a gravitational pull on all others.

## Implementing the Simulation

::: details :neofox_hyper: SEE IT IN ACTION: [N-BODY DEMO](/examples/NBody.md)
The setup for the 3-Body Problem is hard-coded to fully illustrate its three-by-three relationship.

If you're a generalization nerd, [this demo](/examples/NBody.md) is a concrete example that uses the <u>same simulation loop</u>, but focuses on a more generic setup step (with batched setup of Entities and Relations).

It demonstrates how to procedurally set up and simulate not only an arbitrary number of bodies, but also coexisting simulations without changing any of the simulation code.
:::

Here's how we can simulate the 3-body problem using fennecs:

::: code-group

```csharp [Components]
// The Body component represents a celestial body in the simulation.
// It stores the body's position and mass, and is used as both a
// plain component and a relation component.
private class Body
{
    public Vector3 position;
    public float mass { init; get; }
}

// The Velocity component stores the current velocity of a body.
private struct Velocity : Fox<Vector3>
{
    public Vector3 Value { get; set; }
}

// The Force component accumulates the total force acting on a body.
private struct Force : Fox<Vector3>
{
    public Vector3 Value { get; set; }
}

// The Position component represents the current position of a body.
private struct Position : Fox<Vector3>
{
    public Vector3 Value { get; set; }
}
```

```csharp [System Setup]
var world = new World();
const int bodyCount = 3;

// Define the initial positions of the bodies
var p1 = new Vector3(-10, -4, 0);
var p2 = new Vector3(0, 12, 0);
var p3 = new Vector3(7, 0, 4);

// Create the Body components for each celestial body
var body1 = new Body{position = p1, mass = 2.0f};
var body2 = new Body{position = p2, mass = 1.5f};
var body3 = new Body{position = p3, mass = 3.5f};

// Spawn the entities representing the celestial bodies
var sun1 = world.Spawn();
sun1.Add<Force>();
sun1.Add(new Position {Value = body1.position});
sun1.Add<Velocity>();

var sun2 = world.Spawn();
sun2.Add<Force>();
sun2.Add(new Position {Value = body2.position});
sun2.Add<Velocity>();

var sun3 = world.Spawn();
sun3.Add<Force>();
sun3.Add(new Position {Value = body3.position});
sun3.Add<Velocity>();

// Set up the relations between the bodies
sun1.Add(body1);
sun1.AddRelation(sun1, body1);
sun1.AddRelation(sun2, body2);
sun1.AddRelation(sun3, body3);

sun2.Add(body2);
sun2.AddRelation(sun1, body1);
sun2.AddRelation(sun2, body2);
sun2.AddRelation(sun3, body3);

sun3.Add(body3);
sun3.AddRelation(sun1, body1);
sun3.AddRelation(sun2, body2);
sun3.AddRelation(sun3, body3);
```

```csharp [Queries]
// Query to accumulate forces acting on each body
var accumulator = world
    .Query<Force, Position, Body>(Match.Plain, Match.Plain, Match.Entity)
    .Compile();

// Query to update velocities and positions based on the accumulated forces
var integrator = world
    .Query<Force, Velocity, Position>()
    .Compile();

// Query to copy the updated positions back to the Body components
var consolidator = world
    .Query<Position, Body>(Match.Plain, Match.Plain)
    .Compile();        
```

```csharp [Simulation Loop]        
// Main simulation loop
// Assumes a fixed time step of 0.01 seconds (100 fps)

// Clear all forces at the start of each iteration
accumulator.Blit(new Force { Value = Vector3.Zero });

// Accumulate gravitational forces between bodies
accumulator.For(static 
    (ref Force force, ref Position position, ref Body attractor) =>
    {
        var distanceSquared = Vector3.DistanceSquared(attractor.position, position.Value);
        if (distanceSquared < float.Epsilon) return; // Avoid self-interaction
        
        var direction = Vector3.Normalize(attractor.position - position.Value);
        force.Value += direction * attractor.mass / distanceSquared;
    });

// Update velocities and positions based on the accumulated forces
integrator.For(static 
    (ref Force force, ref Velocity velocity, ref Position position, float dt) =>
    {
        velocity.Value += dt * force.Value;
        position.Value += dt * velocity.Value;
    }, 0.01f);

// Copy the updated positions back to the Body components
consolidator.For(static
    (ref Position position, ref Body body) =>
    {
        body.position = position.Value;
    });
```
:::

In this simulation:

1. We define the necessary components: `Body`, `Velocity`, `Force`, and `Position`.
2. We set up the initial state by spawning entities for each celestial body and establishing the relations between them.
3. We create queries to accumulate forces, update velocities and positions, and synchronize the positions back to the `Body` components.
4. In the main simulation loop, we perform the following steps:
   - Clear the accumulated forces.
   - Calculate the gravitational forces between bodies using the `accumulator` query.
   - Update the velocities and positions based on the accumulated forces using the `integrator` query.
   - Copy the updated positions back to the `Body` components using the `consolidator` query.

By leveraging fennecs' efficient query system and component-based architecture, we can simulate the complex interactions of the 3-body problem in a clean and performant manner.


::: warning :neofox_science: DON'T MISTAKE GAME DEV TRICKERY FOR MAGIC!
Even though **fenn**ecs gives tangible speed benefits when iterating over Entities due to cache coherent data layout and loop structure, an `N:N` relation still implies runtime complexity `o(n²)` - and thus, an elegant real-time approximation to simulate the *1-Million-Body-Problem* remains elusive.

To simulate larger systems, you would need to employ optimization techniques such as spatial partitioning, approximation methods, or hardware acceleration to reduce the computational burden.

Also, (if it even has to be said), the above code is a simple simulation and not even remotely scientifically accurate. The "Problem" in "3-Body-Problem" means that for these simulations, small changes in initial conditions quickly lead to vastly different and unpredictable outcomes. Analytical solutions for these systems are difficult to generalize, and numerical simulations quickly diverge.
:::
