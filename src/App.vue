<template>
  <div class="demo">
    <canvas
      ref="map"
      :style="{
        background: '#ddd',
        height: canvasHeight / pixelRatio + 'px',
        width: canvasWidth / pixelRatio + 'px'
      }"
      :width="canvasWidth"
      :height="canvasHeight"
      @wheel="onScroll"
      @mousedown="onMouseDown"
      @mouseup="onMouseUp"
      @mousemove="onMouseMove"
      @mouseover="onMouseOver"
    >
    </canvas>
    <input v-model="api">
  </div>
</template>

<script>
  const demoString = `
 {"debug_info":{"base_address":"CCB46000","uworld_address":"A5B37FC0","persistent_level_address":"B9249000","actors_address":"A2196600","actors_count":61},"timestamp":1524972043,"players":[{"x":793634.94,"y":16548.055,"z":527.1815,"rx":0,"ry":-0,"rz":0.118812986,"rw":0.99291664,"score":0,"name":"BillGaytes"},{"x":793646.6,"y":22693.748,"z":522.11456,"rx":0,"ry":-0,"rz":0.8819213,"rw":0.47139668,"score":0,"name":""},{"x":798984.75,"y":17437.965,"z":522.18,"rx":0,"ry":-0,"rz":0.4821836,"rw":0.8760702,"score":0,"name":""},{"x":798588.25,"y":19550.482,"z":526.15,"rx":0,"ry":-0,"rz":0.6152314,"rw":0.7883466,"score":0,"name":""},{"x":795492,"y":16011.82,"z":527.18,"rx":0,"ry":-0,"rz":0.41642952,"rw":0.909168,"score":0,"name":""},{"x":793762.8,"y":21656.537,"z":544.0913,"rx":0,"ry":-0,"rz":0.6806009,"rw":0.73265433,"score":0,"name":""},{"x":796949.9,"y":15528.714,"z":526.15,"rx":0,"ry":-0,"rz":0.7071067,"rw":0.7071068,"score":0,"name":""},{"x":797644,"y":16537.39,"z":526.14,"rx":0,"ry":-0,"rz":0.49289805,"rw":0.870087,"score":0,"name":""},{"x":794713.5,"y":15285.773,"z":527.18,"rx":-0,"ry":0,"rz":-0.87008715,"rw":0.492898,"score":0,"name":""},{"x":794956.8,"y":16231.308,"z":527.18,"rx":0,"ry":-0,"rz":0.9039892,"rw":0.4275552,"score":0,"name":""},{"x":794542.8,"y":17582.842,"z":527.18,"rx":-0,"ry":0,"rz":-0.42755535,"rw":0.9039892,"score":0,"name":""}]}`
  export default {
    name: 'App',
    props: {
      height: {
        type: Number,
        default: 400
      },
      width: {
        type: Number,
        default: 600
      }
    },
    mounted () {
      this.players = JSON.parse(demoString).players
      this.loadCanvas()
      this.loadImage()

      setInterval(() => {
        if (this.api) {
          fetch(this.api)
            .then(res => res.json())
            .then(data => {
              this.players = data.players
            })
        }
      }, 1000)
    },
    data () {
      return {
        api: '',
        pixelRatio: window.devicePixelRatio,
        canvasHeight: this.height * window.devicePixelRatio,
        canvasWidth: this.width * window.devicePixelRatio,
        mapImage: null,
        indicatorImage: null,
        imageHeight: 0,
        imageWidth: 0,
        indicatorWidth: 0,
        indicatorHeight: 0,
        canvas: null,
        ctx: null,
        focusX: 0,
        focusY: 0,
        scale: 1,
        // renderX: -1,
        // renderY: -1,
        // imageRenderHeight: 0,
        // imageRenderWidth: 0,
        dragging: false,
        lastDragX: -1,
        lastDragY: -1,
        dragOffsetX: 0,
        dragOffsetY: 0,
        overflowAnimation: null,
        players: []
      }
    },
    computed: {
      canvasRatio() {
        return this.canvasWidth / this.canvasHeight
      },
      imageRatio () {
        return this.imageWidth / this.imageHeight
      },
      focusImageX() {
        return this.focusX - this.imageRenderXOrigin
      },
      focusImageY() {
        return this.focusY - this.imageRenderYOrigin
      },
      imageRenderXOrigin() {
        return this.canvasWidth / 2 - this.renderWidth / 2
      },
      imageRenderYOrigin() {
        return this.canvasHeight / 2 - this.renderHeight / 2
      },
      imageRenderWidthOrigin() {
        return this.canvasRatio >= this.imageRatio ? this.canvasHeight : this.imageRenderWidth / this.imageRatio
      },
      imageRenderHeightOrigin() {
        return this.canvasRatio >= this.imageRatio ? this.canvasHeight : this.imageRenderWidth / this.imageRatio
      },
      renderX() {
        return this.imageRenderXOrigin + (this.dragOffsetX * this.scale)
      },
      renderY() {
        return this.imageRenderYOrigin + (this.dragOffsetY * this.scale)
      },
      renderWidth() {
        return this.imageRenderWidthOrigin * this.scale
      },
      renderHeight() {
        return this.imageRenderHeightOrigin * this.scale
      }
    },
    methods: {
      draw() {
        this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
        this.ctx.drawImage(
          this.mapImage,
          this.renderX,
          this.renderY,
          this.renderWidth,
          this.renderHeight)
        this.drawPlayers()
      },
      drawPlayers() {
        this.players.forEach(this.drawPlayer)
      },
      drawPlayer(player) {
        this.ctx.save()
        this.ctx.translate(
          this.renderX + this.convertLocation(player.x) * this.renderWidth / 2048,
          this.renderY + this.convertLocation(player.y) * this.renderHeight / 2048
        )
        this.ctx.rotate(Math.PI * player.rz)
        this.ctx.drawImage(
          this.indicatorImage,
          0,
          0)
        this.ctx.restore()
      },
      resetState() {
        this.focusX = this.canvasWidth / 2
        this.focusY = this.canvasHeight / 2

        this.draw()
      },
      loadCanvas() {
        this.canvas = this.$refs['map']
        this.ctx = this.canvas.getContext('2d')
        this.ctx.imageSmoothingEnabled = false
      },
      loadImage() {
        this.indicatorImage = new Image()
        this.indicatorImage.src = require('./Indicator.svg')
        this.indicatorImage.onload = () => {
          this.indicatorWidth = this.indicatorImage.width * this.pixelRatio
          this.indicatorHeight = this.indicatorImage.height * this.pixelRatio
        }

        this.mapImage = new Image()
        this.mapImage.src = require('./WorldMap.png')
        this.mapImage.onload = () => {
          this.imageWidth = this.mapImage.width * this.pixelRatio
          this.imageHeight = this.mapImage.height * this.pixelRatio

          this.resetState()
        }
      },
      onScroll(e) {
        this.focusX = e.layerX * this.pixelRatio
        this.focusY = e.layerY * this.pixelRatio

        if (e.wheelDelta >= 0) {
          this.scale += 0.1 * this.scale
        } else {
          if (this.scale >= 0.1) {
            this.scale -= 0.1 * this.scale
          }
        }

        setTimeout(() => this.draw())
      },
      onMouseDown(e) {
        this.dragging = true
        this.lastDragX = e.layerX
        this.lastDragY = e.layerY
      },
      onMouseUp() {
        this.dragging = false
      },
      onMouseMove(e) {
        if (this.dragging) {
          this.dragOffsetX += e.layerX - this.lastDragX
          this.dragOffsetY += e.layerY - this.lastDragY

          setTimeout(() => this.draw())

          this.lastDragX = e.layerX
          this.lastDragY = e.layerY
        }
      },
      onMouseOver() {
        this.dragging = false
      },
      convertLocation(num) {
        return 0.002525 * num + 5
      }
    }
  }
</script>

<style>

</style>
