--- 
title: "04 math"
layout: "fluid-notebook"
---

<div class="quarto-embed-container">
  <iframe 
    id="quarto-iframe"
    src="/fluid-book/04_math.html" 
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
# The Maths of Space

To simulate light, we must describe location and movement using **Vectors**.

## Vectors: Arrows in Space

A **Vector** is simply an arrow with:
1.  **Origin**: Where it starts.
2.  **Direction**: Which way it points.
3.  **Magnitude**: How long it is.

In our 2D grid, a vector is often just two numbers: $(x, y)$.

*   $(1, 0)$ points Right.
*   $(0, 1)$ points Down.
*   $(-1, 0)$ points Left.

### Normalisation

Sometimes we care only about direction, not length. If we shrink a vector so its length is exactly $1.0$, we call it a **Unit Vector** or say it is **Normalised**.

[Normalising a vector is like noting the direction a finger points, whilst ignoring the finger's length.]{.aside}


## Distance (Pythagoras)

To determine the distance of a light source, we use the Pythagorean Theorem:

$$ a^2 + b^2 = c^2 $$

Or in code:

```javascript
const dist = Math.sqrt(x*x + y*y);
```

We use this in `Scene.shade` to calculate light fall-off over distance (Inverse Square Law).
</div>
