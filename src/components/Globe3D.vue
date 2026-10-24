<template>
  <div class="globe-overlay">
    <div class="globe-toolbar">
      <div class="globe-info">
        <span class="globe-dot"></span>
        <span class="globe-label">3D Globe · {{ t.trendTitle }}</span>
        <span class="globe-hint">{{ hint }}</span>
      </div>
      <button class="globe-back-btn" @click="$emit('close')">{{ backLabel }}</button>
    </div>
    <div class="globe-canvas" ref="globeCanvas" @pointermove="inspectCity" @pointerleave="hoverCity = null" @click="inspectCity">
      <div class="globe-tools">
        <button @click="autoRotate = !autoRotate" :aria-pressed="autoRotate">{{ autoRotate ? '暂停旋转' : '开始旋转' }}</button>
        <button @click="resetView">中国视角</button>
        <button @click="showClouds = !showClouds" :aria-pressed="showClouds">云层</button>
        <button @click="showGrid = !showGrid" :aria-pressed="showGrid">经纬线</button>
      </div>
      <div v-if="hoverCity" class="city-detail">
        <strong>{{ hoverCity.name }}</strong>
        <span class="globe-legend-react">React <b>{{ hoverCity.react || 0 }}</b></span>
        <span class="globe-legend-vue">Vue <b>{{ hoverCity.vue || 0 }}</b></span>
      </div>
      <div v-if="textureError" class="texture-error">部分地表资源加载失败，已显示基础地球。</div>
      <transition name="fade">
        <div class="globe-loading" v-if="textureLoading">
          <div class="globe-spinner"></div>
          <p>Loading satellite imagery…</p>
        </div>
      </transition>
    </div>
    <div class="globe-legend">
      <span class="globe-legend-react">■ React</span>
      <span class="globe-legend-vue">■ Vue</span>
      <span class="globe-legend-arc">柱高表示岗位数量 · 指向城市查看详情</span>
    </div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const GR = 5.5      // globe radius
const MAX_H = 0.85   // max spike height

const TEX = {
  day: '/textures/earth-day.jpg',
  clouds: '/textures/earth-clouds.png',
  specular: '/textures/earth-specular.jpg',
  night: '/textures/earth-night.jpg',
  topo:  '/textures/earth-topology.png',
}

export default {
  name: 'Globe3D',
  props: {
    cityData: { type: Array, default: () => [] },
    t:        { type: Object, default: () => ({}) },
  },
  data() {
    return {
      textureLoading: true,
      textureError: false,
      autoRotate: true,
      showClouds: true,
      showGrid: false,
      hoverCity: null,
      animFrameId:     null,
      scene:           null,
      camera:          null,
      renderer:        null,
      controls:        null,
      globeGroup:      null,
      moonPivot:       null,
      moonMesh:        null,
      twinkleMat:      null,
      userInteracting: false,
      spikePairs:     [],
      time:           0,
    }
  },
  computed: {
    hint()      { return this.t.hint3d  || 'Drag to rotate · Scroll to zoom' },
    backLabel() { return this.t.back2d  || '← 2D View' },
  },
  watch: {
    cityData: {
      deep: true,
      handler() {
        if (!this.cityGroup) return
        this.releaseObjects(this.cityGroup)
        this.cityGroup.clear()
        this.cityLabels = []
        this.cityTargets = []
        this.buildSpikes()
      },
    },
  },
  mounted() {
    this.cityLabels = []
    this.cityTargets = []
    this.loadedTextures = new Set()
    this.raycaster = new THREE.Raycaster()
    this.lastFrame = 0
    this.destroyed = false
    this.$nextTick(() => {
      this.initScene()
      this.animLoop()
    })
    window.addEventListener('resize', this.onResize)
  },
  beforeDestroy() {
    this.destroyed = true
    clearTimeout(this.interactionTimer)
    if (this.animFrameId) cancelAnimationFrame(this.animFrameId)
    window.removeEventListener('resize', this.onResize)
    this.disposeScene()
  },
  methods: {

    /* ─────────────────── coordinate helper ─────────────────── */
    ll2v(lat, lon, r) {
      const phi   = (90 - lat) * Math.PI / 180
      const theta = lon       * Math.PI / 180
      return new THREE.Vector3(
        r * Math.sin(phi) * Math.cos(theta),
        r * Math.cos(phi),
        r * Math.sin(phi) * Math.sin(theta),
      )
    },

    /* ─────────────────── texture helper ─────────────────── */
    // Three.js SphereGeometry UV runs opposite in azimuth to our ll2v.
    // Applying repeat.x = -1 + offset.x = 1 flips U so the texture
    // aligns with geographic coordinates without any mesh rotation.
    loadTex(url, onLoad, color = false) {
      const tex = new THREE.TextureLoader().load(url, loaded => {
        if (this.destroyed) { loaded.dispose(); return }
        if (onLoad) onLoad(loaded)
      }, undefined, () => {
        if (this.destroyed) return
        this.textureError = true
        this.textureLoading = false
      })
      tex.wrapS = THREE.RepeatWrapping
      tex.offset.x = 1
      tex.repeat.x = -1
      tex.colorSpace = color ? THREE.SRGBColorSpace : THREE.NoColorSpace
      tex.anisotropy = Math.min(8, this.renderer.capabilities.getMaxAnisotropy())
      this.loadedTextures.add(tex)
      return tex
    },

    /* ─────────────────── scene bootstrap ─────────────────── */
    initScene() {
      const el = this.$refs.globeCanvas
      const W  = el.clientWidth  || 800
      const H  = el.clientHeight || 600

      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0x01020a)


      this.camera = new THREE.PerspectiveCamera(48, W / H, 0.1, 600)
      this.camera.position.set(0, 8.5, 15.5)
      this.camera.lookAt(0, 0, 0)

      this.renderer = new THREE.WebGLRenderer({ antialias: true })
      this.renderer.outputColorSpace = THREE.SRGBColorSpace
      this.renderer.toneMapping = THREE.ACESFilmicToneMapping
      this.renderer.toneMappingExposure = 1.15
      this.renderer.setSize(W, H)
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
      el.appendChild(this.renderer.domElement)

      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping  = true
      this.controls.dampingFactor  = 0.05
      this.controls.enablePan = false
      this.controls.minDistance    = 7.5
      this.controls.maxDistance    = 35
      this.controls.addEventListener('start', () => { clearTimeout(this.interactionTimer); this.userInteracting = true; this.hoverCity = null })
      this.controls.addEventListener('end',   () => {
        this.interactionTimer = setTimeout(() => { this.userInteracting = false }, 2000)
      })

      this.buildLights()
      this.buildStars()
      this.buildGlobeGroup()
      this.buildAtmosphere()
      this.buildMoon()
    },

    /* ─────────────────── lighting ─────────────────── */
    buildLights() {
      this.scene.add(new THREE.AmbientLight(0x9baec9, 0.22))
      this.sunDirection = new THREE.Vector3(-0.65, 0.55, 1).normalize()
      const sun = new THREE.DirectionalLight(0xfff4e5, 2.4)
      sun.position.copy(this.sunDirection).multiplyScalar(30)
      this.scene.add(sun)
    },

    /* ─────────────────── star field (twinkling) ─────────────────── */
    buildStars() {
      const n      = 1600
      const pos    = new Float32Array(n * 3)
      const phases = new Float32Array(n)
      for (let i = 0; i < n; i++) {
        const r     = 100 + Math.random() * 180
        const theta = Math.random() * 2 * Math.PI
        const phi   = Math.acos(2 * Math.random() - 1)
        pos[i*3]   = r * Math.sin(phi) * Math.cos(theta)
        pos[i*3+1] = r * Math.sin(phi) * Math.sin(theta)
        pos[i*3+2] = r * Math.cos(phi)
        phases[i]  = Math.random() * Math.PI * 2
      }
      const geo = new THREE.BufferGeometry()
      geo.setAttribute('position', new THREE.BufferAttribute(pos, 3))
      geo.setAttribute('phase',    new THREE.BufferAttribute(phases, 1))
      this.twinkleMat = new THREE.ShaderMaterial({
        uniforms: { uTime: { value: 0 } },
        vertexShader: `
          attribute float phase;
          uniform float uTime;
          varying float vAlpha;
          void main() {
            float t = sin(uTime * 1.8 + phase);
            vAlpha = 0.22 + 0.16 * (t * 0.5 + 0.5);
            vec4 mvPos = modelViewMatrix * vec4(position, 1.0);
            gl_PointSize = 1.0 + 0.6 * sin(phase) * sin(phase);
            gl_Position  = projectionMatrix * mvPos;
          }
        `,
        fragmentShader: `
          varying float vAlpha;
          void main() {
            float d = length(gl_PointCoord - vec2(0.5));
            if (d > 0.5) discard;
            float b = 1.0 - smoothstep(0.0, 0.5, d);
            gl_FragColor = vec4(0.88, 0.94, 1.0, vAlpha * b);
          }
        `,
        transparent: true,
        depthWrite:  false,
        blending:    THREE.AdditiveBlending,
      })
      this.scene.add(new THREE.Points(geo, this.twinkleMat))
    },

    /* ─────────────────── moon ─────────────────── */
    buildMoon() {
      // Pivot group controls orbital tilt and revolution
      this.moonPivot = new THREE.Group()
      this.moonPivot.rotation.z = 0.42   // Artistic orbital tilt for this compressed scene
      this.scene.add(this.moonPivot)

      // Faint orbital path ring
      const orbitPts = []
      for (let i = 0; i <= 128; i++) {
        const a = (i / 128) * Math.PI * 2
        orbitPts.push(new THREE.Vector3(13.5 * Math.cos(a), 0, 13.5 * Math.sin(a)))
      }
      this.moonPivot.add(new THREE.Line(
        new THREE.BufferGeometry().setFromPoints(orbitPts),
        new THREE.LineBasicMaterial({
          color: 0x3355aa, transparent: true, opacity: 0.12,
          blending: THREE.AdditiveBlending, depthWrite: false,
        }),
      ))

      // Moon sphere
      const moonGeo = new THREE.SphereGeometry(0.88, 32, 32)
      const moonMat = new THREE.MeshPhongMaterial({ color: 0x999999, shininess: 3 })
      this.moonMesh = new THREE.Mesh(moonGeo, moonMat)
      this.moonMesh.position.set(13.5, 0, 0)
      this.moonPivot.add(this.moonMesh)

      this.loadTex('/textures/moon_1024.jpg', tex => {
        moonMat.map = tex
        moonMat.bumpMap = tex
        moonMat.bumpScale = 0.012
        moonMat.color.set(0xffffff)
        moonMat.needsUpdate = true
      }, true)
    },

    /* ─────────────────── globe group ─────────────────── */
    buildGlobeGroup() {
      this.globeGroup = new THREE.Group()
      // Rotate so China (lon ≈ 115°E) faces the camera.
      // Positive Y rotation brings longitude 115 E toward the camera at +Z.
      this.globeGroup.rotation.y = 0.44
      this.scene.add(this.globeGroup)

      this.buildEarth()
      this.buildGridOverlay()
      this.cityGroup = new THREE.Group()
      this.globeGroup.add(this.cityGroup)
      this.buildSpikes()
    },

    /* ─────────────────── satellite Earth ─────────────────── */
    buildEarth() {
      const mat = new THREE.MeshPhongMaterial({
        color: 0x30577a, shininess: 24, specular: 0x52606b,
        emissive: 0xffffff, emissiveIntensity: 0,
      })
      mat.onBeforeCompile = shader => {
        shader.uniforms.sunDirection = { value: this.sunDirection }
        shader.fragmentShader = 'uniform vec3 sunDirection;\n' + shader.fragmentShader
        shader.fragmentShader = shader.fragmentShader.replace('#include <emissivemap_fragment>', `
          #include <emissivemap_fragment>
          vec3 sunView = normalize((viewMatrix * vec4(sunDirection, 0.0)).xyz);
          float daylight = dot(normalize(vNormal), sunView);
          totalEmissiveRadiance *= 1.0 - smoothstep(-0.22, 0.12, daylight);
        `)
      }
      this.globeGroup.add(new THREE.Mesh(new THREE.SphereGeometry(GR, 96, 64), mat))
      this.loadTex(TEX.day, tex => {
        mat.map = tex
        mat.color.set(0xffffff)
        mat.needsUpdate = true
        this.textureLoading = false
      }, true)
      this.loadTex(TEX.night, tex => {
        mat.emissiveMap = tex
        mat.emissiveIntensity = 1.7
        mat.needsUpdate = true
      }, true)
      this.loadTex(TEX.topo, tex => { mat.bumpMap = tex; mat.bumpScale = 0.018; mat.needsUpdate = true })
      this.loadTex(TEX.specular, tex => { mat.specularMap = tex; mat.needsUpdate = true })
      const clouds = new THREE.MeshPhongMaterial({
        transparent: true, opacity: 0.72, depthWrite: false,
        shininess: 0, side: THREE.FrontSide,
      })
      this.cloudMesh = new THREE.Mesh(new THREE.SphereGeometry(GR * 1.006, 80, 56), clouds)
      this.cloudMesh.visible = false
      this.globeGroup.add(this.cloudMesh)
      this.loadTex(TEX.clouds, tex => { clouds.map = tex; clouds.needsUpdate = true; this.cloudsReady = true }, true)
    },

    /* ─────────────────── holographic grid overlay ─────────────────── */
    buildGridOverlay() {
      this.gridGroup = new THREE.Group()
      this.globeGroup.add(this.gridGroup)
      const R = GR + 0.04
      const dim    = () => new THREE.LineBasicMaterial({ color: 0x00d4ff, transparent: true, opacity: 0.07 })
      const bright = () => new THREE.LineBasicMaterial({ color: 0x00fff2, transparent: true, opacity: 0.18 })

      // latitude circles every 30°
      for (let lat = -60; lat <= 60; lat += 30) {
        const pts = []
        for (let lon = 0; lon <= 362; lon += 4) pts.push(this.ll2v(lat, lon, R))
        this.gridGroup.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(pts), lat === 0 ? bright() : dim()))
      }
      // longitude lines every 30°
      for (let lon = 0; lon < 360; lon += 30) {
        const pts = []
        for (let lat = -88; lat <= 88; lat += 4) pts.push(this.ll2v(lat, lon, R))
        this.gridGroup.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(pts), dim()))
      }
    },

    /* ─────────────────── atmosphere glow ─────────────────── */
    buildAtmosphere() {
      const material = new THREE.ShaderMaterial({
        uniforms: { sunDirection: { value: this.sunDirection } },
        vertexShader: `
          varying vec3 vNormal;
          varying vec3 vView;
          varying vec3 vWorldNormal;
          void main() {
            vec4 mv = modelViewMatrix * vec4(position, 1.0);
            vNormal = normalize(normalMatrix * normal);
            vView = normalize(-mv.xyz);
            vWorldNormal = normalize(mat3(modelMatrix) * normal);
            gl_Position = projectionMatrix * mv;
          }
        `,
        fragmentShader: `
          uniform vec3 sunDirection;
          varying vec3 vNormal;
          varying vec3 vView;
          varying vec3 vWorldNormal;
          void main() {
            float facing = abs(dot(normalize(vNormal), normalize(vView)));
            float rim = pow(1.0 - facing, 4.0);
            float sun = smoothstep(-0.3, 0.7, dot(normalize(vWorldNormal), sunDirection));
            float alpha = rim * (0.06 + sun * 0.42);
            gl_FragColor = vec4(mix(vec3(0.12, 0.22, 0.5), vec3(0.25, 0.57, 1.0), sun), alpha);
          }
        `,
        transparent: true, depthWrite: false,
        side: THREE.BackSide, blending: THREE.AdditiveBlending,
      })
      this.scene.add(new THREE.Mesh(new THREE.SphereGeometry(GR * 1.025, 96, 64), material))
    },

    /* ─────────────────── city spikes ─────────────────── */
    buildSpikes() {
      this.spikePairs = []
      this.hoverCity = null
      const data = this.cityData.filter(c => Number.isFinite(c.lat) && Number.isFinite(c.lon))
      if (!data.length) return
      const maxVal = Math.max(...data.map(d => Math.max(d.react || 0, d.vue || 0)), 1)
      this.spikePairs = []

      data.forEach(city => {
        const rH = Math.max(((city.react || 0) / maxVal) * MAX_H, 0.06)
        const vH = Math.max(((city.vue   || 0) / maxVal) * MAX_H, 0.06)
        const dir = this.ll2v(city.lat, city.lon, 1).normalize()
        const up  = new THREE.Vector3(0, 1, 0)
        const tan = new THREE.Vector3().crossVectors(dir, up).normalize()

        const { spike: rs, tip: rt } = this.makeSpike(dir, tan, -0.012, rH, 0xff4757, 0xff1133)
        const { spike: vs, tip: vt } = this.makeSpike(dir, tan,  0.012, vH, 0xffd32a, 0xffbb00)
        ;[rs, rt, vs, vt].forEach(object => {
          object.userData.city = city
          this.cityTargets.push(object)
        })
        rs.visible = rt.visible = (city.react || 0) > 0
        vs.visible = vt.visible = (city.vue || 0) > 0
        this.spikePairs.push({ spike: rs, tip: rt, dir: rs.userData.dir, fullH: rH })
        this.spikePairs.push({ spike: vs, tip: vt, dir: vs.userData.dir, fullH: vH })

        this.addCityDot(city)
        this.addCityLabel(dir, Math.max(rH, vH), city)
      })

      // start flat — animate upward in animLoop
      this.spikePairs.forEach(p => {
        p.spike.scale.y = 0.001
        p.spike.position.copy(p.dir.clone().multiplyScalar(GR))
        p.tip.position.copy(p.dir.clone().multiplyScalar(GR))
      })
    },

    makeSpike(dir, tan, lateralOffset, height, color, emissive) {
      const sDir = dir.clone().addScaledVector(tan, lateralOffset).normalize()
      const geo  = new THREE.CylinderGeometry(0.018, 0.027, height, 8)
      const mat  = new THREE.MeshPhongMaterial({
        color, emissive, emissiveIntensity: 0.6,
        transparent: true, opacity: 0.9, shininess: 90,
      })
      const spike = new THREE.Mesh(geo, mat)
      spike.userData.dir   = sDir.clone()
      spike.position.copy(sDir.clone().multiplyScalar(GR + height / 2))
      spike.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), sDir)
      this.cityGroup.add(spike)

      const tipGeo = new THREE.SphereGeometry(0.033, 8, 8)
      const tipMat = new THREE.MeshBasicMaterial({
        color: emissive, transparent: true, opacity: 0.95,
        blending: THREE.AdditiveBlending, depthWrite: false,
      })
      const tip = new THREE.Mesh(tipGeo, tipMat)
      tip.userData.dir  = sDir.clone()
      tip.position.copy(sDir.clone().multiplyScalar(GR + height))
      this.cityGroup.add(tip)

      return { spike, tip }
    },

    addCityDot(city) {
      const pos = this.ll2v(city.lat, city.lon, GR + 0.06)

      // glowing dot
      const dotGeo = new THREE.SphereGeometry(0.055, 8, 8)
      const dotMat = new THREE.MeshBasicMaterial({
        color: 0x00fff2, transparent: true, opacity: 0.85,
        blending: THREE.AdditiveBlending, depthWrite: false,
      })
      const dot = new THREE.Mesh(dotGeo, dotMat)
      dot.position.copy(pos)
      dot.userData.city = city
      this.cityTargets.push(dot)
      this.cityGroup.add(dot)

      // pulsing ring lying on the sphere surface
      const ringGeo = new THREE.RingGeometry(0.065, 0.08, 24)
      const ringMat = new THREE.MeshBasicMaterial({
        color: 0x00fff2, transparent: true, opacity: 0.3,
        side: THREE.DoubleSide, blending: THREE.AdditiveBlending, depthWrite: false,
      })
      const ring = new THREE.Mesh(ringGeo, ringMat)
      ring.position.copy(pos)
      ring.lookAt(new THREE.Vector3(0, 0, 0))
      ring.userData.isPulse = true
      ring.userData.phase   = Math.random() * Math.PI * 2
      this.cityGroup.add(ring)
    },

    addCityLabel(dir, maxH, city) {
      const canvas = document.createElement('canvas')
      canvas.width  = 320
      canvas.height = 72
      const ctx = canvas.getContext('2d')
      ctx.clearRect(0, 0, 320, 72)
      ctx.font         = '500 28px Arial'
      ctx.textAlign    = 'center'
      ctx.textBaseline = 'middle'
      ctx.shadowColor  = '#00fff2'
      ctx.shadowBlur   = 3
      ctx.fillStyle    = 'rgba(200,245,255,0.92)'
      ctx.fillText(city.name, 160, 36)
      const tex = new THREE.CanvasTexture(canvas)
      const mat = new THREE.SpriteMaterial({ map: tex, transparent: true, depthWrite: false })
      const sprite = new THREE.Sprite(mat)
      sprite.scale.set(1.9, 0.4275, 1)
      sprite.position.copy(dir.clone().multiplyScalar(GR + maxH + 0.18))
      this.cityGroup.add(sprite)
      this.cityLabels.push({ sprite, dir, city, priority: Number(city.react || 0) + Number(city.vue || 0) })
      this.cityLabels.sort((a, b) => b.priority - a.priority)
    },

    /* ─────────────────── arc network ─────────────────── */
    /* ─────────────────── animation loop ─────────────────── */
    resetView() {
      this.camera.position.set(0, 8.5, 15.5)
      this.controls.target.set(0, 0, 0)
      this.globeGroup.rotation.y = 0.44
      this.controls.update()
      this.hoverCity = null
    },

    updateLabels() {
      const width = this.$refs.globeCanvas.clientWidth
      const height = this.$refs.globeCanvas.clientHeight
      const boxes = []
      const world = new THREE.Vector3()
      const projected = new THREE.Vector3()
      const normal = new THREE.Vector3()
      const view = new THREE.Vector3()
      const limit = this.camera.position.length() < 11 ? 18 : 8
      this.cityLabels.forEach(({ sprite, dir, city }) => {
        normal.copy(dir).transformDirection(this.globeGroup.matrixWorld)
        sprite.getWorldPosition(world)
        view.copy(this.camera.position).sub(normal.clone().multiplyScalar(GR)).normalize()
        projected.copy(world).project(this.camera)
        sprite.visible = false
        if (normal.dot(view) < 0.18 || projected.z > 1 || projected.z < -1) return
        const x = (projected.x + 1) * width / 2
        const y = (1 - projected.y) * height / 2
        const pixels = height / (2 * Math.tan(THREE.MathUtils.degToRad(this.camera.fov / 2)) * world.distanceTo(this.camera.position))
        const w = Math.max(36, city.name.length * 28 / 320 * sprite.scale.x * pixels) + 12
        const h = Math.max(18, sprite.scale.y * pixels * 0.65) + 6
        const box = { l: x - w / 2, r: x + w / 2, t: y - h / 2, b: y + h / 2 }
        if (box.l < 0 || box.r > width || box.t < 65 || box.b > height) return
        if (boxes.length >= limit || boxes.some(b => box.l < b.r && box.r > b.l && box.t < b.b && box.b > b.t)) return
        boxes.push(box)
        sprite.visible = true
      })
    },

    inspectCity(event) {
      if (!this.renderer || (this.userInteracting && event.type !== 'click')) return
      const rect = this.renderer.domElement.getBoundingClientRect()
      const pointer = new THREE.Vector2((event.clientX - rect.left) / rect.width * 2 - 1, -(event.clientY - rect.top) / rect.height * 2 + 1)
      this.raycaster.setFromCamera(pointer, this.camera)
      const intersections = this.raycaster.intersectObjects(this.cityTargets.filter(object => object.visible), false)
      const hit = intersections.find(item => {
        const world = item.object.getWorldPosition(new THREE.Vector3())
        return world.clone().normalize().dot(this.camera.position.clone().sub(world).normalize()) > 0.08
      })
      this.hoverCity = hit ? hit.object.userData.city : null
      this.renderer.domElement.style.cursor = hit ? 'pointer' : 'grab'
    },

    animLoop(now = 0) {
      if (this.destroyed) return
      this.animFrameId = requestAnimationFrame(this.animLoop)
      const dt = this.lastFrame ? Math.min((now - this.lastFrame) / 1000, 0.05) : 0
      this.lastFrame = now
      this.time += dt
      this.spikePairs.forEach(p => {
        const next = THREE.MathUtils.lerp(p.spike.scale.y, 1, 1 - Math.exp(-4 * dt))
        p.spike.scale.y = next
        p.spike.position.copy(p.dir).multiplyScalar(GR + p.fullH * next / 2)
        p.tip.position.copy(p.dir).multiplyScalar(GR + p.fullH * next)
      })
      if (!this.userInteracting && !this.hoverCity && this.autoRotate) this.globeGroup.rotation.y += dt * 0.025
      if (this.cloudMesh) {
        this.cloudMesh.visible = !!this.cloudsReady && this.showClouds
        this.cloudMesh.rotation.y += dt * 0.008
      }
      if (this.gridGroup) this.gridGroup.visible = this.showGrid
      if (this.moonPivot) this.moonPivot.rotation.y += dt * 0.015
      if (this.twinkleMat) this.twinkleMat.uniforms.uTime.value = this.time
      this.controls.update()
      this.scene.updateMatrixWorld(true)
      this.camera.updateMatrixWorld(true)
      this.updateLabels()
      this.renderer.render(this.scene, this.camera)
    },

    /* ─────────────────── resize & cleanup ─────────────────── */
    onResize() {
      const el = this.$refs.globeCanvas
      if (!el || !this.renderer) return
      const W = el.clientWidth, H = el.clientHeight
      if (!W || !H) return
      this.camera.aspect = W / H
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(W, H)
    },

    releaseObjects(root) {
      root.traverse(obj => {
        if (obj.geometry) obj.geometry.dispose()
        if (!obj.material) return
        const materials = Array.isArray(obj.material) ? obj.material : [obj.material]
        materials.forEach(material => {
          Object.values(material).forEach(value => { if (value && value.isTexture) value.dispose() })
          material.dispose()
        })
      })
    },
    disposeScene() {
      if (this.controls) this.controls.dispose()
      if (this.scene) this.releaseObjects(this.scene)
      this.loadedTextures.forEach(tex => tex.dispose())
      this.loadedTextures.clear()
      if (this.renderer) {
        this.renderer.dispose()
        this.renderer.domElement.remove()
      }
    },
  },
}
</script>

<style scoped>
.globe-overlay {
  position: fixed;
  inset: 0;
  z-index: 2000;
  display: flex;
  flex-direction: column;
  background: #01020a;
}

/* ── toolbar ── */
.globe-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 24px;
  background: rgba(1, 2, 10, 0.95);
  border-bottom: 1px solid rgba(0, 180, 255, 0.2);
  flex-shrink: 0;
}
.globe-info { display: flex; align-items: center; gap: 14px; }
.globe-dot {
  width: 10px; height: 10px; border-radius: 50%;
  background: #00d4ff;
  box-shadow: 0 0 14px #00d4ff, 0 0 30px rgba(0,212,255,0.4);
  animation: gPulse 2s ease-in-out infinite;
  flex-shrink: 0;
}
@keyframes gPulse {
  0%,100% { opacity:1; box-shadow: 0 0 14px #00d4ff, 0 0 30px rgba(0,212,255,0.4); }
  50%     { opacity:.45; box-shadow: 0 0 5px #00d4ff; }
}
.globe-label { font-size: 16px; font-weight: 700; color: #fff; letter-spacing: 2px; }
.globe-hint  { font-size: 12px; color: rgba(0,212,255,0.5); letter-spacing: 1px; }
.globe-back-btn {
  padding: 8px 22px; font-size: 13px; font-weight: 600;
  color: #00d4ff; background: transparent;
  border: 1px solid rgba(0,212,255,0.4); border-radius: 6px;
  cursor: pointer; letter-spacing: 1px; font-family: inherit;
  transition: all .25s ease;
}
.globe-back-btn:hover {
  background: rgba(0,212,255,0.1); border-color: #00d4ff;
  box-shadow: 0 0 18px rgba(0,212,255,0.25);
}

/* ── canvas ── */
.globe-canvas {
  min-height: 0;
  flex: 1; overflow: hidden; position: relative;
}
.globe-canvas canvas { display: block; width: 100% !important; height: 100% !important; }

/* ── satellite loading overlay ── */
.globe-loading {
  position: absolute; inset: 0;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  background: rgba(1, 2, 10, 0.75);
  pointer-events: none; z-index: 5;
}
.globe-loading p {
  color: rgba(0,212,255,0.6); font-size: 13px;
  letter-spacing: 2px; margin-top: 14px;
}
.globe-spinner {
  width: 44px; height: 44px;
  border: 3px solid rgba(0,212,255,0.15);
  border-top-color: #00d4ff;
  border-radius: 50%;
  animation: gSpin .85s linear infinite;
}
@keyframes gSpin { to { transform: rotate(360deg); } }
.fade-enter-active, .fade-leave-active { transition: opacity 0.6s ease; }
.fade-enter, .fade-leave-to { opacity: 0; }

/* ── legend ── */
.globe-legend {
  display: flex; justify-content: center; gap: 36px;
  padding: 10px;
  background: rgba(1,2,10,0.95);
  border-top: 1px solid rgba(0,212,255,0.1);
  flex-shrink: 0;
}
.globe-legend-react { font-size:13px; font-weight:700; color:#ff4757; letter-spacing:2px; text-shadow:0 0 10px rgba(255,71,87,.6); }
.globe-legend-vue   { font-size:13px; font-weight:700; color:#ffd32a; letter-spacing:2px; text-shadow:0 0 10px rgba(255,211,42,.6); }
.globe-legend-arc   { font-size:13px; font-weight:700; color:rgba(0,212,255,.55); letter-spacing:2px; }
.globe-tools { position: absolute; top: 18px; left: 24px; z-index: 6; display: flex; flex-wrap: wrap; gap: 8px; }
.globe-tools button { cursor: pointer; border: 1px solid #33465e; border-radius: 6px; padding: 8px 12px; background: rgba(8, 18, 32, .85); color: #bac9dd; font: inherit; font-size: 12px; }
.globe-tools button[aria-pressed="true"] { border-color: #6fabc7; color: #e1f7ff; }
.globe-tools button:hover, .globe-tools button:focus-visible { background: #1d364c; }
.city-detail { position: absolute; right: 24px; bottom: 24px; z-index: 6; pointer-events: none; min-width: 160px; display: grid; gap: 12px; padding: 18px; border: 1px solid #3c556d; border-radius: 10px; background: rgba(6, 16, 30, .92); color: #ecf3ff; }
.city-detail span { display: flex; justify-content: space-between; gap: 24px; }
.texture-error { position: absolute; bottom: 12px; left: 24px; color: #c5d3df; font-size: 12px; z-index: 6; }
@media (max-width: 600px) {
  .globe-toolbar { padding: 10px 12px; }
  .globe-hint { display: none; }
  .globe-label { font-size: 12px; letter-spacing: 0; }
  .globe-back-btn { padding: 7px 10px; }
  .globe-tools { top: 12px; left: 12px; gap: 5px; }
  .globe-tools button { padding: 7px 8px; }
  .globe-legend { gap: 12px; flex-wrap: wrap; }
  .globe-legend span { font-size: 11px; letter-spacing: 0; }
}
</style>
