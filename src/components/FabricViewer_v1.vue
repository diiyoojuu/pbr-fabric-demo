<template>
    <div ref="canvasContainer" class="viewer-container"></div>
    <div v-if="loading" class="loading-overlay">
        材质加载中... {{ loadingProgress }}%
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
// 引入轨道控制器，用于鼠标旋转缩放
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
// 引入 EXRLoader 用于加载 EXR 环境贴图
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader.js';

const canvasContainer = ref(null);
const loading = ref(true);
const loadingProgress = ref(0);

// Three.js 核心变量
let scene, camera, renderer, controls, mesh;
let animationId;

// --- 初始化 Three.js 场景 ---
const initThree = () => {
    // 1. 创建场景
    scene = new THREE.Scene();
    // 设置背景色，或者稍后用 HDR 贴图填充
    scene.background = new THREE.Color(0x333333);

    // 2. 创建相机 (透视相机)
    const width = canvasContainer.value.clientWidth;
    const height = canvasContainer.value.clientHeight;
    camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100);
    camera.position.set(0, 0, 3); // 将相机向后拉

    // 3. 创建渲染器
    renderer = new THREE.WebGLRenderer({
        antialias: true, // 开启抗锯齿
        alpha: true // 允许透明背景
    });
    renderer.setSize(width, height);
    renderer.setPixelRatio(window.devicePixelRatio); // 适配高清屏

    // --- 关键：PBR 渲染所需的重要设置 ---
    // 使用真实的光照计算模式
    renderer.useLegacyLights = false;
    // 设置色调映射，避免高光过曝，使画面更像照片
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;
    // 设置颜色空间为 sRGB，保证贴图颜色显示正确 (Three.js r152+ 新语法)
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    // 将渲染出的 canvas 添加到 DOM 中
    canvasContainer.value.appendChild(renderer.domElement);

    // 4. 添加控制器
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true; // 开启阻尼惯性，体验更好
    controls.dampingFactor = 0.05;

    // 5. 添加基础灯光 (作为 HDR 的补充)
    // 虽然 PBR 主要靠 HDR，但加一点方向光可以突出立体感
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
    directionalLight.position.set(5, 5, 5);

    // 1. 设置一个显眼的背景色（如深蓝色），如果背景变蓝了，说明渲染器在工作
    scene.background = new THREE.Color(0x1a2a3a);

    // 2. 添加一个极强的环境光，直接照亮所有材质
    const ambientLight = new THREE.AmbientLight(0xffffff, 3.0);
    scene.add(ambientLight);

    scene.add(directionalLight);
};

// --- 加载 PBR 材质和环境 ---
const loadAssets = async () => {
    const loadingManager = new THREE.LoadingManager();
    const textureLoader = new THREE.TextureLoader(loadingManager);
    // 强制开启跨域支持，解决部分缓存读取权限问题
    textureLoader.setCrossOrigin('anonymous');
    //   const rgbeLoader = new RGBELoader(loadingManager);
    // 1. 实例化 EXRLoader
    const exrLoader = new EXRLoader(loadingManager);

    loadingManager.onProgress = (url, loaded, total) => {
        loadingProgress.value = Math.floor((loaded / total) * 100);
    };
    loadingManager.onLoad = () => {
        loading.value = false;
    };

    try {
        // 1. 加载 HDR 环境贴图 (这对 PBR 至关重要!)
        // 假设文件放在 public/assets/hdri/ 目录下
        const envMap = await exrLoader.loadAsync('/assets/exr/royal_esplanade_4k.exr');
        envMap.mapping = THREE.EquirectangularReflectionMapping;
        // scene.environment = envMap; // 设置为场景的环境光
        // scene.background = envMap; // 如果你想让背景也是这个图片，取消注释

        // 2. 加载 PBR 纹理贴图
        // 假设文件放在 public/assets/textures/fabric/ 目录下
        const baseColorMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_basecolor.jpg');
        const normalMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_normal.jpg');
        const roughnessMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_roughness.jpg');
        const aoMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_ao.jpg');

        // const baseColorMap = await textureLoader.loadAsync('/assets/textures/fabric/Fabric079_2K-JPG/Color.jpg');
        // const normalMap = await textureLoader.loadAsync('/assets/textures/fabric/Fabric079_2K-JPG/NormalGL.jpg');
        // const roughnessMap = await textureLoader.loadAsync('/assets/textures/fabric/Fabric079_2K-JPG/Roughness.jpg');
        // const aoMap = await textureLoader.loadAsync('/assets/textures/fabric/Fabric079_2K-JPG/fabric_ao.jpg');

        // 确保颜色贴图使用正确的颜色空间
        baseColorMap.colorSpace = THREE.SRGBColorSpace;

        // --- 重点：创建 PBR 材质 ---
        const material = new THREE.MeshStandardMaterial({
            map: baseColorMap,          // 基础色
            normalMap: normalMap,       // 法线
            roughnessMap: roughnessMap, // 粗糙度
            aoMap: aoMap,               // 环境光遮蔽
            side: THREE.DoubleSide,     // 双面渲染
            roughness: 3.0,             // 材质本身的基础粗糙度值，会与贴图相乘
            metalness: 0.0              // 织物通常不是金属，设为 0
        });

        // 设置纹理重复，让纹理更密集，看起来更像精细面料
        const repeatFactor = 4;
        [baseColorMap, normalMap, roughnessMap, aoMap].forEach(texture => {
            texture.wrapS = THREE.RepeatWrapping;
            texture.wrapT = THREE.RepeatWrapping;
            texture.repeat.set(repeatFactor, repeatFactor);
        });


        // 3. 创建几何体并在场景中显示
        // 为了更好的展示布料褶皱，这里使用一个稍微复杂的几何体，比如 TorusKnot(环面纽结)
        // 如果有鞋子模型(GLTF)更好，但用几何体演示材质足够了。
        const geometry = new THREE.TorusKnotGeometry(0.6, 0.25, 128, 32);
        //也可以用简单的球体：
        // const geometry = new THREE.SphereGeometry(0.8, 64, 64);

        // 这一步是为了让 AO 贴图生效，需要第二套 UV 坐标
        geometry.attributes.uv2 = geometry.attributes.uv;

        mesh = new THREE.Mesh(geometry, material);
        scene.add(mesh);

    } catch (error) {
        console.error('资源加载失败:', error);
        loading.value = false;
    }
};

// --- 渲染循环 ---
const animate = () => {
    animationId = requestAnimationFrame(animate);
    controls.update(); // 更新控制器状态
    renderer.render(scene, camera);
};

// --- 处理窗口大小调整 ---
const onWindowResize = () => {
    if (!canvasContainer.value) return;
    const width = canvasContainer.value.clientWidth;
    const height = canvasContainer.value.clientHeight;
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);
};

// --- Vue 生命周期钩子 ---
onMounted(async () => {
    initThree();
    await loadAssets();
    animate();
    window.addEventListener('resize', onWindowResize);
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    cancelAnimationFrame(animationId);
    // 简单的内存清理
    if (renderer) renderer.dispose();
    if (scene) scene.clear();
});
</script>

<style scoped>
.viewer-container {
    width: 100%;
    height: 100vh;
    /* 全屏显示 */
    position: relative;
    background-color: #f0f0f0;
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1;
}

.loading-overlay {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    background: rgba(0, 0, 0, 0.7);
    padding: 20px;
    border-radius: 8px;
    pointer-events: none;
}
</style>