<template>
	<div
		class="w-full h-screen flex flex-col items-center justify-around bg-blue-950"
	>
		<h1 class="h-16 text-4xl text-white">WebGPU Conway Game of Life</h1>
		<canvas
			class="w-[512px] aspect-square"
			width="512"
			height="512"
			id="gameDisplay"
		></canvas>
	</div>
</template>

<script setup lang="js">

const GRID_SIZE = 128;
const UPDATE_INTERVAL_IN_MS = 8;
const WORKGROUP_SIZE = 8;
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
    cellStateArray[i] = Math.random() > 0.5 ? 1 : 0;
  }
  // Calling this copies the data in the array to GPU and the TypedArray can
  // be modified without impacting the buffer
  gpuDevice.queue.writeBuffer(cellStateStorageBuffers[0], 0, cellStateArray)

  const vertexBufferLayout = {
    arrayStride: 8, // number bytes to skip to reach the next vertex, 4bytes * 2 coords per vertex
    attributes: [{
      format:"float32x2", // two 32 bit floats per vertex
      offset:0, // where in the buffer this attribute starts
      shaderLocation: 0 // position in the vertex shader, must be unique among attributes

    }]
  }

  const simulationShaderCode = `
    @group(0) @binding(0) var<uniform> grid: vec2f;
    @group(0) @binding(1) var<storage> cellStateIn: array<u32>;
    @group(0) @binding(2) var<storage, read_write> cellStateOut: array<u32>;

    fn cellIndex(cell: vec2u) -> u32{
      return (cell.y % u32(grid.y)) * u32(grid.x) + (cell.x % u32(grid.x));
    }

    fn cellActive(x: u32, y:u32) -> u32 {
      return cellStateIn[cellIndex(vec2(x,y))];
    }

    @compute
    @workgroup_size(${WORKGROUP_SIZE}, ${WORKGROUP_SIZE})
    fn computeMain(@builtin(global_invocation_id) cell: vec3u) {
      let activeNeighbors = 
      cellActive(cell.x+1, cell.y+1) +
      cellActive(cell.x+1, cell.y) +
      cellActive(cell.x+1, cell.y-1) +
      cellActive(cell.x, cell.y-1) +
      cellActive(cell.x-1, cell.y-1) +
      cellActive(cell.x-1, cell.y) +
      cellActive(cell.x-1, cell.y+1) +
      cellActive(cell.x, cell.y+1);

      let i = cellIndex(cell.xy);

      // Conway's game of life rules:
      switch activeNeighbors {
        case 2: { // Active cells with 2 neighbors stay active.
          cellStateOut[i] = cellStateIn[i];
        }
        case 3: { // Cells with 3 neighbors become or stay active.
          cellStateOut[i] = 1;
        }
        default: { // Cells with < 2 or > 3 neighbors become inactive.
          cellStateOut[i] = 0;
        }
      }

    }
  `

  const simulationShaderModule = gpuDevice.createShaderModule({
    label: "Game of Life Simulation Shader",
    code: simulationShaderCode
  })

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

    const bindGroupLayout = gpuDevice.createBindGroupLayout({
      label: "Cell Bind Group Layout",
      entries: [{
        binding: 0,
        visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE | GPUShaderStage.FRAGMENT,
        buffer: {} // Grid Uniform Buffer
      },
      {
      binding: 1,
      visibility: GPUShaderStage.VERTEX | GPUShaderStage.COMPUTE,
      buffer: { type: "read-only-storage"} // Cell state input buffer
      },
      {
      binding: 2,
      visibility: GPUShaderStage.COMPUTE,
      buffer: { type: "storage"} // Cell state output buffer
      }]
    })

  const cellPipelineLayout = gpuDevice.createPipelineLayout({
    label: "Cell Render Pipeline Layout",
    bindGroupLayouts: [ bindGroupLayout ],
  })

  const cellRenderPipeline = gpuDevice.createRenderPipeline({
    label: "Cell Render Pipeline",
    layout: cellPipelineLayout,
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

  const simulationPipeline = gpuDevice.createComputePipeline({
    label: "Simulation pipeline",
    layout: cellPipelineLayout,
    compute: {
      module: simulationShaderModule,
      entryPoint: "computeMain",
    }
  })


  const bindGroups = [
    gpuDevice.createBindGroup({
      label: "Cell Renderer Bind Group A",
      layout: bindGroupLayout,
      entries: [{
        binding: 0,
        resource: {buffer: uniformBuffer}
      },
      {
        binding: 1,
        resource: {buffer: cellStateStorageBuffers[0]}
      }, {
      binding: 2, // New Entry
      resource: { buffer: cellStateStorageBuffers[1] }
    }]
    }),
    gpuDevice.createBindGroup({
      label: "Cell renderer bind group B",
      layout: bindGroupLayout,
      entries: [{
        binding: 0,
        resource: { buffer: uniformBuffer }
      }, {
        binding: 1,
        resource: { buffer: cellStateStorageBuffers[1] }
      },  {
      binding: 2, // New Entry
      resource: { buffer: cellStateStorageBuffers[0] }
    }],

    })
  ]
  const renderUpdatedGrid = () => {
    const gpuCommandEncoder = gpuDevice.createCommandEncoder();

    const computePass = gpuCommandEncoder.beginComputePass();

    computePass.setPipeline(simulationPipeline);
    computePass.setBindGroup(0, bindGroups[step % 2]);

    const workgroupCount = Math.ceil(GRID_SIZE / WORKGROUP_SIZE);
    computePass.dispatchWorkgroups(workgroupCount, workgroupCount);

    computePass.end();

    step++;

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
