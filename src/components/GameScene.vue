<script lang="ts" setup>
import { ref, onMounted, onBeforeUnmount, reactive } from 'vue'
import * as THREE from 'three'
import * as CANNON from 'cannon-es'
import { GLTFLoader } from 'three/addons'

const container = ref<HTMLDivElement | null>(null)
const isGameOver = ref(false)

let textureLoader: THREE.TextureLoader
let gltfLoader: GLTFLoader
let renderer: THREE.WebGLRenderer

let camera: THREE.PerspectiveCamera | null = null
let scene: THREE.Scene | null = null
let world: CANNON.World | null = null

//Three Mesh
let table: THREE.Mesh | null = null
let frog: THREE.Mesh | null = null
let frog_mouse: THREE.Mesh | null = null
let frog_tongue: THREE.Mesh | null = null

let handle: THREE.Mesh | null = null
let eightMesh: THREE.Mesh | null = null
let eats: THREE.Mesh[] = []

let frogTongueMaterial: THREE.ShaderMaterial | null = null
let frogMouseMaterial: THREE.ShaderMaterial | null = null

let dragPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0)
let raycaster = new THREE.Raycaster()

//CANNON Body
let boxTable: CANNON.Body | null = null
let boxFrog: CANNON.Body | null = null

let isDragging = false
let dragOffset = new THREE.Vector3()
let deviceBound = 0
let gameTime = 10
let tongueTime = 0

const joyStickRadius = 60

let interval: any | null = null

let handleAnimT = 0; // parameter for the figure eight

// Add this line for the text sprite
let infoTextSprite: THREE.Sprite | null = null

// Add these refs for joystick state
const showJoystick = ref(false)
const joystickBase = ref<HTMLDivElement | null>(null)
const joystickKnob = ref<HTMLDivElement | null>(null)
let joystickStart = { x: 0, y: 0 }
const joystickDelta = reactive({ x: 0, y: 0 })
let joystickActive = false

onMounted(() => {
    textureLoader = new THREE.TextureLoader()
    gltfLoader = new GLTFLoader()

    initRenderer()
    initScene()
    initInteractions()
})

onBeforeUnmount(() => {
    releaseInteractions()
    renderer.dispose()
})

const initRenderer = () => {
    renderer = new THREE.WebGLRenderer({ antialias: true })
    renderer.setPixelRatio(window.devicePixelRatio)
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setAnimationLoop(animate)
    renderer.shadowMap.enabled = true
    renderer.shadowMap.type = THREE.PCFSoftShadowMap
    container.value?.appendChild(renderer.domElement)
}

const initScene = async () => {
    camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100)

    scene = new THREE.Scene()
    scene.background = new THREE.Color(0x527aff)

    world = new CANNON.World({ gravity: new CANNON.Vec3(0, -4, 0) })
    const floorMaterial = new CANNON.Material('floorMaterial')
    const eatMaterial = new CANNON.Material('eatMaterial')
    const contactMaterial = new CANNON.ContactMaterial(floorMaterial, eatMaterial, {
        friction: 0.4,
        restitution: 0
    })
    world.addContactMaterial(contactMaterial)

    await initLevel()
    onResize()
    initFrog()
    initEats()
    initHandle()
}

const initLevel = async () => {
    const hemiLight = new THREE.HemisphereLight(0xFFFFFF)
    scene?.add(hemiLight)

    const dirLight = new THREE.DirectionalLight(0xFFFFFF, 2.8)
    dirLight.position.set(5, 10, -5)
    dirLight.castShadow = true
    dirLight.shadow.camera.zoom = 1
    dirLight.shadow.mapSize.width = 4096;
    dirLight.shadow.mapSize.height = 4096;
    scene?.add(dirLight)

    const spotLight = new THREE.SpotLight(0x0000FF)
    spotLight.position.set(2, 2, 2)
    scene?.add(spotLight)

    const floor = new THREE.Mesh(new THREE.BoxGeometry(5.0, 0.05, 5.0), new THREE.ShadowMaterial({ color: 0x000000, opacity: 0.5 }))
    floor.position.set(0, 0, 0)
    floor.receiveShadow = true
    scene?.add(floor)

    // const gltf = await gltfLoader.parseAsync(getArrayBufferFromBase64(table_base64), '')
    const gltf = await gltfLoader.loadAsync('model/table.glb')
    table = gltf.scene.children[0].children[0] as THREE.Mesh
    table.material = new THREE.MeshStandardMaterial({ color: 0xFF80BB, metalness: 0.25, roughness: 0.5 })
    table.scale.set(1.0, 1.0, 0.5)
    table.position.set(0, -0.35, 0)
    table.rotation.set(Math.PI / 2, 0, Math.PI / 2)
    table.castShadow = true
    table.receiveShadow = true
    scene?.add(table)

    const floorMaterial = new CANNON.Material('floorMaterial')
    boxTable = new CANNON.Body({
        mass: 0,
        shape: new CANNON.Box(new CANNON.Vec3(0.75, 0.1, 1.4)),
        position: new CANNON.Vec3(0, 0.13, 0),
        material: floorMaterial
    })
    boxTable.material = floorMaterial
    boxTable.material.restitution = 0
    world?.addBody(boxTable)
}

const initFrog = () => {
    frog = new THREE.Mesh(new THREE.CylinderGeometry(0.15, 0.15, 0.003), new THREE.ShadowMaterial({ color: 0xFFFFFF, opacity: 0 }))
    frog.position.set(0.0, 0.235, 0.9)
    frog.castShadow = false
    frog.receiveShadow = true
    scene?.add(frog)

    // const frogTexture = textureLoader.load(frog_base64)
    const frogTexture = textureLoader.load('texture/frog.png')
    const frogMaterial = new THREE.MeshStandardMaterial({ map: frogTexture, transparent: true })
    const plane = new THREE.Mesh(new THREE.PlaneGeometry(0.295, 0.295), frogMaterial)
    plane.position.set(0, 0.002, 0)
    plane.rotation.set(-Math.PI / 2, 0, 0)
    plane.receiveShadow = true
    frog.add(plane)

    const circle = new THREE.Mesh(new THREE.CircleGeometry(0.125), new THREE.MeshBasicMaterial({ color: 0x970101 }))
    circle.position.set(0, 0.0018, 0)
    circle.rotation.set(-Math.PI / 2, 0, 0)
    frog.add(circle)

    frogMouseMaterial = new THREE.ShaderMaterial({
        uniforms: {
            cylinderCenter: { value: new THREE.Vector3(0.0, 0.235, 0.65) }, // center of cylinder
            cylinderRadius: { value: 0.125 },
            cylinderHeight: { value: 0.3 },
            circleWorldMatrix: { value: new THREE.Matrix4() }
        },
        vertexShader: `
                varying vec3 vWorldPosition;
                void main() {
                    vec4 worldPosition = modelMatrix * vec4(position, 1.0);
                    vWorldPosition = worldPosition.xyz;
                    gl_Position = projectionMatrix * viewMatrix * worldPosition;
                }
            `,
        fragmentShader: `
                uniform vec3 cylinderCenter;
                uniform float cylinderRadius;
                uniform float cylinderHeight;
                varying vec3 vWorldPosition;
                void main() {
                    // Project point onto cylinder's axis (assume Y axis)
                    float yMin = cylinderCenter.y - cylinderHeight / 2.0;
                    float yMax = cylinderCenter.y + cylinderHeight / 2.0;
                    // Check if within height
                    if (vWorldPosition.y < yMin || vWorldPosition.y > yMax) discard;
                    // Check if within radius (XZ plane)
                    vec2 d = vWorldPosition.xz - cylinderCenter.xz;
                    if (length(d) > cylinderRadius) discard;
                    gl_FragColor = vec4(0.3, 0.0, 0.0, 1.0); // dark red
                }
            `,
        side: THREE.DoubleSide,
        transparent: false
    })

    frog_mouse = new THREE.Mesh(new THREE.CircleGeometry(0.1), frogMouseMaterial)
    frog_mouse.position.set(0, 0.0019, 0.08)
    frog_mouse.rotation.set(-Math.PI / 2, 0, 0)
    frog.add(frog_mouse)

    frogTongueMaterial = new THREE.ShaderMaterial({
        uniforms: {
            time: { value: 0.0 },
            amplitude: { value: 0.02 }, // wave amplitude
            frequency: { value: 8.0 },  // wave frequency
            speed: { value: 2.0 }       // wave speed
        },
        vertexShader: `
            uniform float time;
            uniform float amplitude;
            uniform float frequency;
            uniform float speed;
            void main() {
                vec3 pos = position;
                float height = 0.18; // match your cylinder's height
                float t = (pos.y + height / 2.0) / height; // 0 at bottom, 1 at top
                float mask = smoothstep(0.5, 1.0, t); // wave strongest at top, fades out halfway down

                pos.x += sin(pos.y * frequency + time * speed) * amplitude * mask;

                gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
            }
        `,
        fragmentShader: `
            void main() {
                gl_FragColor = vec4(1.0, 0.7529, 0.7961, 1.0); // pink color (#FFC0CB)
            }
        `,
        side: THREE.DoubleSide
    });

    frog_tongue = new THREE.Mesh(
        new THREE.CylinderGeometry(0.025, 0.025, 0.05, 16, 32),
        frogTongueMaterial
    );
    frog_tongue.position.set(0.0005, 0.0, 0.085);
    frog.add(frog_tongue);

    boxFrog = new CANNON.Body({ mass: 100, shape: new CANNON.Cylinder(0.15, 0.15, 0.003) })
    boxFrog.position.copy(new CANNON.Vec3(frog.position.x, frog.position.y, frog.position.z))
    world?.addBody(boxFrog)
    boxFrog.addEventListener('collide', onCollision)
}

const initEats = () => {
    if (interval) {
        clearInterval(interval)
        interval = null
    }

    let lastTime = performance.now()
    interval = setInterval(() => {
        const currentTime = performance.now()
        const deltaTime = (currentTime - lastTime) / 1000
        gameTime -= deltaTime
        lastTime = currentTime

        if (gameTime <= 0) {
            clearInterval(interval)
            interval = null
            isGameOver.value = true
            showJoystick.value = false
            if (handle)
                handle.visible = false
            if (eightMesh)
                eightMesh.visible = false
            if (infoTextSprite)
                infoTextSprite.visible = false
            showFrog(false)
            return
        }

        const { x, z } = getRandomPointInRotatedRect(0, -0.15, 1.4, 0.7, -45)
        initCrate(x, z)
    }, 1000 / 20)
}

const initCrate = (x: number, z: number) => {
    const eat = new THREE.Mesh(new THREE.BoxGeometry(0.062, 0.05, 0.062),
        [
            new THREE.MeshBasicMaterial({ color: 0xFFFFFF }),
            new THREE.MeshBasicMaterial({ color: 0xFFFFFF }),
            new THREE.MeshBasicMaterial({ color: 0xFFFFFF }),
            new THREE.MeshBasicMaterial({ color: 0xFFFFFF }),
            new THREE.MeshBasicMaterial({ color: 0x6cb6f1 }),
            new THREE.MeshBasicMaterial({ color: 0xFFFFFF })
        ])
    eat.position.set(x, 3.0, z)
    eat.rotation.set(0.0, -Math.PI / 4, 0.0)
    eat.scale.set(1.0, 1.0, 1.0)
    eat.castShadow = true
    scene?.add(eat)

    const eatMaterial = new CANNON.Material('eatMaterial')
    const boxEat = new CANNON.Body({
        mass: 0.8,
        shape: new CANNON.Box(new CANNON.Vec3(0.031, 0.025, 0.031)),
        material: eatMaterial
    })
    boxEat.material = eatMaterial
    boxEat.material.restitution = 0
    boxEat.position.copy(new CANNON.Vec3(eat.position.x, eat.position.y, eat.position.z))
    boxEat.quaternion.set(eat.quaternion.x, eat.quaternion.y, eat.quaternion.z, eat.quaternion.w)
    world?.addBody(boxEat)

    eat.userData = { body: boxEat, scale: 1.0, targetScale: 0.0, animate: false, removed: false }
    eats.push(eat)
}

const initHandle = async () => {
    // const handTexture = textureLoader.load(hand_base64)
    const handTexture = textureLoader.load('texture/hand.png')
    handle = new THREE.Mesh(
        new THREE.PlaneGeometry(0.15, 0.15),
        new THREE.MeshBasicMaterial({
            map: handTexture,
            transparent: true
        })
    )
    handle.position.copy(new THREE.Vector3(0.0, 0.5, 0.85))
    handle.rotation.set(-Math.PI / 2, 0, 0)
    scene?.add(handle)

    // Add this block to visualize the figure-eight path
    const a = 0.2;
    const center = new THREE.Vector3(0.0, 0.3, 0.72)

    // Define the lemniscate curve
    class LemniscateCurve extends THREE.Curve<THREE.Vector3> {
        getPoint(t: number, optionalTarget = new THREE.Vector3()) {
            // t in [0,1]
            const theta = t * 2 * Math.PI;
            const x = a * Math.sin(theta);
            const z = a * Math.sin(theta) * Math.cos(theta);
            return optionalTarget.set(center.x + x, center.y, center.z + z);
        }
    }

    const curve = new LemniscateCurve();
    const geometry = new THREE.TubeGeometry(curve, 100, 0.02, 8, true);
    const material = new THREE.MeshBasicMaterial({ color: 0xffffff, opacity: 0.7, transparent: true });
    eightMesh = new THREE.Mesh(geometry, material);
    scene?.add(eightMesh);

    // --- Load the custom font before drawing the text ---
    const font = new FontFace('PoppinsBold', 'url(/Poppins-Bold.ttf)');
    await font.load();
    (document as any).fonts.add(font);

    // Now create the canvas and draw the text using the custom font
    const canvas = document.createElement('canvas');
    canvas.width = 1536;
    canvas.height = 128;
    const ctx = canvas.getContext('2d')!;
    ctx.font = 'bold 64px PoppinsBold'; // Use the loaded font family
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillStyle = '#fff';
    ctx.shadowColor = '#000';
    ctx.shadowBlur = 8;
    ctx.fillText('Clean the maps to unlock the Minecraft level', canvas.width / 2, canvas.height / 2);

    const texture = new THREE.CanvasTexture(canvas);
    const materialText = new THREE.SpriteMaterial({ map: texture, transparent: true });
    infoTextSprite = new THREE.Sprite(materialText);
    infoTextSprite.scale.set(0.6, 0.07, 1); // your preferred scale
    infoTextSprite.position.set(0.05, 0.85, 0.8);
    scene?.add(infoTextSprite);
    // --- end block ---
}

const animateHandle = () => {
    if (handle && handle.visible && eightMesh && eightMesh.visible) {
        // Parameters for the figure eight
        const a = 0.2; // size of the eight, adjust as needed
        const center = new THREE.Vector3(0.06, 0.5, 0.85)// center position

        // Animate t
        handleAnimT += 0.06; // speed, adjust as needed

        // Lemniscate of Gerono (horizontal 8 in XZ plane)
        const x = a * Math.sin(handleAnimT);
        const z = a * Math.sin(handleAnimT) * Math.cos(handleAnimT);

        handle.position.set(center.x + x, center.y, center.z + z);
    }
}

const initInteractions = () => {
    window.addEventListener('resize', onResize)
    window.addEventListener('keydown', onKeyDown)
    renderer.domElement.addEventListener('mousedown', onInputDown)
    renderer.domElement.addEventListener('mousemove', onInputMove)
    renderer.domElement.addEventListener('mouseup', onInputUp)
    renderer.domElement.addEventListener('touchstart', onInputDown)
    renderer.domElement.addEventListener('touchmove', onInputMove)
    renderer.domElement.addEventListener('touchend', onInputUp)
}

const releaseInteractions = () => {
    window.removeEventListener('resize', onResize)
    window.removeEventListener('keydown', onKeyDown)
    renderer.domElement.removeEventListener('mousedown', onInputDown)
    renderer.domElement.removeEventListener('mousemove', onInputMove)
    renderer.domElement.removeEventListener('mouseup', onInputUp)
    renderer.domElement.removeEventListener('touchstart', onInputDown)
    renderer.domElement.removeEventListener('touchmove', onInputMove)
    renderer.domElement.removeEventListener('touchend', onInputUp)
}

const animate = () => {
    world?.step(1 / 60)

    if (frog && boxFrog) {
        frog.position.y = boxFrog.position.y
        boxFrog.position.x = frog.position.x
        boxFrog.position.z = frog.position.z
    }

    eats = eats.filter(eat => {
        if (eat.userData.removed) {
            world?.removeBody(eat.userData.body)
            scene?.remove(eat)
            return false
        }

        if (eat.userData.animate) {
            eat.userData.scale = THREE.MathUtils.lerp(eat.userData.scale, eat.userData.targetScale, 0.1)
            eat.userData.pos.x = THREE.MathUtils.lerp(eat.userData.pos.x, eat.userData.targetPos.x, 0.05)
            eat.userData.pos.y = THREE.MathUtils.lerp(eat.userData.pos.y, eat.userData.targetPos.y, 0.05)
            eat.userData.pos.z = THREE.MathUtils.lerp(eat.userData.pos.z, eat.userData.targetPos.z, 0.05)
            eat.scale.set(eat.userData.scale, eat.userData.scale, eat.userData.scale);
            eat.position.copy(eat.userData.pos)

            if (Math.abs(eat.userData.scale - eat.userData.targetScale) < 0.6)
                eat.userData.removed = true;
        }
        else {
            eat.position.copy(eat.userData.body.position)
        }

        return true
    })

    if (frogTongueMaterial) {
        tongueTime += 1 / 60;
        frogTongueMaterial.uniforms.time.value = tongueTime;
    }

    animateHandle()

    // --- Add this block to pulse the text scale ---
    if (infoTextSprite) {
        const t = performance.now() * 0.002;
        const scale = 1.0 + Math.sin(t * 2) * 0.12; // Pulsing between 0.88 and 1.12
        infoTextSprite.scale.set(scale, 0.12 + Math.sin(t * 2) * 0.015, 1);
    }
    // --- end block ---

    renderer.render(scene!, camera!)
}

const getArrayBufferFromBase64 = (base64String: string) => {
    const binaryString = atob(base64String)
    const bytes = new Uint8Array(binaryString.length)
    for (let i = 0; i < binaryString.length; i++)
        bytes[i] = binaryString.charCodeAt(i)
    return bytes.buffer
}

const getRandomPointInRotatedRect = (centerX: number, centerY: number, width: number, height: number, rotationDegrees: number) => {
    const rotation = rotationDegrees * (Math.PI / 180);
    const localX = (Math.random() - 0.5) * width;
    const localY = (Math.random() - 0.5) * height;
    const rotatedX = localX * Math.cos(rotation) - localY * Math.sin(rotation);
    const rotatedY = localX * Math.sin(rotation) + localY * Math.cos(rotation);
    const worldX = centerX + rotatedX;
    const worldY = centerY + rotatedY;

    return { x: worldX, z: worldY };
}

const getIntersects = (event: MouseEvent | TouchEvent) => {
    const rect = renderer.domElement.getBoundingClientRect()
    let input = new THREE.Vector2()
    if (event instanceof MouseEvent) {
        input.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
        input.y = -((event.clientY - rect.top) / rect.height) * 2 + 1
    }
    else if (event instanceof TouchEvent) {
        input.x = ((event.touches[0].clientX - rect.left) / rect.width) * 2 - 1
        input.y = -((event.touches[0].clientY - rect.top) / rect.height) * 2 + 1
    }

    raycaster.setFromCamera(input, camera!)
    return raycaster.intersectObjects(scene!.children, true)
}

const clampTableBounds = (position: THREE.Vector3) => {
    position.x = Math.max(-1.5 / 2 + 0.26 / 2, Math.min(1.5 / 2 - 0.26 / 2, position.x))
    position.z = Math.max(-2.8 / 2 + 0.35 / 2, Math.min(2.8 / 2 - deviceBound / 2, position.z))
}

const findEat = (boxBody: CANNON.Body) => {
    return eats.find(eat => eat.userData.body === boxBody)
}

const showFrog = (visible: boolean) => {
    if (frog) {
        frog.visible = visible
        if (frog.children[0])
            frog.children[0].visible = visible

        if (frog_mouse)
            frog_mouse.visible = visible
    }
}

const onResize = () => {
    if (!camera || !renderer)
        return

    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()

    if ((window.innerWidth / window.innerHeight) < 0.5) {
        camera.fov = 52.5
        camera.position.set(0.0275, 3.7, 1.8)
        camera.lookAt(0.0275, 0.7, 0.0)
        camera.updateProjectionMatrix()
        deviceBound = 0.3
    }
    else if ((window.innerWidth / window.innerHeight) < 0.55) {
        camera.fov = 52.5
        camera.position.set(0.0275, 3.0, 1.8)
        camera.lookAt(0.0275, 0.1, 0.0)
        camera.updateProjectionMatrix()
        deviceBound = 0.3
    }
    else if ((window.innerWidth / window.innerHeight) < 0.65) {
        camera.fov = 50
        camera.position.set(0.0275, 3.0, 1.8)
        camera.lookAt(0.0275, 0.2, 0.0)
        camera.updateProjectionMatrix()
        deviceBound = 0.3
    }
    else {
        camera.fov = 50
        camera.position.set(0.0275, 3.0, 1.8)
        camera.lookAt(0.0275, 0.5, 0.0)
        camera.updateProjectionMatrix()
        deviceBound = 0.55
    }

    renderer.setSize(window.innerWidth, window.innerHeight)
}

const onKeyDown = (event: KeyboardEvent) => {
    if (!frog)
        return

    const step = 0.025
    switch (event.key) {
        case 'ArrowUp':
        case 'w':
            frog!.position.z -= step
            break
        case 'ArrowDown':
        case 's':
            frog!.position.z += step
            break
        case 'ArrowLeft':
        case 'a':
            frog!.position.x -= step
            break
        case 'ArrowRight':
        case 'd':
            frog!.position.x += step
            break
    }

    clampTableBounds(frog!.position)
}

const onInputDown = (event: MouseEvent | TouchEvent) => {
    if (!frog)
        return

    const intersects = getIntersects(event)
    if (intersects.length === 0)
        return

    if (handle)
        handle.visible = false
    if (eightMesh)
        eightMesh.visible = false
    if (infoTextSprite)
        infoTextSprite.visible = false

    // Show joystick at touch/click position
    let clientX = 0, clientY = 0
    if (event instanceof MouseEvent) {
        clientX = event.clientX
        clientY = event.clientY
    } else if (event instanceof TouchEvent) {
        clientX = event.touches[0].clientX
        clientY = event.touches[0].clientY
    }
    joystickStart = { x: clientX, y: clientY }
    joystickDelta.x = 0
    joystickDelta.y = 0
    showJoystick.value = true
    joystickActive = true

    isDragging = false // disable drag logic

    if (event instanceof TouchEvent)
        event.preventDefault()

}

const onInputMove = (event: MouseEvent | TouchEvent) => {
    if (!joystickActive)
        return

    let clientX = 0, clientY = 0
    if (event instanceof MouseEvent) {
        clientX = event.clientX
        clientY = event.clientY
    } else if (event instanceof TouchEvent) {
        clientX = event.touches[0].clientX
        clientY = event.touches[0].clientY
    }
    // Calculate joystick movement (clamp to radius)
    const dx = clientX - joystickStart.x
    const dy = clientY - joystickStart.y

    const dist = Math.sqrt(dx * dx + dy * dy)
    let clampedDx = dx, clampedDy = dy
    if (dist > joyStickRadius) {
        clampedDx = dx * joyStickRadius / dist
        clampedDy = dy * joyStickRadius / dist
    }
    joystickDelta.x = clampedDx
    joystickDelta.y = clampedDy

    // Move frog based on joystick direction
    if (frog) {
        // Map joystick to table movement (adjust speed as needed)
        const moveSpeed = 0.0007
        frog.position.x += clampedDx * moveSpeed
        frog.position.z += clampedDy * moveSpeed

        if (frog_mouse && frogMouseMaterial) {
            frogMouseMaterial.uniforms.cylinderCenter.value.copy(frog.position)

            const offsetX = -0.125
            const offsetZ = 0.045
            const offsetY = 0.0019

            const deltaLength = 0.02;

            const frogMousePos = new THREE.Vector3(offsetX * frog.position.x, offsetY, Math.abs(offsetZ * (frog.position.z - 1.8)))
            frog_mouse.position.copy(frogMousePos)

            frog_tongue.geometry = new THREE.CylinderGeometry(0.025, 0.025, 0.18 - Math.abs(deltaLength * (frog.position.z - 1.8)))
            const tonguePos = new THREE.Vector3(-0.05 * frog.position.x, frog_tongue.position.y, frog_tongue.position.z)
            frog_tongue.position.copy(tonguePos)

        }
        clampTableBounds(frog.position)
    }

    if (event instanceof TouchEvent)
        event.preventDefault()
}

const onInputUp = () => {
    joystickActive = false
    showJoystick.value = false
}

const onCollision = (event: any) => {
    if (isGameOver.value)
        return

    const eat = findEat(event.body)
    if (eat && !eat.userData.animate) {
        eat.userData.pos = eat.position
        eat.userData.targetPos = frog?.position
        eat.userData.animate = true
    }
}

const onRestartGame = async () => {
    eats.forEach(eat => {
        if (eat.userData.body) {
            world?.removeBody(eat.userData.body)
        }
        scene?.remove(eat)
    })
    eats = []

    if (frog && boxFrog) {
        frog.position.set(0.0, 0.235, 0.9)
        boxFrog.position.copy(new CANNON.Vec3(frog.position.x, frog.position.y, frog.position.z))
        if (frog_mouse && frogMouseMaterial) {
            frogMouseMaterial.uniforms.cylinderCenter.value.copy(frog.position)
        }
        showFrog(true)
    }

    gameTime = 10
    isGameOver.value = false

    initEats()
}

</script>

<template>
    <div ref="container"></div>
    <button class="restart-button" v-if="isGameOver" @click="onRestartGame">Restart</button>
    <!-- Joystick UI -->
    <div v-if="showJoystick && !isGameOver" class="joystick-base"
        :style="{ left: joystickStart.x - joyStickRadius + 'px', top: joystickStart.y - joyStickRadius + 'px' }">
        <div class="joystick-knob" :style="{
            left: joyStickRadius + joystickDelta.x + 'px',
            top: joyStickRadius + joystickDelta.y + 'px'
        }"></div>
    </div>
</template>

<style scoped>
.joystick-base {
    position: fixed;
    width: 120px;
    height: 120px;
    border: 1px solid white;
    background: rgba(0, 0, 0, 0.2);
    border-radius: 50%;
    z-index: 1000;
    pointer-events: none;
}

.joystick-knob {
    position: absolute;
    width: 40px;
    height: 40px;
    background: rgba(255, 255, 255, 0.7);
    border-radius: 50%;
    left: 30px;
    top: 30px;
    transform: translate(-50%, -50%);
    pointer-events: none;
}
</style>