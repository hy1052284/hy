# hy



      const [r,g,b] = hexToRgb01(m.color);
      colors[i*3+0] = r; colors[i*3+1] = g; colors[i*3+2] = b;
    }

    const geom = new THREE.BufferGeometry();
    geom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
    geom.setAttribute("color", new THREE.BufferAttribute(colors, 3));

    const mat = new THREE.PointsMaterial({
      size: 0.035,
      vertexColors: true,
      transparent: true,
      opacity: 0.95,
      depthWrite: false
    });

    const cloud = new THREE.Points(geom, mat);
    scene.add(cloud);

    // ---------- Modes / Templates ----------
    const MODES = ["DATA", "HEART", "SATURN", "FIREWORKS"];
    let modeIndex = 0;

    function setMode(i){
      modeIndex = (i + MODES.length) % MODES.length;
      document.getElementById("modeLine").innerHTML = "<b>Mode:</b> " + MODES[modeIndex];
      buildTargetPositions(MODES[modeIndex]);
    }

    const targetPositions = new Float32Array(POINTS * 3);

    function buildTargetPositions(mode){
      if(mode === "DATA"){
        targetPositions.set(basePositions);
        return;
      }

      if(mode === "HEART"){
        // 2D heart formula projected in 3D
        for(let i=0;i<POINTS;i++){
          const t = (i/POINTS) * Math.PI * 2;
          const r = Math.sqrt(Math.random());
          const x2 = 16*Math.pow(Math.sin(t),3);
          const y2 = 13*Math.cos(t)-5*Math.cos(2*t)-2*Math.cos(3*t)-Math.cos(4*t);
          const x = (x2/18) * (1.6*r);
          const y = (y2/18) * (1.6*r);
          const z = (Math.random()-0.5) * 0.9;
          targetPositions[i*3+0] = x;
          targetPositions[i*3+1] = y;
          targetPositions[i*3+2] = z;
        }
        return;
      }

      if(mode === "SATURN"){
        // ring + core
        for(let i=0;i<POINTS;i++){
          const isCore = (i % 6 === 0);
          if(isCore){
            const u = (Math.random()-0.5)*0.6;
            const v = (Math.random()-0.5)*0.6;
            const w = (Math.random()-0.5)*0.6;
            targetPositions[i*3+0]=u; targetPositions[i*3+1]=v; targetPositions[i*3+2]=w;
          }else{
            const a = Math.random()*Math.PI*2;
            const rad = 2.2 + (Math.random()-0.5)*0.35;
            const x = Math.cos(a)*rad;
            const z = Math.sin(a)*rad;
            const y = (Math.random()-0.5)*0.25;
            targetPositions[i*3+0]=x; targetPositions[i*3+1]=y; targetPositions[i*3+2]=z;
          }
        }
        return;
      }

      if(mode === "FIREWORKS"){
        // multiple bursts
        for(let i=0;i<POINTS;i++){
          const burst = i % mines.length;
          const bx = mines[burst].center[0]*1.5;
          const by = mines[burst].center[1]*1.5;
          const bz = mines[burst].center[2]*1.5;
          const theta = Math.random()*Math.PI*2;
          const phi = Math.acos(2*Math.random()-1);
          const r = 0.2 + Math.random()*2.6;
          const x = bx + r*Math.sin(phi)*Math.cos(theta);
          const y = by + r*Math.cos(phi);
          const z = bz + r*Math.sin(phi)*Math.sin(theta);
          targetPositions[i*3+0]=x; targetPositions[i*3+1]=y; targetPositions[i*3+2]=z;
        }
        return;
      }
    }

    // tween strength
    let morph = 0.08;

    // ---------- Hand control state ----------
    let handFound = false;
    let gesture = "none";

    // rotate controls
    let yaw = 0, pitch = 0;
    let zoom = 10; // camera distance target

    // smoothing
    let targetYaw = 0, targetPitch = 0, targetZoom = 10;

    function setStatus(text){ document.getElementById("status").textContent = text; }
    function setGesture(text){
      document.getElementById("gestureLine").innerHTML = "<b>Gesture:</b> " + text;
    }

    // ---------- MediaPipe Hands (JS) ----------
    import { Hands } from "https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4/hands.js";
    import { Camera } from "https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils@0.3/camera_utils.js";

    const video = document.getElementById("video");

    function dist(a,b){
      const dx=a.x-b.x, dy=a.y-b.y;
      return Math.sqrt(dx*dx + dy*dy);
    }

    function classifyGesture(landmarks){
      const thumbTip = landmarks[4];
      const indexTip = landmarks[8];
      const middleTip= landmarks[12];
      const ringTip  = landmarks[16];
      const pinkyTip = landmarks[20];
      const wrist    = landmarks[0];

      const pinchD = dist(thumbTip, indexTip);
      const fistScore = (dist(indexTip,wrist)+dist(middleTip,wrist)+dist(ringTip,wrist)+dist(pinkyTip,wrist))/4;

      if(pinchD < 0.05) return { g:"pinch", pinchD, fistScore };
      if(fistScore < 0.25) return { g:"fist", pinchD, fistScore };
      return { g:"open", pinchD, fistScore };
    }

    let lastSwitch = 0;

    const hands = new Hands({
      locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4/${file}`
    });
    hands.setOptions({
      maxNumHands: 1,
      modelComplexity: 1,
      minDetectionConfidence: 0.6,
      minTrackingConfidence: 0.6
    });

    hands.onResults((results) => {
      if(!results.multiHandLandmarks || results.multiHandLandmarks.length === 0){
        handFound = false;
        setStatus("Camera: hand not found");
        setGesture("none");
        return;
      }

      handFound = true;
      const lm = results.multiHandLandmarks[0];
      const { g, pinchD } = classifyGesture(lm);
      gesture = g;
      setStatus("Camera: tracking ✓");
      setGesture(g);

      // Use landmark 5 (index MCP) as stable center
      const cx = lm[5].x; // 0..1
      const cy = lm[5].y; // 0..1

      // Open hand move => rotate (center-based)
      if(g === "open" || g === "pinch"){
        targetYaw   = (cx - 0.5) * 1.8;
        targetPitch = (cy - 0.5) * 1.2;
      }

      // Pinch => zoom
      if(g === "pinch"){
        const z = 6 + (pinchD * 90); // pinch closer => smaller pinchD => closer zoom
        targetZoom = Math.min(18, Math.max(5.5, z));
      }

      // Fist => mode switch (debounced)
      const now = performance.now();
      if(g === "fist" && (now - lastSwitch) > 650){
        setMode(modeIndex + 1);
        lastSwitch = now;
      }
    });

    async function startCamera(){
      try{
        setStatus("Camera: requesting permission…");
        const cam = new Camera(video, {
          onFrame: async () => { await hands.send({ image: video }); },
          width: 640,
          height: 480
        });
        cam.start();
        setStatus("Camera: started ✓");
      }catch(e){
        console.error(e);
        setStatus("Camera: permission blocked");
      }
    }

    // start in DATA mode
    setMode(0);
    startCamera();

    // ---------- Animate ----------
    function animate(){
      requestAnimationFrame(animate);

      // smooth camera control
      yaw   += (targetYaw - yaw) * 0.08;
      pitch += (targetPitch - pitch) * 0.08;
      zoom  += (targetZoom - zoom) * 0.08;

      camera.position.x = Math.sin(yaw) * zoom;
      camera.position.z = Math.cos(yaw) * zoom;
      camera.position.y = 0.8 + (-pitch * 2.0);
      camera.lookAt(0, 0, 0);

      // morph towards template
      const posAttr = geom.getAttribute("position");
      const p = posAttr.array;

      for(let i=0;i<p.length;i++){
        p[i] += (targetPositions[i] - p[i]) * morph;
      }
      posAttr.needsUpdate = true;

      // slight breathing effect when hand found
      if(handFound){
        cloud.rotation.y += 0.0022;
      }else{
        cloud.rotation.y += 0.0011;
      }

      renderer.render(scene, camera);
    }
    animate();

    addEventListener("resize", () => {
      camera.aspect = innerWidth/innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(innerWidth, innerHeight);
    });

    // Prevent page scroll on touch
    document.body.addEventListener("touchmove", (e)=>e.preventDefault(), { passive:false });
  </script>
</body>
</html>
