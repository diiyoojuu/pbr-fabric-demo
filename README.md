# High-Fidelity 3D Fabric/Shoe Demo (Web 3D 高保真面料展示器)

![Vue 3](https://img.shields.io/badge/Vue-3.x-4FC08D?style=flat-square)
![Three.js](https://img.shields.io/badge/Three.js-r150+-000000?style=flat-square)
![Status](https://img.shields.io/badge/Status-Interview%20Ready-blue)

这是一个基于 **Vue 3** 和 **Three.js** 开发的 Web 端高保真 3D 面料展示 Demo。
该项目旨在解决纺织/鞋服行业在 C2M（用户直连制造）场景下的数字化展示痛点，重点实现了**PBR（基于物理的渲染）材质还原**、**飞织/经编面料的细节模拟**以及**高性能的材质动态切换**。

## 🔗 在线演示 (Live Demo)

👉 **访问地址：[https://pbr-fabric-demo.vercel.app/](https://pbr-fabric-demo.vercel.app/)**

*(注：建议使用 PC 端浏览器访问以获得最佳纹理细节体验)*

---

## ✨ 核心功能 (Key Features)

### 1. 高保真面料渲染 (High-Fidelity Rendering)
*   **PBR 工作流**：使用 `MeshPhysicalMaterial` 替代基础材质，精准控制粗糙度 (Roughness)、金属度 (Metalness) 和法线 (Normal)。
*   **Sheen (光泽/绒毛感)**：针对“飞织”和“棉麻”材质，专门启用了 `Sheen` 属性，模拟织物表面微小纤维在侧光下的泛白与散射效果，拒绝“塑料感”。
*   **HDR 环境光照**：集成高动态范围成像 (HDRI) 照明，提供真实的漫反射和镜面反射环境。

### 2. 动态材质/纹理系统 (Dynamic Material System)
*   **混合映射逻辑**：
    *   **平铺模式**：支持动态加载平铺纹理（Tiling Textures），自动计算 UV 重复率，适用于 C2M 自定义选料场景。
*   **异步加载与容错**：使用 `Promise.all` 并行加载纹理组，并包含完整的错误降级处理。

### 3. 性能优化 (Performance Optimization)
*   **显存管理**：实现了自动垃圾回收机制。在切换材质时，严格执行 `dispose()` 清理旧纹理占用的 GPU 内存，防止显存泄漏。
*   **色彩空间校正**：严格遵循 `SRGBColorSpace` 和 `ACESFilmicToneMapping`，确保 Web 端渲染颜色与设计稿一致。
*   **FPS 监控**：内置 Stats.js 性能监控面板。

---

## 🛠 技术栈 (Tech Stack)

*   **核心框架**: Vue 3 (Composition API)
*   **3D 引擎**: Three.js
*   **模型加载**: GLTFLoader, EXRLoader (HDR支持)
*   **工具库**: 
    *   `lil-gui`: 用于实时调试材质物理参数
    *   `stats.js`: 用于性能FPS监控
*   **构建工具**: Vite

---

## 📂 项目结构 (Structure)

```text
├── public/
│   ├── assets/
│   │   ├── models/       # GLB 模型文件 (Draco压缩)
│   │   ├── textures/     # PBR 纹理贴图 (Base, Normal, Roughness, AO)
│   │   └── exr/          # HDR 环境贴图
├── src/
│   ├── App.vue           # 主应用入口
│   ├── components/       # (可选) UI 组件拆分
│   └── main.js
└── README.md

---

## 截图展示 (Screenshots)
<img width="2559" height="1347" alt="ae4a9ffb-4ac4-48ef-b72d-cb31f694a9ea" src="https://github.com/user-attachments/assets/3a958742-c555-479b-8526-7c95f0cc034c" />
<img width="2559" height="1347" alt="1a477468-5435-42ca-860e-3139761dca39" src="https://github.com/user-attachments/assets/a3d35bfa-8070-471c-954d-487a5f36db3e" />



