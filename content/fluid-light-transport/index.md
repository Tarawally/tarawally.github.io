# Preface 

This document provides a **Guided Technical Walkthrough** of the Hybrid Fluid-Light Transport engine.

Rather than focusing on jargon, we explore the underlying architecture. We build our understanding from the first principles of digital rasterisation to the complex simulation of non-linear light transport.

## The Graphics Dichotomy

In real-time computer graphics, we typically face a choice:

1.  **Ray Tracing**: Offers extreme precision (like real-world light) but is computationally expensive, especially for soft, diffuse effects.
2.  **Rasterisation**: The standard for games; it is incredibly fast but struggles with complex "global illumination" (how light bounces around).

This book explores a "third way": **Hybrid Fluid-Light Transport**. We treat light not merely as rays, but as a **fluid substance** that flows across the scene. This yields beautiful, organic lighting at a fraction of the cost.

## The Goal

We aim to demystify the following code:

```javascript
/**
 * @fileoverview Hybrid Fluid-Light Transport Engine.
 * This technique creates soft shadows, colour bleeding, and ambient occlusion
 * purely through 2D pixel-neighbour interactions.
 */
```

By the end of this book, you will understand exactly what that means and how to implement it.

## How to Read This Book

1.  **The Digital Canvas**: We start with the screen itself.
2.  **The Logic**: We learn the grammar of JavaScript.
3.  **The Memory**: We explore how computers remember things.
4.  **The Maths**: We discover how to measure space.
5.  **The Physics**: We simulate the behaviour of light.

Let us begin our journey.