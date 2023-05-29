<template>
  <div
    class="w-full h-screen flex flex-col items-center justify-around bg-blue-950"
  >
    <h1 class="h-16 text-4xl text-white">WebGPU Conway Game of Life</h1>
    <canvas class="w-[512px] aspect-square" width="512" height="512" id="gameDisplay"></canvas>
  </div>
</template>

<script setup lang="js">

const GRID_SIZE = 32;
const UPDATE_INTERVAL_IN_MS = 256;
let step = 0;

const main = async () => {
  let canvas = document.getElementById("gameDisplay");

  if (!navigator.gpu) {
    throw new Error("WebGPU not compatible with browser");
  }

  const gpuAdapter = await navigator.gpu.requestAdapter();
  if (!gpuAdapter) {
    throw new Error("No GPU Adapter Found");
  }

  const gpuDevice = await gpuAdapter.requestDevice();
  const canvasContext = canvas.getContext("webgpu");
  const canvasFormat = navigator.gpu.getPreferredCanvasFormat();

  canvasContext.configure({
    device: gpuDevice,
    format:canvasFormat
  })


  const squareVertices = new Float32Array([
    // Triangle 1
    // x,   y
     0.8,  0.8,
    -0.8,  0.8,
    -0.8, -0.8,
    // Triangle 2
    -0.8, -0.8,
     0.8, -0.8,
     0.8,  0.8
  ])

  const vertexBuffer = gpuDevice.createBuffer({
    label: "Cell Vertices",
    size: squareVertices.byteLength,
    usage: GPUBufferUsage.VERTEX| GPUBufferUsage.COPY_DST,
  })

  gpuDevice.queue.writeBuffer(vertexBuffer, /*bufferOffset=*/0, squareVertices);

  const uniformArray = new Float32Array([GRID_SIZE, GRID_SIZE]);

  const uniformBuffer = gpuDevice.createBuffer({
    label: "Grid Uniforms",
    size: uniformArray.byteLength,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
  })

  gpuDevice.queue.writeBuffer(uniformBuffer, 0, uniformArray);

  const cellStateArray = new Uint32Array(GRID_SIZE * GRID_SIZE)

  const cellStateStorageBuffers = [
    gpuDevice.createBuffer({
    label: "Cell State A",
    size: cellStateArray.byteLength,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  }),
  gpuDevice.createBuffer({
    label: "Cell State B",
     size: cellStateArray.byteLength,
     usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
  })
]

  for (let i = 0; i < cellStateArray.length; i += 3){
    cellStateArray[i] = 1;
  }
  // Calling this copies the data in the array to GPU and the TypedArray can
  // be modified without impacting the buffer
  gpuDevice.queue.writeBuffer(cellStateStorageBuffers[0], 0, cellStateArray)

  for (let i = 0; i < cellStateArray.length; i++) {
  cellStateArray[i] = i % 2;
}


  gpuDevice.queue.writeBuffer(cellStateStorageBuffers[1], 0, cellStateArray)

  const vertexBufferLayout = {
    arrayStride: 8, // number bytes to skip to reach the next vertex, 4bytes * 2 coords per vertex
    attributes: [{
      format:"float32x2", // two 32 bit floats per vertex
      offset:0, // where in the buffer this attribute starts
      shaderLocation: 0 // position in the vertex shader, must be unique among attributes

    }]
  }

  const cellShaderCode = /* wgsl */`
      // Shader Code goes here

      struct VertexInput {
        @location(0) pos: vec2f,
        @builtin(instance_index) instance: u32
      }

      struct VertexOutput {
        @builtin(position) pos: vec4f,
        @location(0) cell: vec2f,
      }

      struct FragInput {
        @location(0) cell: vec2f,
      };


      @group(0) @binding(0) var<uniform> grid: vec2f;
      @group(0) @binding(1) var<storage> cellState: array<u32>;

      @vertex
      fn vertexMain(input: VertexInput) ->
      VertexOutput {
        let i = f32(input.instance); // convert to float
        let cell = vec2f(i % grid.x, floor(i / grid.y));
        let state = f32(cellState[input.instance]);

        let cellOffset = cell / grid * 2;
        let gridPos = (input.pos * state + 1) / grid - 1 + cellOffset;

        var output: VertexOutput;
        output.pos = vec4f(gridPos, 0, 1); // (x, y, z, w)
        output.cell = cell;
        return output;
      }

      @fragment
      fn fragmentMain(input: FragInput) -> @location(0) vec4f {
        let colorSelector = input.cell / grid;
        return vec4f(colorSelector, 1 - colorSelector.x, 1); // (r, g, b, a)
      }
    `


  const cellShaderModule = gpuDevice.createShaderModule({
    label: "Cell Shader",
    code: cellShaderCode
  })

  const cellRenderPipeline = gpuDevice.createRenderPipeline({
    label: "Cell Render Pipeline",
    layout: "auto",
    vertex: {
      module: cellShaderModule,
      entryPoint: "vertexMain",
      buffers: [vertexBufferLayout]
    },
    fragment: {
      module: cellShaderModule,
      entryPoint: "fragmentMain",
      targets: [{
        format:canvasFormat
      }]
    }
  })

  const bindGroups = [
    gpuDevice.createBindGroup({
      label: "Cell Renderer Bind Group",
      layout: cellRenderPipeline.getBindGroupLayout(0),
      entries: [{
        binding: 0,
        resource: {buffer: uniformBuffer}
      },
      {
        binding: 1,
        resource: {buffer: cellStateStorageBuffers[0]}
      }
      ]
    }),
    gpuDevice.createBindGroup({
      label: "Cell renderer bind group B",
      layout: cellRenderPipeline.getBindGroupLayout(0),
      entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
      }, {
        binding: 1,
        resource: { buffer: cellStateStorageBuffers[1] }
      }],

    })
  ]
  const renderUpdatedGrid = () => {
    step++;
    const gpuCommandEncoder = gpuDevice.createCommandEncoder();

    const renderPass = gpuCommandEncoder.beginRenderPass({
      colorAttachments: [{
        view: canvasContext.getCurrentTexture().createView(),
        loadOp: "clear",
        clearValue: [15/255, 25/255, 70/255, 0.1],
        storeOp: "store"
      }]
    })

    renderPass.setPipeline(cellRenderPipeline);
    renderPass.setVertexBuffer(0, vertexBuffer);
    renderPass.setBindGroup(0, bindGroups[step % 2]);
    renderPass.draw(squareVertices.length / 2, GRID_SIZE*GRID_SIZE);

    renderPass.end();
    gpuDevice.queue.submit([gpuCommandEncoder.finish()]);
  }

  setInterval(renderUpdatedGrid, UPDATE_INTERVAL_IN_MS)

}


onMounted(main)
</script>
