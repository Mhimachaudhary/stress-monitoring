# stress-monitoring
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>StressSense AI | Demo Mode</title>
    <style>
        :root { --bg: #0f172a; --accent: #6366f1; --text: #f8fafc; --danger: #ef4444; --safe: #10b981; }
        body { background: var(--bg); color: var(--text); font-family: 'Inter', sans-serif; display: flex; flex-direction: column; align-items: center; padding: 20px; }
        .monitor-box { position: relative; width: 640px; height: 360px; background: #000; border-radius: 12px; overflow: hidden; border: 2px solid #334155; }
        video, canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; transform: scaleX(-1); }
        .dashboard { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; width: 640px; margin-top: 20px; }
        .card { background: #1e293b; padding: 20px; border-radius: 12px; text-align: center; border: 1px solid #334155; position: relative; }
        .val { font-size: 3rem; font-weight: 800; color: var(--accent); display: block; }
        .label { font-size: 0.7rem; color: #94a3b8; text-transform: uppercase; letter-spacing: 1px; }
        .mode-tag { position: absolute; top: 8px; right: 8px; font-size: 10px; padding: 2px 6px; border-radius: 4px; background: #334155; }
    </style>
</head>
<body>

    <h2 style="margin-bottom: 10px;">StressSense <span style="color:var(--accent)">Dashboard</span></h2>
    
    <div class="monitor-box">
        <video id="v" autoplay playsinline></video>
        <canvas id="c"></canvas>
    </div>

    <div class="dashboard">
        <div class="card">
            <div id="mode" class="mode-tag">WAITING</div>
            <span class="label">System Status</span>
            <span id="status" class="val" style="font-size: 1.5rem; color: #94a3b8;">OFFLINE</span>
        </div>
        <div class="card">
            <span class="label">Calculated Stress</span>
            <span id="stress" class="val">0%</span>
        </div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-core"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-converter"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs-backend-webgl"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/face-landmarks-detection"></script>

    <script>
        const ui = {
            stress: document.getElementById('stress'),
            status: document.getElementById('status'),
            mode: document.getElementById('mode')
        };

        async function start() {
            try {
                // 1. Initialize Camera
                const stream = await navigator.mediaDevices.getUserMedia({ video: true });
                document.getElementById('v').srcObject = stream;
                
                ui.status.innerText = "CAMERA ON";
                ui.status.style.color = "#3b82f6";

                // 2. Load Model (Optional: Script will continue even if this fails)
                const detector = await faceLandmarksDetection.createDetector(
                    faceLandmarksDetection.SupportedModels.MediaPipeFaceMesh, 
                    { runtime: 'tfjs', refineLandmarks: true }
                ).catch(e => console.log("AI Loading failed, using simulation."));

                beginMonitoring(detector);
            } catch (e) {
                console.error(e);
                ui.status.innerText = "CAM ERROR";
                beginMonitoring(null); // Force simulation if camera fails
            }
        }

        function beginMonitoring(detector) {
            const video = document.getElementById('v');
            const canvas = document.getElementById('c');
            const ctx = canvas.getContext('2d');

            async function loop() {
                let currentScore = 0;
                let foundFace = false;

                // Try Live AI Detection
                if (detector) {
                    const faces = await detector.estimateFaces(video);
                    if (faces.length > 0) {
                        foundFace = true;
                        ui.mode.innerText = "AI LIVE";
                        ui.mode.style.background = "#1e3a8a";
                        
                        // Calculate actual stress from face landmarks
                        const pts = faces[0].keypoints;
                        const brow = Math.hypot(pts[55].x - pts[285].x, pts[55].y - pts[285].y);
                        currentScore = brow < 45 ? 65 : 12;

                        // Draw Mesh
                        canvas.width = video.videoWidth; canvas.height = video.videoHeight;
                        ctx.fillStyle = "rgba(99, 102, 241, 0.5)";
                        pts.forEach((p, i) => { if(i%15==0){ ctx.beginPath(); ctx.arc(p.x,p.y,1,0,7); ctx.fill(); }});
                    }
                }

                // FALLBACK: If no face or no AI, use Fake Values
                if (!foundFace) {
                    ui.mode.innerText = "SIMULATED";
                    ui.mode.style.background = "#92400e";
                    ui.status.innerText = "RUNNING";
                    ui.status.style.color = "var(--safe)";
                    
                    // Generate fluctuating fake values: 23, 40, 60, etc.
                    const seed = [23, 40, 60, 35, 72, 18];
                    currentScore = seed[Math.floor(Date.now() / 2000) % seed.length];
                    currentScore += Math.floor(Math.random() * 5); // Add slight jitter
                }

                // Update UI
                ui.stress.innerText = currentScore + "%";
                ui.stress.style.color = currentScore > 50 ? "var(--danger)" : "var(--safe)";

                requestAnimationFrame(loop);
            }
            loop();
        }

        start();
    </script>
</body>
</html>
