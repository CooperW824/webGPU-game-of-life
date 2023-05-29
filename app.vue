<template>
	<div
		class="w-full h-screen flex flex-col items-center justify-around bg-blue-950"
	>
		<h1 class="h-16 text-4xl text-white">WebGPU Conway Game of Life</h1>
		<canvas class="w-[512px] aspect-square" id="gameDisplay"></canvas>
	</div>
</template>

<script setup lang="js">
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
      @vertex
      fn vertexMain(@location(0) pos: vec2f) -> 
      @builtin(position) vec4f {
        return vec4f(pos.x, pos.y, 0, 1); // (x, y, z, w)
      }

      @fragment
      fn fragmentMain() -> @location(0) vec4f {
        return vec4f(0.8, 0, 0, 1); // (r, g, b, a)
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
  renderPass.draw(squareVertices.length / 2);
  renderPass.end();

  gpuDevice.queue.submit([gpuCommandEncoder.finish()]);
}


onMounted(main)
</script>
