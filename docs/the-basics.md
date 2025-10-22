<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Biometric Demo — Face Recognition + WebAuthn (Fingerprint-like)</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, Arial; margin: 18px; background: #f7f8fb; color: #111; }
    h1 { font-size: 20px; margin-bottom: 6px; }
    .card { background: white; padding:14px; border-radius:10px; box-shadow: 0 6px 18px rgba(20,20,40,0.06); margin-bottom: 14px; }
    video, canvas { border-radius:8px; max-width: 100%; }
    .row { display:flex; gap:12px; align-items:center; flex-wrap:wrap; }
    button { padding:8px 12px; border-radius:8px; border:0; background:#2563eb; color:white; cursor:pointer; }
    button.secondary { background:#475569; }
    small { color:#555; }
    label{display:block;margin-top:8px}
    .status { margin-top:8px; font-weight:600; color:#0f172a; }
    .hint { font-size:13px; color:#4b5563; }
    pre { background:#0b1220;color:#dfefff;padding:10px;border-radius:8px; overflow:auto; }
  </style>
</head>
<body>
  <h1>Biometric Demo — Face recognition + WebAuthn (fingerprint/OS auth)</h1>
  <div class="card">
    <strong>Instructions:</strong>
    <ul>
      <li>Use a secure origin (https or localhost). Allow camera access when prompted.</li>
      <li>First, register a face. Then try Verify Face.</li>
      <li>For fingerprint-like biometric, use "Register WebAuthn" (this will call platform authenticator; you may be prompted for fingerprint/face/Pin depending on device).</li>
    </ul>
  </div>

  <div class="card" id="cameraCard">
    <h2>Camera / Face</h2>
    <div class="row">
      <video id="video" playsinline autoplay width="320" height="240"></video>
      <canvas id="snapshot" width="320" height="240" style="display:none"></canvas>
    </div>

    <div style="margin-top:10px" class="row">
      <button id="startCam">Start Camera</button>
      <button id="capture" class="secondary">Capture & Register Face</button>
      <button id="verifyFace">Verify Face</button>
      <button id="stopCam" class="secondary">Stop Camera</button>
    </div>

    <div class="status" id="faceStatus">Face: <small id="faceStatusText">No face registered</small></div>
    <div class="hint" id="faceHint">Face descriptor stored locally (demo). Distance threshold: 0.6 (lower = more strict).</div>
  </div>

  <div class="card">
    <h2>WebAuthn (Fingerprint / Platform Authenticator)</h2>
    <div class="row">
      <button id="registerWeAuthn">Register WebAuthn (Enroll)</button>
      <button id="authWeAuthn" class="secondary">Authenticate (Login)</button>
      <button id="clearWeAuthn" class="secondary">Clear WebAuthn Data</button>
    </div>
    <div class="status" id="weStatus">WebAuthn: <small id="weStatusText">No credential registered</small></div>
    <div class="hint">This uses navigator.credentials.create / get. Browser + OS may prompt for fingerprint / face / PIN if using platform authenticator.</div>
  </div>

  <div class="card">
    <h2>Debug / Storage</h2>
    <div class="row">
      <button id="showStorage">Show Stored Items</button>
      <button id="clearStorage" class="secondary">Clear All Stored</button>
    </div>
    <pre id="debugOut">{ }</pre>
  </div>

  <!-- face-api.js CDN -->
  <script src="https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/dist/face-api.min.js"></script>

  <script>
    // ====== Utilities
    const $ = id => document.getElementById(id);
    const logDebug = (obj) => { $('debugOut').textContent = JSON.stringify(obj, null, 2); };

    // ====== Camera & Face Recognition
    const video = $('video');
    const canvas = $('snapshot');
    const ctx = canvas.getContext('2d');
    let stream = null;
    let modelsLoaded = false;

    async function loadFaceModels() {
      if (modelsLoaded) return;
      const MODEL_URL = 'https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/weights';
      try {
        // Load tiny face detector + face recognition model
        await faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL);
        await faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL);
        await faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL);
        modelsLoaded = true;
        console.log('Face-api models loaded');
      } catch (e) {
        console.error('Model load failed; check CDN or network', e);
        alert('Failed to load face models. Check network or CDN.');
      }
    }

    async function startCamera() {
      if (stream) return;
      try {
        stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'user' }, audio: false });
        video.srcObject = stream;
        await video.play();
      } catch (e) {
        alert('Could not access camera. Allow camera access and try again.');
        console.error(e);
      }
    }

    function stopCamera() {
      if (!stream) return;
      stream.getTracks().forEach(t => t.stop());
      stream = null;
      video.srcObject = null;
    }

    async function captureFaceAndRegister() {
      await loadFaceModels();
      if (!stream) {
        await startCamera();
        await new Promise(r => setTimeout(r, 600)); // let video warm
      }
      // draw frame to canvas
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const detections = await faceapi.detectSingleFace(canvas, new faceapi.TinyFaceDetectorOptions()).withFaceLandmarks().withFaceDescriptor();
      if (!detections) {
        $('faceStatusText').textContent = 'No face detected — try better lighting and hold steady.';
        return;
      }
      const descriptor = Array.from(detections.descriptor); // convert Float32Array -> Array
      localStorage.setItem('faceDescriptor', JSON.stringify(descriptor));
      $('faceStatusText').textContent = 'Face registered locally.';
      logDebug({ faceDescriptorStored: descriptor.length });
    }

    async function verifyFace() {
      await loadFaceModels();
      const stored = localStorage.getItem('faceDescriptor');
      if (!stored) {
        $('faceStatusText').textContent = 'No face registered. Use "Capture & Register Face".';
        return;
      }
      if (!stream) {
        await startCamera();
        await new Promise(r => setTimeout(r, 600));
      }
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const detection = await faceapi.detectSingleFace(canvas, new faceapi.TinyFaceDetectorOptions()).withFaceLandmarks().withFaceDescriptor();
      if (!detection) {
        $('faceStatusText').textContent = 'No face detected by verify. Try again.';
        return;
      }
      const probe = detection.descriptor;
      const enrolled = new Float32Array(JSON.parse(stored));
      // compute euclidean distance
      let sum = 0;
      for (let i = 0; i < probe.length; i++) {
        const d = probe[i] - enrolled[i];
        sum += d*d;
      }
      const dist = Math.sqrt(sum);
      const threshold = 0.6; // adjustable
      $('faceStatusText').textContent = `Distance: ${dist.toFixed(4)} — ${dist <= threshold ? 'MATCH' : 'NO MATCH'}`;
      logDebug({ distance: dist, threshold });
    }

    // ====== WebAuthn (simple demo)
    function bufferToBase64(b) {
      return btoa(String.fromCharCode(...new Uint8Array(b)));
    }
    function base64ToBuffer(b64) {
      return Uint8Array.from(atob(b64), c => c.charCodeAt(0));
    }

    // Create (register) a credential using WebAuthn
    async function registerWebAuthn() {
      if (!window.PublicKeyCredential) {
        alert('WebAuthn not supported in this browser.');
        return;
      }
      try {
        // In production, server must provide a challenge. Here we create a random one for demo.
        const challenge = new Uint8Array(32);
        crypto.getRandomValues(challenge);

        const publicKey = {
          challenge,
          rp: { name: "Demo RP" },
          user: {
            id: new Uint8Array(16), // in real app: stable user id bytes
            name: "demo-user",
            displayName: "Demo User"
          },
          pubKeyCredParams: [{ type: "public-key", alg: -7 }], // ES256
          authenticatorSelection: { authenticatorAttachment: "platform", userVerification: "preferred" },
          timeout: 60000,
          attestation: "direct"
        };

        const cred = await navigator.credentials.create({ publicKey });
        if (!cred) {
          $('weStatusText').textContent = 'Registration failed or cancelled.';
          return;
        }

        // store credential id and clientData/attestation in localStorage (demo only)
        const rawId = new Uint8Array(cred.rawId);
        localStorage.setItem('webauthn_cred_id', bufferToBase64(rawId));
        // We don't parse attestation; production must send attestationObject to server for verification.
        $('weStatusText').textContent = 'WebAuthn credential registered (demo).';
        logDebug({ webauthn_cred_id: localStorage.getItem('webauthn_cred_id') });
      } catch (err) {
        console.error(err);
        $('weStatusText').textContent = 'Registration error: ' + (err.message || err);
      }
    }

    // Authenticate using stored credential
    async function authWebAuthn() {
      const credB64 = localStorage.getItem('webauthn_cred_id');
      if (!credB64) {
        $('weStatusText').textContent = 'No credential registered. Use "Register WebAuthn".';
        return;
      }
      try {
        const allowList = [{
          id: base64ToBuffer(credB64),
          type: 'public-key',
          transports: ['internal'] // platform
        }];
        const challenge = new Uint8Array(32);
        crypto.getRandomValues(challenge);

        const publicKey = {
          challenge,
          allowCredentials: allowList,
          timeout: 60000,
          userVerification: 'preferred'
        };

        const assertion = await navigator.credentials.get({ publicKey });
        if (!assertion) {
          $('weStatusText').textContent = 'Authentication cancelled or failed.';
          return;
        }
        // In production: send assertion.response (authenticatorData, clientDataJSON, signature) to server for verification.
        $('weStatusText').textContent = 'Authenticated (demo). You may have used fingerprint/face via OS authenticator.';
        logDebug({ auth_assertion: { id: assertion.id, type: assertion.type }});
      } catch (err) {
        console.error(err);
        $('weStatusText').textContent = 'Authentication error: ' + (err.message || err);
      }
    }

    // ====== Storage debug helpers
    function showStored() {
      const stored = {
        faceDescriptor: localStorage.getItem('faceDescriptor') ? '[stored]' : null,
        webauthn_cred_id: localStorage.getItem('webauthn_cred_id') || null,
        keys: Object.keys(localStorage)
      };
      logDebug(stored);
    }
    function clearAll() {
      localStorage.removeItem('faceDescriptor');
      localStorage.removeItem('webauthn_cred_id');
      $('faceStatusText').textContent = 'No face registered';
      $('weStatusText').textContent = 'No credential registered';
      logDebug({ cleared: true });
    }

    // ====== Hook buttons
    $('startCam').addEventListener('click', () => startCamera());
    $('stopCam').addEventListener('click', () => stopCamera());
    $('capture').addEventListener('click', () => captureFaceAndRegister());
    $('verifyFace').addEventListener('click', () => verifyFace());
    $('registerWeAuthn').addEventListener('click', () => registerWebAuthn());
    $('authWeAuthn').addEventListener('click', () => authWebAuthn());
    $('clearWeAuthn').addEventListener('click', () => { localStorage.removeItem('webauthn_cred_id'); $('weStatusText').textContent = 'No credential registered'; logDebug({ webauthnCleared:true });});
    $('showStorage').addEventListener('click', () => showStored());
    $('clearStorage').addEventListener('click', () => clearAll());

    // show initial state
    (function init() {
      if (localStorage.getItem('faceDescriptor')) $('faceStatusText').textContent = 'Face registered locally.';
      if (localStorage.getItem('webauthn_cred_id')) $('weStatusText').textContent = 'WebAuthn credential registered.';
      showStored();
    })();

    // Stop camera on page unload
    window.addEventListener('beforeunload', stopCamera);
  </script>
</body>
</html>
