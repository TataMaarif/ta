<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>K3 Lab MQTT Dashboard</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.0.1/mqttws31.min.js" type="text/javascript"></script>
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;500;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --safe: #00fa9a; --danger: #ff4b4b; --circle-bg: rgba(255, 255, 255, 0.1);
        }
        body {
            font-family: 'Poppins', sans-serif;
            background: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
            color: white; text-align: center; margin: 0; padding: 40px 20px;
        }
        #statusBanner {
            display: inline-block; padding: 12px 40px; border-radius: 30px;
            font-weight: 700; background: rgba(0, 250, 154, 0.15);
            color: var(--safe); border: 1px solid rgba(0, 250, 154, 0.3);
            margin-bottom: 40px; transition: all 0.4s;
        }
        #statusBanner.bahaya { 
            background: rgba(255, 75, 75, 0.15); color: var(--danger);
            border: 1px solid rgba(255, 75, 75, 0.3); animation: pulse 1.5s infinite; 
        }
        @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } }
        .grid { display: flex; flex-wrap: wrap; justify-content: center; gap: 25px; }
        .card {
            background: rgba(255, 255, 255, 0.05); backdrop-filter: blur(15px);
            padding: 30px; border-radius: 20px; width: 200px; box-shadow: 0 8px 32px rgba(0,0,0,0.3);
        }
        .gauge-container { position: relative; width: 130px; height: 130px; margin: 0 auto; }
        svg { width: 100%; height: 100%; transform: rotate(-90deg); }
        .circle-bg { fill: none; stroke: var(--circle-bg); stroke-width: 8; }
        .circle-value {
            fill: none; stroke-width: 8; stroke-linecap: round;
            stroke-dasharray: 282.7; stroke-dashoffset: 282.7;
            transition: stroke-dashoffset 0.8s ease-out;
        }
        #svg-suhu { stroke: #ff9e80; } #svg-gas { stroke: #b388ff; } #svg-bising { stroke: #69f0ae; }
        .gauge-text { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-size: 1.8em; font-weight: 700; }
        .unit { font-size: 0.4em; color: #ccc; display: block; }
    </style>
</head>
<body>

    <h1>Monitoring K3 Lab (MQTT)</h1>
    <div id="statusBanner">STATUS: MENGHUBUNGKAN...</div>

    <div class="grid">
        <div class="card">
            <h3>Suhu</h3>
            <div class="gauge-container">
                <svg viewBox="0 0 100 100">
                    <circle class="circle-bg" cx="50" cy="50" r="45"></circle>
                    <circle id="svg-suhu" class="circle-value" cx="50" cy="50" r="45"></circle>
                </svg>
                <div class="gauge-text"><span id="valSuhu">0</span><span class="unit">°C</span></div>
            </div>
        </div>

        <div class="card">
            <h3>Gas (VOC)</h3>
            <div class="gauge-container">
                <svg viewBox="0 0 100 100">
                    <circle class="circle-bg" cx="50" cy="50" r="45"></circle>
                    <circle id="svg-gas" class="circle-value" cx="50" cy="50" r="45"></circle>
                </svg>
                <div class="gauge-text"><span id="valGas">0</span><span class="unit">AQI</span></div>
            </div>
        </div>

        <div class="card">
            <h3>Kebisingan</h3>
            <div class="gauge-container">
                <svg viewBox="0 0 100 100">
                    <circle class="circle-bg" cx="50" cy="50" r="45"></circle>
                    <circle id="svg-bising" class="circle-value" cx="50" cy="50" r="45"></circle>
                </svg>
                <div class="gauge-text"><span id="valBising">0</span><span class="unit">dB</span></div>
            </div>
        </div>
    </div>

    <script>
        // PENGATURAN MQTT
        const MQTT_HOST = "broker.hivemq.com";
        const MQTT_PORT = 8000; // Port Websocket untuk Browser
        const CLIENT_ID = "Web_K3_Client_" + Math.random().toString(16).substr(2, 8);

        const client = new Paho.MQTT.Client(MQTT_HOST, MQTT_PORT, CLIENT_ID);

        client.onConnectionLost = onConnectionLost;
        client.onMessageArrived = onMessageArrived;

        client.connect({ onSuccess: onConnect });

        function onConnect() {
            console.log("Connected to MQTT Broker");
            document.getElementById('statusBanner').innerText = "STATUS: AMAN ✓";
            // Subscribe ke topik yang sama dengan ESP32
            client.subscribe("kampus/k3lab/suhu");
            client.subscribe("kampus/k3lab/gas");
            client.subscribe("kampus/k3lab/bising");
            client.subscribe("kampus/k3lab/status");
        }

        function onConnectionLost(responseObject) {
            document.getElementById('statusBanner').innerText = "STATUS: TERPUTUS";
        }

        function onMessageArrived(message) {
            const topic = message.destinationName;
            const payload = message.payloadString;

            if (topic === "kampus/k3lab/suhu") updateUI('valSuhu', 'svg-suhu', payload, 50);
            if (topic === "kampus/k3lab/gas") updateUI('valGas', 'svg-gas', payload, 500);
            if (topic === "kampus/k3lab/bising") updateUI('valBising', 'svg-bising', payload, 120);
            if (topic === "kampus/k3lab/status") {
                const banner = document.getElementById('statusBanner');
                if (payload === "BAHAYA") {
                    banner.innerText = "⚠️ STATUS: BAHAYA ⚠️";
                    banner.className = "bahaya";
                } else {
                    banner.innerText = "STATUS: AMAN ✓";
                    banner.className = "";
                }
            }
        }

        function updateUI(textId, svgId, value, max) {
            document.getElementById(textId).innerText = parseFloat(value).toFixed(1);
            let percent = value / max;
            let offset = 282.7 - (percent * 282.7);
            document.getElementById(svgId).style.strokeDashoffset = offset;
        }
    </script>
</body>
</html>
