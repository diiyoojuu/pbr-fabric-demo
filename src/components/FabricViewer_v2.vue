<template>
    <div class="page-container">
        <!-- 3D 画布 -->
        <div ref="canvasContainer" class="viewer-container"></div>

        <!-- 加载遮罩 -->
        <div v-if="loading" class="loading-overlay">
            <div class="loader-content">
                <div class="spinner"></div>
                <p>资源加载中... {{ loadingProgress }}%</p>
            </div>
        </div>

        <div class="texture-panel">
            <h3>面料选择</h3>
            <div class="texture-list">
                <div v-for="(tex, index) in textureOptions" :key="index" class="texture-item"
                    :class="{ active: currentTextureIndex === index }" @click="handleTextureChange(index)"
                    :style="{ backgroundImage: `url(${tex.preview})` }">
                    <span class="texture-name">{{ tex.name }}</span>
                </div>
            </div>
        </div>

        <!-- 底部环境切换栏 -->
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

// --- 状态与引用 ---
const canvasContainer = ref(null);
const loading = ref(true);
const loadingProgress = ref(0);
const currentEnv = ref('studio');
const currentTextureIndex = ref(0);

// Three.js 核心对象
let scene, camera, renderer, controls, stats, gui;
let shoeMesh = null; // 存储鞋子模型的引用
let fabricMaterial = null; // 存储核心材质引用，方便修改参数
let animationId;

// 环境光配置
const envOptions = {
    room: { name: '房间', url: '/assets/exr/royal_esplanade_4k.exr' },
    outDoor: { name: '草地', url: '/assets/exr/river_alcove_4k.exr' }
};

// 材质切换
const textureOptions = [
    {
        name: '默认 (normal)',
        id: 'normal',
        // 预览图可以用 BaseColor 替代
        preview: '/assets/textures/fabric/base.jpg',
        config: {
            // 贴图路径 (假设你下载了对应图片，如果没有，暂时留空或用颜色模拟)
            map: '/assets/textures/fabric/base.jpg',
            normalMap: '/assets/textures/fabric/normal.jpg',
            roughnessMap: '/assets/textures/fabric/roughness.jpg',
            aoMap: '/assets/textures/fabric/ao.jpg',
            // 特殊参数微调
            repeat: [4, 4], // 纹理密度
            normalScale: new THREE.Vector2(1, 1), // 凹凸程度
            sheen: 1.0, // 针织绒毛感强
            roughness: 0.7,
            sheenRoughness: 0.5,
            sheenColor: 0xffffff,

            // 其他高级属性
            clearcoat: 0.0, // 织物通常没有清漆层
            side: THREE.DoubleSide
        }
    },
    {
        name: '经典飞织 (Knit)',
        id: 'knit',
        // 预览图可以用 BaseColor 替代
        preview: '/assets/textures/knit/base.jpg',
        config: {
            // 贴图路径 (假设你下载了对应图片，如果没有，暂时留空或用颜色模拟)
            map: '/assets/textures/knit/base.jpg',
            normalMap: '/assets/textures/knit/normal.jpg',
            roughnessMap: '/assets/textures/knit/roughness.jpg',
            aoMap: '',
            // 特殊参数微调
            repeat: [4, 4], // 纹理密度
            normalScale: 1.5, // 凹凸程度
            sheen: 1.0, // 针织绒毛感强
            roughness: 0.7
        }
    },
    {
        name: '透气网布 (mesh)', // 对比材质
        id: 'leather',
        preview: '/assets/textures/mesh/base.jpg',
        config: {
            map: '/assets/textures/mesh/base.jpg',
            normalMap: '/assets/textures/mesh/normal.jpg',
            roughnessMap: '/assets/textures/mesh/roughness.jpg',
            aoMap: '',
            aoMap: '/assets/textures/leather/ao.jpg',
            repeat: [6, 6], // 网眼通常更密
            normalScale: 0.8,
            sheen: 0.2, // 网眼通常是化纤，光泽感不同
            roughness: 0.5
        }
    }
];

// --- 初始化场景 ---
const initThree = () => {
    // 1. 场景
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x222222);

    // 2. 相机
    const width = canvasContainer.value.clientWidth;
    const height = canvasContainer.value.clientHeight;
    camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100);
    camera.position.set(2, 1.5, 2); // 调整初始视角

    // 3. 渲染器
    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // 性能优化：限制最大像素比

    // PBR 关键设置
    renderer.useLegacyLights = false;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.0;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    // 开启阴影
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;

    canvasContainer.value.appendChild(renderer.domElement);

    // 4. 控制器
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.minDistance = 0.5;
    controls.maxDistance = 5;

    // 5. 性能监控 (FPS)
    stats = new Stats();
    stats.showPanel(0); // 0: fps, 1: ms
    stats.dom.style.position = 'absolute';
    stats.dom.style.left = '10px';
    stats.dom.style.top = '10px';
    canvasContainer.value.appendChild(stats.dom);

    // 6. 添加调试面板 (GUI)
    initGUI();
};

// --- 初始化 GUI 面板 ---
const initGUI = () => {
    gui = new GUI({ title: '材质参数调试' });
    gui.domElement.style.position = 'absolute';
    gui.domElement.style.right = '10px';
    gui.domElement.style.top = '10px';

    // 初始占位，等材质加载完再添加具体的控制器
};

// --- 加载资源 (核心逻辑) ---
const loadAssets = async () => {
    const loadingManager = new THREE.LoadingManager();
    const textureLoader = new THREE.TextureLoader(loadingManager);
    const exrLoader = new EXRLoader(loadingManager);
    const gltfLoader = new GLTFLoader(loadingManager);

    loadingManager.onProgress = (url, loaded, total) => {
        loadingProgress.value = Math.floor((loaded / total) * 100);
    };
    loadingManager.onLoad = () => {
        loading.value = false;
        setupMaterialGUI(); // 资源加载完毕后绑定 GUI
    };

    try {
        // 1. 加载初始环境光
        await loadEnvironment('room');

        // 2. 加载纹理 (模拟飞织面料)
        // 为了演示效果，这里使用颜色图和法线图。如果没有本地图片，可以暂时注释掉 map/normalMap 的加载
        // 假设你还是用本地的路径，请确保图片存在。
        // 这里为了让你能直接运行，我使用纯色+生成法线的方式，或者你可以解开下面的注释加载图片

        // const baseColorMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_basecolor.jpg');
        // const normalMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_normal.jpg');
        // const roughnessMap = await textureLoader.loadAsync('/assets/textures/fabric/normal/fabric_roughness.jpg');

        // 设置重复
        // [baseColorMap, normalMap, roughnessMap].forEach(t => {
        //   t.wrapS = t.wrapT = THREE.RepeatWrapping;
        //   t.repeat.set(4, 4);
        //   t.colorSpace = THREE.SRGBColorSpace;
        // });

        console.log(textureOptions);

        const baseColorMap = await textureLoader.loadAsync(textureOptions[0].config.map);
        const normalMap = await textureLoader.loadAsync(textureOptions[0].config.normalMap);
        const roughnessMap = await textureLoader.loadAsync(textureOptions[0].config.roughnessMap);        

        console.log(baseColorMap);
        

        // --- 核心考点：MeshPhysicalMaterial ---
        fabricMaterial = new THREE.MeshPhysicalMaterial({
            // color: 0xff3333, // 初始红色
            roughnessMap: roughnessMap,
            // roughness: 0.7,   // 织物通常比较粗糙
            metalness: 0.0,

            // 法线 (模拟编织凹凸)
            normalMap: normalMap,
            normalScale: new THREE.Vector2(1, 1),

            // **Sheen (光泽/绒毛感) - 飞织的关键**
            // 这模拟了光线在微小纤维上的反光，让表面看起来像布而不是塑料
            sheen: 1.0,
            sheenRoughness: 0.5,
            sheenColor: 0xffffff,

            // 其他高级属性
            clearcoat: 0.0, // 织物通常没有清漆层
            side: THREE.DoubleSide
        });

        // 3. 加载鞋子模型
        // 使用开源的 Nike Air Jordan 模型作为演示
        // 来源: Sketchfab / pmndrs
        const gltf = await gltfLoader.loadAsync('/assets/models/MaterialsVariantsShoe.glb');

        const model = gltf.scene;

        // 遍历模型，替换材质
        model.traverse((child) => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;

                // 简单判断：假设名字里带 "shoe" 或 "fabric" 的部分应用我们的飞织材质
                // 在实际项目中，你需要根据模型具体的 mesh name 来区分鞋底(橡胶)和鞋面(飞织)
                // 这里为了演示效果，我们把主要部分替换掉
                if (child.name.includes('shoe') || true) { // 暂时全部替换方便看效果
                    // 保持原有的 UV 映射，替换材质
                    // child.material = fabricMaterial; 

                    // 为了保留鞋子原本的纹理细节但增加飞织感，
                    // 更好的做法是：保留原材质的 map，但升级为 PhysicalMaterial
                    const oldMat = child.material;
                    const newMat = fabricMaterial.clone();
                    if (oldMat.map) newMat.map = oldMat.map; // 保留原纹理
                    child.material = newMat;

                    // 保存第一个主要材质的引用给 GUI 控制
                    if (!shoeMesh) {
                        shoeMesh = child;
                        fabricMaterial = newMat;
                    }
                }
            }
        });

        // 自动居中模型
        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        model.position.sub(center); // 移到原点
        scene.add(model);

        // 添加一个地面接收阴影
        // const plane = new THREE.Mesh(
        //     new THREE.PlaneGeometry(10, 10),
        //     new THREE.MeshStandardMaterial({ color: 0x444444, roughness: 0.8, metalness: 0.2 })
        // );
        // plane.rotation.x = -Math.PI / 2;
        // plane.position.y = box.min.y - 0.06; // 放在鞋底下面一点点
        // plane.receiveShadow = true;
        // scene.add(plane);

    } catch (error) {
        console.error('资源加载失败:', error);
        loading.value = false;
    }
};

// --- 环境光切换逻辑 ---
const loadEnvironment = async (type) => {
    currentEnv.value = type;
    const loader = new EXRLoader();
    const texture = await loader.loadAsync(envOptions[type].url);
    texture.mapping = THREE.EquirectangularReflectionMapping;

    scene.environment = texture;
    scene.background = new THREE.Color(0x333333); // 保持深色背景，突出产品
    scene.background = texture; // 如果想看环境图，解开这行
};

const changeEnvironment = (key) => {
    loadEnvironment(key);
};

// --- 切换材质的核心函数 (面试重点：资源管理与异步加载) ---
const handleTextureChange = async (index) => {
    if (currentTextureIndex.value === index) return;
    currentTextureIndex.value = index;
    loading.value = true; // 显示加载圈

    const preset = textureOptions[index];
    const conf = preset.config;
    const loader = new THREE.TextureLoader();

    try {
        // 1. 并行加载所有贴图 (使用 Promise.all 提高速度)
        // 注意：这里加了 catch 允许部分贴图加载失败（比如没有AO图时）
        const loadTask = (path) => path ? loader.loadAsync(path).catch(() => null) : Promise.resolve(null);

        const [map, normalMap, roughnessMap, aoMap] = await Promise.all([
            loadTask(conf.map),
            loadTask(conf.normalMap),
            loadTask(conf.roughnessMap),
            loadTask(conf.aoMap)
        ]);

        // 2. 纹理预处理 (重要：面试考点)
        const textures = [map, normalMap, roughnessMap, aoMap];
        textures.forEach(t => {
            if (t) {
                // 织物需要平铺
                t.wrapS = t.wrapT = THREE.RepeatWrapping;
                t.repeat.set(conf.repeat[0], conf.repeat[1]);
            }
        });

        // 基础色必须设为 sRGB 空间，否则颜色会发灰
        if (map) map.colorSpace = THREE.SRGBColorSpace;

        // 3. 更新材质属性
        if (fabricMaterial) {
            // 3.1 内存清理：释放旧贴图显存 (非常重要！否则切换几次页面就卡死)
            // 面试话术： "在切换材质时，我注意了手动 dispose 旧的 texture，防止显存泄漏"
            if (fabricMaterial.map) fabricMaterial.map.dispose();
            if (fabricMaterial.normalMap) fabricMaterial.normalMap.dispose();
            if (fabricMaterial.roughnessMap) fabricMaterial.roughnessMap.dispose();
            if (fabricMaterial.aoMap) fabricMaterial.aoMap.dispose();

            // 3.2 应用新贴图
            fabricMaterial.map = map;
            fabricMaterial.normalMap = normalMap;
            fabricMaterial.roughnessMap = roughnessMap;
            fabricMaterial.aoMap = aoMap;

            // 3.3 应用物理参数
            fabricMaterial.normalScale.set(conf.normalScale, conf.normalScale);
            fabricMaterial.roughness = conf.roughness;
            fabricMaterial.sheen = conf.sheen;

            // 3.4 标记更新
            fabricMaterial.needsUpdate = true;
        }

    } catch (e) {
        console.error('材质切换失败', e);
    } finally {
        loading.value = false;
    }
};

// --- 配置 GUI 面板 (在材质加载后调用) ---
const setupMaterialGUI = () => {
    if (!fabricMaterial || !gui) return;

    const params = {
        color: fabricMaterial.color.getHex(),
        roughness: fabricMaterial.roughness,
        normalScale: 1.0,
        sheen: fabricMaterial.sheen,
        sheenRoughness: fabricMaterial.sheenRoughness,
    };

    const matFolder = gui.addFolder('面料属性 (PBR)');

    matFolder.addColor(params, 'color').name('面料颜色').onChange(val => {
        fabricMaterial.color.setHex(val);
    });

    matFolder.add(params, 'roughness', 0, 1).name('粗糙度').onChange(val => {
        fabricMaterial.roughness = val;
    });

    matFolder.add(params, 'normalScale', 0, 3).name('凹凸/法线强度').onChange(val => {
        fabricMaterial.normalScale.set(val, val);
    });

    const sheenFolder = gui.addFolder('飞织绒毛感 (Sheen)');
    sheenFolder.add(params, 'sheen', 0, 2).name('绒毛强度').onChange(val => {
        fabricMaterial.sheen = val;
    });
    sheenFolder.add(params, 'sheenRoughness', 0, 1).name('绒毛粗糙度').onChange(val => {
        fabricMaterial.sheenRoughness = val;
    });

    matFolder.open();
    sheenFolder.open();
};

// --- 渲染循环 ---
const animate = () => {
    animationId = requestAnimationFrame(animate);
    stats.begin(); // 开始监控
    controls.update();
    renderer.render(scene, camera);
    stats.end(); // 结束监控
};

// --- Resize ---
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
    if (gui) gui.destroy(); // 销毁 GUI
    if (renderer) {
        renderer.dispose();
        renderer.forceContextLoss();
    }
});
</script>

<style scoped>
.page-container {
    width: 100%;
    height: 100vh;
    position: relative;
    overflow: hidden;
    font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
}

.viewer-container {
    width: 100%;
    height: 100%;
    background: radial-gradient(circle at center, #2b2b2b 0%, #1a1a1a 100%);
    position: absolute;
    top: 0;
    left: 0;
    z-index: 1;
}

/* Loading */
.loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.85);
    display: flex;
    justify-content: center;
    align-items: center;
    color: white;
    z-index: 999;
}

.spinner {
    width: 40px;
    height: 40px;
    border: 4px solid #fff;
    border-top: 4px solid transparent;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin: 0 auto 10px;
}

@keyframes spin {
    0% {
        transform: rotate(0deg);
    }

    100% {
        transform: rotate(360deg);
    }
}

/* 底部切换栏 */
.bottom-bar {
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 15px;
    background: rgba(0, 0, 0, 0.6);
    padding: 10px 20px;
    border-radius: 30px;
    backdrop-filter: blur(10px);
    z-index: 1;
}

.env-item {
    color: #aaa;
    cursor: pointer;
    padding: 5px 15px;
    border-radius: 20px;
    transition: all 0.3s;
    font-size: 14px;
    font-weight: 500;
}

.env-item:hover {
    color: white;
    background: rgba(255, 255, 255, 0.1);
}

.env-item.active {
    color: #000;
    background: #fff;
    box-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
}

.texture-panel {
    position: absolute;
    left: 20px;
    bottom: 100px;
    /* 在环境光切换栏上方 */
    background: rgba(0, 0, 0, 0.6);
    padding: 15px;
    border-radius: 12px;
    backdrop-filter: blur(8px);
    color: white;
    width: 200px;
    z-index: 1;
}

.texture-panel h3 {
    margin: 0 0 10px 0;
    font-size: 14px;
    opacity: 0.8;
}

.texture-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.texture-item {
    height: 50px;
    border-radius: 8px;
    background-size: cover;
    background-position: center;
    cursor: pointer;
    position: relative;
    border: 2px solid transparent;
    transition: all 0.2s;
    overflow: hidden;
}

.texture-item:hover {
    transform: scale(1.02);
}

.texture-item.active {
    border-color: #4af;
    box-shadow: 0 0 8px rgba(68, 170, 255, 0.5);
}

.texture-name {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    background: rgba(0, 0, 0, 0.6);
    font-size: 12px;
    padding: 4px;
    text-align: center;
}
</style>