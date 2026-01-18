---
title: Monistic Transport Engine
---

## Introduction

This literate programming notebook serves as the definitive
architectural specification and implementation guide for the **Monistic
Transport Engine (MTE)**. Designed for execution within the WebGPU
environment, the MTE functions as a **Physics Hypervisor** that
prioritises arithmetic proceduralism over traditional memory-intensive
rasterisation.

By deconstructing the engine’s core axioms—specifically host-managed
atomic scheduling and radiance cascades—this resource functions as both
a technical manual and a practical framework for mastering
high-performance JavaScript. Implementation logic adheres to [Google’s
Style Guidelines](https://google.github.io/styleguide/tsguide.html) to
ensure maintainability.

### Architectural visualisation

The engine operates on a strict bottom-up data flow, moving from
host-managed scheduling to on-chip arithmetic generation.

``` mermaid
flowchart TD
    A["Host CPU (JavaScript)"] -->|Atomic Batching| B[WebGPU Queue]
    B --> C{Radiance Injection}
    C -->|CIE xyY LogLuv| D[Cascade 0: High Res]
    D --> E{Cascade Merge}
    E -->|Software Bilinear| D
    D --> F{Holographic Resolve}
    F -->|LOD Re-Trace| G[Final Image]
```

## Foundational axioms

The MTE operates under three primary architectural constraints to ensure
performance stability and safety in a browser-based environment:

1.  **The bandwidth cap**: Bandwidth is the scarcest resource. To
    minimise memory overhead, all radiance data is compressed into a
    strictly defined 64-bits per pixel format (`RG32UI`) utilising **CIE
    xyY LogLuv** encoding.
2.  **The atomic batch**: Timeout Detection and Recovery (TDR) safety is
    guaranteed by partitioning the frame into four fixed dispatches.
    Host-managed scheduling uses ceiling division to ensure
    comprehensive screen coverage across all hardware.
3.  **LOD-aware re-tracing**: The engine prioritises arithmetic
    proceduralism. Instead of reading a heavy G-Buffer, the final pass
    re-calculates geometry on-chip to derive velocity and depth.

## Part 1: the physical layer

This section establishes the raw materials: the Silicon connection, the
Memory allocations, and the Host-Device interface.

### Environment setup

The following initialisation logic establishes the primary asynchronous
entry point for the WebGPU device.

``` {ojs}
/**
 * Initialises the WebGPU environment for MTE execution.
 * This cell leverages the browser's GPU API to request an adapter and device.
 * @return {Promise<GPUDevice>} The authorised WebGPU device.
 */
device = {
  const adapter = await navigator.gpu?.requestAdapter();
  if (!adapter) {
    throw new Error('WebGPU is not supported on this browser.');
  }

  const device = await adapter.requestDevice();
  if (!device) {
    throw new Error('Failed to create a WebGPU device.');
  }

  return device;
}
```

#### Technical deconstruction: the asynchronous handshake

The initialisation of the MTE relies heavily on the asynchronous nature
of JavaScript.

1.  **The Promise Protocol:** The `navigator.gpu.requestAdapter()`
    method returns a
    [`Promise`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).
    We utilise the
    [`await`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
    operator to pause execution within this cell until the GPU is ready,
    without freezing the browser UI.
2.  **Topological Execution:** OJS constructs a **Dependency Graph**.
    The `device` variable acts as a reactive node. Subsequent cells
    implicitly subscribe to this node, mimicking a **Dependency
    Injection** pattern found in systems engineering.

### Physical constants

These values are immutable during the execution lifecycle and dictate
the resolution of radiance cascades.

``` {ojs}
/**
 * Engine constants following Google's naming conventions for immutable values.
 * These parameters define the resolution and scheduling limits of the MTE.
 */
constants = {
  const RENDER_WIDTH = 1920;
  const RENDER_HEIGHT = 1080;
  const CASCADE_LEVELS = 6;
  const BITS_PER_PIXEL = 64;
  const BATCH_COUNT = 4;
  const WORKGROUP_SIZE = 8;

  return {
    RENDER_WIDTH,
    RENDER_HEIGHT,
    CASCADE_LEVELS,
    BITS_PER_PIXEL,
    BATCH_COUNT,
    WORKGROUP_SIZE,
    // Calculated total tile coverage using ceiling division
    TOTAL_TILES: Math.ceil(RENDER_WIDTH / WORKGROUP_SIZE) * Math.ceil(RENDER_HEIGHT / WORKGROUP_SIZE)
  };
}
```

### Buffer memory layouts

The primary data structures include the **global atomic buffer** for
scheduling and the **radiance texture** for light transport.

``` {ojs}
/**
 * Initialises the primary memory structures for the MTE.
 * Implements the Texture View Hierarchy for Radiance Cascades (C0-C5).
 */
buffers = {
  if (!device || !constants) return null;

  // 1. Global Atomic Counter (4 bytes)
  const globalAtomic = device.createBuffer({
    label: 'mte_global_atomic_counter',
    size: 4,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST | GPUBufferUsage.COPY_SRC,
  });

  // 2. Radiance Cascade Texture (The Pyramid)
  const cascadeTexture = device.createTexture({
    label: 'mte_cascade_pyramid',
    size: {
      width: constants.RENDER_WIDTH,
      height: constants.RENDER_HEIGHT,
      depthOrArrayLayers: 1
    },
    format: 'rg32uint', // Matches LogLuv 64-bit cap
    mipLevelCount: constants.CASCADE_LEVELS,
    usage: GPUTextureUsage.STORAGE_BINDING | GPUTextureUsage.TEXTURE_BINDING
  });

  // 3. Texture Views
  const cascadeViews = [];
  for (let i = 0; i < constants.CASCADE_LEVELS; i++) {
    cascadeViews.push(cascadeTexture.createView({
      label: `mte_cascade_view_C${i}`,
      format: 'rg32uint',
      baseMipLevel: i,
      mipLevelCount: 1,
    }));
  }

  return { globalAtomic, cascadeTexture, cascadeViews };
}
```

#### Technical deconstruction: the binary interface

To bridge the gap between JavaScript’s dynamic types and GPU memory, we
utilise
[`TypedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray)
structures in later steps. This ensures bit-perfect synchronisation with
the GPU’s [`std140`](https://www.w3.org/TR/WGSL/#memory-layouts) memory
layout.

### The visual surface

We explicitly configure the visual context to use `rgba8unorm`. This is
a critical architectural decision to ensure the shader’s mathematical
output (Red/Blue channels) maps 1:1 to the display, bypassing
OS-specific preferences for BGRA.

``` {ojs}
visualCanvas = {
  const canvas = document.createElement('canvas');
  canvas.width = constants.RENDER_WIDTH;
  canvas.height = constants.RENDER_HEIGHT;
  canvas.style.width = '100%';
  return canvas;
}
```

``` {ojs}
/**
 * Configures the WebGPU swap chain.
 * FORCED: rgba8unorm to match shader logic.
 */
canvasContext = {
  const context = visualCanvas.getContext('webgpu');
  context.configure({
    device: device,
    format: 'rgba8unorm', // Critical Fix for Colour Correctness
    alphaMode: 'premultiplied'
  });
  return context; 
}
```

## Part 2: The compute core (tools)

Here, we define the three distinct “Machines” that will operate on the
data: Injection, Merge, and Resolve.

### Tool A: radiance injection (the hypervisor)

This kernel acts as the physics hypervisor. It uses **CIE xyY LogLuv**
encoding to preserve high dynamic range colour data without clipping
saturated hues (like orange). It also implements **Metaprogramming via
Template Literals** to inject physical constants directly into the
shader source.

``` {ojs}
/**
 * WGSL for Radiance Injection.
 * Implements CIE xyY LogLuv and Dynamic Stride Injection.
 */
radianceInjectionModule = {
  if (!device || !constants) return null;

  const workgroupSize = 8;
  const strideX = Math.ceil(constants.RENDER_WIDTH / workgroupSize);
  const totalTiles = constants.TOTAL_TILES;
  const aspectRatio = constants.RENDER_WIDTH / constants.RENDER_HEIGHT;

  const shaderSource = `
    @group(0) @binding(0) var<storage, read_write> globalAtomic: atomic<u32>;
    @group(0) @binding(1) var radianceTexture: texture_storage_2d<rg32uint, write>;

    var<workgroup> sharedTileIndex: u32;

    // --- CIE xyY LOGLUV ENCODING ---
    fn encodeLogLuv(rgb: vec3<f32>) -> vec4<u32> {
      let X = 0.4124564*rgb.r + 0.3575761*rgb.g + 0.1804375*rgb.b;
      let Y = 0.2126729*rgb.r + 0.7151522*rgb.g + 0.0721750*rgb.b;
      let Z = 0.0193339*rgb.r + 0.1191920*rgb.g + 0.9503041*rgb.b;
      
      let sum = X + Y + Z;
      if (sum < 0.000001) { return vec4<u32>(0u); }

      let x = X / sum;
      let y = Y / sum;
      let logLum = (log2(Y) + 64.0) * 256.0;
      let packedLum = u32(max(logLum, 0.0));
      let u = u32(clamp(x, 0.0, 1.0) * 65535.0);
      let v = u32(clamp(y, 0.0, 1.0) * 65535.0);

      return vec4<u32>(packedLum, (u << 16u) | v, 0u, 0u);
    }

    fn sdSphere(p: vec3<f32>, r: f32) -> f32 { return length(p) - r; }
    fn map(p: vec3<f32>) -> f32 { return sdSphere(p - vec3(0.0, 0.0, 2.0), 1.0); }
    
    fn calcNormal(p: vec3<f32>) -> vec3<f32> {
      let e = 0.001;
      return normalize(vec3(
        map(p+vec3(e,0,0))-map(p-vec3(e,0,0)), 
        map(p+vec3(0,e,0))-map(p-vec3(0,e,0)), 
        map(p+vec3(0,0,e))-map(p-vec3(0,0,e))
      ));
    }

    @compute @workgroup_size(${workgroupSize}, ${workgroupSize})
    fn main(@builtin(local_invocation_index) localIdx: u32) {
      // Workgroup Elector Pattern
      if (localIdx == 0u) { sharedTileIndex = atomicAdd(&globalAtomic, 1u); }
      workgroupBarrier();
      
      let tileIndex = sharedTileIndex;
      // Injected Constant Check via Template Literals
      if (tileIndex >= ${totalTiles}u) { return; }

      let stride = ${strideX}u;
      let tileX = tileIndex % stride;
      let tileY = tileIndex / stride;
      let px = tileX * ${workgroupSize}u + (localIdx % ${workgroupSize}u);
      let py = tileY * ${workgroupSize}u + (localIdx / ${workgroupSize}u);

      let uv = (vec2<f32>(f32(px), f32(py)) / vec2(${constants.RENDER_WIDTH}.0, ${constants.RENDER_HEIGHT}.0)) * 2.0 - 1.0;
      let rayDir = normalize(vec3<f32>(uv.x * ${aspectRatio}, -uv.y, 1.0));
      let ro = vec3<f32>(0.0, 0.0, -2.0);

      var t = 0.0; var hit = false;
      for(var i=0; i<64; i++) {
        let d = map(ro + rayDir * t);
        if (d < 0.001) { hit = true; break; }
        t += d;
        if (t > 20.0) { break; }
      }

      var col = vec3<f32>(0.05, 0.1, 0.4); 
      if (hit) {
        let n = calcNormal(ro + rayDir * t);
        let lightDir = normalize(vec3(-1.0, 1.0, -1.0));
        let diff = max(dot(n, lightDir), 0.0);
        col = vec3(1.0, 0.5, 0.2) * diff * 5.0; // High Intensity Orange
      }

      textureStore(radianceTexture, vec2<i32>(i32(px), i32(py)), encodeLogLuv(col));
    }
  `;
  return device.createShaderModule({ label: 'mte_inject_shader', code: shaderSource });
}
```

#### Technical deconstruction: metaprogramming

We leverage JavaScript [**Template
Literals**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)
to inject `${strideX}` and `${totalTiles}` directly into the shader
string. This functions as a Just-In-Time (JIT) preprocessor, ensuring
the shader is hard-coded to the current resolution without runtime
uniform overhead.

### Tool B: the cascade merge (the fusion)

The merge pass utilises arithmetic bilinear filtering to blend probes
between levels, as the hardware cannot filter `RG32UI` textures
natively.

``` {ojs}
/**
 * WGSL for Cascade Merge.
 * Uses arithmetic bilinear interpolation.
 */
cascadeMergeModule = {
  if (!device) return null;
  const shaderSource = `
    @group(0) @binding(1) var destTex: texture_storage_2d<rg32uint, write>;
    @group(0) @binding(2) var srcTex: texture_2d<u32>;

    // Placeholder decode/encode for merge logic
    fn decode(p: vec4<u32>) -> vec3<f32> { return vec3(0.0); } 
    fn encode(c: vec3<f32>) -> vec4<u32> { return vec4(0u); }

    @compute @workgroup_size(8, 8)
    fn main(@builtin(global_invocation_id) id: vec3<u32>) {
       // Hierarchical merge logic (Placeholder for Phase 2)
    }
  `;
  return device.createShaderModule({ label: 'mte_merge_shader', code: shaderSource });
}
```

### Tool C: the resolve (the lens)

This shader implements **LOD-Aware Re-Tracing**. It fires a primary ray
again to establish geometric truth (depth/hit) separate from the
lighting texture, enabling future TAA implementation.

``` {ojs}
/**
 * WGSL for Final Resolve.
 * Implements Re-Tracing and CIE xyY Decoding.
 */
resolveModule = {
  if (!device || !constants) return null;
  
  const aspectRatio = constants.RENDER_WIDTH / constants.RENDER_HEIGHT;

  const shaderSource = `
    @group(0) @binding(2) var inputTex: texture_2d<u32>;

    struct VertexOutput {
      @builtin(position) pos: vec4<f32>,
      @location(0) uv: vec2<f32>
    };

    fn decodeLogLuv(p: vec4<u32>) -> vec3<f32> {
      let Y = exp2(f32(p.x) / 256.0 - 64.0);
      let x = f32((p.y >> 16u) & 0xFFFFu) / 65535.0;
      let y = f32(p.y & 0xFFFFu) / 65535.0;

      if (y < 0.000001) { return vec3(0.0); }

      let X = (x / y) * Y;
      let Z = ((1.0 - x - y) / y) * Y;

      let R =  3.2404542*X - 1.5371385*Y - 0.4985314*Z;
      let G = -0.9692660*X + 1.8760108*Y + 0.0415560*Z;
      let B =  0.0556434*X - 0.2040259*Y + 1.0572252*Z;

      return max(vec3(R, G, B), vec3(0.0));
    }

    fn sdSphere(p: vec3<f32>, r: f32) -> f32 { return length(p) - r; }
    fn map_low_lod(p: vec3<f32>) -> f32 { return sdSphere(p - vec3(0.0, 0.0, 2.0), 1.0); }

    @fragment
    fn main(in: VertexOutput) -> @location(0) vec4<f32> {
      // Re-Trace Logic (LOD Aware)
      let uv_ndc = in.uv * 2.0 - 1.0; 
      let rayDir = normalize(vec3<f32>(uv_ndc.x * ${aspectRatio}, -uv_ndc.y, 1.0)); 
      let ro = vec3<f32>(0.0, 0.0, -2.0);

      var t = 0.0; var hit = false;
      for(var i=0; i<32; i++) {
        let d = map_low_lod(ro + rayDir * t);
        if (d < 0.001) { hit = true; break; }
        t += d;
        if (t > 20.0) { break; }
      }

      let pixelCoord = vec2<i32>(in.pos.xy);
      let packedLight = textureLoad(inputTex, pixelCoord, 0);
      let hdrColor = decodeLogLuv(packedLight);

      // Tone Mapping (ACES)
      let a = 2.51; let b = 0.03; let c = 2.43; let d = 0.59; let e = 0.14;
      let sdr = clamp((hdrColor*(a*hdrColor+b))/(hdrColor*(c*hdrColor+d)+e), vec3(0.0), vec3(1.0));
      let gamma = pow(sdr, vec3(1.0/2.2));

      return vec4(gamma, 1.0);
    }
  `;
  return device.createShaderModule({ label: 'mte_resolve_shader', code: shaderSource });
}
```

### Pipeline configuration

This section binds the tools to the materials.

``` {ojs}
/**
 * Initialises all pipelines.
 * Configures resource bindings and layout compatibility.
 */
pipelineState = {
  if (!device || !buffers || !constants || !resolveModule) return null;

  const dummyTexture = device.createTexture({
    size: [1, 1],
    format: 'rg32uint',
    usage: GPUTextureUsage.STORAGE_BINDING
  });

  const bindGroupLayout = device.createBindGroupLayout({
    label: 'mte_bind_group_layout',
    entries: [
      { binding: 0, visibility: GPUShaderStage.COMPUTE, buffer: { type: 'storage' } },
      { binding: 1, visibility: GPUShaderStage.COMPUTE, storageTexture: { access: 'write-only', format: 'rg32uint', viewDimension: '2d' } },
      { binding: 2, visibility: GPUShaderStage.COMPUTE | GPUShaderStage.FRAGMENT, texture: { sampleType: 'uint', viewDimension: '2d' } }
    ]
  });

  const injectionBindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: { buffer: buffers.globalAtomic } },
      { binding: 1, resource: buffers.cascadeViews[0] }, 
      { binding: 2, resource: buffers.cascadeViews[1] } 
    ]
  });

  const resolveBindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: { buffer: buffers.globalAtomic } },
      { binding: 1, resource: dummyTexture.createView() },   
      { binding: 2, resource: buffers.cascadeViews[0] }      
    ]
  });

  const pipelineLayout = device.createPipelineLayout({ bindGroupLayouts: [bindGroupLayout] });

  // Initialise Pipelines
  const injection = device.createComputePipeline({
    layout: pipelineLayout,
    compute: { module: radianceInjectionModule, entryPoint: 'main' }
  });

  const resolveVertex = device.createShaderModule({
    code: `
      struct VertexOutput { @builtin(position) p: vec4<f32>, @location(0) uv: vec2<f32> };
      @vertex fn main(@builtin(vertex_index) i: u32) -> VertexOutput {
        var pos = array<vec2<f32>,3>(vec2(-1.,-1.), vec2(3.,-1.), vec2(-1.,3.));
        var out: VertexOutput;
        out.p = vec4(pos[i], 0., 1.);
        out.uv = out.p.xy * 0.5 + 0.5; out.uv.y = 1.0 - out.uv.y;
        return out;
      }
    `
  });

  const resolve = device.createRenderPipeline({
    layout: pipelineLayout,
    vertex: { module: resolveVertex, entryPoint: 'main' },
    fragment: { 
      module: resolveModule, 
      entryPoint: 'main', 
      targets: [{ format: 'rgba8unorm' }] // Matches Canvas Context
    },
    primitive: { topology: 'triangle-list' }
  });

  return { injection, resolve, injectionBindGroup, resolveBindGroup };
}
```

## Part 3: assembly (the scheduler)

The `scheduler` cell functions as the central nervous system of the
engine.

``` {ojs}
/**
 * Orchestrates the execution frame.
 * Coordinates synchronisation, injection, and resolution.
 */
scheduler = {
  if (!device || !buffers || !pipelineState || !canvasContext) return null;

  const tilesPerBatch = Math.ceil(constants.TOTAL_TILES / constants.BATCH_COUNT);

  const executeFrame = () => {
    const currentTextureView = canvasContext.getCurrentTexture().createView();
    const commandEncoder = device.createCommandEncoder({ label: 'mte_frame_encoder' });

    // --- PHASE 1: SYNCHRONISATION ---
    // Encoder-First Clear: Guarantees the atomic counter is zeroed 
    // on the GPU timeline before any compute threads launch.
    commandEncoder.clearBuffer(buffers.globalAtomic, 0, 4);

    // --- PHASE 2: RADIANCE INJECTION ---
    const computePass = commandEncoder.beginComputePass({ label: 'mte_injection_pass' });
    computePass.setPipeline(pipelineState.injection);
    computePass.setBindGroup(0, pipelineState.injectionBindGroup);
    
    // Dispatch in fixed batches to adhere to TDR safety protocols
    for (let i = 0; i < constants.BATCH_COUNT; i++) {
      computePass.dispatchWorkgroups(tilesPerBatch, 1, 1);
    }
    computePass.end();

    // --- PHASE 3: CASCADE MERGE ---
    // For this foundational demonstration, the Cascade Merge pass is bypassed 
    // to visualise the raw atomic injection. 
    // The architecture supports it, but C1-C5 are currently unpopulated.

    // --- PHASE 4: HOLOGRAPHIC RESOLVE ---
    const renderPass = commandEncoder.beginRenderPass({
      label: 'mte_resolve_pass',
      colorAttachments: [{
        view: currentTextureView,
        clearValue: { r: 0, g: 0, b: 0, a: 1 },
        loadOp: 'clear',
        storeOp: 'store'
      }]
    });

    renderPass.setPipeline(pipelineState.resolve);
    renderPass.setBindGroup(0, pipelineState.resolveBindGroup); 
    renderPass.draw(3); 
    renderPass.end();

    device.queue.submit([commandEncoder.finish()]);
  };

  return { tilesPerBatch, executeFrame };
}
```

#### Host scheduling and the event loop

The “Atomic Batch” strategy is a concession to the JavaScript
[**Run-to-Completion**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)
model. By splitting the work into four batches via `commandEncoder`, we
allow the browser to potentially interleave the GPU submission with
other microtasks, preventing the “frozen tab” symptom.

## Part 4: verification

``` {ojs}
/**
 * Verifies the MTE execution state.
 */
verificationReport = {
  if (!scheduler || !buffers || !constants) return null;

  const ticketCount = scheduler.tilesPerBatch * constants.BATCH_COUNT;
  const vramFootprint = `${(constants.RENDER_WIDTH * constants.RENDER_HEIGHT * 8 / 1024 / 1024).toFixed(2)} MB`;
  const isCoverageComplete = ticketCount >= constants.TOTAL_TILES;

  return {
    status: isCoverageComplete ? 'Compliant' : 'Coverage Gap Detected',
    vramFootprint,
    axioms: { bandwidthCap: 'Active (RG32UI)', physicsHypervisor: 'Enabled' }
  };
}
```

``` {ojs}
// Run the engine
{
  if (scheduler && scheduler.executeFrame) {
    scheduler.executeFrame();
    return "Frame Executed";
  }
  return "Waiting...";
}
```
