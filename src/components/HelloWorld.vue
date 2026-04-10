<template>
  <div class="fireworks-container" @click="launchFirework">
    <canvas ref="canvas"></canvas>
    <div class="hint">Click anywhere to launch fireworks!</div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  data() {
    return {
      ctx: null,
      width: 0,
      height: 0,
      particles: [],
      rockets: [],
      animationId: null,
      hue: 0,
      autoLaunchTimer: null
    }
  },
  mounted() {
    this.initCanvas()
    this.animate()
    this.autoLaunch()
    window.addEventListener('resize', this.initCanvas)
  },
  beforeDestroy() {
    cancelAnimationFrame(this.animationId)
    clearInterval(this.autoLaunchTimer)
    window.removeEventListener('resize', this.initCanvas)
  },
  methods: {
    initCanvas() {
      const canvas = this.$refs.canvas
      this.width = canvas.width = window.innerWidth
      this.height = canvas.height = window.innerHeight
      this.ctx = canvas.getContext('2d')
    },
    autoLaunch() {
      // Launch 2-3 fireworks at a time, every 400ms
      this.autoLaunchTimer = setInterval(() => {
        const count = 2 + Math.floor(Math.random() * 2)
        for (let i = 0; i < count; i++) {
          const x = Math.random() * this.width * 0.8 + this.width * 0.1
          this.createRocket(x, this.height)
        }
      }, 400)
    },
    launchFirework(e) {
      this.createRocket(e.clientX, this.height)
    },
    createRocket(x, y) {
      const targetY = Math.random() * this.height * 0.3 + this.height * 0.1
      const targetX = x + (Math.random() - 0.5) * 100
      this.rockets.push({
        x, y,
        targetX, targetY,
        speed: 3 + Math.random() * 2,
        angle: Math.atan2(targetY - y, targetX - x),
        hue: Math.random() * 360,
        trail: []
      })
    },
    explode(rocket) {
      const count = 80 + Math.floor(Math.random() * 60)
      const hue = rocket.hue
      for (let i = 0; i < count; i++) {
        const angle = (Math.PI * 2 / count) * i + Math.random() * 0.5
        const speed = Math.random() * 6 + 2
        const life = 60 + Math.random() * 40
        this.particles.push({
          x: rocket.x,
          y: rocket.y,
          vx: Math.cos(angle) * speed,
          vy: Math.sin(angle) * speed,
          life,
          maxLife: life,
          hue: hue + Math.random() * 30 - 15,
          saturation: 50 + Math.random() * 50,
          brightness: 50 + Math.random() * 30,
          size: Math.random() * 2.5 + 0.5,
          decay: 0.015 + Math.random() * 0.015
        })
      }
      // Ring effect
      if (Math.random() > 0.5) {
        for (let i = 0; i < 36; i++) {
          const angle = (Math.PI * 2 / 36) * i
          const speed = 4 + Math.random()
          this.particles.push({
            x: rocket.x,
            y: rocket.y,
            vx: Math.cos(angle) * speed,
            vy: Math.sin(angle) * speed,
            life: 40,
            maxLife: 40,
            hue: hue + 180,
            saturation: 100,
            brightness: 70,
            size: 2,
            decay: 0.025
          })
        }
      }
    },
    animate() {
      const ctx = this.ctx
      // Semi-transparent overlay for trail effect
      ctx.fillStyle = 'rgba(0, 0, 0, 0.15)'
      ctx.fillRect(0, 0, this.width, this.height)

      // Update and draw rockets
      for (let i = this.rockets.length - 1; i >= 0; i--) {
        const r = this.rockets[i]
        r.trail.push({ x: r.x, y: r.y })
        if (r.trail.length > 8) r.trail.shift()

        r.x += Math.cos(r.angle) * r.speed
        r.y += Math.sin(r.angle) * r.speed

        // Draw rocket trail
        for (let t = 0; t < r.trail.length; t++) {
          const alpha = t / r.trail.length
          ctx.beginPath()
          ctx.arc(r.trail[t].x, r.trail[t].y, 1.5, 0, Math.PI * 2)
          ctx.fillStyle = `hsla(${r.hue}, 100%, 70%, ${alpha})`
          ctx.fill()
        }

        // Draw rocket head
        ctx.beginPath()
        ctx.arc(r.x, r.y, 2.5, 0, Math.PI * 2)
        ctx.fillStyle = `hsl(${r.hue}, 100%, 80%)`
        ctx.fill()

        if (r.y <= r.targetY) {
          this.explode(r)
          this.rockets.splice(i, 1)
        }
      }

      // Update and draw particles
      for (let i = this.particles.length - 1; i >= 0; i--) {
        const p = this.particles[i]
        p.x += p.vx
        p.y += p.vy
        p.vy += 0.04 // gravity
        p.vx *= 0.99  // friction
        p.vy *= 0.99
        p.life -= 1

        if (p.life <= 0) {
          this.particles.splice(i, 1)
          continue
        }

        const alpha = p.life / p.maxLife
        const radius = Math.max(p.size * alpha, 0.1)
        ctx.beginPath()
        ctx.arc(p.x, p.y, radius, 0, Math.PI * 2)
        ctx.fillStyle = `hsla(${p.hue}, ${p.saturation}%, ${p.brightness}%, ${alpha})`
        ctx.fill()

        // Sparkle glow
        if (alpha > 0.3) {
          ctx.beginPath()
          ctx.arc(p.x, p.y, radius * 3, 0, Math.PI * 2)
          ctx.fillStyle = `hsla(${p.hue}, ${p.saturation}%, ${p.brightness}%, ${alpha * 0.15})`
          ctx.fill()
        }
      }

      // Draw ground
      this.drawGround(ctx)
      // Draw bench and couple sitting on it
      const benchX = this.width * 0.15
      const groundY = this.height - 35
      this.drawBench(ctx, benchX, groundY)
      this.drawSeatedPerson(ctx, benchX - 10, groundY, true, '#e74c3c')
      this.drawSeatedPerson(ctx, benchX + 10, groundY, false, '#3498db')
      this.drawHeart(ctx, benchX, groundY)

      this.animationId = requestAnimationFrame(this.animate)
    },
    drawGround(ctx) {
      // Grass hill silhouette
      ctx.fillStyle = '#0a1a0a'
      ctx.beginPath()
      ctx.moveTo(0, this.height)
      ctx.lineTo(0, this.height - 30)
      ctx.quadraticCurveTo(this.width * 0.15, this.height - 50, this.width * 0.3, this.height - 25)
      ctx.quadraticCurveTo(this.width * 0.5, this.height - 10, this.width * 0.7, this.height - 25)
      ctx.quadraticCurveTo(this.width * 0.85, this.height - 50, this.width, this.height - 30)
      ctx.lineTo(this.width, this.height)
      ctx.closePath()
      ctx.fill()
    },
    drawBench(ctx, x, y) {
      const s = Math.min(this.width, this.height) / 500
      ctx.save()
      ctx.translate(x, y)
      ctx.scale(s, s)

      // Bench seat (wooden plank)
      ctx.fillStyle = '#5c3a1e'
      ctx.fillRect(-30, -2, 60, 5)
      // Seat highlight
      ctx.fillStyle = '#7a5230'
      ctx.fillRect(-28, -2, 56, 2)

      // Back rest
      ctx.fillStyle = '#5c3a1e'
      ctx.fillRect(-28, -22, 3, 20)
      ctx.fillRect(25, -22, 3, 20)
      ctx.fillRect(-28, -20, 56, 3)
      ctx.fillRect(-28, -12, 56, 3)

      // Legs
      ctx.fillStyle = '#3e2712'
      ctx.fillRect(-25, 3, 3, 18)
      ctx.fillRect(22, 3, 3, 18)
      // Cross bars
      ctx.fillRect(-25, 14, 50, 2)

      ctx.restore()
    },
    drawSeatedPerson(ctx, x, y, facingRight, color) {
      const s = Math.min(this.width, this.height) / 500
      const dir = facingRight ? 1 : -1

      ctx.save()
      ctx.translate(x, y)
      ctx.scale(s, s)

      // Head - tilted up looking at fireworks
      ctx.beginPath()
      ctx.arc(dir * 2, -38, 6, 0, Math.PI * 2)
      ctx.fillStyle = '#f5d0a9'
      ctx.fill()

      // Hair
      ctx.beginPath()
      ctx.arc(dir * 2, -41, 6, Math.PI, Math.PI * 2)
      ctx.fillStyle = facingRight ? '#3a2a1a' : '#5a3825'
      ctx.fill()
      // Girl longer hair
      if (!facingRight) {
        ctx.beginPath()
        ctx.moveTo(-4, -41)
        ctx.quadraticCurveTo(-9, -30, -7, -22)
        ctx.lineTo(-4, -24)
        ctx.quadraticCurveTo(-6, -32, -2, -40)
        ctx.fillStyle = '#5a3825'
        ctx.fill()
      }

      // Body (torso sitting upright)
      ctx.beginPath()
      ctx.moveTo(0, -32)
      ctx.lineTo(-5, -8)
      ctx.lineTo(5, -8)
      ctx.closePath()
      ctx.fillStyle = color
      ctx.fill()

      // Seated legs (bent at knee, extending forward)
      ctx.strokeStyle = '#2c3e50'
      ctx.lineWidth = 3
      ctx.lineCap = 'round'
      // Thigh (horizontal on bench)
      ctx.beginPath()
      ctx.moveTo(-2, -6)
      ctx.lineTo(dir * 8, -4)
      ctx.stroke()
      ctx.beginPath()
      ctx.moveTo(2, -6)
      ctx.lineTo(dir * 10, -3)
      ctx.stroke()
      // Lower legs (hanging down)
      ctx.beginPath()
      ctx.moveTo(dir * 8, -4)
      ctx.lineTo(dir * 6, 16)
      ctx.stroke()
      ctx.beginPath()
      ctx.moveTo(dir * 10, -3)
      ctx.lineTo(dir * 9, 16)
      ctx.stroke()
      // Shoes
      ctx.fillStyle = '#1a1a2e'
      ctx.beginPath()
      ctx.ellipse(dir * 6, 17, 3, 1.5, 0, 0, Math.PI * 2)
      ctx.fill()
      ctx.beginPath()
      ctx.ellipse(dir * 9, 17, 3, 1.5, 0, 0, Math.PI * 2)
      ctx.fill()

      // Inner arm - leaning on partner / holding hands
      ctx.strokeStyle = '#f5d0a9'
      ctx.lineWidth = 2.5
      ctx.beginPath()
      ctx.moveTo(-dir * 1, -26)
      ctx.lineTo(-dir * 8, -16)
      ctx.stroke()

      // Outer arm - resting or pointing up
      ctx.beginPath()
      ctx.moveTo(dir * 2, -28)
      ctx.lineTo(dir * 14, -44)
      ctx.stroke()
      // Hand dot
      ctx.beginPath()
      ctx.arc(dir * 14, -44, 1.8, 0, Math.PI * 2)
      ctx.fillStyle = '#f5d0a9'
      ctx.fill()

      ctx.restore()
    },
    drawHeart(ctx, x, y) {
      const s = Math.min(this.width, this.height) / 500
      const time = Date.now() / 1000
      const beat = 1 + Math.sin(time * 3) * 0.15 // heartbeat pulse

      ctx.save()
      ctx.translate(x, y)
      ctx.scale(s * beat, s * beat)

      // Small floating heart above their heads
      const floatY = -72 + Math.sin(time * 1.5) * 3
      ctx.translate(0, floatY)

      ctx.fillStyle = '#ff4466'
      ctx.beginPath()
      ctx.moveTo(0, -3)
      ctx.bezierCurveTo(-5, -8, -10, -3, 0, 5)
      ctx.moveTo(0, -3)
      ctx.bezierCurveTo(5, -8, 10, -3, 0, 5)
      ctx.fill()

      // Heart glow
      ctx.shadowColor = '#ff4466'
      ctx.shadowBlur = 8
      ctx.fill()

      ctx.restore()
    }
  }
}
</script>

<style scoped>
.fireworks-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: #000;
  overflow: hidden;
  cursor: crosshair;
}
canvas {
  display: block;
}
.hint {
  position: absolute;
  bottom: 30px;
  width: 100%;
  text-align: center;
  color: rgba(255, 255, 255, 0.5);
  font-size: 16px;
  font-family: Arial, sans-serif;
  pointer-events: none;
  letter-spacing: 2px;
}
</style>
