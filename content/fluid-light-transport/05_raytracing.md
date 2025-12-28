# Casting Rays

With our grid (Canvas) and maths (Vectors) established, simulation begins. The first step is locating the light.

## Reverse Ray Tracing

In reality, light travels from a bulb to your eye. In computer graphics, we work in reverse. We shoot a ray from the "Camera" (your eye) through every screen pixel to see what it strikes.

```{mermaid}
sequenceDiagram
    participant Camera
    participant Screen
    participant Sphere

    Camera->>Screen: Shoot Ray(x,y)
    Screen->>Sphere: Check Intersection
    alt Hit
        Sphere-->>Screen: Return Colour
    else Miss
        Sphere-->>Screen: Return Black
    end
```

## Intersection: The Sphere

Our scene comprises spheres. To check if a ray hits a sphere, we use algebra. A sphere is defined as all points at a distance $r$ from a centre point $C$.

If we shoot a line, we solve a quadratic equation to find intersection:

$$ t = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

This is implemented in `Scene.trace` within `src/engine.js`.

```javascript
/* src/engine.js */
trace: function(ro, rd) {
    // ...
    // The Discriminant (d) tells us if we hit.
    const d = b * b - c;             // <1>
    if (d > 0) {                     // <2>
        // We hit the sphere!
    }
}
```

1.  Calculates the discriminant $b^2 - 4ac$.
2.  If positive, the line crosses the sphere. If negative, it misses.

## Injection

When a ray strikes a light source, we "inject" that energy into our fluid grid (`State.lattice`). This initiates our fluid simulation.