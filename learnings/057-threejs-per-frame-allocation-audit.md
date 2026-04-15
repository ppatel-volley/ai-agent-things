# Learning 057: Three.js Per-Frame Allocation Audit — The segmentCenter Pattern

**Severity:** Critical
**Sources:** Tempest 2026 — WebGL context loss on Fire TV, graphics engineer audit
**Category:** Three.js, Performance, React Three Fiber

## Principle

In a Three.js game loop (`useFrame`), every `new THREE.Vector3()`, `new THREE.Color()`, or `new THREE.Matrix4()` allocates a heap object that the garbage collector must eventually reclaim. On low-end devices like Fire TV, thousands of allocations per second cause GC stalls that drop frames and can trigger WebGL context loss.

The most dangerous pattern is a utility function like `segmentCenter()` that allocates internally and gets called 200+ times per frame from multiple consumers.

## The Audit Results (Tempest 2026)

Before optimisation:
- `tubeGeometry.segmentCenter()` — 3 Vector3 allocations per call × ~220 calls/frame = **~39,600 allocs/sec**
- `ParticleSystem` — `new THREE.Color(p.color)` per particle per frame = **~3,000 allocs/sec**
- `PlayerBlaster` — 3 Vector3 + 1 Matrix4 per player per frame = **~960 allocs/sec**
- `SpringCamera` — 5 Vector3 per frame = **~300 allocs/sec**
- **Total: ~44,000 object allocations per second**

After optimisation: **~0 allocations per second**.

## The Patterns

### Pattern 1: Utility functions that allocate internally

```typescript
// WRONG — allocates 3 Vector3 objects per call
function segmentCenter(index: number, depth: number): THREE.Vector3 {
    const rimCenter = new THREE.Vector3().addVectors(a, b).multiplyScalar(0.5)
    const farCenter = new THREE.Vector3().addVectors(c, d).multiplyScalar(0.5)
    return new THREE.Vector3().lerpVectors(farCenter, rimCenter, depth)
}
```

```typescript
// CORRECT — scratch variables, optional output parameter
const _rimCenter = new THREE.Vector3()
const _farCenter = new THREE.Vector3()
const _result = new THREE.Vector3()

function segmentCenter(index: number, depth: number, out?: THREE.Vector3): THREE.Vector3 {
    _rimCenter.addVectors(a, b).multiplyScalar(0.5)
    _farCenter.addVectors(c, d).multiplyScalar(0.5)
    const target = out ?? _result
    return target.lerpVectors(_farCenter, _rimCenter, depth)
}
```

### Pattern 2: Per-frame allocations in useFrame

```typescript
// WRONG — allocates inside useFrame
useFrame(() => {
    const tangent = new THREE.Vector3().subVectors(a, b)
    const forward = new THREE.Vector3().subVectors(c, d)
    const mat = new THREE.Matrix4()
})

// CORRECT — useRef scratch variables
const _tangent = useRef(new THREE.Vector3())
const _forward = useRef(new THREE.Vector3())
const _mat = useRef(new THREE.Matrix4())

useFrame(() => {
    _tangent.current.subVectors(a, b)
    _forward.current.subVectors(c, d)
})
```

### Pattern 3: new THREE.Color in loops

```typescript
// WRONG — 50 Color allocations per frame
for (const particle of particles) {
    const c = new THREE.Color(particle.color)
    colors[i] = c.r * brightness
}

// CORRECT — reuse single Color
const _color = useRef(new THREE.Color())
// in useFrame:
_color.current.set(particle.color)
colors[i] = _color.current.r * brightness
```

## Draw Call Reduction (Same Audit)

| Before | After | Change |
|--------|-------|--------|
| 16 separate meshes for tube fills | 1 merged BufferGeometry | -15 draw calls |
| 32 separate Lines for lane dividers | 2 LineSegments (core + glow) | -30 draw calls |
| 120 enemy pool objects in scene graph | Same (pool pattern) | Could be InstancedMesh |
| ~200 total draw calls worst case | ~30-40 draw calls | **~5x reduction** |

Key technique: merge geometries that share the same material into a single `BufferGeometry` with all vertices concatenated. One draw call instead of N.

## Red Flags to Watch For

- `new THREE.Vector3()` or `new THREE.Color()` inside `useFrame()` — always wrong
- A utility function called >10x per frame that returns `new THREE.Vector3()` — needs scratch vars
- `useMemo` that depends on a prop that changes every frame (e.g. `enemies` array) — recreates objects every render
- Individual `<mesh>` elements per particle/projectile — use Points or LineSegments with pre-allocated buffers
- `THREE.LineBasicMaterial` cloned per line — share a single material instance

## Prevention

1. **Grep for `new THREE.` inside `useFrame` callbacks** — every match is a potential per-frame allocation
2. **Count draw calls** using `renderer.info.render.calls` in a useFrame debug log
3. **Profile on the target device** (Fire TV) — desktop Chrome hides GC pressure
4. **Use `useRef` for all scratch math objects** in R3F components
5. **Utility functions should accept an `out` parameter** for the return value
