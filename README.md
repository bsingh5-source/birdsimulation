<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LS Investigation: For the Birds Simulator</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f4f7f6;
            color: #333;
            padding: 20px;
            max-width: 950px;
            margin: 0 auto;
        }
        h1 {
            color: #2c3e50;
            border-bottom: 2px solid #34495e;
            padding-bottom: 10px;
            margin-bottom: 5px;
        }
        p { margin-top: 5px; color: #555; }
        
        /* Layout split: Controls left, Visuals right */
        .container {
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
            margin-top: 20px;
        }
        .left-panel {
            flex: 1;
            min-width: 350px;
        }
        .right-panel {
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            display: flex;
            flex-direction: column;
            align-items: center;
            min-width: 280px;
        }
        
        .setup-box, .results-box {
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }
        label {
            font-weight: bold;
            display: block;
            margin-top: 15px;
            margin-bottom: 5px;
        }
        select, input, button {
            width: 100%;
            padding: 10px;
            font-size: 16px;
            border-radius: 4px;
            border: 1px solid #ccc;
            box-sizing: border-box;
        }
        button {
            background-color: #27ae60;
            color: white;
            font-weight: bold;
            border: none;
            margin-top: 20px;
            cursor: pointer;
            transition: background 0.2s;
        }
        button:hover { background-color: #219653; }
        button:disabled { background-color: #bdc3c7; cursor: not-allowed; }
        
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 15px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 10px;
            text-align: center;
        }
        th { background-color: #ecf0f1; }
        
        .status {
            font-weight: bold;
            font-size: 16px;
            margin-top: 15px;
            padding: 10px;
            border-radius: 4px;
            text-align: center;
        }
        .success { background-color: #d4edda; color: #155724; }
        .fail { background-color: #f8d7da; color: #721c24; }
        
        /* Canvas Styling */
        canvas {
            border: 2px solid #34495e;
            background-color: #eef2f3;
            border-radius: 4px;
        }
    </style>
</head>
<body>

    <h1>LS Investigation — For the Birds Simulator</h1>
    <p>Test engineered solutions, observe the drop animation, and record data in your Student Answer Packet.</p>

    <div class="container">
        <div class="left-panel">
            <div class="setup-box">
                <h3>[Designing Solutions Setup]</h3>
                
                <label for="birdSelect">Select a Bird Species (Table 1):</label>
                <select id="birdSelect" onchange="onSettingsChange()">
                    <option value="chickadee">Black-capped Chickadee (Mass: 12g, Drop Height: 1.0m)</option>
                    <option value="cardinal">Northern Cardinal (Mass: 45g, Drop Height: 1.2m)</option>
                    <option value="robin">American Robin (Mass: 75g, Drop Height: 1.5m)</option>
                </select>

                <div id="controlInfo" style="margin-top:8px; color:#555; font-style:italic; font-size: 14px;">
                    Baseline Control Deformation (No Mesh): 2.5 cm
                </div>

                <label for="meshHeight">Independent Variable - Height of mesh from floor (cm):</label>
                <input type="number" id="meshHeight" min="0" max="25" step="0.5" value="0" oninput="onSettingsChange()">

                <button id="runBtn" onclick="startSimulationPipeline()">Run 3 Trials</button>
            </div>

            <div id="results" class="results-box" style="display: none;">
                <h3>Simulation Data Collected</h3>
                <table>
                    <thead>
                        <tr>
                            <th>Trial 1</th>
                            <th>Trial 2</th>
                            <th>Trial 3</th>
                            <th>Average Deformation</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr>
                            <td id="t1">-</td>
                            <td id="t2">-</td>
                            <td id="t3">-</td>
                            <td id="tAvg" style="font-weight:bold;">-</td>
                        </tr>
                    </tbody>
                </table>
                <div id="statusMessage" class="status"></div>
            </div>
        </div>

        <div class="right-panel">
            <h3 style="margin-top: 0; margin-bottom: 10px;">Drop Zone Visualizer</h3>
            <canvas id="simCanvas" width="260" height="420"></canvas>
            <div style="margin-top: 10px; font-size: 13px; color: #666; text-align: center;">
                Scale: 10px = 1cm mesh height clearance
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById("simCanvas");
        const ctx = canvas.getContext("2d");

        const BIRD_DATA = {
            chickadee: { minMesh: 6.0, maxDef: 2.5, color: "#e67e22", radius: 8 },
            cardinal: { minMesh: 10.0, maxDef: 4.2, color: "#e74c3c", radius: 12 },
            robin: { minMesh: 15.0, maxDef: 5.8, color: "#3498db", radius: 15 }
        };

        // Canvas coordinate constants
        const FLOOR_Y = 380;
        const CEILING_Y = 50;
        const SCALE = 10; // 10 pixels per cm of mesh height

        // Animation state variables
        let animationId = null;
        let birdY = CEILING_Y;
        let birdVelocity = 0;
        let currentTrialIndex = 0;
        let simulatedTrials = [];
        let isAnimating = false;
        let currentMeshStretch = 0;

        // Run initial canvas draw
        onSettingsChange();

        function onSettingsChange() {
            if (isAnimating) return;
            
            const birdKey = document.getElementById("birdSelect").value;
            const bird = BIRD_DATA[birdKey];
            document.getElementById("controlInfo").innerText = `Baseline Control Deformation (No Mesh): ${bird.maxDef} cm`;
            
            let meshHeight = parseFloat(document.getElementById("meshHeight").value);
            if (isNaN(meshHeight) || meshHeight < 0) meshHeight = 0;

            drawCanvas(meshHeight, bird, CEILING_Y, 0);
        }

        function drawCanvas(meshHeight, bird, currentBirdY, currentStretch) {
            // Clear space
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw Foam/Clay Floor
            ctx.fillStyle = "#bdc3c7";
            ctx.fillRect(0, FLOOR_Y, canvas.width, canvas.height - FLOOR_Y);
            ctx.fillStyle = "#7f8c8d";
            ctx.fillRect(0, FLOOR_Y, canvas.width, 4); // floor top line

            // Calculate mesh Y position
            let meshBaseY = FLOOR_Y - (meshHeight * SCALE);

            // Draw Safety Mesh Netting
            if (meshHeight > 0) {
                ctx.strokeStyle = "#2c3e50";
                ctx.lineWidth = 3;
                ctx.beginPath();
                // If stretching from bird impact, draw a dynamic V shape downward
                ctx.moveTo(10, meshBaseY);
                ctx.lineTo(canvas.width / 2, meshBaseY + currentStretch);
                ctx.lineTo(canvas.width - 10, meshBaseY);
                ctx.stroke();
                
                // Labels for mesh
                ctx.fillStyle = "#2c3e50";
                ctx.font = "12px Arial";
                ctx.fillText(`Mesh Netting (${meshHeight} cm)`, 15, meshBaseY - 8);
            }

            // Draw Bird Object
            ctx.fillStyle = bird.color;
            ctx.beginPath();
            ctx.arc(canvas.width / 2, currentBirdY, bird.radius, 0, Math.PI * 2);
            ctx.fill();
            ctx.strokeStyle = "#2c3e50";
            ctx.lineWidth = 1;
            ctx.stroke();

            // Lab Floor label
            ctx.fillStyle =
