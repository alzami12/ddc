<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Rasenshuriken — Chakra Weapon (Three.js)</title>
<style>
  html, body {
    margin: 0; padding: 0; width: 100%; height: 100%;
    background: radial-gradient(ellipse at center, #050914 0%, #000000 100%);
    overflow: hidden;
    font-family: 'Segoe UI', Arial, sans-serif;
  }
  #canvas { position: fixed; top: 0; left: 0; width: 100%; height: 100%; display: block; }
  #panel {
    position: fixed; top: 14px; left: 14px; z-index: 5;
    color: #bdf6ff; background: rgba(5, 15, 30, 0.6);
    border: 1px solid rgba(100, 220, 255, 0.35);
    border-radius: 10px; padding: 12px 16px; font-size: 13px; line-height: 1.6em;
    max-width: 280px; backdrop-filter: blur(4px);
  }
  #panel b { color: #e6ffff; }
  #panel button {
    margin-top: 8px; width: 100%;
    background: linear-gradient(135deg, #1a6fa8, #0a3a66);
    color: #eafcff; border: 1px solid #4fd6ff55;
    padding: 8px 10px; border-radius: 8px; cursor: pointer;
    font-size: 13px; letter-spacing: 0.3px;
  }
  #panel button:hover { background: linear-gradient(135deg, #2483c4, #0d4a80); }
  #stats {
    position: fixed; bottom: 12px; left: 14px; z-index: 5;
    color: #7fdfff99; font-size: 11px; letter-spacing: 0.4px;
  }
</style>
</head>
<body>

<canvas id="canvas"></canvas>

<div id="panel">
  <b>Rasenshuriken — Chakra Weapon</b><br/>
  Procedural, game-ready Three.js model.<br/>
  Drag to orbit • Scroll to zoom<br/>
  <button id="exportBtn">Export as .GLB</button>
</div>
<div id="stats">Triangles: <span id="triCount">–</span></div>

<!-- Three.js core + orbit controls + GLTF exporter (for GLB download) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/exporters/GLTFExporter.js"></script>

<script>
/* =========================================================================
   RASENSHURIKEN — procedurally generated, game-ready chakra weapon
   -------------------------------------------------------------------------
   Structure (each is its own mesh/group so it can be exported as separate
   GLB nodes and animated independently in a game engine):

     Rasenshuriken (root Group)
       ├── core            (Group: pivot for the central energy sphere)
       │     ├── coreSphere      - inner bright emissive sphere
       │     ├── coreEnergyShell - translucent swirling layer
       │     └── coreRings[]     - thin torus "chakra ring" details
       ├── blades          (Group: pivot for the 4 shuriken blades, spins)
       │     └── blade x4        - sharp aerodynamic blade meshes
       ├── trails          (Group: curved trail ribbons behind blades)
       └── particles       (Points: small glowing sparks around the weapon)

   All materials are MeshStandardMaterial/MeshPhysicalMaterial (PBR) with
   emissive channels, keeping material slots minimal (4 total) for
   real-time performance. Geometry uses low-to-medium poly primitives
   (icosahedron, cone, custom shuriken blade shape) rather than dense
   sculpts, keeping the asset lightweight for WebGL / low-end GPUs.
   ========================================================================= */

(function () {
  'use strict';

  // ---------------------------------------------------------------------
  // RENDERER / SCENE / CAMERA
  // ---------------------------------------------------------------------
  const canvas = document.getElementById('canvas');
  const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.outputEncoding = THREE.sRGBEncoding;

  const scene = new THREE.Scene();

  const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.set(2.6, 1.6, 2.6);

  const controls = new THREE.OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.08;
  controls.minDistance = 1.2;
  controls.maxDistance = 8;

  // Ambient + key/rim lights so the PBR materials read well from all angles
  scene.add(new THREE.AmbientLight(0x1a2b44, 0.9));
  const keyLight = new THREE.DirectionalLight(0xbfe9ff, 0.8);
  keyLight.position.set(3, 4, 2);
  scene.add(keyLight);
  const rimLight = new THREE.DirectionalLight(0x3388ff, 0.6);
  rimLight.position.set(-3, -1, -3);
  scene.add(rimLight);

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });

  // ---------------------------------------------------------------------
  // ROOT GROUP — central pivot for the whole weapon (spin / scale / move
  // this single node in-game to animate the entire Rasenshuriken).
  // ---------------------------------------------------------------------
  const rasenshuriken = new THREE.Group();
  rasenshuriken.name = 'Rasenshuriken';
  scene.add(rasenshuriken);

  // Shared material palette — kept to a minimal set of slots for
  // real-time efficiency (4 materials reused across all geometry).
  const materials = {
    core: new THREE.MeshPhysicalMaterial({
      color: 0x5fd8ff,
      emissive: 0x1fa8ff,
      emissiveIntensity: 2.4,
      roughness: 0.2,
      metalness: 0.05,
      clearcoat: 0.6,
      transparent: true,
      opacity: 0.95
    }),
    energyShell: new THREE.MeshBasicMaterial({
      color: 0x8fe9ff,
      transparent: true,
      opacity: 0.22,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      side: THREE.DoubleSide
    }),
    blade: new THREE.MeshPhysicalMaterial({
      color: 0x4fc8ff,
      emissive: 0x1a7fdd,
      emissiveIntensity: 1.4,
      roughness: 0.15,
      metalness: 0.1,
      transparent: true,
      opacity: 0.85,
      side: THREE.DoubleSide
    }),
    bladeEdge: new THREE.MeshBasicMaterial({
      color: 0xeaffff,
      transparent: true,
      opacity: 0.9,
      blending: THREE.AdditiveBlending,
      depthWrite: false
    })
  };

  // ---------------------------------------------------------------------
  // 1. ENERGY CORE — spherical chakra core with layered translucent shell
  //    and fine "chakra ring" details for a swirling energy look.
  // ---------------------------------------------------------------------
  const core = new THREE.Group();
  core.name = 'EnergyCore';

  const coreSphere = new THREE.Mesh(
    new THREE.IcosahedronGeometry(0.32, 3), // low-medium poly, smooth enough for glow
    materials.core
  );
  coreSphere.name = 'CoreSphere';
  core.add(coreSphere);

  const coreEnergyShell = new THREE.Mesh(
    new THREE.IcosahedronGeometry(0.42, 2),
    materials.energyShell
  );
  coreEnergyShell.name = 'CoreEnergyShell';
  core.add(coreEnergyShell);

  // Thin swirling chakra rings around the core (layered tori, cheap geometry)
  const coreRings = [];
  const ringSpecs = [
    { r: 0.5,  tube: 0.006, rx: 0.3, ry: 0.0, rz: 0.1 },
    { r: 0.56, tube: 0.005, rx: 1.1, ry: 0.4, rz: 0.0 },
    { r: 0.47, tube: 0.006, rx: 0.0, ry: 1.0, rz: 0.7 }
  ];
  ringSpecs.forEach((spec, i) => {
    const ring = new THREE.Mesh(
      new THREE.TorusGeometry(spec.r, spec.tube, 6, 48),
      materials.energyShell
    );
    ring.rotation.set(spec.rx, spec.ry, spec.rz);
    ring.name = 'CoreRing' + i;
    coreRings.push(ring);
    core.add(ring);
  });

  rasenshuriken.add(core);

  // ---------------------------------------------------------------------
  // 2. SHURIKEN BLADES — four sharp, symmetrical, aerodynamic blades
  //    arranged around the core, forming a perfect rotating star.
  // ---------------------------------------------------------------------
  const blades = new THREE.Group();
  blades.name = 'Blades';

  /**
   * Build a single sharp shuriken-style blade as an extruded 2D shape.
   * Low poly count (a single extrude with a handful of points) keeps this
   * game-ready while still reading as "sharp and aerodynamic".
   */
  function createBladeGeometry() {
    const shape = new THREE.Shape();
    // Define a curved, tapering blade silhouette (points in local XY plane)
    shape.moveTo(0, 0.08);
    shape.quadraticCurveTo(0.35, 0.18, 0.95, 0.05);   // upper curved edge to sharp tip
    shape.quadraticCurveTo(1.05, 0.0, 0.95, -0.05);   // tip
    shape.quadraticCurveTo(0.4, -0.14, 0.05, -0.09);  // lower curved edge back
    shape.quadraticCurveTo(-0.02, 0, 0, 0.08);        // back to base, slight inner curve

    const extrudeSettings = {
      steps: 1,
      depth: 0.02,
      bevelEnabled: true,
      bevelThickness: 0.006,
      bevelSize: 0.008,
      bevelSegments: 2
    };
    const geo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
    geo.center();
    return geo;
  }

  const bladeGeometry = createBladeGeometry();

  // Thin glowing edge strip along the blade's leading edge for the
  // "bright white-blue edge" highlight requested.
  function createBladeEdgeGeometry() {
    const shape = new THREE.Shape();
    shape.moveTo(0, 0.085);
    shape.quadraticCurveTo(0.35, 0.185, 0.95, 0.05);
    shape.quadraticCurveTo(1.0, 0.02, 0.95, 0.0);
    shape.quadraticCurveTo(0.4, 0.1, 0.05, 0.07);
    shape.quadraticCurveTo(0.0, 0.08, 0, 0.085);
    const geo = new THREE.ExtrudeGeometry(shape, { steps: 1, depth: 0.005, bevelEnabled: false });
    geo.center();
    return geo;
  }
  const bladeEdgeGeometry = createBladeEdgeGeometry();

  const BLADE_COUNT = 4;
  for (let i = 0; i < BLADE_COUNT; i++) {
    const bladeGroup = new THREE.Group();
    bladeGroup.name = 'Blade' + i;

    const bladeMesh = new THREE.Mesh(bladeGeometry, materials.blade);
    bladeMesh.position.x = 0.35; // offset outward from the pivot
    bladeGroup.add(bladeMesh);

    const edgeMesh = new THREE.Mesh(bladeEdgeGeometry, materials.bladeEdge);
    edgeMesh.position.x = 0.35;
    edgeMesh.position.z = 0.013;
    bladeGroup.add(edgeMesh);

    // Rotate each blade evenly around the center -> perfect 4-point star
    bladeGroup.rotation.z = (i / BLADE_COUNT) * Math.PI * 2;
    blades.add(bladeGroup);
  }
  rasenshuriken.add(blades);

  // ---------------------------------------------------------------------
  // 3. ENERGY TRAILS — thin curved ribbons suggesting motion/speed behind
  //    each blade tip. Cheap plane-strip geometry.
  // ---------------------------------------------------------------------
  const trails = new THREE.Group();
  trails.name = 'Trails';

  function createTrailGeometry() {
    const curve = new THREE.QuadraticBezierCurve3(
      new THREE.Vector3(0.35, 0, 0),
      new THREE.Vector3(0.15, 0.25, 0),
      new THREE.Vector3(-0.15, 0.4, 0)
    );
    const points = curve.getPoints(12);
    const positions = [];
    for (let i = 0; i < points.length - 1; i++) {
      const w = 0.03 * (1 - i / points.length); // taper the trail width
      positions.push(points[i].x, points[i].y + w, 0);
      positions.push(points[i].x, points[i].y - w, 0);
      positions.push(points[i + 1].x, points[i + 1].y + w * 0.8, 0);
      positions.push(points[i].x, points[i].y - w, 0);
      positions.push(points[i + 1].x, points[i + 1].y + w * 0.8, 0);
      positions.push(points[i + 1].x, points[i + 1].y - w * 0.8, 0);
    }
    const geo = new THREE.BufferGeometry();
    geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
    geo.computeVertexNormals();
    return geo;
  }
  const trailGeometry = createTrailGeometry();
  const trailMaterial = new THREE.MeshBasicMaterial({
    color: 0x66d9ff,
    transparent: true,
    opacity: 0.35,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    side: THREE.DoubleSide
  });

  for (let i = 0; i < BLADE_COUNT; i++) {
    const trailMesh = new THREE.Mesh(trailGeometry, trailMaterial);
    trailMesh.rotation.z = (i / BLADE_COUNT) * Math.PI * 2;
    trailMesh.name = 'Trail' + i;
    trails.add(trailMesh);
  }
  rasenshuriken.add(trails);

  // ---------------------------------------------------------------------
  // 4. PARTICLES — small glowing chakra sparks orbiting the weapon.
  // ---------------------------------------------------------------------
  const PARTICLE_COUNT = 220;
  const particlePositions = new Float32Array(PARTICLE_COUNT * 3);
  const particleRadii = new Float32Array(PARTICLE_COUNT);
  const particleSpeeds = new Float32Array(PARTICLE_COUNT);
  const particleAngles = new Float32Array(PARTICLE_COUNT);
  const particleHeights = new Float32Array(PARTICLE_COUNT);

  for (let i = 0; i < PARTICLE_COUNT; i++) {
    particleRadii[i] = 0.45 + Math.random() * 0.55;
    particleAngles[i] = Math.random() * Math.PI * 2;
    particleHeights[i] = (Math.random() - 0.5) * 0.25;
    particleSpeeds[i] = 0.6 + Math.random() * 1.6;
  }

  const particleGeometry = new THREE.BufferGeometry();
  particleGeometry.setAttribute('position', new THREE.BufferAttribute(particlePositions, 3));
  const particleMaterial = new THREE.PointsMaterial({
    color: 0xaef2ff,
    size: 0.02,
    transparent: true,
    opacity: 0.85,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    sizeAttenuation: true
  });
  const particles = new THREE.Points(particleGeometry, particleMaterial);
  particles.name = 'ChakraParticles';
  rasenshuriken.add(particles);

  // ---------------------------------------------------------------------
  // LIGHTING FROM THE WEAPON ITSELF — bright cyan point light at the core
  // ---------------------------------------------------------------------
  const coreLight = new THREE.PointLight(0x55e0ff, 3.5, 5, 2);
  rasenshuriken.add(coreLight);

  // ---------------------------------------------------------------------
  // TRIANGLE COUNT (for the on-screen perf readout)
  // ---------------------------------------------------------------------
  function countTriangles(root) {
    let tris = 0;
    root.traverse((obj) => {
      if (obj.isMesh && obj.geometry) {
        const geo = obj.geometry;
        if (geo.index) tris += geo.index.count / 3;
        else if (geo.attributes.position) tris += geo.attributes.position.count / 3;
      }
    });
    return Math.round(tris);
  }
  document.getElementById('triCount').textContent = countTriangles(rasenshuriken).toLocaleString();

  // ---------------------------------------------------------------------
  // ANIMATION LOOP — continuous spin on the root pivot, blades counter-
  // detailed with their own faster spin, particles orbiting, pulsing glow.
  // ---------------------------------------------------------------------
  const clock = new THREE.Clock();

  function animate() {
    requestAnimationFrame(animate);
    const delta = clock.getDelta();
    const t = clock.getElapsedTime();

    // Whole-weapon slow tumble so it reads well from every angle
    rasenshuriken.rotation.y += delta * 0.3;

    // Blades spin rapidly around the central pivot — this is the primary
    // "rotating star" motion, kept on its own group for easy game-side control.
    blades.rotation.z += delta * 4.5;
    trails.rotation.z += delta * 4.5; // trails follow the blades exactly

    // Core swirls independently for the "inner energy" look
    core.rotation.y += delta * 1.8;
    core.rotation.x += delta * 0.6;
    coreRings.forEach((ring, i) => {
      ring.rotation.z += delta * (1.2 + i * 0.4);
    });

    // Pulse the core emissive intensity for a "living chakra" feel
    materials.core.emissiveIntensity = 2.1 + Math.sin(t * 5) * 0.5;
    coreLight.intensity = 3.2 + Math.sin(t * 6) * 0.7;

    // Animate orbiting particles along spiral paths around the weapon
    const posAttr = particleGeometry.attributes.position;
    for (let i = 0; i < PARTICLE_COUNT; i++) {
      const angle = particleAngles[i] + t * particleSpeeds[i];
      const r = particleRadii[i];
      posAttr.array[i * 3]     = Math.cos(angle) * r;
      posAttr.array[i * 3 + 1] = particleHeights[i] + Math.sin(t * 2 + i) * 0.03;
      posAttr.array[i * 3 + 2] = Math.sin(angle) * r;
    }
    posAttr.needsUpdate = true;

    controls.update();
    renderer.render(scene, camera);
  }
  animate();

  // ---------------------------------------------------------------------
  // GLB EXPORT — lets a developer download this procedural model as a
  // real .glb asset for use in any Three.js game or other GLTF-compatible
  // engine, complete with materials, hierarchy, and node names intact.
  // ---------------------------------------------------------------------
  document.getElementById('exportBtn').addEventListener('click', () => {
    const exporter = new THREE.GLTFExporter();
    exporter.parse(
      rasenshuriken,
      (result) => {
        const blob = new Blob([result], { type: 'application/octet-stream' });
        const url = URL.createObjectURL(blob);
        const link = document.createElement('a');
        link.href = url;
        link.download = 'rasenshuriken.glb';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(url);
      },
      (error) => {
        console.error('GLTFExporter error:', error);
        alert('Export failed — see console for details.');
      },
      { binary: true }
    );
  });

})();
</script>
</body>
</html>
