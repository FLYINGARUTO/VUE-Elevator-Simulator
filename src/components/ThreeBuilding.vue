<script setup>
import { onMounted, onUnmounted, ref, reactive, computed } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { CSS2DRenderer, CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js'

// --- Constants --- //
const FLOORS = 10
const ELEVATOR_COUNT = 3
const TICK_MS = 900 // How often the elevator state updates
const DOOR_MS = 5000 // How long doors stay open

// --- 3D Scene Dimensions --- //
const floorHeight = 3.6
const elevatorCabHalfHeight = 1.5

// --- Vue Refs and Reactive State --- //
const canvasHost = ref(null)
const elevators = reactive(Array.from({ length: ELEVATOR_COUNT }, (_, i) => createElevator(i)))
const floorCalls = reactive(Array.from({ length: FLOORS }, () => ({ up: false, down: false })))

// --- Shared Three.js and Logic Variables --- //
let renderer, scene, camera, controls, labelRenderer
let tickTimer = null
let elevatorMeshes = []
let cabMatIdle, cabMatOpen

// --- Elevator Logic --- //
function createElevator(id) {
  return {
    id,
    name: `E${id + 1}`,
    currentFloor: 0,
    direction: 'idle', // 'up' | 'down' | 'idle'
    targets: [],
    doorsOpen: false,
    moving: false,
    floorButtons: Array(FLOORS).fill(false), // Internal panel buttons
  }
}

function call(floor, dir) {
  if (floor < 0 || floor >= FLOORS) return
  floorCalls[floor][dir] = true
  schedule(floor, dir)
}

function pressInCarButton(e, floor) {
  if (e.targets.includes(floor)) return // Already going there
  e.floorButtons[floor] = true
  insertTarget(e, floor)
}

function schedule(floor, dir) {
  let bestElevator = null
  let minCost = Infinity

  for (const e of elevators) {
    let cost
    const distanceToCall = Math.abs(e.currentFloor - floor)

    if (e.direction === 'idle') {
      // Cost for an idle elevator is just the distance to the call floor.
      cost = distanceToCall
    } else if (e.direction === dir && (dir === 'up' ? floor >= e.currentFloor : floor <= e.currentFloor)) {
      // This elevator is moving towards the call floor and in the same direction.
      // This is the best case (piggybacking). Cost is just the distance.
      cost = distanceToCall
    } else {
      // The elevator is busy with other tasks or moving in the opposite direction.
      // The cost is the distance to its final target, plus the distance from there to the new call.
      const lastTarget = e.targets[e.targets.length - 1]
      const distToFinishTrip = Math.abs(lastTarget - e.currentFloor)
      const distFromEndToNewCall = Math.abs(floor - lastTarget)
      cost = distToFinishTrip + distFromEndToNewCall
    }

    if (cost < minCost) {
      minCost = cost
      bestElevator = e
    }
  }

  if (bestElevator) {
    insertTarget(bestElevator, floor)
  }
}

function insertTarget(e, floor) {
  if (e.targets.includes(floor)) return

  e.targets.push(floor)
  if (e.direction === 'up') {
    e.targets.sort((a, b) => a - b)
  } else if (e.direction === 'down') {
    e.targets.sort((a, b) => b - a)
  }

  if (e.direction === 'idle') {
    updateDirection(e)
  }
}

function updateDirection(e) {
  if (e.targets.length === 0) {
    e.direction = 'idle'
    e.moving = false
  } else {
    const target = e.targets[0]
    if (target > e.currentFloor) e.direction = 'up'
    else if (target < e.currentFloor) e.direction = 'down'
    else e.direction = 'idle' // Should not happen if logic is correct
  }
}

// --- Core Update Loop (Tick) --- //
function tick() {
  // 1. Update elevator state
  for (const e of elevators) {
    if (e.doorsOpen) continue

    if (e.targets.length === 0) {
      if (e.moving) {
        e.moving = false
        updateDirection(e)
      }
      continue
    }

    const target = e.targets[0]
    if (target === e.currentFloor) {
      // Arrived at target
      // Check if this stop is for an external call BEFORE clearing the flags
      const isExternalCallStop = floorCalls[target].up || floorCalls[target].down

      e.targets.shift()
      e.doorsOpen = true
      e.moving = false
      // Clear call lights for this floor and the internal button
      floorCalls[target].up = false
      floorCalls[target].down = false
      e.floorButtons[target] = false

      const elevatorMesh = elevatorMeshes[e.id]

      // Only show the panel if someone was waiting outside (an external call)
      if (isExternalCallStop) {
        // Create and show the internal button panel
        const panelDiv = document.createElement('div')
        panelDiv.className = 'in-car-panel'
        for (let i = 0; i < FLOORS; i++) {
          const btn = document.createElement('button')
          btn.textContent = i
          btn.className = e.floorButtons[i] ? 'active' : ''
          btn.onclick = () => pressInCarButton(e, i)
          panelDiv.appendChild(btn)
        }
        const panelObject = new CSS2DObject(panelDiv)
        panelObject.name = `panel-${e.id}` // Name it for later removal

        // Add panel directly to the scene, not to the car
        scene.add(panelObject)

        // Position it in world space, to the right of the correct shaft
        const shaftXPos = (e.id - (ELEVATOR_COUNT - 1) / 2) * 7.5 // 7.5 is the spacing between shafts
        panelObject.position.set(shaftXPos + 3.5, e.currentFloor * floorHeight + floorHeight / 2, 0)
      }

      setTimeout(() => {
        e.doorsOpen = false
        updateDirection(e)

        // If a panel was shown, remove it from the scene now that the doors are closing.
        if (isExternalCallStop) {
          const panelToRemove = scene.getObjectByName(`panel-${e.id}`)
          if (panelToRemove) {
            scene.remove(panelToRemove)
            // Important: Clean up the DOM element to prevent memory leaks
            if (panelToRemove.element) {
              panelToRemove.element.remove()
            }
          }
        }
      }, DOOR_MS)
    } else {
      // Moving towards target
      e.moving = true
      if (target > e.currentFloor) {
        e.direction = 'up'
        e.currentFloor++
      } else {
        e.direction = 'down'
        e.currentFloor--
      }
    }
  }

  // 2. Sync 3D Meshes with the state
  for (let i = 0; i < elevatorMeshes.length; i++) {
    const state = elevators[i]
    const mesh = elevatorMeshes[i]
    if (!mesh) continue

    // Set the target Y position for the animation loop to smoothly interpolate to.
    mesh.targetY = state.currentFloor * floorHeight + floorHeight / 2
    mesh.material = state.doorsOpen ? cabMatOpen : cabMatIdle
  }
}

// --- Lifecycle Hooks --- //
onMounted(() => {
  const host = canvasHost.value
  if (!host) return

  // --- Scene and Camera --- //
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf3f5f7)
  camera = new THREE.PerspectiveCamera(65, host.clientWidth / host.clientHeight, 0.1, 1000)
  camera.position.set(40, 25, 40)

  // --- Renderer --- //
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(host.clientWidth, host.clientHeight)
  host.appendChild(renderer.domElement)

  // CSS2D Renderer for labels
  labelRenderer = new CSS2DRenderer()
  labelRenderer.setSize(host.clientWidth, host.clientHeight)
  labelRenderer.domElement.style.position = 'absolute'
  labelRenderer.domElement.style.top = '0px'
  labelRenderer.domElement.style.pointerEvents = 'none' // Let mouse events pass through to the 3D canvas
  host.appendChild(labelRenderer.domElement)

  // --- Controls --- //
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.08
  controls.minDistance = 20
  controls.maxDistance = 120
  controls.target.set(0, 18, 0)

  // --- Lighting --- //
  scene.add(new THREE.AmbientLight(0xffffff, 0.7))
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.9)
  dirLight.position.set(30, 40, 20)
  scene.add(dirLight)

  // --- Building Geometry --- //
  const buildingWidth = 24
  const buildingDepth = 14
  const buildingHeight = FLOORS * floorHeight

  // Transparent shell
  const shellGeom = new THREE.BoxGeometry(buildingWidth, buildingHeight, buildingDepth)
  const shellMat = new THREE.MeshPhongMaterial({ color: 0xdddddd, transparent: true, opacity: 0.15, side: THREE.DoubleSide })
  const shell = new THREE.Mesh(shellGeom, shellMat)
  shell.position.set(0, buildingHeight / 2, 0)
  scene.add(shell)

  // Floor planes
  for (let i = 0; i < FLOORS; i++) {
    const y = i * floorHeight
    const floorGeom = new THREE.PlaneGeometry(buildingWidth - 1.5, buildingDepth - 1.5)
    const floorMat = new THREE.MeshBasicMaterial({ color: 0xbfc7d1, transparent: true, opacity: 0.25, side: THREE.DoubleSide })
    const floor = new THREE.Mesh(floorGeom, floorMat)
    floor.rotation.x = -Math.PI / 2
    floor.position.set(0, y + 0.02, 0)
    scene.add(floor)
    const edges = new THREE.LineSegments(new THREE.EdgesGeometry(floorGeom), new THREE.LineBasicMaterial({ color: 0xa0a7b1 }))
    edges.rotation.x = -Math.PI / 2
    edges.position.set(0, y + 0.02, 0)
    scene.add(edges)
  }

  // Shafts
  const shaftWidth = 3.2
  const shaftDepth = 3.2
  const shaftX = [-7.5, 0, 7.5]
  shaftX.forEach((xPos) => {
    const shaftGeom = new THREE.BoxGeometry(shaftWidth, buildingHeight - 0.4, shaftDepth)
    const shaftEdges = new THREE.LineSegments(new THREE.EdgesGeometry(shaftGeom), new THREE.LineBasicMaterial({ color: 0x6b7a89 }))
    shaftEdges.position.set(xPos, buildingHeight / 2, 0)
    scene.add(shaftEdges)
  })

  // Floor Labels
  for (let i = 0; i < FLOORS; i++) {
    const floorDiv = document.createElement('div')
    floorDiv.className = 'floor-label'
    floorDiv.textContent = `${i}`
    floorDiv.style.color = '#36404a'
    floorDiv.style.fontSize = '14px'
    floorDiv.style.textShadow = '0 0 4px white, 0 0 4px white' // Make it more readable

    const floorLabel = new CSS2DObject(floorDiv)
    floorLabel.position.set(buildingWidth / 2 + 1.5, i * floorHeight, 0)
    scene.add(floorLabel)
  }

  // Elevator Cabs
  const cabGeom = new THREE.BoxGeometry(2.4, elevatorCabHalfHeight * 2, 2.4)
  cabMatIdle = new THREE.MeshStandardMaterial({ color: 0x4466ff, metalness: 0.3, roughness: 0.6 })
  cabMatOpen = new THREE.MeshStandardMaterial({ color: 0x4caf50, metalness: 0.2, roughness: 0.8 })
  elevatorMeshes = shaftX.map((x) => {
    const mesh = new THREE.Mesh(cabGeom, cabMatIdle)
    mesh.position.set(x, elevatorCabHalfHeight, 0) // Start at floor 0
    scene.add(mesh)
    return mesh
  })

  // Ground
  const ground = new THREE.Mesh(new THREE.PlaneGeometry(160, 160), new THREE.MeshPhongMaterial({ color: 0xe8ebef, side: THREE.DoubleSide }))
  ground.rotation.x = -Math.PI / 2
  ground.position.y = -0.001
  scene.add(ground)
  const grid = new THREE.GridHelper(160, 40, 0x999999, 0xcccccc)
  grid.position.y = 0
  scene.add(grid)

  // --- Start Animation and Logic --- //
  const moveSpeed = 0.08 // Controls the smoothness of elevator movement
  let running = true
  function animate() {
    if (!running) return
    requestAnimationFrame(animate)

    // Smoothly move elevators
    elevatorMeshes.forEach((mesh, i) => {
      if (mesh.targetY !== undefined) {
        mesh.position.y += (mesh.targetY - mesh.position.y) * moveSpeed
      }

      // Dynamically update panel position to stay to the right of the car from camera's view
      const panel = scene.getObjectByName(`panel-${i}`)
      if (panel) {
        const right = new THREE.Vector3()
        camera.getWorldDirection(right) // Get camera's forward vector
        right.cross(camera.up).normalize() // Cross with up to get right vector
        const offset = right.multiplyScalar(7) // 7 units to the right
        panel.position.copy(mesh.position).add(offset)
      }
    })

    controls.update()
    renderer.render(scene, camera)
    labelRenderer.render(scene, camera) // Render the labels
  }
  animate()

  // Set initial state and start the logic timer
  tick()
  tickTimer = setInterval(tick, TICK_MS)

  // --- Event Listeners --- //
  const onResize = () => {
    const w = host.clientWidth
    const h = host.clientHeight
    camera.aspect = w / h
    camera.updateProjectionMatrix()
    renderer.setSize(w, h)
    labelRenderer.setSize(w, h)
  }
  window.addEventListener('resize', onResize)

  // --- Cleanup --- //
  onUnmounted(() => {
    running = false
    if (tickTimer) clearInterval(tickTimer)
    window.removeEventListener('resize', onResize)
    controls.dispose()
    renderer.dispose()
    // No dispose method on CSS2DRenderer, just remove the element
    if (labelRenderer && labelRenderer.domElement.parentElement === host) {
      host.removeChild(labelRenderer.domElement)
    }
    if (renderer.domElement.parentElement === host) {
      host.removeChild(renderer.domElement)
    }
    // Dispose geometries and materials if needed
  })
})

// --- UI Methods --- //
const floors = computed(() => Array.from({ length: FLOORS }, (_, i) => FLOORS - 1 - i))
function reset() {
  for (let i = 0; i < elevators.length; i++) {
    Object.assign(elevators[i], createElevator(i))
  }
  for (let f = 0; f < FLOORS; f++) {
    floorCalls[f].up = false
    floorCalls[f].down = false
  }
  tick() // Sync view immediately after reset
}
</script>

<template>
  <div class="three-host" ref="canvasHost">
    <div class="overlay">
      <div class="title">3D 大楼 · 三口电梯</div>
      <div class="hint">拖拽旋转 · 滚轮缩放 · 点击楼层呼叫</div>

      <!-- Floor Call Panel -->
      <div class="floors">
        <div v-for="f in floors" :key="f" class="floor">
          <span class="label">{{ f }}</span>
          <div class="calls">
            <button
              class="btn-up"
              :class="{ active: floorCalls[f]?.up }"
              @click="call(f, 'up')"
              :disabled="f === FLOORS - 1"
            >
              ⬆️
            </button>
            <button
              class="btn-down"
              :class="{ active: floorCalls[f]?.down }"
              @click="call(f, 'down')"
              :disabled="f === 0"
            >
              ⬇️
            </button>
          </div>
        </div>
      </div>

      <!-- Elevator Status Panel -->
      <div class="stats">
        <div v-for="e in elevators" :key="e.id" class="stat">
          <strong>{{ e.name }}</strong>
          <div>楼层: {{ e.currentFloor }}</div>
          <div>方向: {{ e.direction }}</div>
          <div>队列: {{ e.targets.join(',') || '-' }}</div>
          <div :class="['door-status', { open: e.doorsOpen }]">
            {{ e.doorsOpen ? '门开' : '门关' }}
          </div>
        </div>
      </div>

      <div class="ops">
        <button class="reset" @click="reset">重置状态</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.three-host {
  width: min(1200px, 95vw);
  height: min(720px, 85vh);
  position: relative;
  border: 1px solid #dfe3e8;
  border-radius: 8px;
  background: #f3f5f7;
  overflow: hidden;
  box-shadow: 0 8px 20px rgba(0,0,0,0.06);
}

.overlay {
  position: absolute;
  left: 12px;
  top: 10px;
  z-index: 10;
  background: rgba(255,255,255,0.8);
  border: 1px solid #e6e9ee;
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 13px;
  color: #36404a;
  backdrop-filter: blur(3px);
  display: grid;
  grid-template-columns: auto auto;
  gap: 12px 20px;
  user-select: none;
}
.title {
  grid-column: 1 / -1;
  font-size: 16px;
  font-weight: 600;
  border-bottom: 1px solid #eee;
  padding-bottom: 6px;
}
.hint {
  grid-column: 1 / -1;
  font-size: 12px;
  color: #777;
  margin-top: -8px;
}

.floors {
  display: flex;
  flex-direction: column;
  max-height: 420px;
  overflow-y: auto;
  padding-right: 5px;
}
.floor { display: flex; align-items: center; gap: 8px; height: 28px; }
.label { width: 24px; font-weight: 600; text-align: right; }
.calls { display: flex; gap: 6px; }
.calls button {
  width: 30px;
  height: 24px;
  padding: 0;
  font-size: 14px;
  line-height: 1;
  border-radius: 4px;
  background-color: #f0f2f5;
  transition: background-color 0.2s;
}
.calls button:hover {
  background-color: #e4e6e9;
}
.calls button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
.calls .active {
  background: #ffd54f;
  border-color: #fbc02d;
}

.stats {
  display: grid;
  gap: 8px;
  align-content: start;
}
.stat {
  background: #f7f9fb;
  border: 1px solid #e6e9ee;
  border-radius: 4px;
  padding: 6px 10px;
  font-size: 12px;
}
.door-status {
  font-weight: bold;
  color: #d32f2f;
}
.door-status.open {
  color: #388e3c;
}

.ops {
  grid-column: 1 / -1;
}
.reset {
  padding: 6px 12px;
  width: 100%;
  background-color: #e53935;
  color: white;
  border: none;
  border-radius: 4px;
}
.reset:hover {
  background-color: #c62828;
}

.floor-label {
  color: #ffffff;
  font-family: Arial, sans-serif;
  font-size: 14px;
  text-shadow: 1px 1px 2px black;
}
</style>

<style>
/* 
Styles for elements rendered by CSS2DRenderer (like in-car panels) must be global,
as they are appended to the body/host outside of the Vue component's scoped style context.
*/
.in-car-panel {
  pointer-events: auto; /* Make the panel itself clickable */
  background: rgba(0, 0, 0, 0.5);
  border-radius: 5px;
  padding: 5px;
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 4px;
  width: 100px;
}

.in-car-panel button {
  background: #555;
  color: white;
  border: 1px solid #777;
  border-radius: 3px;
  cursor: pointer;
  font-size: 12px;
  padding: 2px;
}

.in-car-panel button:hover {
  background: #666;
}

.in-car-panel button.active {
  background: #f9a825;
  color: black;
  border-color: #fbc02d;
}
</style>