<template>
    <div class="content">
        <div class="header">
            <el-button type="primary" @click="modelExplode">展开</el-button>
            <el-button type="primary" @click="modelRestore">收起</el-button>
            <el-button type="primary" @click="addIcon">添加图标</el-button>
        </div>
        <div id="MAIN" class="main"></div>
        <!-- 新增：模型模块名称显示标签 -->
        <div id="nameTag" class="model-name-tag" style="display: none"></div>
    </div>
</template>

<script setup>
import { ref } from "vue";
// 引入threeJS
import * as THREE from "three";

// 引入轨道控制器
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

// 引入模型加载器
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
const loader = new GLTFLoader();

// 引入将原生DOM转化为3D并且渲染到3D场景的方法(引入3d转换器与渲染器)
import { CSS3DRenderer } from "three/addons/renderers/CSS3DRenderer.js";
import { onMounted } from "vue";

// 引入dat.gui库
import * as dat from "dat.gui";
const gui = new dat.GUI();

import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { OutlinePass } from "three/examples/jsm/postprocessing/OutlinePass";
const raycaster = new THREE.Raycaster();
let composer; // 效果合成器
let outlinePass;

// 引入gsap动画库
import gsap from "gsap";

// 声明场景、摄像机、3D渲染器、轨道控制器、3D原生DOM渲染器等全局变量
let scene, camera, renderer, controls, css3dRenderer, mainDom, model;
// 模型缓存数组，用于切换模型时不用重新加载模型，算是一种性能优化方式 (暂时不用 不需要切换模型)
let modelPool = [];

// 页面图标数组
const iconArray = ref([]);

// 1.初始化场景
const init = () => {
    //   获取dom
    mainDom = document.getElementById("MAIN");
    // 创建场景
    scene = new THREE.Scene();
    // 创建摄像机
    camera = new THREE.PerspectiveCamera(75, mainDom.clientWidth / mainDom.clientHeight, 0.1, 1000);
    //   初始化相机位置
    camera.position.z = 7;
    // 创建渲染器（画布）
    renderer = new THREE.WebGLRenderer({ antialias: true });
    //   设置画布大小
    renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    // 将渲染器添加到dom
    mainDom.append(renderer.domElement);
};

// 2.创建轨道控制器
const cretaeControls = () => {
    // 创建轨道控制器
    controls = new OrbitControls(camera, renderer.domElement);
    // 垂直方向旋转角度
    // controls.maxPolarAngle = Math.PI * 0.25;
    // 开启阻尼效果
    controls.enableDamping = true;
    //   禁止滚动缩放
    // controls.enableZoom = false;
};

// 3.场景适配
const resizeRender = () => {
    window.addEventListener("resize", () => {
        // 自适应摄像机的宽度比
        camera.aspect = mainDom.clientWidth / mainDom.clientHeight;
        // 自适应画布的宽高比
        renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        // 重新更新垂体空间
        camera.updateProjectionMatrix();
        // 更新 OutlinePass 尺寸
        if (outlinePass) {
            // 新增：防错 - 避免outlinePass未初始化
            outlinePass.resolution.set(mainDom.clientWidth, mainDom.clientHeight);
        }
        // 新增：更新合成器尺寸
        if (composer) {
            composer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        }
    });
};

// 4. 循环渲染
const renderLoop = () => {
    // 渲染更新渲染
    requestAnimationFrame(renderLoop);
    // 通过轨道控制器调用场景更新
    controls.update();
    // 将场景和摄像机渲染到画布上面
    // renderer.render(scene, camera);
    if (composer) {
        composer.render();
        // composer.render会调用renderer.render，所以这里不需要再调用一次
    }
};

// 5. 创建坐标轴
const createHelper = () => {
    // 创建坐标轴
    const axesHelper = new THREE.AxesHelper(3);
    // 将坐标轴添加到场景中
    scene.add(axesHelper);
};

// 6. 引入模型文件
const createModel = () => {
    loader.load(
        "/public/AHU-9.glb",
        (gltf) => {
            gltf.scene.traverse((obj) => {
                const mesh = obj;
                // 记录原位置、移动位置
                mesh.fromPosition = mesh.position.clone();
                // multiplyScalar 放大倍数，将原位置放大10倍，展示爆炸之后的位置效果
                mesh.toPosition = mesh.position.clone().multiplyScalar(10);
                // 新增：如果模型子模块没有名称，给一个默认名称（可选）
                if (!mesh.name || mesh.name === "") {
                    mesh.name = `模块_${Math.floor(Math.random() * 1000)}`;
                }
            });

            // 储存模型数据
            model = gltf.scene;
            // 模型分解标识
            model.decomposition = false;
            // 模型缓存，用于切换模型时不用重新加载模型，算是一种性能优化方式
            modelPool.push(model);
            scene.add(gltf.scene);
        },
        function (xhr) {
            const percent = Math.floor((xhr.loaded / xhr.total) * 100); // 计算加载进度百分比
            // console.log(`模型加载进度：${percent}%`);
            if (percent == 100) {
                // 当模型加载完之后添加控制器
                setTimeout(() => {
                    createGui();
                }, 1000);
            }
        },
    );
};

// 7. 引入光照(模型文件引入之后是一团黑，需要光照)
const createLight = () => {
    const ambient = new THREE.AmbientLight(0xffffff, 1.4);
    scene.add(ambient);
    const directionalLight = new THREE.DirectionalLight(0xffffff, 3.5); //光源，color:灯光颜色，intensity:光照强度
    directionalLight.position.set(400, 200, 300);
    scene.add(directionalLight);
    // 新增：添加第二个平行光，提升亮度
    const directionalLight2 = new THREE.DirectionalLight(0xffffff, 2);
    directionalLight2.position.set(-400, -200, -300);
    scene.add(directionalLight2);
};

// 8. 根据dat.gui添加控制器
const createGui = () => {
    const folder = gui.addFolder("模型位置");
    console.log("model", model);
    folder.add(model.position, "x").min(-10).max(10).step(0.01).name("移动x轴");
    folder.add(model.position, "y").min(-10).max(10).step(0.01).name("移动y轴");
    folder.add(model.position, "z").min(-10).max(10).step(0.01).name("移动z轴");

    const folder1 = gui.addFolder("模型旋转角度");
    folder1.add(model.rotation, "x").min(-10).max(10).step(0.01).name("旋转x轴");
    folder1.add(model.rotation, "y").min(-10).max(10).step(0.01).name("旋转y轴");
    folder1.add(model.rotation, "z").min(-10).max(10).step(0.01).name("旋转z轴");
};

// 9.点击展开按钮实现模型分解
const modelExplode = () => {
    console.log("model", model);
    model.decomposition = true;
    model.traverse((obj) => {
        const mesh = obj;
        console.log("xyz", { x: mesh.toPosition.x, y: mesh.toPosition.y, z: mesh.toPosition.z });
        gsap.to(mesh.position, {
            x: mesh.toPosition.x,
            y: mesh.toPosition.y,
            z: mesh.toPosition.z,
            ease: "Power2.inOut",
            duration: 5,
        });
    });
};

// 10. 点击实现模型还原
const modelRestore = () => {
    model.decomposition = false;
    model.traverse((obj) => {
        const mesh = obj;
        gsap.to(mesh.position, {
            x: mesh.fromPosition.x,
            y: mesh.fromPosition.y,
            z: mesh.fromPosition.z,
            ease: "Power2.inOut",
            duration: 5,
        });
    });
};

// 11.鼠标滑过模型高亮当前
const selectMesh = (e) => {
    // 创建鼠标向量
    const mouse = new THREE.Vector2();
    // 计算鼠标点击位置的归一化设备坐标（NDC）
    // NDC 坐标系的范围是 [-1, 1]，左下角为 (-1, -1)，右上角为 (1, 1)
    if (!mainDom) return;
    // 全屏模式下获取鼠标点击位置的归一化设备坐标
    // mouse.x = (e.clientX / mainDom.clientWidth) * 2 - 1;
    // mouse.y = -(e.clientY / mainDom.clientHeight) * 2 + 1;

    let x = e.clientX;
    let y = e.clientY;
    // 非全屏模式下获取鼠标点击位置的归一化设备坐标
    mouse.x = ((x - mainDom.getBoundingClientRect().left) / mainDom.offsetWidth) * 2 - 1;
    mouse.y = -((y - mainDom.getBoundingClientRect().top) / mainDom.offsetHeight) * 2 + 1;

    // 更新射线的起点和方向
    raycaster.setFromCamera(mouse, camera);

    // 执行射线与物体的相交测试
    const intersects = raycaster.intersectObjects(scene.children);
    let nameTagDom = document.getElementById("nameTag");
    // console.log("intersects", intersects);
    // 检查是否有相交的物体
    if (intersects.length > 0) {
        const selectedObject = intersects[0].object;
        outlinePass.selectedObjects = [selectedObject];

        // 新增：显示模块名称
        const meshName = selectedObject.name || "未知模块";
        nameTagDom.textContent = meshName;
        nameTagDom.style.display = "block";
        // 定位标签到鼠标位置（偏移10px避免遮挡鼠标）
        nameTagDom.style.left = `${e.clientX + 10}px`;
        nameTagDom.style.top = `${e.clientY + 10}px`;
    } else {
        outlinePass.selectedObjects = [];
        nameTagDom.style.display = "none";
    }
};

// 12.效果合成器
const initComposer = () => {
    // 设置鼠标滑过物体时显示轮廓线会导致原有的光照变暗，应该是轮廓线导致的，所以将光照的亮度调高
    outlinePass = new OutlinePass(
        new THREE.Vector2(mainDom.clientWidth, mainDom.clientHeight),
        scene,
        camera,
    );
    console.log("outlinePass", outlinePass);
    outlinePass.visibleEdgeColor.set(0xff0000); // 设置轮廓线颜色为红色
    outlinePass.edgeStrength = 10; // 设置轮廓线强度

    // 添加 OutlinePass 到渲染器的通道中
    composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    composer.addPass(outlinePass);
};

// 13. 往页面上添加图标
const addIcon = () => {};

// 初始化渲染
onMounted(() => {
    init();
    cretaeControls();
    resizeRender();
    renderLoop();
    // createHelper();
    createModel();
    createLight();
    initComposer();
});
window.addEventListener("mousemove", selectMesh, false);
</script>

<style scoped lang="scss">
.content {
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
    .header {
        display: flex;
        align-items: center;
        gap: 16px;
        padding: 16px;
        box-sizing: border-box;
    }
    .main {
        flex: 1;
    }
}
// 新增：名称标签样式
.model-name-tag {
    position: fixed;
    padding: 4px 8px;
    background: rgba(0, 0, 0, 0.8);
    color: #fff;
    font-size: 14px;
    border-radius: 4px;
    pointer-events: none; // 新增：避免标签遮挡鼠标事件
    z-index: 9999;
    white-space: nowrap;
}
</style>
