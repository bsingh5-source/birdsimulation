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
            max-width: 800px;
            margin: 0 auto;
        }
        h1 {
            color: #2c3e50;
            border-bottom: 2px solid #34495e;
            padding-bottom: 10px;
        }
        .setup-box {
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
        button:hover {
            background-color: #219653;
        }
        .results-box {
            background-color: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            display: none;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 15px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 12px;
            text-align: center;
        }
        th {
            background-color: #ecf0f1;
        }
        .status {
            font-weight: bold;
            font-size: 18px;
            margin-top: 15px;
            padding: 10px;
            border-radius: 4px;
            text-align: center;
        }
        .success { background-color: #d4edda; color: #155724; }
        .fail { background-color: #f8d7da; color: #721c24; }
    </style>
</head>
<body>

    <h1>LS Investigation — For the Birds Simulator</h1>
    <p>Use this simulator to test your engineered solutions and collect data for your Student Answer Packet.</p>

    <div class="setup-box">
        <h3>[Designing Solutions Setup]</h3>
        
        <label for="birdSelect">Select a Bird Species (Table 1):</label>
        <select id="birdSelect" onchange="updateControlInfo()">
            <option value="chickadee">Black-capped Chickadee (Mass: 12g, Drop Height: 1.0m)</option>
            <option value="cardinal">Northern Cardinal (Mass: 45g, Drop Height: 1.2m)</option>
            <option value="robin">American Robin (Mass: 75g, Drop Height: 1.5m)</option>
        </select>

        <div id="controlInfo" style="margin-top:10px; color:#555; font-style:italic;">
            Baseline Control Deformation (No Mesh): 2.5 cm
        </div>

        <label for="meshHeight">Independent Variable - Height of mesh from floor (cm):</label>
        <input type="number" id="meshHeight" min="0" step="0.5" placeholder="e.g., 5.0">

        <button onclick="runSimulation()">Run 3 Trials</button>
    </div>

    <div id="results" class="results-box">
        <h3>Simulation Results</h3>
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

    <script>
        const BIRD_DATA = {
            chickadee: { minMesh: 6.0, maxDef: 2.5 },
            cardinal: { minMesh: 10.0, maxDef: 4.2 },
            robin: { minMesh: 15.0, maxDef: 5.8 }
        };

        function updateControlInfo() {
            const bird = document.getElementById("birdSelect").value;
            document.getElementById("controlInfo").innerText = `Baseline Control Deformation (No Mesh): ${BIRD_DATA[bird].maxDef} cm`;
        }

        function calculateDeformation(meshHeight, bird) {
            let minNeeded = bird.minMesh;
            let maxDef = bird.maxDef;

            if (meshHeight >= minNeeded) {
                return 0.0;
            } else {
                let fractionOfImpact = 1.0 - (meshHeight / minNeeded);
                let baseDef = maxDef * fractionOfImpact;
                
                // Introduce slight simulation variance (+/- 0.1 or 0.2)
                let variances = [-0.2, -0.1, 0.0, 0.1, 0.2];
                let randomVariance = variances[Math.floor(Math.random() * variances.length)];
                let finalDef = baseDef + randomVariance;
                
                return Math.max(0, Math.round(finalDef * 10) / 10);
            }
        }

        function runSimulation() {
            const birdKey = document.getElementById("birdSelect").value;
            const meshHeight = parseFloat(document.getElementById("meshHeight").value);

            if (isNaN(meshHeight) || meshHeight < 0) {
                alert("Please enter a valid height (0 or greater).");
                return;
            }

            const bird = BIRD_DATA[birdKey];
            
            // Generate 3 trials
            let t1 = calculateDeformation(meshHeight, bird);
            let t2 = calculateDeformation(meshHeight, bird);
            let t3 = calculateDeformation(meshHeight, bird);
            let avg = Math.round(((t1 + t2 + t3) / 3) * 10) / 10;

            // Display in Table
            document.getElementById("t1").innerText = t1 + " cm";
            document.getElementById("t2").innerText = t2 + " cm";
            document.getElementById("t3").innerText = t3 + " cm";
            document.getElementById("tAvg").innerText = avg + " cm";

            // Update Safety Status Message
            const statusDiv = document.getElementById("statusMessage");
            if (avg === 0) {
                statusDiv.innerText = "SUCCESS: Bird survived! No mortality detected.";
                statusDiv.className = "status success";
            } else {
                statusDiv.innerText = "CAUTION: Bird mortality occurred due to floor impact.";
                statusDiv.className = "status fail";
            }

            document.getElementById("results").style.display = "block";
        }
    </script>
</body>
</html>
