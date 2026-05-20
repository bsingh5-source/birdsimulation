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
        
        .container {
            display: flex;
            gap: 20px;
            flex-wrap: wrap;
            margin-top: 20px;
        }
        .left-panel {
            flex: 1;
            min-width: 380px;
        }
        .right-panel {
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            display: flex;
            flex-direction: column;
            align-items: center;
            min-width: 320px;
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
            <canvas id="simCanvas" width="340" height="440"></canvas>
            <div style="margin-top: 10px; font-size: 13px; color: #666; text-align: center;">
                Use the Measuring Tape (left) to verify mesh height clearance window.
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById("simCanvas");
        const ctx = canvas.getContext("2d");

        const BIRD_DATA = {
            chickadee: { minMesh: 6.0, maxDef: 2.5, color: "#e67e22", radius: 9 },
            cardinal: { minMesh: 10.0, maxDef: 4.2, color: "#e74c3c", radius: 13 },
            robin: { minMesh: 15.0, maxDef: 5.8, color: "#3498db", radius: 16 }
        };

        // Canvas Layout Coordinates
        const FLOOR_Y = 390;
        const CEILING_Y = 40;
        const SCALE = 13; // 13 pixels = 1 cm of vertical mesh clearance height
        const RULER_X = 45; // X position alignment for measuring tape

        let animationId = null;
        let birdY = CEILING_Y;
        let birdVelocity = 0;
        let currentTrialIndex = 0;
        let simulatedTrials = [];
        let isAnimating = false;
        let currentMeshStretch = 0;

        // Render structural environment on initial load
        onSettingsChange();

        function onSettingsChange() {
            if (isAnimating) return;
            
            const birdKey = document.getElementById("birdSelect").value;
            const bird = BIRD_DATA[birdKey];
            document.getElementById("controlInfo").innerText = `Baseline Control Deformation (No Mesh): ${bird.maxDef} cm`;
            
            let meshHeight = parseFloat(document.getElementById("meshHeight").value);
            if (isNaN(meshHeight) || meshHeight < 0) meshHeight = 0;
            if (meshHeight > 25) meshHeight = 25;

            drawCanvas(meshHeight, bird, CEILING_Y, 0);
        }

        function drawCanvas(meshHeight, bird, currentBirdY, currentStretch) {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // 1. DRAW WINDOW IMPACT FLOOR (The structural impact plate)
            ctx.fillStyle = "#a6b8c7";
            ctx.fillRect(0, FLOOR_Y, canvas.width, canvas.height - FLOOR_Y);
            ctx.fillStyle = "#2c3e50";
            ctx.fillRect(0, FLOOR_Y, canvas.width, 5); // Glass surface line

            // Glass accent stripes
            ctx.strokeStyle = "rgba(255,255,255,0.6)";
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(100, FLOOR_Y + 15); ctx.lineTo(130, FLOOR_Y + 35);
            ctx.moveTo(220, FLOOR_Y + 15); ctx.lineTo(250, FLOOR_Y + 35);
            ctx.stroke();

            // 2. DRAW VERTICAL MEASURING TAPE (Ruler on left margin)
            ctx.strokeStyle = "#7f8c8d";
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(RULER_X, CEILING_Y);
            ctx.lineTo(RULER_X, FLOOR_Y);
            ctx.stroke();

            for (let cm = 0; cm <= 25; cm++) {
                let yTick = FLOOR_Y - (cm * SCALE);
                if (yTick < CEILING_Y - 5) break;

                ctx.beginPath();
                if (cm % 5 === 0) {
                    // Major Ticks (5cm intervals)
                    ctx.moveTo(RULER_X - 18, yTick);
                    ctx.lineTo(RULER_X, yTick);
                    ctx.strokeStyle = "#2c3e50";
                    ctx.lineWidth = 2;
                    ctx.stroke();
                    
                    ctx.fillStyle = "#2c3e50";
                    ctx.font = "bold 11px Arial";
                    ctx.fillText(cm + " cm", RULER_X - 42, yTick + 4);
                } else {
                    // Minor Ticks (1cm intervals)
                    ctx.moveTo(RULER_X - 8, yTick);
                    ctx.lineTo(RULER_X, yTick);
                    ctx.strokeStyle = "#95a5a6";
                    ctx.lineWidth = 1;
                    ctx.stroke();
                }
            }

            // 3. DRAW SAFETY MESH WINDOW PROTECTION
            let meshBaseY = FLOOR_Y - (meshHeight * SCALE);
            if (meshHeight > 0) {
                let midX = (canvas.width + RULER_X) / 2;
                let currentMidY = meshBaseY + currentStretch;

                ctx.strokeStyle = "#d35400";
                ctx.lineWidth = 3;
                ctx.beginPath();
                ctx.moveTo(RULER_X + 10, meshBaseY);
                ctx.lineTo(midX, currentMidY);
                ctx.lineTo(canvas.width - 15, meshBaseY);
                ctx.stroke();

                // Generate Net Weave Crosshatch Look
                ctx.strokeStyle = "rgba(211, 84, 0, 0.4)";
                ctx.lineWidth = 1;
                for (let offset = -20; offset <= 20; offset += 8) {
                    ctx.beginPath();
                    ctx.moveTo(RULER_X + 10, meshBaseY + offset);
                    ctx.lineTo(midX, currentMidY + offset);
                    ctx.lineTo(canvas.width - 15, meshBaseY + offset);
                    ctx.stroke();
                }
                
                ctx.fillStyle = "#d35400";
                ctx.font = "bold 11px Arial";
                ctx.fillText(`Mesh Netting Setup`, canvas.width - 120, meshBaseY - 8);
            }

            // 4. DRAW BIRD OBJECT (Ball representing the organism)
            let birdX = (canvas.width + RULER_X) / 2;
            ctx.fillStyle = bird.color;
            ctx.beginPath();
            ctx.arc(birdX, currentBirdY, bird.radius, 0, Math.PI * 2);
            ctx.fill();
            ctx.strokeStyle = "#2c3e50";
            ctx.lineWidth = 2;
            ctx.stroke();

            // Minimalist bird face features to identify direction
            ctx.fillStyle = "#fff";
            ctx.beginPath();
            ctx.arc(birdX + (bird.radius * 0.3), currentBirdY - (bird.radius * 0.2), bird.radius * 0.2, 0, Math.PI * 2);
            ctx.fill();
            ctx.fillStyle = "#000";
            ctx.beginPath();
            ctx.arc(birdX + (bird.radius * 0.3), currentBirdY - (bird.radius * 0.2), bird.radius * 0.08, 0, Math.PI * 2);
            ctx.fill();

            // Label Floor Area
            ctx.fillStyle = "#2c3e50";
            ctx.font = "bold 12px Arial";
            ctx.fillText("WINDOW / FLOOR SURFACE", RULER_X + 40, FLOOR_Y + 25);
        }

        function calculateDeformationData(meshHeight, bird) {
            let minNeeded = bird.minMesh;
            let maxDef = bird.maxDef;

            if (meshHeight >= minNeeded) {
                return 0.0;
            } else {
                let fractionOfImpact = 1.0 - (meshHeight / minNeeded);
                let baseDef = maxDef * fractionOfImpact;
                let variances = [-0.2, -0.1, 0.0, 0.1, 0.2];
                let randomVariance = variances[Math.floor(Math.random() * variances.length)];
                return Math.max(0, Math.round((baseDef + randomVariance) * 10) / 10);
            }
        }

        function startSimulationPipeline() {
            let meshHeight = parseFloat(document.getElementById("meshHeight").value);
            if (isNaN(meshHeight) || meshHeight < 0) {
                alert("Please enter a valid height (0 or greater).");
                return;
            }
            if (meshHeight > 25) meshHeight = 25;

            const birdKey = document.getElementById("birdSelect").value;
            const bird = BIRD_DATA[birdKey];

            isAnimating = true;
            document.getElementById("runBtn").disabled = true;
            document.getElementById("birdSelect").disabled = true;
            document.getElementById("meshHeight").disabled = true;
            document.getElementById("results").style.display = "none";

            simulatedTrials = [
                calculateDeformationData(meshHeight, bird),
                calculateDeformationData(meshHeight, bird),
                calculateDeformationData(meshHeight, bird)
            ];
            
            currentTrialIndex = 0;
            executeSingleDropAnimation(meshHeight, bird);
        }

        function executeSingleDropAnimation(meshHeight, bird) {
            birdY = CEILING_Y;
            birdVelocity = 1; 
            currentMeshStretch = 0;
            let hasHitMesh = false;
            let expectedDeformation = simulatedTrials[currentTrialIndex];

            function animate() {
                let meshBaseY = FLOOR_Y - (meshHeight * SCALE);

                if (!hasHitMesh) {
                    birdY += birdVelocity;
                    birdVelocity += 0.5; // Acceleration constant

                    if (meshHeight > 0 && birdY >= meshBaseY) {
                        hasHitMesh = true;
                    } else if (birdY >= FLOOR_Y - bird.radius) {
                        birdY = FLOOR_Y - bird.radius;
                        endTrial();
                        return;
                    }
                } else {
                    birdY += birdVelocity;
                    currentMeshStretch = birdY - meshBaseY;

                    if (expectedDeformation === 0) {
                        birdVelocity *= 0.65; // Net dampens impact velocity completely
                        if (birdVelocity < 0.4) {
                            birdVelocity = 0;
                            setTimeout(endTrial, 400);
                            return;
                        }
                    } else {
                        // Net fails to arrest velocity, stretching down into floor impact
                        if (birdY >= FLOOR_Y - bird.radius) {
                            birdY = FLOOR_Y - bird.radius;
                            currentMeshStretch = FLOOR_Y - meshBaseY;
                            endTrial();
                            return;
                        }
                    }
                }

                drawCanvas(meshHeight, bird, birdY, currentMeshStretch);
                animationId = requestAnimationFrame(animate);
            }

            function endTrial() {
                cancelAnimationFrame(animationId);
                currentTrialIndex++;
                
                if (currentTrialIndex < 3) {
                    setTimeout(() => {
                        executeSingleDropAnimation(meshHeight, bird);
                    }, 600);
                } else {
                    finalizePipeline();
                }
            }

            animate();
        }

        function finalizePipeline() {
            isAnimating = false;
            document.getElementById("runBtn").disabled = false;
            document.getElementById("birdSelect").disabled = false;
            document.getElementById("meshHeight").disabled = false;

            let t1 = simulatedTrials[0];
            let t2 = simulatedTrials[1];
            let t3 = simulatedTrials[2];
            let avg = Math.round(((t1 + t2 + t3) / 3) * 10) / 10;

            document.getElementById("t1").innerText = t1 + " cm";
            document.getElementById("t2").innerText = t2 + " cm";
            document.getElementById("t3").innerText = t3 + " cm";
            document.getElementById("tAvg").innerText = avg + " cm";

            const statusDiv = document.getElementById("statusMessage");
            if (avg === 0) {
                statusDiv.innerText = "SUCCESS: Mesh height successfully preserved bird safety clearance window.";
                statusDiv.className = "status success";
            } else {
                statusDiv.innerText = "CRITICAL MORTALITY: Insufficient deceleration height. Floor impact noted.";
                statusDiv.className = "status fail";
            }

            document.getElementById("results").style.display = "block";
            onSettingsChange();
        }
    </script>
</body>
</html>
