<template>
  <main id="sidebar" ref="sidebar">
    <div class="content">
      <div class="content__header">
        <h1>lazy-brush</h1>
      </div>
      <div class="content__aside">
        <p>
          Draw smooth curves and straight lines with your mouse, finger or any
          pointing device.
        </p>

        <div class="slider">
          <label for="lazy">Brush radius</label>
          <input
            class="range"
            type="range"
            id="slider_brush"
            name="radius"
            min="2"
            max="50"
            v-model="brushRadius"
            ref="slider_brush"
            @input="handleSliderBrush"
          />
        </div>

        <div class="slider">
          <label for="lazy">Lazy radius</label>
          <input
            class="range"
            type="range"
            id="slider_lazy"
            name="lazy"
            min="2"
            max="175"
            v-model="chainLength"
            ref="slider_lazy"
            @input="handleSliderLazy"
          />
        </div>
      </div>

      <div class="flex content__bottom">
        <div class="flex-auto">
          <button
            class="button-disable"
            id="button_lazy"
            ref="button_lazy"
            @click="handleButtonLazy"
          >
            On
          </button>
        </div>
        <div class="flex-1">
          <button
            class="button-clear"
            id="button_clear"
            ref="button_clear"
            @click="handleButtonClear"
          >
            Clear
          </button>
        </div>

        <div class="flex-auto">
          <a
            href="https://github.com/dulnan/lazy-brush"
            class="button button-github"
          >
            <svg
              aria-labelledby="simpleicons-github-icon"
              role="img"
              viewBox="0 0 24 24"
              xmlns="http://www.w3.org/2000/svg"
            >
              <title id="simpleicons-github-icon">GitHub icon</title>
              <path
                d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"
              />
            </svg>
          </a>
        </div>
        <div class="flex-auto hidden-desktop">
          <button
            class="button-menu"
            id="button_menu"
            ref="button_menu"
            @click="handleButtonMenu"
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              width="24px"
              height="24px"
            >
              <path
                d="M 12 3 A 2 2 0 0 0 10 5 A 2 2 0 0 0 12 7 A 2 2 0 0 0 14 5 A 2 2 0 0 0 12 3 z M 12 10 A 2 2 0 0 0 10 12 A 2 2 0 0 0 12 14 A 2 2 0 0 0 14 12 A 2 2 0 0 0 12 10 z M 12 17 A 2 2 0 0 0 10 19 A 2 2 0 0 0 12 21 A 2 2 0 0 0 14 19 A 2 2 0 0 0 12 17 z"
              />
            </svg>
          </button>
        </div>
      </div>
    </div>
  </main>

  <div class="canvas-container" id="canvas_container" ref="canvasContainer">
    <canvas
      class="lazy-canvas"
      ref="canvas_interface"
      @mousedown="handlePointerDown"
      @mouseup="handlePointerUp"
      @mousemove="(e) => handlePointerMove(e.clientX, e.clientY)"
      @contextmenu="(e) => handleContextMenu(e)"
      @touchstart="(e) => handleTouchStart(e)"
      @touchend="(e) => handleTouchEnd(e)"
      @touchmove="(e) => handleTouchMove(e)"
    ></canvas>
    <canvas class="lazy-canvas" ref="canvas_drawing"></canvas>
    <canvas class="lazy-canvas" ref="canvas_temp"></canvas>
    <canvas class="lazy-canvas" ref="canvas_grid"></canvas>
  </div>
</template>

<script setup>
import { onMounted, ref, reactive } from "vue"
// import Scene from "../classes/Scene.js"

import { LazyBrush, Point } from "lazy-brush" //笔刷
import { Catenary } from "catenary-curve" //悬线链
import ResizeObserver from "resize-observer-polyfill"
const styleVariables = {
  colorPrimary: "#f2530b",
  colorBlack: "#0a0302",
  colorCatenary: "#0a0302",
}
const LAZY_RADIUS = 60
const BRUSH_RADIUS = 12.5

function midPointBtw(p1, p2) {
  return {
    x: p1.x + (p2.x - p1.x) / 2,
    y: p1.y + (p2.y - p1.y) / 2,
  }
}

const sidebar = ref("")
const canvasContainer = ref("")

const countext = reactive({
  interface: ref(""),
  drawing: ref(""),
  temp: ref(""),
  grid: ref(""),
})

// 四个canvas
const catenary = new Catenary()
const lazy = new LazyBrush({
  radius: LAZY_RADIUS,
  enabled: true,
  initialPoint: {
    x: window.innerWidth / 2,
    y: window.innerHeight / 2,
  },
})

let mouseHasMoved = true
let valuesChanged = true
let isDrawing = false
let isPressing = false

const points = []

let brushRadius = ref(BRUSH_RADIUS) //线条宽度
let chainLength = ref(LAZY_RADIUS) //链子长度

const canvas_interface = ref("")
const canvas_drawing = ref("")
const canvas_temp = ref("")
const canvas_grid = ref("")

let context = {}

let dpi = 1

onMounted(() => {
  context.interface = canvas_interface.value.getContext("2d")
  context.drawing = canvas_drawing.value.getContext("2d")
  context.temp = canvas_temp.value.getContext("2d")
  context.grid = canvas_grid.value.getContext("2d")

  const observeCanvas = new ResizeObserver((entries, observer) =>
    handleCanvasResize(entries, observer)
  )
  observeCanvas.observe(canvasContainer.value)
  const observeSidebar = new ResizeObserver((entries, observer) =>
    handleSidebarResize(entries, observer)
  )
  observeSidebar.observe(sidebar.value)

  loop()

  window.setTimeout(() => {
    const initX = window.innerWidth / 2
    const initY = window.innerHeight / 2
    lazy.update({ x: initX - chainLength / 4, y: initY }, { both: true })
    lazy.update({ x: initX + chainLength / 4, y: initY }, { both: false })
    mouseHasMoved = true
    valuesChanged = true
    // clearCanvas()
  }, 100)

  function handleCanvasResize(entries, observer) {
    dpi = window.devicePixelRatio

    for (const entry of entries) {
      const { width, height } = entry.contentRect

      setCanvasSize(canvas_interface.value, width, height, 1.25)
      setCanvasSize(canvas_drawing.value, width, height, 1)
      setCanvasSize(canvas_temp.value, width, height, 1)
      setCanvasSize(canvas_grid.value, width, height, 2)

      drawGrid(context.grid)
      loop({ once: true })
    }
  }
})

function handleTouchStart(e) {
  const x = e.changedTouches[0].clientX
  const y = e.changedTouches[0].clientY
  lazy.update({ x: x, y: y }, { both: true })
  handlePointerDown(e)

  mouseHasMoved = true
}

function handleTouchMove(e) {
  e.preventDefault()
  handlePointerMove(e.changedTouches[0].clientX, e.changedTouches[0].clientY)
}

function handleTouchEnd(e) {
  handlePointerUp(e)
  const brush = lazy.getBrushCoordinates()
  lazy.update({ x: brush.x, y: brush.y }, { both: true })
  mouseHasMoved = true
}

// 右键
function handleContextMenu(e) {
  e.preventDefault()
  if (e.button === 2) {
    clearCanvas()
  }
}

function handleButtonMenu(e) {
  e.preventDefault()
  document.body.classList.toggle("menu-visible")
}

function handleButtonClear(e) {
  e.preventDefault()
  clearCanvas()
}

function handleButtonLazy(e) {
  e.preventDefault()
  valuesChanged = true
  button_lazy.value.classList.toggle("disabled")

  if (lazy.isEnabled()) {
    button_lazy.value.innerHTML = "Off"
    lazy.disable()
  } else {
    button_lazy.value.innerHTML = "On"
    lazy.enable()
  }
}

function handleSidebarResize(entries, observer) {
  for (const entry of entries) {
    const { left, top, width, height } = entry.contentRect
    loop({ once: true })
  }
}

function handleSliderBrush(e) {
  const val = parseInt(e.target.value)
  valuesChanged = true
  brushRadius = val
  console.log(
    "🚀 ~ file: HelloWorld.vue ~ line 294 ~ handleSliderBrush ~ val",
    val
  )
}

function handleSliderLazy(e) {
  valuesChanged = true
  const val = parseInt(e.target.value)
  chainLength = val
  lazy.setRadius(val)
  console.log(
    "🚀 ~ file: HelloWorld.vue ~ line 306 ~ handleSliderLazy ~ val",
    val
  )
}

function setCanvasSize(canvas, width, height, maxDpi = 4) {
  let dp = dpi
  // reduce canvas size for hidpi desktop screens
  if (window.innerWidth > 1024) {
    dp = Math.min(dpi, maxDpi)
  }

  canvas.width = width * dp
  canvas.height = height * dp
  canvas.style.width = width
  canvas.style.height = height
  canvas.getContext("2d").scale(dp, dp)
}

function handlePointerDown(e) {
  e.preventDefault()
  isPressing = true
}

function handlePointerUp(e) {
  e.preventDefault()
  isDrawing = false
  isPressing = false
  points.length = 0

  const dpi = window.innerWidth > 1024 ? 1 : window.devicePixelRatio
  const width = canvas.temp.width / dpi
  const height = canvas.temp.height / dpi

  context.drawing.drawImage(canvas.temp, 0, 0, width, height)
  context.temp.clearRect(0, 0, width, height)
}

// 鼠标移动
function handlePointerMove(x, y) {
  const hasChanged = lazy.update({ x: x, y: y })
  const isDisabled = !lazy.isEnabled()

  context.temp.lineJoin = "round"
  context.temp.lineCap = "round"
  context.temp.strokeStyle = styleVariables.colorPrimary

  if ((isPressing && hasChanged && !isDrawing) || (isDisabled && isPressing)) {
    isDrawing = true
    points.push(lazy.brush.toObject())
  }

  if (isDrawing && (lazy.brushHasMoved() || isDisabled)) {
    context.temp.clearRect(
      0,
      0,
      context.temp.canvas.width,
      context.temp.canvas.height
    )
    context.temp.lineWidth = brushRadius * 2
    points.push(lazy.brush.toObject())

    var p1 = points[0]
    var p2 = points[1]

    context.temp.moveTo(p2.x, p2.y)
    context.temp.beginPath()

    for (var i = 1, len = points.length; i < len; i++) {
      // we pick the point between pi+1 & pi+2 as the
      // end point and p1 as our control point
      var midPoint = midPointBtw(p1, p2)
      context.temp.quadraticCurveTo(p1.x, p1.y, midPoint.x, midPoint.y)
      p1 = points[i]
      p2 = points[i + 1]
    }
    // Draw last line as a straight line while
    // we wait for the next point to be able to calculate
    // the bezier control point
    context.temp.lineTo(p1.x, p1.y)
    context.temp.stroke()
  }

  mouseHasMoved = true
}

function clearCanvas() {
  valuesChanged = true
  context.drawing.clearRect(
    0,
    0,
    canvas_drawing.value.width,
    canvas_drawing.value.height
  )
  context.temp.clearRect(
    0,
    0,
    canvas_temp.value.width,
    canvas_temp.value.height
  )
  console.log(
    "🚀 ~ file: HelloWorld.vue ~ line 394 ~ clearCanvas ~ clearCanvas",
    "clearCanvas"
  )
}

function loop({ once = false } = {}) {
  if (mouseHasMoved || valuesChanged) {
    const pointer = lazy.getPointerCoordinates()
    const brush = lazy.getBrushCoordinates()

    drawInterface(context.interface, pointer, brush)
    mouseHasMoved = false
    valuesChanged = false
  }

  if (!once) {
    window.requestAnimationFrame(() => {
      loop()
    })
  }
}

// 画网格
function drawGrid(ctx) {
  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)

  ctx.beginPath()
  ctx.setLineDash([5, 1])
  ctx.setLineDash([])
  // ctx.strokeStyle = styleVariables.colorInterfaceGrid
  ctx.strokeStyle = "rgba(150,150,150,0.17)"
  ctx.lineWidth = 0.5

  const gridSize = 25

  let countX = 0
  while (countX < ctx.canvas.width) {
    countX += gridSize
    ctx.moveTo(countX, 0)
    ctx.lineTo(countX, ctx.canvas.height)
  }
  ctx.stroke()

  let countY = 0
  while (countY < ctx.canvas.height) {
    countY += gridSize
    ctx.moveTo(0, countY)
    ctx.lineTo(ctx.canvas.width, countY)
  }
  ctx.stroke()
}

function drawInterface(ctx, pointer, brush) {
  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height)

  // Draw brush point
  ctx.beginPath()
  ctx.fillStyle = styleVariables.colorPrimary
  ctx.arc(brush.x, brush.y, brushRadius, 0, Math.PI * 2, true)
  ctx.fill()

  // Draw mouse point
  ctx.beginPath()
  ctx.fillStyle = styleVariables.colorBlack
  ctx.arc(pointer.x, pointer.y, 4, 0, Math.PI * 2, true)
  ctx.fill()

  //Draw catharina
  if (lazy.isEnabled()) {
    ctx.beginPath()
    ctx.lineWidth = 2
    ctx.lineCap = "round"
    ctx.setLineDash([2, 4])
    ctx.strokeStyle = styleVariables.colorCatenary
    catenary.drawToCanvas(context.interface, brush, pointer, chainLength)
    ctx.stroke()
  }

  // Draw mouse point
  ctx.beginPath()
  ctx.fillStyle = "#222222"
  ctx.arc(brush.x, brush.y, 2, 0, Math.PI * 2, true)
  ctx.fill()
}
</script>

<style lang="scss" scoped></style>
