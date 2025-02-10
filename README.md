<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>INCU Analyzer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mqtt/4.3.7/mqtt.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: Arial, sans-serif;
        }

        body {
            padding: 20px;
            background: #f0f2f5;
        }

        .title {
            text-align: center;
            font-size: 2.5em;
            margin-bottom: 30px;
            color: #2c3e50;
        }

        .control-buttons {
            display: flex;
            justify-content: center;
            gap: 20px;
            margin-bottom: 30px;
        }

        .control-btn {
            padding: 15px 30px;
            font-size: 1.1em;
            border: none;
            border-radius: 8px;
            background: #3498db;
            color: white;
            cursor: pointer;
            transition: transform 0.3s, background 0.3s;
        }

        .control-btn:hover {
            transform: scale(1.05);
        }

        .control-btn.active {
            background: #e74c3c;
        }

        .input-container {
            display: flex;
            justify-content: center;
            gap: 20px;
            margin-bottom: 30px;
        }

        .input-group {
            display: flex;
            flex-direction: column;
            gap: 5px;
        }

        .input-group label {
            font-weight: bold;
            color: #2c3e50;
        }

        .input-group input {
            padding: 10px;
            border: 1px solid #bdc3c7;
            border-radius: 5px;
            font-size: 1em;
        }

        .timer-display {
            text-align: center;
            font-size: 2em;
            margin-bottom: 30px;
            color: #2c3e50;
        }

        .sensor-grid {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 20px;
            margin-bottom: 30px;
            max-width: 900px;
            margin-left: auto;
            margin-right: auto;
        }

        .sensor-box {
            background: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            text-align: center;
            transition: transform 0.3s;
        }

        .sensor-box:hover {
            transform: translateY(-5px);
        }

        .sensor-box h3 {
            margin-bottom: 10px;
            color: #2c3e50;
        }

        .sensor-value {
            font-size: 1.5em;
            color: #3498db;
        }

        .data-table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
            background: white;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }

        .data-table th, .data-table td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }

        .data-table th {
            background: #3498db;
            color: white;
        }

        .data-table tr:hover {
            background: #f5f6fa;
        }
    </style>
</head>
<body>
    <h1 class="title">INCU ANALYZER</h1>

    <div class="control-buttons">
        <button id="saveBtn" class="control-btn">Play Saving Data</button>
        <button id="resetBtn" class="control-btn">Reset Data</button>
        <button id="exportBtn" class="control-btn">Export Data</button>
    </div>

    <div class="input-container">
        <div class="input-group">
            <label>Interval (seconds)</label>
            <input type="number" id="intervalInput" min="1" value="2">
        </div>
        <div class="input-group">
            <label>Timer (HH:MM:SS)</label>
            <input type="text" id="timerInput" placeholder="00:00:00" pattern="[0-9]{2}:[0-9]{2}:[0-9]{2}">
        </div>
    </div>

    <div class="timer-display" id="timerDisplay">00:00:00</div>

    <div class="sensor-grid">
        <div class="sensor-box">
            <h3>T1</h3>
            <div class="sensor-value" id="t1Value">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>T2</h3>
            <div class="sensor-value" id="t2Value">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>T3</h3>
            <div class="sensor-value" id="t3Value">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>T4</h3>
            <div class="sensor-value" id="t4Value">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>T5</h3>
            <div class="sensor-value" id="t5Value">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>TM</h3>
            <div class="sensor-value" id="tmValue">0.0 °C</div>
        </div>
        <div class="sensor-box">
            <h3>Flow</h3>
            <div class="sensor-value" id="flowValue">0.0 m/s</div>
        </div>
        <div class="sensor-box">
            <h3>Noise</h3>
            <div class="sensor-value" id="noiseValue">0.0 dB</div>
        </div>
        <div class="sensor-box">
            <h3>RH</h3>
            <div class="sensor-value" id="rhValue">0.0 %</div>
        </div>
    </div>

    <table class="data-table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Time</th>
                <th>T1 (°C)</th>
                <th>T2 (°C)</th>
                <th>T3 (°C)</th>
                <th>T4 (°C)</th>
                <th>T5 (°C)</th>
                <th>TM (°C)</th>
                <th>Flow (m/s)</th>
                <th>Noise (dB)</th>
                <th>RH (%)</th>
            </tr>
        </thead>
        <tbody id="dataTableBody"></tbody>
    </table>

    <script>
        // MQTT Configuration
        const client = mqtt.connect('ws://your-mqtt-broker:port');
        const topic = 'incu/sensors';
        let isRecording = false;
        let timerInterval;
        let sensorInterval;
        let remainingTime;
        let tableData = [];

        client.on('connect', () => {
            console.log('Connected to MQTT broker');
            client.subscribe(topic);
        });

        client.on('message', (topic, message) => {
            if (isRecording) {
                const data = JSON.parse(message.toString());
                updateSensorValues(data);
                if (sensorInterval) {
                    addTableRow(data);
                }
            }
        });

        function updateSensorValues(data) {
            document.getElementById('t1Value').textContent = `${data.t1.toFixed(1)} °C`;
            document.getElementById('t2Value').textContent = `${data.t2.toFixed(1)} °C`;
            document.getElementById('t3Value').textContent = `${data.t3.toFixed(1)} °C`;
            document.getElementById('t4Value').textContent = `${data.t4.toFixed(1)} °C`;
            document.getElementById('t5Value').textContent = `${data.t5.toFixed(1)} °C`;
            document.getElementById('tmValue').textContent = `${data.tm.toFixed(1)} °C`;
            document.getElementById('flowValue').textContent = `${data.flow.toFixed(1)} m/s`;
            document.getElementById('noiseValue').textContent = `${data.noise.toFixed(1)} dB`;
            document.getElementById('rhValue').textContent = `${data.rh.toFixed(1)} %`;
        }

        function addTableRow(data) {
            const now = new Date();
            const row = {
                date: now.toLocaleDateString(),
                time: now.toLocaleTimeString(),
                ...data
            };
            tableData.push(row);
            
            const tr = document.createElement('tr');
            tr.innerHTML = `
                <td>${row.date}</td>
                <td>${row.time}</td>
                <td>${row.t1.toFixed(1)}</td>
                <td>${row.t2.toFixed(1)}</td>
                <td>${row.t3.toFixed(1)}</td>
                <td>${row.t4.toFixed(1)}</td>
                <td>${row.t5.toFixed(1)}</td>
                <td>${row.tm.toFixed(1)}</td>
                <td>${row.flow.toFixed(1)}</td>
                <td>${row.noise.toFixed(1)}</td>
                <td>${row.rh.toFixed(1)}</td>
            `;
            document.getElementById('dataTableBody').appendChild(tr);
        }

        document.getElementById('saveBtn').addEventListener('click', function() {
            if (!isRecording) {
                // Start recording
                const timerValue = document.getElementById('timerInput').value;
                const [hours, minutes, seconds] = timerValue.split(':').map(Number);
                remainingTime = (hours * 3600 + minutes * 60 + seconds) * 1000;
                
                if (remainingTime <= 0) {
                    alert('Please set a valid timer');
                    return;
                }

                const interval = parseInt(document.getElementById('intervalInput').value) * 1000;
                if (interval < 1000) {
                    alert('Interval must be at least 1 second');
                    return;
                }

                this.textContent = 'Stop Saving Data';
                this.classList.add('active');
                isRecording = true;

                // Start timer
                const startTime = Date.now();
                timerInterval = setInterval(() => {
                    const elapsed = Date.now() - startTime;
                    remainingTime = Math.max(0, remainingTime - 1000);
                    
                    const h = Math.floor(remainingTime / 3600000);
                    const m = Math.floor((remainingTime % 3600000) / 60000);
                    const s = Math.floor((remainingTime % 60000) / 1000);
                    
                    document.getElementById('timerDisplay').textContent = 
                        `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;

                    if (remainingTime <= 0) {
                        stopRecording();
                    }
                }, 1000);

                // Start sensor reading
                sensorInterval = setInterval(() => {
                    client.publish('incu/request', 'get_data');
                }, interval);

            } else {
                stopRecording();
            }
        });

        function stopRecording() {
            isRecording = false;
            clearInterval(timerInterval);
            clearInterval(sensorInterval);
            document.getElementById('saveBtn').textContent = 'Play Saving Data';
            document.getElementById('saveBtn').classList.remove('active');
            document.getElementById('timerDisplay').textContent = '00:00:00';
        }

        document.getElementById('resetBtn').addEventListener('click', function() {
            tableData = [];
            document.getElementById('dataTableBody').innerHTML = '';
            // Reset timer display
            document.getElementById('timerDisplay').textContent = '00:00:00';
            // Clear any running intervals
            clearInterval(timerInterval);
            clearInterval(sensorInterval);
            // Reset button state
            document.getElementById('saveBtn').textContent = 'Play Saving Data';
            document.getElementById('saveBtn').classList.remove('active');
            isRecording = false;
        });

        document.getElementById('exportBtn').addEventListener('click', async function() {
            try {
                const response = await fetch('http://localhost:5000/export', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(tableData)
                });

                if (response.ok) {
                    const blob = await response.blob();
                    const url = window.URL.createObjectURL(blob);
                    const a = document.createElement('a');
                    a.href = url;
                    a.download = 'incu_report.xlsx';
                    a.click();
                    window.URL.revokeObjectURL(url);
                    alert('Data exported successfully!');
                } else {
                    alert('Failed to export data');
                }
            } catch (error) {
                console.error('Export error:', error);
                alert('Error during export');
            }
        });
    </script>
</body>
</html>
