--- 
title: "06 fluids"
layout: "fluid-notebook"
---

<div class="quarto-embed-container">
  <iframe 
    id="quarto-iframe"
    src="/fluid-book/06_fluids.html" 
    width="100%" 
    style="border:none; min-height: 800px;" 
    onload="initQuartoIframe(this)">
  </iframe>
</div>

<script>
function initQuartoIframe(iframe) {
  // Sync theme
  const isDark = document.documentElement.getAttribute('saved-theme') === 'dark' || 
                 document.body.classList.contains('dark');
  iframe.contentWindow.postMessage({ type: 'themechange', theme: isDark ? 'dark' : 'light' }, '*');
  
  // Listen for height updates from the iframe
  window.addEventListener('message', function(event) {
    if (event.data && event.data.type === 'resize') {
      iframe.style.height = event.data.height + 'px';
    }
  });
}

// Watch for Quartz theme changes
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    if (mutation.type === 'attributes' && mutation.attributeName === 'saved-theme') {
      const isDark = document.documentElement.getAttribute('saved-theme') === 'dark';
      const iframe = document.getElementById('quarto-iframe');
      if (iframe) {
        iframe.contentWindow.postMessage({ type: 'themechange', theme: isDark ? 'dark' : 'light' }, '*');
      }
    }
  });
});
observer.observe(document.documentElement, { attributes: true });
</script>

<div style="display: none;">
# Simulating Fluids

This is the engine's heart. After injecting light energy into the grid, we treat it as a fluid.

## The Fluid Analogy

*   **Light Intensity** $\approx$ **Fluid Pressure** (Quantity)
*   **Light Direction** $\approx$ **Fluid Velocity** (Flow direction)

## Cellular Automata

We use **Cellular Automata** (CA). Imagine a checkerboard where every square observes its neighbours to determine its next state.

### 1. Advection (Movement)

If a pixel has velocity pointing Right, it pushes its energy to the Right neighbour.

### 2. Diffusion (Spreading)

Even without velocity, energy spreads. This creates soft shadows and ambient occlusion.

## Interactive Simulation

Below is a live version of the fluid logic running in this book.




::: {.panel-tabset}

## Visualisation


## Raw Data


:::

## Surface Continuity Check

In a 2D grid, light might accidentally "bleed" from a foreground object onto a background one. We prevent this with a **Surface Continuity Check**.

We examine the **Depth** (distance from the camera) of two pixels. A large difference implies distinct objects, so we block the flow of light between them.

```javascript
/* src/engine.js */
const depthDiff = Math.abs(
  State.lattice[ptr + FIELD.DEPTH] - State.lattice[nPtr + FIELD.DEPTH]
);

if (depthDiff < 0.5) {
  // Surfaces are connected! Flow energy.
  State.lattice[nPtr + FIELD.R] += State.lattice[ptr + FIELD.R] * transfer;
}
```
</div>
