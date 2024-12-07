<!DOCTYPE html>
<html lang="en">
<head>
  <script src="https://cdn.jsdelivr.net/npm/gpu.js@2.9.1/dist/gpu-browser.min.js"></script>
  <style>
    body {
      text-align: center;
      margin: 0;
      overflow: hidden;
      pointer-events: none; /* Disables mouse input on the body */
    }
    canvas {
      padding: 0;
      margin: 0;
      display: block;
    }
    div {
      position: absolute;
      color: white;
      font-size: 20px;
      font-family: Arial, sans-serif;
      margin-left: auto;
      margin-right: auto;
      left: 0;
      right: 0;
      top: 0;
    }
    #fpsCounter {
      position: absolute;
      top: 10px;
      left: 10px;
      color: white;
      font-size: 24px;
      font-family: Arial, sans-serif;
    }
    #totalRenderedFPS {
      position: absolute;
      top: 40px;
      left: 10px;
      color: white;
      font-size: 24px;
      font-family: Arial, sans-serif;
    }
    #score {
      position: absolute;
      top: 80px;
      left: 10px;
      color: white;
      font-size: 32px;
      font-family: Arial, sans-serif;
    }
  </style>
</head>
<body>
  <div id="fpsCounter">Current FPS: 0</div>
  <div id="totalRenderedFPS">Total Rendered FPS: 0</div>
  <div id="score" style="display: none;">Score: 0</div>

  <script>
    // Mandelbrot iteration function
	  // Disable user interaction on the page
    document.body.addEventListener('mousemove', (e) => e.preventDefault(), { passive: false });
    document.body.addEventListener('mousedown', (e) => e.preventDefault(), { passive: false });
    document.body.addEventListener('mouseup', (e) => e.preventDefault(), { passive: false });
    document.body.addEventListener('keydown', (e) => e.preventDefault(), { passive: false });
    document.body.addEventListener('contextmenu', (e) => e.preventDefault(), { passive: false });
    function f(x, c) {
      return [
        (x[0] * x[0]) - (x[1] * x[1]) + c[0],
        (x[0] * x[1]) + (x[0] * x[1]) + c[1],
      ];
    }

    // Palette generation for GPU intensive color calculation
    function palette(t, a, b, c, d) {
      return [
        a[0] + b[0] * Math.cos(6.28318 * (c[0] * t + d[0])),
        a[1] + b[1] * Math.cos(6.28318 * (c[1] * t + d[1])),
        a[2] + b[2] * Math.cos(6.28318 * (c[2] * t + d[2])),
      ];
    }

    // Calculate the length of a vector for escape condition
    function vectorLength(vector) {
      return Math.sqrt(vector[0] * vector[0] + vector[1] * vector[1]);
    }

    const gpu = new GPU();

    // Add the necessary functions to the GPU kernel
    gpu
      .addFunction(f)
      .addFunction(palette)
      .addFunction(vectorLength);

    // GPU kernel to calculate Mandelbrot set with higher resolution and iterations
    const calculateMandelbrotSet = gpu.createKernel(function(zoomCenter, zoomSize, maxIterations) {
      let x = [0, 0];
      let c = [
        zoomCenter[0] + ((this.thread.x / this.output.x) * 4 - 2) * (zoomSize / 4),
        zoomCenter[1] + ((this.thread.y / this.output.y) * 4 - 2) * (zoomSize / 4)
      ];
      let escaped = false;
      let iterations = 0;
      for (let i = 0; i < maxIterations; i++) {
        iterations = i;
        x = f(x, c);
        if (vectorLength(x) > 2) {
          escaped = true;
          break;
        }
      }
      if (escaped) {
        const pixel = palette(iterations / maxIterations, [0, 0, 0], [.59, .55, .75], [.1, .2, .3], [.75, .75, .75]);
        this.color(pixel[0], pixel[1], pixel[2], 1);
      } else {
        this.color(.85, .99, 1, 1);
      }
    })
      .setGraphical(true)
      .setOutput([1000, 1000]); // Updated resolution to 1200x1200

    // Zoom parameters
    let zoomCenter = [0, 0], // Always zoom at the center
        zoomSize = 4,
        maxIterations = 1000, // High iterations for more complex detail
        zoomFactor = 0.98, // Very fast zoom-in factor
        zoomOutFactor = 1.02, // Very fast zoom-out factor
        stopZooming = false,
        isZoomingIn = true, // Flag to toggle between zooming in and zooming out
        zoomDirectionChangeThreshold = 0.01, // Threshold to switch zoom direction
        stopZoomingSize = 0.1; // Stop zooming when this zoom size is reached

    // FPS variables
    let lastFrameTime = performance.now();
    let frameCount = 0;
    let fps = 0;

    // Total Frames rendered variable
    let totalRenderedFrames = 0;

    // Initial rendering of the Mandelbrot set
    calculateMandelbrotSet(zoomCenter, zoomSize, maxIterations);
    const canvas = calculateMandelbrotSet.canvas;
    document.body.appendChild(canvas);

    // Function to update and render the FPS counters
    function updateFPS() {
      frameCount++;
      const now = performance.now();
      const deltaTime = now - lastFrameTime;

      // Update the FPS counter every second
      if (deltaTime >= 1000) {
        fps = frameCount; // FPS in last second
        frameCount = 0;
        lastFrameTime = now;
      }

      // Increment the total rendered frames
      totalRenderedFrames++;

      // Update the FPS display
      document.getElementById('fpsCounter').innerText = `Current FPS: ${fps}`;
      document.getElementById('totalRenderedFPS').innerText = `Total Rendered Frames: ${totalRenderedFrames}`;
    }

    // Render function that alternates between zooming in and zooming out
    function render() {
      calculateMandelbrotSet(zoomCenter, zoomSize, maxIterations);

      // If zooming in, we reduce zoom size and speed it up
      if (isZoomingIn) {
        zoomSize *= zoomFactor; // Very fast zoom-in speed
        
        // When we reach a certain zoom size, switch to zooming out
        if (zoomSize < stopZoomingSize) {
          isZoomingIn = false;
        }
      } 
      // If zooming out, we increase zoom size and speed it up
      else {
        zoomSize *= zoomOutFactor; // Very fast zoom-out speed

        // When we reach a certain zoom size, switch back to zooming in
        if (zoomSize > 4) {
          isZoomingIn = true;
        }
      }

      // Adjust the number of iterations as we zoom in and out
      if (isZoomingIn) {
        maxIterations -= 5; // Speed up zoom by reducing iterations faster
        if (maxIterations < 50) {
          maxIterations = 50;
        }
      } else {
        maxIterations += 10; // More iterations when zooming out
        if (maxIterations > 3000) { // Cap maxIterations to prevent overloading
          maxIterations = 3000;
        }
      }

      // Update FPS counters
      updateFPS();
    }

    // Set a fixed frame rate using setInterval (not tied to screen refresh)
    const frameRate = 250000; // Target 120 FPS or any value you want for fast zoom
    const interval = 1000 / frameRate; // Interval in milliseconds
    
    const intervalID = setInterval(render, interval); // Start rendering at fixed frame rate

    // Function to stop rendering after a specific amount of time (e.g., 10 seconds)
    const stopTime = 2500; // Stop after 10 seconds (10000ms)
    setTimeout(function() {
      clearInterval(intervalID); // Stop the rendering loop
	  document.write(`<h1>Score (total rendered frames): ${totalRenderedFrames}</h1>`);
     // document.getElementById('fpsCounter').style.display = 'none'; // Hide FPS counter
      //document.getElementById('totalRenderedFPS').style.display = 'none'; // Hide total FPS counter
     // document.getElementById('score').innerText = `Score: ${totalRenderedFrames}`; // Display the score
     // document.getElementById('score').style.display = 'block'; // Show score
    }, stopTime);


  </script>
</body>
</html>