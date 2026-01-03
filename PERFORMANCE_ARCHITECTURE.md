# Performance Architecture Plan

## Overview

This document outlines smart, performant solutions for:
1. Collision detection (aircraft + vehicle)
2. Ground snapping (vehicle)
3. Custom 3D object rendering (Minecraft blocks, etc.)

---

## 1. Smart Collision Detection

### The Problem
`scene.clampToHeight()` with 3D Tiles is expensive - it "incurs two scene draws" per call. Current implementation:
- Aircraft: 2 calls every 8 frames (~15 ops/sec at 60fps)
- Car: 2 calls every frame + ground clamping = 180 ops/sec

### Solution: Tiered Height Sampling with Caching

```
┌─────────────────────────────────────────────────────────────┐
│                    HEIGHT CACHE SYSTEM                       │
├─────────────────────────────────────────────────────────────┤
│  Level 1: Local Grid Cache (fast, interpolated)             │
│  Level 2: Globe.getHeight (sync, terrain only)              │
│  Level 3: clampToHeight (expensive, 3D tiles)               │
└─────────────────────────────────────────────────────────────┘
```

#### Architecture: HeightFieldCache

```typescript
// packages/web/src/cesium/terrain/HeightFieldCache.ts

interface CachedHeight {
  height: number;
  timestamp: number;
  source: 'grid' | 'globe' | 'clamp';
}

interface GridCell {
  longitude: number;
  latitude: number;
  height: number;
  validUntil: number;
}

export class HeightFieldCache {
  private grid: Map<string, GridCell> = new Map();
  private readonly GRID_RESOLUTION = 0.0001; // ~11m at equator
  private readonly CACHE_TTL = 2000; // 2 seconds
  private readonly GRID_SIZE = 5; // 5x5 grid around vehicle

  private pendingSamples: Map<string, Promise<number>> = new Map();
  private lastSampleTime = 0;
  private readonly SAMPLE_INTERVAL = 500; // Refresh grid every 500ms

  // Scratch variables
  private static readonly scratchCarto = new Cesium.Cartographic();

  constructor(
    private scene: Cesium.Scene,
    private globe: Cesium.Globe
  ) {}

  /**
   * Get height at position - uses tiered approach
   * Fast path: interpolate from cached grid
   * Medium path: Globe.getHeight (terrain only)
   * Slow path: clampToHeight (3D tiles)
   */
  public getHeight(
    position: Cesium.Cartesian3,
    needsPrecision: boolean = false
  ): number | null {
    const carto = Cesium.Cartographic.fromCartesian(
      position,
      undefined,
      HeightFieldCache.scratchCarto
    );

    // Try cached grid interpolation first (fast)
    const cached = this.interpolateFromGrid(carto.longitude, carto.latitude);
    if (cached !== null && !needsPrecision) {
      return cached;
    }

    // Fall back to Globe.getHeight (sync, terrain only)
    const globeHeight = this.globe.getHeight(carto);
    if (globeHeight !== undefined) {
      return globeHeight;
    }

    // Only use expensive clampToHeight if absolutely needed
    if (needsPrecision) {
      const clamped = this.scene.clampToHeight(position);
      if (clamped) {
        return Cesium.Cartographic.fromCartesian(clamped).height;
      }
    }

    return null;
  }

  /**
   * Update the height grid around the vehicle
   * Call this periodically (every 500ms), NOT every frame
   */
  public async updateGrid(
    centerPosition: Cesium.Cartesian3,
    exclude?: any[]
  ): Promise<void> {
    const now = performance.now();
    if (now - this.lastSampleTime < this.SAMPLE_INTERVAL) {
      return;
    }
    this.lastSampleTime = now;

    const center = Cesium.Cartographic.fromCartesian(centerPosition);
    const halfSize = Math.floor(this.GRID_SIZE / 2);

    // Sample grid points asynchronously
    const samplePromises: Promise<void>[] = [];

    for (let dx = -halfSize; dx <= halfSize; dx++) {
      for (let dy = -halfSize; dy <= halfSize; dy++) {
        const lon = center.longitude + dx * this.GRID_RESOLUTION;
        const lat = center.latitude + dy * this.GRID_RESOLUTION;
        const key = this.getGridKey(lon, lat);

        // Skip if still valid
        const existing = this.grid.get(key);
        if (existing && existing.validUntil > now) {
          continue;
        }

        // Sample this point
        samplePromises.push(this.samplePoint(lon, lat, key, exclude));
      }
    }

    // Don't await - let it happen in background
    Promise.all(samplePromises).catch(() => {});
  }

  private async samplePoint(
    lon: number,
    lat: number,
    key: string,
    exclude?: any[]
  ): Promise<void> {
    // Avoid duplicate requests
    if (this.pendingSamples.has(key)) {
      return;
    }

    const position = Cesium.Cartesian3.fromRadians(lon, lat, 1000);

    const samplePromise = new Promise<number>((resolve) => {
      // Use requestAnimationFrame to spread work
      requestAnimationFrame(() => {
        const clamped = this.scene.clampToHeight(position, exclude);
        if (clamped) {
          resolve(Cesium.Cartographic.fromCartesian(clamped).height);
        } else {
          resolve(0);
        }
      });
    });

    this.pendingSamples.set(key, samplePromise);

    try {
      const height = await samplePromise;
      this.grid.set(key, {
        longitude: lon,
        latitude: lat,
        height,
        validUntil: performance.now() + this.CACHE_TTL
      });
    } finally {
      this.pendingSamples.delete(key);
    }
  }

  private interpolateFromGrid(lon: number, lat: number): number | null {
    // Find the 4 nearest grid points
    const baseLon = Math.floor(lon / this.GRID_RESOLUTION) * this.GRID_RESOLUTION;
    const baseLat = Math.floor(lat / this.GRID_RESOLUTION) * this.GRID_RESOLUTION;

    const p00 = this.grid.get(this.getGridKey(baseLon, baseLat));
    const p10 = this.grid.get(this.getGridKey(baseLon + this.GRID_RESOLUTION, baseLat));
    const p01 = this.grid.get(this.getGridKey(baseLon, baseLat + this.GRID_RESOLUTION));
    const p11 = this.grid.get(this.getGridKey(baseLon + this.GRID_RESOLUTION, baseLat + this.GRID_RESOLUTION));

    // Need at least 3 points for reasonable interpolation
    const validPoints = [p00, p10, p01, p11].filter(p => p !== undefined);
    if (validPoints.length < 3) {
      return null;
    }

    // Bilinear interpolation
    const tx = (lon - baseLon) / this.GRID_RESOLUTION;
    const ty = (lat - baseLat) / this.GRID_RESOLUTION;

    const h00 = p00?.height ?? p10?.height ?? p01?.height ?? 0;
    const h10 = p10?.height ?? p00?.height ?? p11?.height ?? 0;
    const h01 = p01?.height ?? p00?.height ?? p11?.height ?? 0;
    const h11 = p11?.height ?? p10?.height ?? p01?.height ?? 0;

    const h0 = h00 * (1 - tx) + h10 * tx;
    const h1 = h01 * (1 - tx) + h11 * tx;

    return h0 * (1 - ty) + h1 * ty;
  }

  private getGridKey(lon: number, lat: number): string {
    const lonKey = Math.round(lon / this.GRID_RESOLUTION);
    const latKey = Math.round(lat / this.GRID_RESOLUTION);
    return `${lonKey},${latKey}`;
  }

  public clearCache(): void {
    this.grid.clear();
    this.pendingSamples.clear();
  }
}
```

#### Smart Collision Detector

```typescript
// packages/web/src/cesium/collision/SmartCollisionDetector.ts

export interface CollisionConfig {
  // Aircraft settings
  aircraftMinAltitude: number;      // Only check when below this (e.g., 500m)
  aircraftCheckInterval: number;    // Frames between checks when low
  aircraftHighAltInterval: number;  // Frames between checks when high

  // Car settings
  carCheckInterval: number;         // Frames between full checks
  carProbeDistance: number;

  // Performance
  maxChecksPerFrame: number;        // Spread work across frames
}

export class SmartCollisionDetector {
  private heightCache: HeightFieldCache;
  private frameCount = 0;
  private config: CollisionConfig;

  constructor(
    scene: Cesium.Scene,
    config: Partial<CollisionConfig> = {}
  ) {
    this.heightCache = new HeightFieldCache(scene, scene.globe);
    this.config = {
      aircraftMinAltitude: 500,
      aircraftCheckInterval: 8,
      aircraftHighAltInterval: 30,
      carCheckInterval: 4,
      carProbeDistance: 2.0,
      maxChecksPerFrame: 2,
      ...config
    };
  }

  /**
   * Check aircraft collision - smart altitude-based throttling
   */
  public checkAircraftCollision(
    position: Cesium.Cartesian3,
    heading: number,
    exclude?: any[]
  ): { collision: boolean; groundHeight?: number } {
    this.frameCount++;

    const carto = Cesium.Cartographic.fromCartesian(position);
    const altitude = carto.height;

    // Determine check interval based on altitude
    const interval = altitude > this.config.aircraftMinAltitude
      ? this.config.aircraftHighAltInterval
      : this.config.aircraftCheckInterval;

    // Not time to check yet
    if (this.frameCount % interval !== 0) {
      return { collision: false };
    }

    // Update height cache in background
    this.heightCache.updateGrid(position, exclude);

    // Use cached height (fast)
    const groundHeight = this.heightCache.getHeight(position, false);

    if (groundHeight === null) {
      return { collision: false };
    }

    // Simple altitude check
    const buffer = 5.0; // 5m safety buffer
    if (altitude <= groundHeight + buffer) {
      // Confirm with precise check before declaring crash
      const preciseHeight = this.heightCache.getHeight(position, true);
      if (preciseHeight !== null && altitude <= preciseHeight + 0.5) {
        return { collision: true, groundHeight: preciseHeight };
      }
    }

    return { collision: false, groundHeight };
  }

  /**
   * Check car collision - uses cached heights mostly
   */
  public checkCarCollision(
    position: Cesium.Cartesian3,
    heading: number,
    velocity: number,
    exclude?: any[]
  ): { front: boolean; back: boolean; groundHeight?: number } {
    this.frameCount++;

    // Update cache periodically
    if (this.frameCount % 30 === 0) {
      this.heightCache.updateGrid(position, exclude);
    }

    // Only check every N frames
    if (this.frameCount % this.config.carCheckInterval !== 0) {
      return { front: false, back: false };
    }

    // Skip collision check if moving slowly
    if (Math.abs(velocity) < 1.0) {
      return { front: false, back: false };
    }

    const vehicleHeight = Cesium.Cartographic.fromCartesian(position).height;

    // Calculate probe positions (reuse existing logic but with cached heights)
    const probeDistance = this.config.carProbeDistance;

    // ... probe calculation ...
    // Use this.heightCache.getHeight() instead of scene.clampToHeight()

    return { front: false, back: false };
  }
}
```

---

## 2. Smart Ground Snapping

### The Problem
Car calls `scene.clampToHeight()` every frame in rover mode = 60 expensive ops/sec

### Solution: Predictive Caching + Interpolation

```
Frame 1: Use cached height (interpolated)
Frame 2: Use cached height (interpolated)
Frame 3: Use cached height (interpolated)
Frame 4: Use cached height (interpolated)
...
Frame N: Background refresh of nearby grid
```

#### Architecture: SmartTerrainClamping

```typescript
// packages/web/src/cesium/vehicles/car/SmartTerrainClamping.ts

export class SmartTerrainClamping {
  private heightCache: HeightFieldCache;
  private lastKnownHeight: number = 0;
  private lastPosition: Cesium.Cartesian3 = new Cesium.Cartesian3();
  private velocity: Cesium.Cartesian3 = new Cesium.Cartesian3();

  private frameCount = 0;
  private readonly FULL_CHECK_INTERVAL = 15; // Full clamp every 15 frames

  // Scratch variables (no allocations in hot path)
  private static readonly scratchCarto = new Cesium.Cartographic();
  private static readonly scratchResult = new Cesium.Cartesian3();

  constructor(
    private scene: Cesium.Scene,
    private groundOffset: number = 0
  ) {
    this.heightCache = new HeightFieldCache(scene, scene.globe);
  }

  /**
   * Clamp vehicle to ground - smart version
   * Uses cached heights most frames, only does expensive check periodically
   */
  public clampToGround(
    position: Cesium.Cartesian3,
    exclude?: any[]
  ): Cesium.Cartesian3 {
    this.frameCount++;

    // Update velocity estimate
    Cesium.Cartesian3.subtract(position, this.lastPosition, this.velocity);
    Cesium.Cartesian3.clone(position, this.lastPosition);

    // Background cache update (non-blocking)
    if (this.frameCount % 30 === 0) {
      this.heightCache.updateGrid(position, exclude);
    }

    let groundHeight: number;

    // Every N frames, do a full precise check
    if (this.frameCount % this.FULL_CHECK_INTERVAL === 0) {
      const precise = this.heightCache.getHeight(position, true);
      if (precise !== null) {
        groundHeight = precise;
        this.lastKnownHeight = precise;
      } else {
        groundHeight = this.lastKnownHeight;
      }
    } else {
      // Use fast cached/interpolated height
      const cached = this.heightCache.getHeight(position, false);
      if (cached !== null) {
        // Smooth transition to avoid popping
        groundHeight = Cesium.Math.lerp(
          this.lastKnownHeight,
          cached,
          0.3
        );
        this.lastKnownHeight = groundHeight;
      } else {
        groundHeight = this.lastKnownHeight;
      }
    }

    // Apply height without allocation
    const carto = Cesium.Cartographic.fromCartesian(
      position,
      undefined,
      SmartTerrainClamping.scratchCarto
    );

    carto.height = groundHeight + this.groundOffset;

    return Cesium.Cartographic.toCartesian(
      carto,
      undefined,
      SmartTerrainClamping.scratchResult
    );
  }
}
```

### Performance Comparison

| Approach | Ops/sec | CPU Impact |
|----------|---------|------------|
| Current (every frame) | 60 clampToHeight | Very High |
| Smart (cached + interpolated) | 4 clampToHeight + 56 interpolations | Low |

---

## 3. Custom 3D Object Rendering (Minecraft Blocks)

### Three Approaches

#### Option A: Cesium Primitives (Recommended for < 10,000 objects)

Best for: Static or semi-static structures, good integration with globe

```typescript
// packages/web/src/cesium/objects/VoxelRenderer.ts

interface VoxelBlock {
  x: number;
  y: number;
  z: number;
  type: number; // Block type ID
  color: Cesium.Color;
}

export class VoxelRenderer {
  private primitive: Cesium.Primitive | null = null;
  private geometryInstances: Cesium.GeometryInstance[] = [];
  private scene: Cesium.Scene;
  private dirty = true;

  // Block size in meters
  private readonly BLOCK_SIZE = 1.0;

  constructor(scene: Cesium.Scene) {
    this.scene = scene;
  }

  /**
   * Add blocks to the renderer
   * Call rebuild() after adding all blocks
   */
  public addBlock(
    longitude: number,
    latitude: number,
    altitude: number,
    color: Cesium.Color
  ): void {
    const position = Cesium.Cartesian3.fromDegrees(longitude, latitude, altitude);

    const instance = new Cesium.GeometryInstance({
      geometry: Cesium.BoxGeometry.fromDimensions({
        dimensions: new Cesium.Cartesian3(
          this.BLOCK_SIZE,
          this.BLOCK_SIZE,
          this.BLOCK_SIZE
        ),
        vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT
      }),
      modelMatrix: Cesium.Transforms.eastNorthUpToFixedFrame(position),
      attributes: {
        color: Cesium.ColorGeometryInstanceAttribute.fromColor(color)
      }
    });

    this.geometryInstances.push(instance);
    this.dirty = true;
  }

  /**
   * Add multiple blocks at once (more efficient)
   */
  public addBlocks(blocks: VoxelBlock[], origin: Cesium.Cartesian3): void {
    const originTransform = Cesium.Transforms.eastNorthUpToFixedFrame(origin);

    for (const block of blocks) {
      // Local position relative to origin
      const localMatrix = Cesium.Matrix4.fromTranslation(
        new Cesium.Cartesian3(
          block.x * this.BLOCK_SIZE,
          block.y * this.BLOCK_SIZE,
          block.z * this.BLOCK_SIZE
        )
      );

      const modelMatrix = Cesium.Matrix4.multiply(
        originTransform,
        localMatrix,
        new Cesium.Matrix4()
      );

      const instance = new Cesium.GeometryInstance({
        geometry: Cesium.BoxGeometry.fromDimensions({
          dimensions: new Cesium.Cartesian3(
            this.BLOCK_SIZE,
            this.BLOCK_SIZE,
            this.BLOCK_SIZE
          ),
          vertexFormat: Cesium.PerInstanceColorAppearance.VERTEX_FORMAT
        }),
        modelMatrix,
        attributes: {
          color: Cesium.ColorGeometryInstanceAttribute.fromColor(block.color)
        }
      });

      this.geometryInstances.push(instance);
    }

    this.dirty = true;
  }

  /**
   * Rebuild the primitive from all instances
   * Call this after adding blocks - batches everything into one draw call
   */
  public rebuild(): void {
    if (!this.dirty || this.geometryInstances.length === 0) {
      return;
    }

    // Remove old primitive
    if (this.primitive) {
      this.scene.primitives.remove(this.primitive);
    }

    // Create new batched primitive
    this.primitive = new Cesium.Primitive({
      geometryInstances: this.geometryInstances,
      appearance: new Cesium.PerInstanceColorAppearance({
        closed: true,
        translucent: false
      }),
      asynchronous: true, // Build on web worker
      releaseGeometryInstances: false // Keep for later modification
    });

    this.scene.primitives.add(this.primitive);
    this.dirty = false;

    console.log(`Built voxel primitive with ${this.geometryInstances.length} blocks`);
  }

  public destroy(): void {
    if (this.primitive) {
      this.scene.primitives.remove(this.primitive);
      this.primitive = null;
    }
    this.geometryInstances = [];
  }
}
```

**Performance:** Can handle 2,000-10,000 blocks efficiently with batching.

---

#### Option B: Three.js Integration (Recommended for > 10,000 objects)

Best for: Massive voxel worlds, complex shaders, instanced rendering

```typescript
// packages/web/src/threejs/ThreejsIntegration.ts

import * as THREE from 'three';
import * as Cesium from 'cesium';

export class ThreejsIntegration {
  private threeScene: THREE.Scene;
  private threeCamera: THREE.PerspectiveCamera;
  private threeRenderer: THREE.WebGLRenderer;
  private cesiumViewer: Cesium.Viewer;

  // For coordinate conversion
  private readonly minWGS84: [number, number] = [0, 0];
  private readonly maxWGS84: [number, number] = [0, 0];

  constructor(cesiumViewer: Cesium.Viewer) {
    this.cesiumViewer = cesiumViewer;

    // Create Three.js scene
    this.threeScene = new THREE.Scene();

    // Camera with far plane for globe scale
    this.threeCamera = new THREE.PerspectiveCamera(
      45,
      window.innerWidth / window.innerHeight,
      1,
      10000000 // 10,000 km far plane
    );

    // Renderer - renders to overlay canvas
    this.threeRenderer = new THREE.WebGLRenderer({
      alpha: true,
      antialias: true
    });
    this.threeRenderer.setSize(window.innerWidth, window.innerHeight);
    this.threeRenderer.domElement.style.position = 'absolute';
    this.threeRenderer.domElement.style.top = '0';
    this.threeRenderer.domElement.style.left = '0';
    this.threeRenderer.domElement.style.pointerEvents = 'none';

    // Add canvas on top of Cesium
    document.body.appendChild(this.threeRenderer.domElement);

    // Start render loop
    this.startRenderLoop();
  }

  /**
   * Add instanced voxels - VERY efficient for many identical objects
   */
  public addVoxelChunk(
    blocks: { x: number; y: number; z: number; color: number }[],
    originLon: number,
    originLat: number,
    originAlt: number
  ): THREE.InstancedMesh {
    const geometry = new THREE.BoxGeometry(1, 1, 1);
    const material = new THREE.MeshLambertMaterial();

    // InstancedMesh can render 100,000+ cubes in ONE draw call
    const mesh = new THREE.InstancedMesh(
      geometry,
      material,
      blocks.length
    );

    const matrix = new THREE.Matrix4();
    const color = new THREE.Color();

    for (let i = 0; i < blocks.length; i++) {
      const block = blocks[i];
      matrix.setPosition(block.x, block.y, block.z);
      mesh.setMatrixAt(i, matrix);
      mesh.setColorAt(i, color.setHex(block.color));
    }

    mesh.instanceMatrix.needsUpdate = true;
    if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;

    // Position mesh at geographic origin
    const origin = Cesium.Cartesian3.fromDegrees(originLon, originLat, originAlt);
    mesh.position.set(origin.x, origin.y, origin.z);

    // Align with globe surface
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(origin);
    const rotation = new THREE.Matrix4();
    rotation.set(
      transform[0], transform[4], transform[8], 0,
      transform[1], transform[5], transform[9], 0,
      transform[2], transform[6], transform[10], 0,
      0, 0, 0, 1
    );
    mesh.setRotationFromMatrix(rotation);

    this.threeScene.add(mesh);

    console.log(`Added voxel chunk with ${blocks.length} blocks`);
    return mesh;
  }

  private startRenderLoop(): void {
    const render = () => {
      requestAnimationFrame(render);
      this.syncCamera();
      this.threeRenderer.render(this.threeScene, this.threeCamera);
    };
    render();
  }

  /**
   * Sync Three.js camera to Cesium camera
   */
  private syncCamera(): void {
    // Disable Three.js auto matrix updates
    this.threeCamera.matrixAutoUpdate = false;

    // Copy Cesium's inverse view matrix to Three.js camera
    const civm = this.cesiumViewer.camera.inverseViewMatrix;
    this.threeCamera.matrixWorld.set(
      civm[0], civm[4], civm[8], civm[12],
      civm[1], civm[5], civm[9], civm[13],
      civm[2], civm[6], civm[10], civm[14],
      civm[3], civm[7], civm[11], civm[15]
    );

    // Sync FOV
    const frustum = this.cesiumViewer.camera.frustum as Cesium.PerspectiveFrustum;
    if (frustum.fovy) {
      this.threeCamera.fov = Cesium.Math.toDegrees(frustum.fovy);
      this.threeCamera.updateProjectionMatrix();
    }
  }

  public destroy(): void {
    document.body.removeChild(this.threeRenderer.domElement);
    this.threeRenderer.dispose();
  }
}
```

**Performance:** InstancedMesh can render 100,000+ cubes in a single draw call.

---

#### Option C: glTF with GPU Instancing (Best of Both Worlds)

Best for: Pre-designed block types, maximum performance, full Cesium integration

```typescript
// packages/web/src/cesium/objects/GltfVoxelRenderer.ts

export class GltfVoxelRenderer {
  private model: Cesium.Model | null = null;
  private scene: Cesium.Scene;

  constructor(scene: Cesium.Scene) {
    this.scene = scene;
  }

  /**
   * Load a pre-built glTF with GPU instancing
   * The glTF should use EXT_mesh_gpu_instancing extension
   */
  public async loadVoxelModel(
    url: string, // e.g., "/models/voxel-chunk.glb"
    position: Cesium.Cartesian3
  ): Promise<void> {
    this.model = await Cesium.Model.fromGltfAsync({
      url,
      modelMatrix: Cesium.Transforms.eastNorthUpToFixedFrame(position),
      scale: 1.0
    });

    this.scene.primitives.add(this.model);
  }

  /**
   * Generate a glTF with instances programmatically
   * Uses EXT_mesh_gpu_instancing for efficient rendering
   */
  public static generateInstancedGltf(
    blocks: { x: number; y: number; z: number }[]
  ): Blob {
    // This would generate a glTF binary with:
    // 1. Base cube mesh
    // 2. EXT_mesh_gpu_instancing extension
    // 3. Instance transforms for each block

    // The actual implementation would use a glTF library
    // like @gltf-transform/core

    throw new Error('Implementation requires glTF library');
  }
}
```

---

### Recommendation Matrix

| Scenario | Approach | Why |
|----------|----------|-----|
| < 1,000 blocks, static | Cesium Primitives | Simple, good integration |
| 1,000 - 10,000 blocks | Cesium Primitives (batched) | Still manageable |
| > 10,000 blocks | Three.js InstancedMesh | True GPU instancing |
| Massive world (Minecraft-scale) | Three.js + chunking | LOD, streaming |
| Pre-designed structures | glTF with instances | Best performance |

---

## 4. Integration Plan

### Phase 1: Height Cache System (Week 1)
1. Implement `HeightFieldCache` class
2. Add to VehicleManager as shared resource
3. Migrate Car ground clamping to use cache
4. Migrate collision detection to use cache

### Phase 2: Three.js Integration (Week 2)
1. Add Three.js dependency
2. Implement `ThreejsIntegration` class
3. Create VoxelManager for block placement
4. Test with 10,000+ blocks

### Phase 3: Optimization (Week 3)
1. Add chunking for large worlds
2. Implement LOD for distant chunks
3. Add frustum culling
4. Profile and tune

---

## Sources

- [CesiumJS Globe.getHeight](https://community.cesium.com/t/cesium-equivalent-of-terrain-getheight-x-y/13187)
- [Cesium Custom Geometry & Appearances](https://cesium.com/learn/cesiumjs-learn/cesiumjs-geometry-appearances/)
- [Cesium Three.js Integration](https://github.com/CesiumGS/cesium-threejs-experiment)
- [CesiumJS Performance Issue #11923](https://github.com/CesiumGS/cesium/issues/11923)
- [sampleTerrainMostDetailed](https://community.cesium.com/t/how-to-decrease-sampleterrain-requests-and-improve-its-performance/16247)
