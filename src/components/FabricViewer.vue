<template>
    <div class="page-container">
        <!-- 3D 渲染容器 -->
        <div ref="canvasContainer" class="viewer-container"></div>

        <!-- 加载状态遮罩 -->
        <transition name="fade">
            <div v-if="loading" class="loading-overlay">
                <div class="loader-content">
                    <div class="spinner"></div>
                    <p>正在渲染高保真模型... {{ loadingProgress }}%</p>
                </div>
            </div>
        </transition>

        <!-- 左侧材质选择面板 -->
        <div class="texture-panel">
            <h3>面料 / 材质选择</h3>
            <div class="texture-list">
                <div v-for="(tex, index) in textureOptions" :key="index" class="texture-item"
                    :class="{ active: currentTextureIndex === index }" @click="handleTextureChange(index)"
                    :style="{ backgroundImage: `url(${tex.preview})` }">
                    <span class="texture-name">{{ tex.name }}</span>
                </div>
            </div>
        </div>

        <!-- 底部环境光切换栏 -->
        <div class="bottom-bar">
            <div v-for="(env, key) in envOptions" :key="key" class="env-item" :class="{ active: currentEnv === key }"
                @click="changeEnvironment(key)">
                {{ env.name }}
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import GUI from 'lil-gui';
import Stats from 'stats.js';

// ==========================================
// 1. 状态管理与配置
// ==========================================
const canvasContainer = ref(null);
const loading = ref(true);
const loadingProgress = ref(0);
const currentEnv = ref('room');
const currentTextureIndex = ref(-1); // 初始设为 -1，保证第一次点击或初始化能触发

// Three.js 核心全局变量
let scene, camera, renderer, controls, stats, gui;
let shoeModelGroup = null; // 保存模型组引用，用于后续遍历修改
let fabricMaterial = null; // 保存核心材质引用
let animationId;

// --- 环境光配置 (HDR) ---
// 面试话术：使用 EXR/HDR 格式的高动态范围贴图，模拟真实物理光照反射
const envOptions = {
    room: { name: '室内摄影棚', url: '/assets/exr/royal_esplanade_4k.exr' },
    outDoor: { name: '户外自然光', url: '/assets/exr/river_alcove_4k.exr' }
};

// --- 材质预设配置 (Data-Driven) ---
// 面试话术：通过配置表驱动材质切换，支持 C2M 个性化定制场景
const textureOptions = [
    {
        name: '默认针织 (Normal)',
        id: 'normal',
        preview: '/assets/textures/fabric/base.jpg', // UI显示的缩略图
        config: {
            // PBR 贴图路径
            map: '/assets/textures/fabric/base.jpg',
            normalMap: '/assets/textures/fabric/normal.jpg',
            roughnessMap: '/assets/textures/fabric/roughness.jpg',
            aoMap: '/assets/textures/fabric/ao.jpg',

            // 材质物理参数
            color: null, // null 代表使用贴图原色，不进行染色
            repeat: [4, 4], // 纹理重复次数（控制密度）
            normalScale: 1.0, // 法线强度（控制凹凸感）
            roughness: 0.7, // 粗糙度
            sheen: 1.0, // **关键属性**：模拟织物表面的泛白绒毛感
            sheenRoughness: 0.5,
            sheenColor: 0xffffff,
        }
    },
    {
        name: '红色飞织 (Red Knit)',
        id: 'knit',
        preview: '/assets/textures/knit/base.jpg',
        config: {
            // 假设这个预设没有贴图，只用纯色模拟，或者路径不同
            map: '/assets/textures/knit/base.jpg',
            normalMap: '/assets/textures/knit/normal.jpg',
            roughnessMap: '/assets/textures/knit/roughness.jpg',
            aoMap: null, // 没有 AO 图

            color: 0xff3333, // 演示：在纹理基础上叠加红色
            repeat: [4, 4],
            normalScale: 1.5, // 飞织纹理凹凸感更强
            roughness: 0.8,
            sheen: 1.0
        }
    },
    {
        name: '科技网布 (Mesh)',
        id: 'mesh',
        preview: '/assets/textures/mesh/base.jpg',
        config: {
            map: '/assets/textures/mesh/base.jpg',
            normalMap: '/assets/textures/mesh/normal.jpg',
            roughnessMap: '/assets/textures/mesh/roughness.jpg',
            aoMap: '/assets/textures/mesh/ao.jpg',

            color: null,
            repeat: [6, 6], // 网眼更密
            normalScale: 0.8,
            roughness: 0.5, // 稍微光滑一点
            sheen: 0.2 // 化纤材质绒毛感较弱
        }
    }
];

// ==========================================
// 2. Three.js 核心初始化
// ==========================================
const initThree = () => {
    // 1. 创建场景
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x222222);

    // 2. 创建相机
    const width = canvasContainer.value.clientWidth;
    const height = canvasContainer.value.clientHeight;
    camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100);
    camera.position.set(2, 1.5, 2);

    // 3. 创建渲染器 (配置高性能渲染参数)
    renderer = new THREE.WebGLRenderer({
        antialias: true, // 抗锯齿
        alpha: true // 允许背景透明
    });
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 限制像素比，平衡性能与画质

    // --- 真实感渲染关键设置 ---
    renderer.useLegacyLights = false; // 使用物理正确光照
    renderer.toneMapping = THREE.ACESFilmicToneMapping; // 电影级色调映射，防止高光过曝
    renderer.toneMappingExposure = 1.0;
    renderer.outputColorSpace = THREE.SRGBColorSpace; // 保证纹理颜色还原准确
    renderer.shadowMap.enabled = true; // 开启阴影
    renderer.shadowMap.type = THREE.PCFSoftShadowMap; // 软阴影

    canvasContainer.value.appendChild(renderer.domElement);

    // 4. 轨道控制器
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true; // 开启阻尼（惯性），交互更顺滑
    controls.minDistance = 0.5;
    controls.maxDistance = 5;

    // 5. 性能监控 (FPS) - 面试加分项：关注性能
    stats = new Stats();
    stats.showPanel(0);
    stats.dom.style.position = 'absolute';
    stats.dom.style.left = '10px';
    stats.dom.style.top = '10px';
    canvasContainer.value.appendChild(stats.dom);

    // 6. GUI 面板初始化
    initGUI();
};

const initGUI = () => {
    gui = new GUI({ title: '参数调试 (Dev)' });
    gui.domElement.style.position = 'absolute';
    gui.domElement.style.right = '10px';
    gui.domElement.style.top = '10px';
};

// ==========================================
// 3. 资源加载与场景构建
// ==========================================
const loadAssets = async () => {
    // 使用 LoadingManager 统一管理所有加载进度
    const loadingManager = new THREE.LoadingManager();
    const gltfLoader = new GLTFLoader(loadingManager);

    loadingManager.onProgress = (url, loaded, total) => {
        loadingProgress.value = Math.floor((loaded / total) * 100);
    };
    loadingManager.onLoad = () => {
        // 当所有资源(包括材质切换里的贴图)都就绪后，隐藏 Loading
        // 注意：这里可能需要延时一点，防止材质切换还没完成
    };

    try {
        // 1. 并行加载环境光
        await loadEnvironment('room');

        // 2. 加载 3D 模型
        // 面试话术：实际项目中我会使用 Dracro 压缩的 GLB 模型以减少体积
        const gltf = await gltfLoader.loadAsync('/assets/models/MaterialsVariantsShoe.glb');

        console.log('gltf', gltf);

        const model = gltf.scene;
        shoeModelGroup = model; // 保存引用，供切换材质使用

        // 3. 初始化材质对象
        // 这里我们创建一个空的 PhysicalMaterial，稍后由 handleTextureChange 填充数据
        fabricMaterial = new THREE.MeshPhysicalMaterial({
            side: THREE.DoubleSide,
            color: 0xffffff // 默认白底，防止染色
        });

        // 4. 遍历模型，赋予材质
        model.traverse((child) => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;

                console.log(`正在处理部件: ${child.name}`);

                // 情况一：这是一个多材质物体 (材质是个数组)
                // 这个官方鞋子模型通常就是这种情况！
                if (Array.isArray(child.material)) {
                    console.log('检测到多重材质，原始材质列表:', child.material.map(m => m.name));

                    // 我们需要创建一个新数组，保留不想改的，替换想改的
                    // child.material 是引用，直接修改它的元素即可
                    child.material.forEach((mat, index) => {
                        // 这里需要你根据打印出来的名字判断
                        // 通常鞋面材质名字里不带 "sole" (鞋底) 或者 "interior" (内衬)
                        // 对于 MaterialsVariantsShoe 这个模型：
                        // 通常 Index 0 是鞋面，Index 1 是鞋底/内衬 (具体看控制台打印)

                        const matName = mat.name.toLowerCase();

                        // 逻辑：如果名字里【不包含】'sole' 且【不包含】'rubber'，我们就认为是鞋面
                        // 如果你的控制台打印出的名字没有任何特征，可以尝试只替换 index === 0
                        if (!matName.includes('sole') && !matName.includes('rubber')) {
                            console.log(`--> 替换索引 ${index} (${mat.name}) 为飞织材质`);

                            // 【关键】保留数组结构，只替换这一个元素
                            child.material[index] = fabricMaterial;
                        } else {
                            console.log(`--> 保留索引 ${index} (${mat.name}) 原样`);
                            // 可选：给原样保留的鞋底也加上环境光反射，否则看起来太假
                            mat.envMap = scene.environment;
                            mat.needsUpdate = true;
                        }
                    });
                }
                // 情况二：这是一个单材质物体
                else {
                    // 如果名字包含 'shoe' 且不包含 'sole'
                    if (child.name.toLowerCase().includes('shoe') && !child.name.toLowerCase().includes('sole')) {
                        child.material = fabricMaterial;
                    }
                }
            }
        });

        // 5. 模型居中校正
        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center);

        // 放大模型
        model.scale.set(3, 3, 3);

        scene.add(model);

        // 6. 【关键修复】初始化完成后，立即调用材质切换函数应用第0个预设
        // 这样确保了初始效果和配置表里的第一个完全一致
        await handleTextureChange(0);

        // 资源全部就绪
        loading.value = false;
        setupMaterialGUI(); // 绑定 GUI

    } catch (error) {
        console.error('致命错误: 资源加载失败', error);
        loading.value = false;
    }
};

// ==========================================
// 4. 核心功能：材质切换 System
// ==========================================
const handleTextureChange = async (index) => {
    // 防止重复点击，但在初始化时(index=0, current=-1)允许执行
    if (currentTextureIndex.value === index && index !== 0) return;

    currentTextureIndex.value = index;
    loading.value = true;

    const preset = textureOptions[index];
    const conf = preset.config;
    const loader = new THREE.TextureLoader();

    try {
        // 1. 异步并行加载所有贴图
        // 使用 catch 容错，即使缺一张 AO 图也不影响其他图显示
        const loadTask = (path) => path ? loader.loadAsync(path).catch(e => null) : Promise.resolve(null);

        const [map, normalMap, roughnessMap, aoMap] = await Promise.all([
            loadTask(conf.map),
            loadTask(conf.normalMap),
            loadTask(conf.roughnessMap),
            loadTask(conf.aoMap)
        ]);

        // 2. 纹理预处理
        const textures = [map, normalMap, roughnessMap, aoMap];
        textures.forEach(t => {
            if (t) {
                t.wrapS = t.wrapT = THREE.RepeatWrapping; // 开启平铺模式
                t.repeat.set(conf.repeat[0], conf.repeat[1]); // 设置密度
            }
        });

        // 颜色空间校正 (必须，否则颜色发灰)
        if (map) map.colorSpace = THREE.SRGBColorSpace;

        // 3. 应用到材质
        if (fabricMaterial) {
            // A. 显存清理 (防止内存泄漏)
            // 面试话术：在 WebGL 中频繁切换贴图必须手动 dispose，否则会撑爆显存
            if (fabricMaterial.map) fabricMaterial.map.dispose();
            if (fabricMaterial.normalMap) fabricMaterial.normalMap.dispose();
            if (fabricMaterial.roughnessMap) fabricMaterial.roughnessMap.dispose();
            if (fabricMaterial.aoMap) fabricMaterial.aoMap.dispose();

            // B. 属性赋值
            fabricMaterial.map = map || null;
            fabricMaterial.normalMap = normalMap || null;
            fabricMaterial.roughnessMap = roughnessMap || null;
            fabricMaterial.aoMap = aoMap || null;

            // C. 颜色与物理参数重置
            // 如果配置里没有 color，必须重置为白色，否则会被上一次的颜色染色
            fabricMaterial.color.setHex(conf.color !== null && conf.color !== undefined ? conf.color : 0xffffff);

            fabricMaterial.normalScale.set(
                conf.normalScale !== undefined ? conf.normalScale : 1,
                conf.normalScale !== undefined ? conf.normalScale : 1
            );
            fabricMaterial.roughness = conf.roughness;
            fabricMaterial.sheen = conf.sheen || 0;
            if (conf.sheenColor) fabricMaterial.sheenColor.setHex(conf.sheenColor);

            // D. 标记更新
            fabricMaterial.needsUpdate = true;
        }

    } catch (e) {
        console.error('材质切换异常:', e);
    } finally {
        loading.value = false;
    }
};

// ==========================================
// 5. 辅助功能：环境光 & GUI
// ==========================================
const loadEnvironment = async (type) => {
    currentEnv.value = type;
    const loader = new EXRLoader();
    try {
        const texture = await loader.loadAsync(envOptions[type].url);
        texture.mapping = THREE.EquirectangularReflectionMapping;
        scene.environment = texture;
        scene.background = texture; // 可选：设置背景
    } catch (e) {
        console.warn('HDR加载失败，回退到默认光照');
    }
};

const changeEnvironment = (key) => loadEnvironment(key);

const setupMaterialGUI = () => {
    if (!fabricMaterial || !gui) return;
    // 先清理旧的 GUI 文件夹（如果有）
    // 此处简化，实际可用 gui.folders 遍历清理

    const params = {
        color: fabricMaterial.color.getHex(),
        roughness: fabricMaterial.roughness,
        normalScale: 1.0,
        sheen: fabricMaterial.sheen,
    };

    const folder = gui.addFolder('实时材质调试');
    folder.addColor(params, 'color').name('基础颜色').onChange(v => fabricMaterial.color.setHex(v));
    folder.add(params, 'roughness', 0, 1).name('粗糙度').onChange(v => fabricMaterial.roughness = v);
    folder.add(params, 'normalScale', 0, 3).name('法线强度').onChange(v => fabricMaterial.normalScale.set(v, v));
    folder.add(params, 'sheen', 0, 2).name('绒毛感(Sheen)').onChange(v => fabricMaterial.sheen = v);
    folder.open();
};

// ==========================================
// 6. 渲染循环与生命周期
// ==========================================
const animate = () => {
    animationId = requestAnimationFrame(animate);
    stats.begin();
    controls.update();
    renderer.render(scene, camera);
    stats.end();
};

const onWindowResize = () => {
    if (!canvasContainer.value) return;
    const w = canvasContainer.value.clientWidth;
    const h = canvasContainer.value.clientHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
};

onMounted(async () => {
    initThree();
    await loadAssets();
    animate();
    window.addEventListener('resize', onWindowResize);
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    cancelAnimationFrame(animationId);
    if (gui) gui.destroy();
    if (renderer) {
        renderer.dispose();
        renderer.forceContextLoss();
    }
    // 销毁场景中的对象
    if (scene) {
        scene.traverse((child) => {
            if (child.isMesh) {
                child.geometry.dispose();
                if (child.material) {
                    // 处理多材质数组的情况
                    if (Array.isArray(child.material)) {
                        child.material.forEach(m => m.dispose());
                    } else {
                        child.material.dispose();
                    }
                }
            }
        });
    }
});
</script>

<style scoped>
/* 基础容器 */
.page-container {
    width: 100%;
    height: 100vh;
    position: relative;
    overflow: hidden;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    user-select: none;
}

.viewer-container {
    width: 100%;
    height: 100%;
    /* 使用径向渐变背景，突出中心模型 */
    background: radial-gradient(circle at center, #3a3a3a 0%, #1a1a1a 100%);
}

/* Loading 遮罩 */
.loading-overlay {
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.9);
    display: flex;
    justify-content: center;
    align-items: center;
    color: white;
    z-index: 100;
    backdrop-filter: blur(10px);
}

.fade-enter-active,
.fade-leave-active {
    transition: opacity 0.5s;
}

.fade-enter-from,
.fade-leave-to {
    opacity: 0;
}

.spinner {
    width: 50px;
    height: 50px;
    border: 3px solid rgba(255, 255, 255, 0.3);
    border-radius: 50%;
    border-top-color: #fff;
    animation: spin 1s ease-in-out infinite;
    margin: 0 auto 15px;
}

@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}

/* 材质面板 */
.texture-panel {
    position: absolute;
    left: 20px;
    top: 50%;
    transform: translateY(-50%);
    background: rgba(20, 20, 20, 0.7);
    padding: 20px;
    border-radius: 16px;
    backdrop-filter: blur(12px);
    color: white;
    width: 140px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
    z-index: 10;
}

.texture-panel h3 {
    margin: 0 0 15px 0;
    font-size: 14px;
    color: #888;
    text-transform: uppercase;
    letter-spacing: 1px;
}

.texture-list {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.texture-item {
    height: 60px;
    border-radius: 10px;
    background-size: cover;
    background-position: center;
    cursor: pointer;
    position: relative;
    border: 2px solid rgba(255, 255, 255, 0.1);
    transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
    overflow: hidden;
}

.texture-item:hover {
    transform: translateY(-2px);
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.texture-item.active {
    border-color: #4cd964;
    box-shadow: 0 0 0 4px rgba(76, 217, 100, 0.2);
}

.texture-name {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    background: linear-gradient(to top, rgba(0, 0, 0, 0.8), transparent);
    font-size: 12px;
    padding: 6px;
    text-align: center;
    font-weight: 500;
}

/* 底部环境切换 */
.bottom-bar {
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 8px;
    background: rgba(20, 20, 20, 0.7);
    padding: 8px;
    border-radius: 20px;
    backdrop-filter: blur(12px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    z-index: 10;
}

.env-item {
    color: #ccc;
    cursor: pointer;
    padding: 8px 20px;
    border-radius: 14px;
    transition: all 0.3s;
    font-size: 13px;
    font-weight: 600;
}

.env-item:hover {
    color: white;
    background: rgba(255, 255, 255, 0.1);
}

.env-item.active {
    color: #111;
    background: #fff;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}
</style>