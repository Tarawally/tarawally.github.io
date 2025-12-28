--- 
title: "01 canvas"
layout: "fluid-notebook"
---

<div class="quarto-embed-container">
  <iframe 
    id="quarto-iframe"
    src="/fluid-book/01_canvas.html" 
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
# The Digital Canvas

Before painting with light, we must understand our canvas.

## The Pixel

Screens comprise millions of tiny squares called **Pixels** (Picture Elements). Each pixel is not a single colour but a blend of three primary lights: **Red**, **Green**, and **Blue** (RGB).

> [!note]- Deep Dive: Additive Colour
> Unlike mixing paint (where red + blue = purple), mixing light is **additive**.
> *   Red + Green = Yellow
> *   Red + Blue = Magenta
> *   Green + Blue = Cyan
> *   Red + Green + Blue = White

## The Cartesian Grid

In mathematics, graphs typically place $(0,0)$ in the centre with $Y$ increasing *upwards*. Computer graphics differ:

*   The **Origin** $(0,0)$ sits at the **Top-Left** corner.
*   **X** increases to the **Right**.
*   **Y** increases **Downwards**.


## The Canvas Element

We access this grid via the HTML5 [Canvas API](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) using the `<canvas>` element.

```html
<!-- MAIN RENDERING SURFACE -->
<canvas id="canvas" tabindex="0"></canvas>
```

In `src/engine.js`, we establish a connection to this element to draw upon it.

```javascript
// Canvas Context configuration
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d', {
  alpha: false,
  willReadFrequently: true,
});
```

The [context](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext) (`ctx`) is our tool for modifying the canvas.

> [!tip] Performance: Resizing
> Resizing the browser window is computationally expensive. Our engine employs a **debounce** function to ensure the grid is only recalculated once the user stops adjusting the window, preventing the simulation from freezing.
</div>
