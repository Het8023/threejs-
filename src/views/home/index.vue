<template>
    <div class="content">
        <div class="header">
            <el-button type="primary" @click="modelExplode">展开</el-button>
            <el-button type="primary" @click="modelRestore">收起</el-button>
            <el-button type="primary" @click="addIcon">添加图标</el-button>
        </div>
        <div id="MAIN" class="main"></div>
        <div id="nameTag" class="model-name-tag" style="display: none"></div>
        <div id="contextMenu" class="context-menu" style="display: none">
            <div class="menu-item" @click="deleteSelectedCircle">删除该色块</div>
        </div>
    </div>
</template>

<script setup>
import { ref, onUnmounted } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";
import { onMounted } from "vue";
import * as dat from "dat.gui";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { OutlinePass } from "three/examples/jsm/postprocessing/OutlinePass";
import gsap from "gsap";

const raycaster = new THREE.Raycaster();
let composer;
let outlinePass;

let scene, camera, renderer, controls, mainDom, model;
let modelPool = [];
const iconArray = ref([]);

let circleBlocks = [];
let selectedCircle = null;
let isDragging = false;
let mouse = new THREE.Vector2();

let dragPlane = new THREE.Plane();
let dragRay = new THREE.Raycaster();
let dragIntersection = new THREE.Vector3();
let dragNormal = new THREE.Vector3();

let labelRenderer;
let contextMenuDom;
let gui;
let colorGuiFolder = null;
let colorNameController = null;

// 1. 初始化场景
const init = () => {
    mainDom = document.getElementById("MAIN");
    contextMenuDom = document.getElementById("contextMenu");
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(75, mainDom.clientWidth / mainDom.clientHeight, 0.1, 1000);
    camera.position.z = 7;
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    mainDom.appendChild(renderer.domElement);

    labelRenderer = new CSS2DRenderer();
    labelRenderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    labelRenderer.domElement.style.position = "absolute";
    labelRenderer.domElement.style.top = "0";
    labelRenderer.domElement.style.pointerEvents = "none";
    mainDom.appendChild(labelRenderer.domElement);
};

// 2. 轨道控制器
const cretaeControls = () => {
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
};

// 3. 适配
const resizeRender = () => {
    window.addEventListener("resize", () => {
        camera.aspect = mainDom.clientWidth / mainDom.clientHeight;
        renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        camera.updateProjectionMatrix();
        if (outlinePass) outlinePass.resolution.set(mainDom.clientWidth, mainDom.clientHeight);
        if (composer) composer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        if (labelRenderer) labelRenderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    });
};

// 4. 渲染
const renderLoop = () => {
    requestAnimationFrame(renderLoop);
    controls.update();

    // 文字缩放比例从60改为20，文字大小更适配
    circleBlocks.forEach((item) => {
        if (item.label) {
            const scale = item.mesh.scale.x;
            item.label.element.style.fontSize = `${15 * scale}px`; // 核心调整：缩放系数降低
        }
    });

    if (composer) composer.render();
    if (labelRenderer) labelRenderer.render(scene, camera);
};

// 5. 坐标轴
const createHelper = () => {
    const axes = new THREE.AxesHelper(3);
    scene.add(axes);
};

// 6. 模型（修复异步加载问题）
const createModel = () => {
    const loader = new GLTFLoader();
    loader.load(
        "/public/AHU-9.glb",
        (gltf) => {
            gltf.scene.traverse((obj) => {
                obj.fromPosition = obj.position.clone();
                obj.toPosition = obj.position.clone().multiplyScalar(10);
                if (!obj.name) obj.name = `模块_${Math.floor(Math.random() * 1000)}`;
            });
            model = gltf.scene;
            model.decomposition = false;
            modelPool.push(model);
            scene.add(model);

            // 模型加载完成后再创建GUI（核心修复）
            createGui();
        },
        (xhr) => {
            const percent = Math.floor((xhr.loaded / xhr.total) * 100);
            console.log(`模型加载进度：${percent}%`);
        },
        (err) => {
            console.error("模型加载失败：", err);
        },
    );
};

// 7. 灯光
const createLight = () => {
    const ambient = new THREE.AmbientLight(0xffffff, 1.4);
    scene.add(ambient);
    const dir1 = new THREE.DirectionalLight(0xffffff, 3.5);
    dir1.position.set(400, 200, 300);
    scene.add(dir1);
    const dir2 = new THREE.DirectionalLight(0xffffff, 2);
    dir2.position.set(-400, -200, -300);
    scene.add(dir2);
};

// 8. GUI（增加判空容错）
const createGui = () => {
    // 判空：model不存在则直接返回
    if (!model) {
        console.warn("模型未加载完成，暂不创建GUI");
        return;
    }

    gui = new dat.GUI();
    const f1 = gui.addFolder("模型位置");
    f1.add(model.position, "x", -10, 10, 0.01).name("X轴");
    f1.add(model.position, "y", -10, 10, 0.01).name("Y轴");
    f1.add(model.position, "z", -10, 10, 0.01).name("Z轴");

    const f2 = gui.addFolder("模型旋转");
    f2.add(model.rotation, "x", -10, 10, 0.01).name("X轴");
    f2.add(model.rotation, "y", -10, 10, 0.01).name("Y轴");
    f2.add(model.rotation, "z", -10, 10, 0.01).name("Z轴");
};

// 9. 展开（增加判空）
const modelExplode = () => {
    if (!model) {
        alert("模型未加载完成，无法展开");
        return;
    }
    model.decomposition = true;
    model.traverse((obj) => {
        gsap.to(obj.position, {
            x: obj.toPosition.x,
            y: obj.toPosition.y,
            z: obj.toPosition.z,
            duration: 5,
        });
    });
};

// 10. 收起（增加判空）
const modelRestore = () => {
    if (!model) {
        alert("模型未加载完成，无法收起");
        return;
    }
    model.decomposition = false;
    model.traverse((obj) => {
        gsap.to(obj.position, {
            x: obj.fromPosition.x,
            y: obj.fromPosition.y,
            z: obj.fromPosition.z,
            duration: 5,
        });
    });
};

// 11. 鼠标滑过
const selectMesh = (e) => {
    if (!mainDom) return;
    const rect = mainDom.getBoundingClientRect();
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);
    const tag = document.getElementById("nameTag");

    if (e.type === "click") {
        if (intersects.length > 0) {
            const obj = intersects[0].object;
            const isBlock = circleBlocks.some((b) => b.mesh === obj);
            if (isBlock) {
                selectedCircle = circleBlocks.find((b) => b.mesh === obj);
                outlinePass.selectedObjects = [selectedCircle.mesh];
                updateColorGui();
            } else {
                selectedCircle = null;
                outlinePass.selectedObjects = [obj];
                clearColorGui();
            }
        } else {
            selectedCircle = null;
            outlinePass.selectedObjects = [];
            clearColorGui();
        }
    } else {
        if (intersects.length > 0 && !selectedCircle) {
            outlinePass.selectedObjects = [intersects[0].object];
            tag.textContent = intersects[0].object.name || "未知";
            tag.style.display = "block";
            tag.style.left = e.clientX + 10 + "px";
            tag.style.top = e.clientY + 10 + "px";
        } else {
            if (!selectedCircle) outlinePass.selectedObjects = [];
            tag.style.display = "none";
        }
    }
};

// 12. 效果合成器 —— 完全没动！
const initComposer = () => {
    outlinePass = new OutlinePass(
        new THREE.Vector2(mainDom.clientWidth, mainDom.clientHeight),
        scene,
        camera,
    );
    outlinePass.visibleEdgeColor.set(0xff0000);
    outlinePass.edgeStrength = 10;
    composer = new EffectComposer(renderer);
    composer.addPass(new RenderPass(scene, camera));
    composer.addPass(outlinePass);
};

// ==================== 色块 GUI（增强版：旋转+大小） ====================
const updateColorGui = () => {
    clearColorGui();
    if (!selectedCircle || !gui) return;

    colorGuiFolder = gui.addFolder(`色块：${selectedCircle.mesh.name}`);
    colorGuiFolder.add(selectedCircle.mesh.position, "x", -20, 20, 0.01).name("X");
    colorGuiFolder.add(selectedCircle.mesh.position, "y", -20, 20, 0.01).name("Y");
    colorGuiFolder.add(selectedCircle.mesh.position, "z", -20, 20, 0.01).name("Z");

    // 旋转 XYZ
    colorGuiFolder.add(selectedCircle.mesh.rotation, "x", -Math.PI, Math.PI, 0.01).name("旋转 X");
    colorGuiFolder.add(selectedCircle.mesh.rotation, "y", -Math.PI, Math.PI, 0.01).name("旋转 Y");
    colorGuiFolder.add(selectedCircle.mesh.rotation, "z", -Math.PI, Math.PI, 0.01).name("旋转 Z");

    // 大小缩放
    colorGuiFolder
        .add(selectedCircle.mesh.scale, "x", 0.1, 5, 0.01)
        .name("大小")
        .onChange((v) => {
            selectedCircle.mesh.scale.setScalar(v);
        });

    // 名称
    colorNameController = colorGuiFolder
        .add({ name: selectedCircle.mesh.name }, "name")
        .name("名称");
    colorNameController.onFinishChange((val) => {
        selectedCircle.mesh.name = val;
        selectedCircle.label.element.textContent = val;
        colorGuiFolder.name(`色块：${val}`);
    });

    colorGuiFolder.open();
};

const clearColorGui = () => {
    if (colorGuiFolder && gui) {
        gui.removeFolder(colorGuiFolder);
        colorGuiFolder = null;
    }
    colorNameController = null;
};

// ==================== 色块创建 ====================
const createTextLabel = (text) => {
    const div = document.createElement("div");
    div.style.color = "#fff";
    div.style.fontSize = "8px"; // 基础字号从20px改为8px，大幅缩小
    div.style.fontWeight = "bold";
    div.style.textShadow = "1px 1px 2px #000";
    div.style.transform = "translate(-50%, -50%)"; // 保持居中
    div.style.textAlign = "center";
    div.style.whiteSpace = "nowrap";
    div.style.pointerEvents = "none";
    div.textContent = text;

    const label = new CSS2DObject(div);
    label.position.set(0, 0, 0); // 色块正中心
    return label;
};

const addIcon = () => {
    const geo = new THREE.CircleGeometry(0.5, 32);
    const mat = new THREE.MeshBasicMaterial({
        color: new THREE.Color(Math.random(), Math.random(), Math.random()),
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.8,
    });
    const mesh = new THREE.Mesh(geo, mat);
    // mesh.position.set(
    //     (Math.random() - 0.5) * 10,
    //     (Math.random() - 0.5) * 6,
    //     (Math.random() - 0.5) * 8 - 5,
    // );
    mesh.position.set(-5, 3, 0);
    const name = `图标_${circleBlocks.length + 1}`;
    mesh.name = name;
    const label = createTextLabel(name);
    mesh.add(label); // 文字挂在色块上
    scene.add(mesh);
    circleBlocks.push({ mesh, label });
};

// ==================== 3D拖拽 ====================
const getMouseNormalized = (clientX, clientY) => {
    const rect = mainDom.getBoundingClientRect();
    return new THREE.Vector2(
        ((clientX - rect.left) / rect.width) * 2 - 1,
        -((clientY - rect.top) / rect.height) * 2 + 1,
    );
};

const initDragPlane = (mesh, mp) => {
    dragNormal.set(0, 0, 1).applyQuaternion(camera.quaternion);
    dragPlane.setFromNormalAndCoplanarPoint(dragNormal, mesh.position);
    dragRay.setFromCamera(mp, camera);
    dragRay.ray.intersectPlane(dragPlane, dragIntersection);
};

// ==================== 鼠标事件（彻底禁用右键） ====================
const onMouseDown = (e) => {
    if (e.button === 0) {
        const mp = getMouseNormalized(e.clientX, e.clientY);
        raycaster.setFromCamera(mp, camera);
        const arr = circleBlocks.map((i) => i.mesh);
        const intersects = raycaster.intersectObjects(arr, true);
        if (intersects.length > 0) {
            selectedCircle = circleBlocks.find((i) => i.mesh === intersects[0].object);
            isDragging = true;
            initDragPlane(selectedCircle.mesh, mp);
            controls.enabled = false;
            outlinePass.selectedObjects = [selectedCircle.mesh];
            updateColorGui();
        } else {
            isDragging = false;
            controls.enabled = true;
        }
    }
};

const onMouseMove = (e) => {
    if (!isDragging || !selectedCircle) return;
    const mp = getMouseNormalized(e.clientX, e.clientY);
    dragRay.setFromCamera(mp, camera);
    if (dragRay.ray.intersectPlane(dragPlane, dragIntersection)) {
        selectedCircle.mesh.position.copy(dragIntersection);
        if (colorGuiFolder) colorGuiFolder.updateDisplay();
    }
};

const onMouseUp = () => {
    isDragging = false;
    controls.enabled = true;
};

// 滚轮缩放
const onMouseWheel = (e) => {
    if (!selectedCircle) return;
    e.preventDefault();
    const s = THREE.MathUtils.clamp(
        selectedCircle.mesh.scale.x * (e.deltaY > 0 ? 0.9 : 1.1),
        0.1,
        5,
    );
    selectedCircle.mesh.scale.setScalar(s);
    if (colorGuiFolder) colorGuiFolder.updateDisplay();
};

// 右键菜单
const openContextMenu = (e) => {
    e.preventDefault();
    e.stopPropagation();
    const mp = getMouseNormalized(e.clientX, e.clientY);
    raycaster.setFromCamera(mp, camera);
    const intersects = raycaster.intersectObjects(
        circleBlocks.map((b) => b.mesh),
        true,
    );
    if (intersects.length > 0) {
        selectedCircle = circleBlocks.find((b) => b.mesh === intersects[0].object);
        contextMenuDom.style.display = "block";
        contextMenuDom.style.left = e.clientX + "px";
        contextMenuDom.style.top = e.clientY + "px";
    } else {
        contextMenuDom.style.display = "none";
    }
};

const closeContextMenu = () => {
    contextMenuDom.style.display = "none";
};

// 删除
const deleteSelectedCircle = () => {
    if (!selectedCircle) return;
    scene.remove(selectedCircle.mesh);
    selectedCircle.mesh.remove(selectedCircle.label);
    selectedCircle.mesh.geometry.dispose();
    selectedCircle.mesh.material.dispose();
    circleBlocks = circleBlocks.filter((b) => b !== selectedCircle);
    selectedCircle = null;
    outlinePass.selectedObjects = [];
    clearColorGui();
    closeContextMenu();
};

// ==================== 生命周期 ====================
onMounted(() => {
    init();
    cretaeControls();
    resizeRender();
    renderLoop();
    createModel(); // 模型加载后自动创建GUI
    createLight();
    initComposer();
    // 移除原来的 setTimeout(createGui)，改为模型加载完成后创建

    renderer.domElement.addEventListener("mousedown", onMouseDown);
    window.addEventListener("mousemove", onMouseMove);
    window.addEventListener("mouseup", onMouseUp);
    renderer.domElement.addEventListener("wheel", onMouseWheel);
    renderer.domElement.addEventListener("click", selectMesh);
    window.addEventListener("mousemove", selectMesh);
    window.addEventListener("click", closeContextMenu);

    // 关键：全局禁止右键
    renderer.domElement.addEventListener("contextmenu", (e) => {
        e.preventDefault();
        e.stopPropagation();
    });
    window.addEventListener("contextmenu", (e) => {
        if (mainDom.contains(e.target)) {
            e.preventDefault();
            e.stopPropagation();
        }
    });

    // 右键打开自定义菜单
    renderer.domElement.addEventListener("contextmenu", openContextMenu);
});

onUnmounted(() => {
    // 清理所有事件监听
    renderer.domElement.removeEventListener("mousedown", onMouseDown);
    window.removeEventListener("mousemove", onMouseMove);
    window.removeEventListener("mouseup", onMouseUp);
    renderer.domElement.removeEventListener("wheel", onMouseWheel);
    renderer.domElement.removeEventListener("click", selectMesh);
    window.removeEventListener("mousemove", selectMesh);
    window.removeEventListener("click", closeContextMenu);
    renderer.domElement.removeEventListener("contextmenu", (e) => {
        e.preventDefault();
        e.stopPropagation();
    });
    window.removeEventListener("contextmenu", (e) => {
        if (mainDom.contains(e.target)) {
            e.preventDefault();
            e.stopPropagation();
        }
    });
    renderer.domElement.removeEventListener("contextmenu", openContextMenu);

    // 清理Three.js资源
    if (renderer) renderer.dispose();
    if (scene) {
        scene.traverse((obj) => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) obj.material.dispose();
        });
    }
});
</script>

<style scoped lang="scss">
.content {
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
}
.header {
    padding: 16px;
    display: flex;
    gap: 12px;
}
.main {
    flex: 1;
    position: relative;
}
.model-name-tag {
    position: fixed;
    padding: 4px 8px;
    background: rgba(0, 0, 0, 0.8);
    color: #fff;
    font-size: 14px;
    border-radius: 4px;
    pointer-events: none;
    z-index: 9999;
}
.context-menu {
    position: fixed;
    width: 120px;
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    z-index: 99999;
    .menu-item {
        padding: 8px 12px;
        cursor: pointer;
        &:hover {
            background: #f5f5f5;
        }
    }
}
</style>
