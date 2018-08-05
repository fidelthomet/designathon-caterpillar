<template>
  <div
    ref="worm"
    class="Worm">
    <canvas
      ref="canvas"
      :width="width"
      :height="height"/>
  </div>
</template>

<script>
import * as shape from 'd3-shape'
import * as spp from 'svg-path-properties'
import * as three from 'three'
import debounce from 'lodash.debounce'

export default {
  props: {
    options: {
      type: Object,
      default () {
        return {
          points: 6,
          depth: 8,
          density: 0.1,
          lineLength: 50,
          fill: false,
          speed: 0.5
        }
      }
    }
  },
  data () {
    return {
      points: this.initPoints(),
      levelPoints: [],
      linearPoints: [],
      width: 0,
      height: 0,
      lineCount: 100,
      lines: [],
      resized: false
    }
  },
  computed: {
    path () {
      const line = shape.line()
        .x(d => d.x)
        .y(d => d.y)
        .curve(shape.curveCatmullRomClosed.alpha(0))

      return line(this.points.map(point => ({ ...point.position })))
    },
    path2 () {
      if (this.linearPoints.length === 0) return
      return 'M' + this.linearPoints.map(p => `${p.x},${p.y}`).join('L')
    }
  },
  mounted () {
    const { update, updateDimensions } = this
    requestAnimationFrame(update)
    window.addEventListener('resize', debounce(updateDimensions, 150))
    updateDimensions()
  },
  methods: {
    update (t = 0) {
      const { options, updatePoints, pointsToPath, simplifyPath, pathToAnchors, anchorsToLines, draw, update } = this
      const time = t * 0.0001 * options.speed
      const points = updatePoints(time)
      const path = pointsToPath(points)
      const simplePath = simplifyPath(path)
      const anchors = pathToAnchors(simplePath, time)
      const lines = anchorsToLines(anchors)
      draw(lines)
      requestAnimationFrame(update)
    },
    updateDimensions () {
      const { $refs } = this
      const rect = $refs.worm.getBoundingClientRect()
      this.width = rect.width
      this.height = rect.height
      this.resized = true
    },
    updatePoints (time) {
      const { points, width, height } = this
      return points.map(point => {
        const offset = time + point.base.offset
        let x = (Math.cos(offset) * point.base.radius * width * 0.01) + width * 0.5
        let y = (Math.sin(offset) * point.base.radius * height * 0.01) + height * 0.5
        point.levels.forEach((level, i) => {
          x += (Math.cos(time * (i + 1) + level.offset) * level.radius * width * 0.015)
          y += (Math.sin(time * (i + 1) + level.offset) * level.radius * height * 0.015)
        })
        return { x, y }
      })
    },
    pointsToPath (points) {
      const line = shape.line()
        .x(d => d.x)
        .y(d => d.y)
        .curve(shape.curveCatmullRomClosed.alpha(0))

      return line(points)
    },
    simplifyPath (svgPath, resolution = 50) {
      const svgCurvePoints = svgPath.replace(/M/, '').split('C').map(d => d.split(',').map(d => +d))
      const threeCurves = svgCurvePoints.filter((d, i) => i > 0).map((d, i) => {
        const p1 = new three.Vector2(svgCurvePoints[i][svgCurvePoints[i].length - 2], svgCurvePoints[i][svgCurvePoints[i].length - 1])
        const p2 = new three.Vector2(d[0], d[1])
        const p3 = new three.Vector2(d[2], d[3])
        const p4 = new three.Vector2(d[4], d[5])
        return new three.CubicBezierCurve3(p1, p2, p3, p4)
      })
      const curvePath = new three.CurvePath()
      threeCurves.forEach(p => {
        curvePath.add(p)
      })
      const curvePoints = curvePath.getPoints(resolution)
      return 'M' + curvePoints.map(p => `${p.x},${p.y}`).join('L')
    },
    pathToAnchors (path, time) {
      const { mappable, resized, options } = this
      const properties = spp.svgPathProperties(path)

      const length = properties.getTotalLength()
      if (resized) {
        this.lineCount = Math.floor(length * options.density)
        this.resized = false
        console.log(this.lineCount)
      }
      return mappable(this.lineCount).map(i => {
        const anchorOffset = length / this.lineCount * i
        const timeOffset = length * ((time % (Math.PI * 2)) / (Math.PI * 2))
        return properties.getPointAtLength((length + anchorOffset - timeOffset) % length)
      })
    },
    anchorsToLines (anchors) {
      const { options } = this
      return anchors.map((p, i) => {
        const p1 = anchors[(i + 1) % anchors.length]
        const p2 = anchors[(anchors.length + i - 1) % anchors.length]

        const diff = { x: p1.x - p2.x, y: p1.y - p2.y }
        const length = Math.sqrt(diff.x * diff.x + diff.y * diff.y)
        const orthagonal = { y: -diff.y / length, x: diff.x / length }

        const lineLength = options.lineLength * 1.5 + (Math.sin((i / anchors.length) * Math.PI * 8)) * options.lineLength * 0.55

        return {
          x1: p.x - orthagonal.y * lineLength,
          y1: p.y - orthagonal.x * lineLength,
          x2: p.x + orthagonal.y * lineLength,
          y2: p.y + orthagonal.x * lineLength
        }
      })
    },
    draw (lines) {
      if (this.$refs.canvas == null) return
      const ctx = this.$refs.canvas.getContext('2d')
      ctx.clearRect(0, 0, this.width, this.height)
      ctx.strokeStyle = '#F00C77'
      lines.forEach((line, i) => {
        ctx.beginPath()
        ctx.moveTo(line.x1, line.y1)
        ctx.lineTo(line.x2, line.y2)
        ctx.stroke()
      })
    },
    initPoints () {
      const { options, mappable } = this
      return mappable(options.points).map(i => {
        return {
          base: {
            offset: Math.PI * 2 / options.points * i,
            radius: 24 + Math.random() * 24
          },
          levels: mappable(options.depth).map(i => {
            return {
              offset: Math.PI * 2 * Math.random(),
              radius: 6 + Math.random() * 6 / (i + 1)
            }
          }),
          position: {
            x: 0,
            y: 0
          }
        }
      })
    },
    mappable (length) {
      return '.'.repeat(length).split('').map((x, i) => i)
    }
  }
}
</script>

<style scoped>
.Worm {
  position: fixed;
  width: 100vw;
  height: 100vh;
}

canvas {
  position: absolute;
  width: 100%;
  height: 100%;
}
</style>
