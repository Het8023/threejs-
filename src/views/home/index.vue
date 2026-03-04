<template>
    <!-- 根容器：包含整个页面的结构 -->
    <div class="content">
        <!-- 头部操作栏：放置功能按钮 -->
        <div class="header">
            <!-- 展开模型按钮：点击触发modelExplode函数 -->
            <el-button type="primary" @click="modelExplode">展开</el-button>
            <!-- 收起模型按钮：点击触发modelRestore函数 -->
            <el-button type="primary" @click="modelRestore">收起</el-button>
            <!-- 编辑按钮：仅非编辑状态显示 -->
            <el-button type="warning" @click="enterEditMode" v-if="!isEditMode">编辑</el-button>
            <!-- 添加色块按钮：仅编辑状态显示 -->
            <el-button type="success" @click="addIcon" v-if="isEditMode">添加图标</el-button>
            <!-- 取消编辑按钮：仅编辑状态显示 -->
            <el-button type="danger" @click="exitEditMode" v-if="isEditMode">取消编辑</el-button>
        </div>
        <!-- 3D场景主容器：Three.js渲染的画布会挂载到这里 -->
        <div id="MAIN" class="main"></div>
        <!-- 模型名称标签：鼠标滑过模型时显示物体名称 -->
        <div id="nameTag" class="model-name-tag" style="display: none"></div>
        <!-- 自定义右键菜单：右键点击色块时显示 -->
        <div id="contextMenu" class="context-menu" style="display: none">
            <!-- 复制色块菜单项 -->
            <div class="menu-item" @click="copySelectedCircle">复制该色块</div>
            <!-- 粘贴色块菜单项 -->
            <div class="menu-item" @click="pasteCircle">粘贴色块</div>
            <!-- 删除色块菜单项：点击触发deleteSelectedCircle函数 -->
            <div class="menu-item" @click="deleteSelectedCircle">删除该色块</div>
        </div>
    </div>
</template>

<script setup>
// 导入Vue核心API：ref用于响应式数据，onUnmounted用于组件卸载时清理资源
import { ref, onUnmounted } from "vue";
// 导入Three.js核心库：3D场景的基础
import * as THREE from "three";
// 导入轨道控制器：实现鼠标拖拽旋转、缩放场景
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
// 导入GLTF模型加载器：加载3D模型文件（.glb/.gltf）
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
// 导入2D标签渲染器：在3D场景中显示HTML文字标签
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";
// 导入Vue生命周期钩子：组件挂载完成时初始化3D场景
import { onMounted } from "vue";
// 导入dat.GUI：创建可视化的参数调整面板
import * as dat from "dat.gui";
// 导入后处理相关库：实现模型/色块的描边高亮效果
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass";
import { OutlinePass } from "three/examples/jsm/postprocessing/OutlinePass";
// 导入gsap：实现平滑的动画效果（模型展开/收起）
import gsap from "gsap";

// ==================== 核心新增：编辑状态响应式变量 ====================
const isEditMode = ref(false); // 标记是否处于编辑模式

// ==================== 全局变量定义 ====================
// 射线检测器：用于检测鼠标点击/划过的3D物体
const raycaster = new THREE.Raycaster();
// 后处理合成器：管理描边效果的渲染
let composer;
// 描边通道：控制选中物体的描边样式（颜色、强度）
let outlinePass;

// 核心3D对象：场景、相机、渲染器、控制器、主容器DOM、加载的模型
let scene, camera, renderer, controls, mainDom, model;
// 模型池：存储加载的所有3D模型
let modelPool = [];
// 图标数组（响应式）：存储色块数据（目前未使用，预留扩展）
const iconArray = ref([]);

// 色块数组：存储所有创建的圆形色块（包含mesh和label）
let circleBlocks = [];
// 选中的色块：记录当前被点击选中的色块
let selectedCircle = null;
// 拖拽状态：标记是否正在拖拽色块
let isDragging = false;
// 鼠标坐标：存储归一化的鼠标位置（用于射线检测）
let mouse = new THREE.Vector2();

// 拖拽相关：用于计算3D拖拽的平面和交点
let dragPlane = new THREE.Plane(); // 拖拽平面
let dragRay = new THREE.Raycaster(); // 拖拽射线
let dragIntersection = new THREE.Vector3(); // 射线与拖拽平面的交点
let dragNormal = new THREE.Vector3(); // 拖拽平面的法向量

// 2D标签渲染器：渲染色块上的文字标签
let labelRenderer;
// 右键菜单DOM元素：自定义右键菜单的DOM引用
let contextMenuDom;
// dat.GUI实例：参数调整面板
let gui;
// 色块GUI文件夹：选中色块时显示的参数面板
let colorGuiFolder = null;
// 色块名称控制器：用于修改色块名称的GUI控件
let colorNameController = null;

// 新增：存储复制的色块数据（深拷贝）
let copiedCircleData = null;
// 新增：存储右键点击位置（全局）
window.rightClickPosition = null;

// ==================== 1. 初始化3D场景 ====================
const init = () => {
    // 获取3D场景主容器DOM
    mainDom = document.getElementById("MAIN");
    // 获取右键菜单DOM元素
    contextMenuDom = document.getElementById("contextMenu");

    // 创建Three.js场景：所有3D物体的容器
    scene = new THREE.Scene();

    // 创建透视相机：模拟人眼视角（参数：视角75°，宽高比=容器宽高比，近裁剪面0.1，远裁剪面1000）
    camera = new THREE.PerspectiveCamera(75, mainDom.clientWidth / mainDom.clientHeight, 0.1, 1000);
    // 设置相机初始位置：z轴方向7个单位（离场景中心的距离）
    camera.position.z = 7;

    // 创建WebGL渲染器：负责绘制3D场景（antialias=true开启抗锯齿，让边缘更平滑）
    renderer = new THREE.WebGLRenderer({ antialias: true });
    // 设置渲染器尺寸为容器大小
    renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    // 将渲染器的画布添加到主容器中
    mainDom.appendChild(renderer.domElement);

    // 创建2D标签渲染器：用于渲染色块上的文字
    labelRenderer = new CSS2DRenderer();
    // 设置标签渲染器尺寸
    labelRenderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    // 设置标签渲染器的样式：绝对定位、顶部对齐、不拦截鼠标事件
    labelRenderer.domElement.style.position = "absolute";
    labelRenderer.domElement.style.top = "0";
    labelRenderer.domElement.style.pointerEvents = "none";
    // 将标签渲染器的画布添加到主容器
    mainDom.appendChild(labelRenderer.domElement);
};

// ==================== 2. 创建轨道控制器 ====================
const cretaeControls = () => {
    // 创建轨道控制器：关联相机和渲染器画布，实现鼠标交互
    controls = new OrbitControls(camera, renderer.domElement);
    // 开启阻尼效果：让旋转/缩放更平滑（需要在渲染循环中更新）
    controls.enableDamping = true;
};

// ==================== 3. 窗口适配：窗口大小变化时更新渲染器 ====================
const resizeRender = () => {
    // 监听窗口大小变化事件
    window.addEventListener("resize", () => {
        // 更新相机宽高比
        camera.aspect = mainDom.clientWidth / mainDom.clientHeight;
        // 更新渲染器尺寸
        renderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        // 更新相机投影矩阵（必须调用，否则视角会变形）
        camera.updateProjectionMatrix();

        // 如果描边通道存在，更新其分辨率
        if (outlinePass) outlinePass.resolution.set(mainDom.clientWidth, mainDom.clientHeight);
        // 如果后处理合成器存在，更新其尺寸
        if (composer) composer.setSize(mainDom.clientWidth, mainDom.clientHeight);
        // 如果标签渲染器存在，更新其尺寸
        if (labelRenderer) labelRenderer.setSize(mainDom.clientWidth, mainDom.clientHeight);
    });
};

// ==================== 4. 渲染循环：持续渲染3D场景 ====================
const renderLoop = () => {
    // 请求下一帧动画：创建无限循环（浏览器刷新率）
    requestAnimationFrame(renderLoop);
    // 更新轨道控制器：启用阻尼时必须调用
    controls.update();

    // 遍历所有色块：让文字大小跟随色块缩放比例变化
    circleBlocks.forEach((item) => {
        if (item.label) {
            // 获取色块的缩放比例（x/y/z一致）
            const scale = item.mesh.scale.x;
            // 设置文字字号：基础系数15 * 缩放比例（保证文字大小适配色块）
            item.label.element.style.fontSize = `${15 * scale}px`;
        }
    });

    // 如果后处理合成器存在，执行渲染（描边效果）
    if (composer) composer.render();
    // 如果标签渲染器存在，渲染2D文字标签
    if (labelRenderer) labelRenderer.render(scene, camera);
};

// ==================== 5. 创建坐标轴辅助器：调试用 ====================
const createHelper = () => {
    // 创建坐标轴辅助器：长度3个单位（红=x，绿=y，蓝=z）
    const axes = new THREE.AxesHelper(3);
    // 将坐标轴添加到场景中
    scene.add(axes);
};

// ==================== 6. 加载3D模型（修复异步加载问题） ====================
const createModel = () => {
    // 创建GLTF模型加载器
    const loader = new GLTFLoader();
    // 加载模型文件（路径：public下的AHU-9.glb）
    loader.load(
        "/public/AHU-9.glb",
        // 加载成功回调
        (gltf) => {
            // 遍历模型的所有子物体
            gltf.scene.traverse((obj) => {
                // 保存物体初始位置：用于模型收起时恢复
                obj.fromPosition = obj.position.clone();
                // 计算物体展开后的位置：初始位置*10（放大展开距离）
                obj.toPosition = obj.position.clone().multiplyScalar(10);
                // 给无名称的物体设置随机名称
                if (!obj.name) obj.name = `模块_${Math.floor(Math.random() * 1000)}`;
            });
            // 保存加载的模型
            model = gltf.scene;
            // 标记模型未展开
            model.decomposition = false;
            // 将模型添加到模型池
            modelPool.push(model);
            // 将模型添加到场景
            scene.add(model);

            // 核心修复：模型加载完成后再创建GUI（避免model为undefined）
            createGui();
        },
        // 加载进度回调
        (xhr) => {
            // 计算加载进度百分比
            const percent = Math.floor((xhr.loaded / xhr.total) * 100);
            console.log(`模型加载进度：${percent}%`);
        },
        // 加载失败回调
        (err) => {
            console.error("模型加载失败：", err);
        },
    );
};

// ==================== 7. 创建灯光：让3D物体可见 ====================
const createLight = () => {
    // 创建环境光：均匀照亮所有物体（颜色白色，强度1.4）
    const ambient = new THREE.AmbientLight(0xffffff, 1.4);
    scene.add(ambient);

    // 创建方向光1：模拟主光源（颜色白色，强度3.5）
    const dir1 = new THREE.DirectionalLight(0xffffff, 3.5);
    // 设置光源位置
    dir1.position.set(400, 200, 300);
    scene.add(dir1);

    // 创建方向光2：补光（颜色白色，强度2）
    const dir2 = new THREE.DirectionalLight(0xffffff, 2);
    // 设置补光位置（与主光源相反）
    dir2.position.set(-400, -200, -300);
    scene.add(dir2);
};

// ==================== 8. 创建GUI面板：可视化调整参数 ====================
const createGui = () => {
    // 判空：model不存在则直接返回（避免报错）
    if (!model) {
        console.warn("模型未加载完成，暂不创建GUI");
        return;
    }

    // 创建dat.GUI实例
    gui = new dat.GUI();
    // 初始隐藏GUI（非编辑模式）
    if (gui.domElement) gui.domElement.style.display = isEditMode.value ? "block" : "none";

    // ==================== 场景设置文件夹（背景色 + 坐标轴开关 + 长度调整） ====================
    const sceneFolder = gui.addFolder("场景设置");

    // 1. 背景颜色修改（修复null值问题）
    let currentBgColor = "#000000"; // 默认背景色
    if (scene.background && scene.background.isColor) {
        currentBgColor = `#${scene.background.getHexString()}`;
    } else {
        scene.background = new THREE.Color(currentBgColor);
    }
    const bgColorObj = { color: currentBgColor };
    sceneFolder
        .addColor(bgColorObj, "color")
        .name("背景颜色")
        .onChange((newColor) => {
            scene.background.set(newColor);
        });

    // 2. 坐标轴显示/隐藏开关 + 长度调整（修复parameters.size未定义问题）
    // let axesHelper = scene.getObjectByName("axesHelper");
    // // 定义初始长度（兼容不同Three.js版本）
    // let initialAxesLength = 3;
    // if (!axesHelper) {
    //     axesHelper = new THREE.AxesHelper(initialAxesLength); // 默认长度3
    //     axesHelper.name = "axesHelper"; // 命名方便查找
    //     scene.add(axesHelper); // 添加到场景
    // } else {
    //     // 兼容方案：优先从geometry获取长度，失败则用默认值
    //     if (axesHelper.geometry && axesHelper.geometry.boundingBox) {
    //         axesHelper.geometry.computeBoundingBox();
    //         initialAxesLength = axesHelper.geometry.boundingBox.max.x || 3;
    //     } else if (axesHelper.parameters?.size) {
    //         initialAxesLength = axesHelper.parameters.size;
    //     }
    // }

    // // 2.1 显示/隐藏开关
    // const axesVisibleObj = { showAxes: axesHelper.visible };
    // sceneFolder
    //     .add(axesVisibleObj, "showAxes")
    //     .name("显示坐标轴")
    //     .onChange((isShow) => {
    //         axesHelper.visible = isShow;
    //     });

    // // 2.2 坐标轴长度调整（修复size未定义问题）
    // const axesLengthObj = { axesLength: initialAxesLength };
    // sceneFolder
    //     .add(axesLengthObj, "axesLength")
    //     .name("坐标轴长度")
    //     .min(1) // 最小长度1
    //     .max(10) // 最大长度10
    //     .step(0.1) // 调整步长0.1
    //     .onChange((newLength) => {
    //         // 销毁旧的坐标轴，创建新长度的坐标轴
    //         scene.remove(axesHelper);
    //         if (axesHelper.geometry) axesHelper.geometry.dispose(); // 释放几何资源
    //         if (axesHelper.material) axesHelper.material.dispose(); // 释放材质资源
    //         // 创建新坐标轴
    //         axesHelper = new THREE.AxesHelper(newLength);
    //         axesHelper.name = "axesHelper";
    //         axesHelper.visible = axesVisibleObj.showAxes; // 继承显示状态（直接取对象属性，更稳定）
    //         scene.add(axesHelper);
    //         // 更新存储的长度值
    //         axesLengthObj.axesLength = newLength;
    //     });

    // sceneFolder.open(); // 默认展开场景设置文件夹

    // 创建模型位置文件夹
    const f1 = gui.addFolder("模型位置");
    f1.add(model.position, "x", -10, 10, 0.01).name("X轴");
    f1.add(model.position, "y", -10, 10, 0.01).name("Y轴");
    f1.add(model.position, "z", -10, 10, 0.01).name("Z轴");

    // 创建模型旋转文件夹
    const f2 = gui.addFolder("模型旋转");
    f2.add(model.rotation, "x", -10, 10, 0.01).name("X轴");
    f2.add(model.rotation, "y", -10, 10, 0.01).name("Y轴");
    f2.add(model.rotation, "z", -10, 10, 0.01).name("Z轴");
};

// ==================== 核心新增：控制GUI显示/隐藏 ====================
const toggleGuiDisplay = (show) => {
    if (gui && gui.domElement) {
        gui.domElement.style.display = show ? "block" : "none";
    }
};

// ==================== 9. 展开模型：带动画效果 ====================
const modelExplode = () => {
    // 判空：模型未加载完成则提示
    if (!model) {
        alert("模型未加载完成，无法展开");
        return;
    }
    // 标记模型为展开状态
    model.decomposition = true;
    // 遍历模型所有子物体
    model.traverse((obj) => {
        // 使用gsap实现平滑动画：5秒内移动到展开位置
        gsap.to(obj.position, {
            x: obj.toPosition.x,
            y: obj.toPosition.y,
            z: obj.toPosition.z,
            duration: 5,
        });
    });
};

// ==================== 10. 收起模型：带动画效果 ====================
const modelRestore = () => {
    // 判空：模型未加载完成则提示
    if (!model) {
        alert("模型未加载完成，无法收起");
        return;
    }
    // 标记模型为收起状态
    model.decomposition = false;
    // 遍历模型所有子物体
    model.traverse((obj) => {
        // 使用gsap实现平滑动画：5秒内恢复到初始位置
        gsap.to(obj.position, {
            x: obj.fromPosition.x,
            y: obj.fromPosition.y,
            z: obj.fromPosition.z,
            duration: 5,
        });
    });
};

// ==================== 核心新增：进入编辑模式 ====================
const enterEditMode = () => {
    isEditMode.value = true;
    toggleGuiDisplay(true); // 显示GUI面板
    // 清空选中状态，确保编辑模式初始状态干净
    selectedCircle = null;
    outlinePass.selectedObjects = [];
    clearColorGui();
};

// ==================== 核心新增：退出编辑模式 ====================
const exitEditMode = () => {
    isEditMode.value = false;
    toggleGuiDisplay(false); // 隐藏GUI面板
    // 重置交互状态
    isDragging = false;
    selectedCircle = null;
    outlinePass.selectedObjects = [];
    clearColorGui();
    closeContextMenu(); // 关闭右键菜单
    controls.enabled = true; // 恢复场景控制器
};

// ==================== 11. 鼠标滑过/点击检测：选中3D物体（新增编辑模式判断） ====================
const selectMesh = (e) => {
    // 非编辑模式：直接返回，不处理选中逻辑
    if (!isEditMode.value) return;

    // 修复1：判断点击目标是否为GUI元素，若是则跳过
    if (e.target.closest(".dg")) return;

    // 主容器不存在则返回
    if (!mainDom) return;
    // 获取主容器的位置和尺寸
    const rect = mainDom.getBoundingClientRect();
    // 将鼠标坐标转换为归一化设备坐标（-1到1之间）
    mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
    // 设置射线检测器：从相机位置向鼠标方向发射射线
    raycaster.setFromCamera(mouse, camera);
    // 检测射线与场景中所有物体的交点（递归检测子物体）
    const intersects = raycaster.intersectObjects(scene.children, true);
    // 获取模型名称标签DOM
    const tag = document.getElementById("nameTag");

    // 如果是点击事件
    if (e.type === "click") {
        // 检测到点击了物体
        if (intersects.length > 0) {
            // 获取点击的第一个物体
            const obj = intersects[0].object;
            // 判断是否点击的是色块
            const isBlock = circleBlocks.some((b) => b.mesh === obj);
            if (isBlock) {
                // 选中该色块
                selectedCircle = circleBlocks.find((b) => b.mesh === obj);
                // 设置描边选中的物体为该色块
                outlinePass.selectedObjects = [selectedCircle.mesh];
                // 更新色块的GUI面板
                updateColorGui();
            } else {
                // 未点击色块，清空选中状态
                selectedCircle = null;
                // 描边选中的模型物体
                outlinePass.selectedObjects = [obj];
                // 清空色块GUI面板
                clearColorGui();
            }
        } else {
            // 未点击任何物体，清空选中状态
            selectedCircle = null;
            // 清空描边
            outlinePass.selectedObjects = [];
            // 清空色块GUI面板
            clearColorGui();
        }
    } else {
        // 鼠标滑过事件
        if (intersects.length > 0 && !selectedCircle) {
            // 描边滑过的物体
            outlinePass.selectedObjects = [intersects[0].object];
            // 设置名称标签内容
            tag.textContent = intersects[0].object.name || "未知";
            // 显示名称标签
            tag.style.display = "block";
            // 设置标签位置（鼠标右侧10px）
            tag.style.left = e.clientX + 10 + "px";
            tag.style.top = e.clientY + 10 + "px";
        } else {
            // 未滑过物体或已选中色块，隐藏标签
            if (!selectedCircle) outlinePass.selectedObjects = [];
            tag.style.display = "none";
        }
    }
};

// ==================== 12. 初始化后处理：实现选中物体描边 ====================
const initComposer = () => {
    // 创建描边通道：参数（分辨率、场景、相机）
    outlinePass = new OutlinePass(
        new THREE.Vector2(mainDom.clientWidth, mainDom.clientHeight),
        scene,
        camera,
    );
    // 设置描边颜色为红色
    outlinePass.visibleEdgeColor.set(0xff0000);
    // 设置描边强度（数值越大，描边越粗）
    outlinePass.edgeStrength = 10;
    // 创建后处理合成器
    composer = new EffectComposer(renderer);
    // 添加基础渲染通道：渲染场景
    composer.addPass(new RenderPass(scene, camera));
    // 添加描边通道：在基础渲染上叠加描边
    composer.addPass(outlinePass);
};

// ==================== 色块GUI面板：选中色块时显示 ====================
// ==================== 色块GUI面板：选中色块时显示（新增颜色修改功能） ====================
const updateColorGui = (updateOnly = false) => {
    // 非编辑模式：直接返回
    if (!isEditMode.value) return;

    // 先清空之前的色块GUI
    clearColorGui();
    // 未选中色块或GUI不存在则返回
    if (!selectedCircle || !gui) return;

    // 如果是更新名称，先记录当前展开状态
    let wasOpen = true;

    // 创建色块参数文件夹：显示最新的色块名称
    colorGuiFolder = gui.addFolder(`色块：${selectedCircle.mesh.name}`);

    // 添加色块X轴位置调整：范围-20到20，步长0.01
    colorGuiFolder.add(selectedCircle.mesh.position, "x", -20, 20, 0.01).name("X");
    // 添加Y轴位置调整
    colorGuiFolder.add(selectedCircle.mesh.position, "y", -20, 20, 0.01).name("Y");
    // 添加Z轴位置调整
    colorGuiFolder.add(selectedCircle.mesh.position, "z", -20, 20, 0.01).name("Z");

    // 旋转XYZ调整：范围-π到π（完整圆周），步长0.01
    colorGuiFolder.add(selectedCircle.mesh.rotation, "x", -Math.PI, Math.PI, 0.01).name("旋转 X");
    colorGuiFolder.add(selectedCircle.mesh.rotation, "y", -Math.PI, Math.PI, 0.01).name("旋转 Y");
    colorGuiFolder.add(selectedCircle.mesh.rotation, "z", -Math.PI, Math.PI, 0.01).name("旋转 Z");

    // 大小缩放调整：范围0.1到5，步长0.01
    colorGuiFolder
        .add(selectedCircle.mesh.scale, "x", 0.1, 5, 0.01)
        .name("大小")
        .onChange((v) => {
            // 保证XYZ缩放比例一致（均匀缩放）
            selectedCircle.mesh.scale.setScalar(v);
        });

    // 色块名称修改：绑定临时对象的name属性
    colorNameController = colorGuiFolder
        .add({ name: selectedCircle.mesh.name }, "name")
        .name("名称");

    // 名称修改完成后触发（核心修复：重新创建文件夹实现实时更新）
    colorNameController.onFinishChange((val) => {
        // 更新色块mesh的名称
        selectedCircle.mesh.name = val;
        // 更新色块标签的文字
        selectedCircle.label.element.textContent = val;

        // 关键：重新调用updateColorGui，强制重建文件夹
        updateColorGui(true);
    });

    // ==================== 新增：色块颜色修改功能 ====================
    // 1. 格式化当前色块颜色为十六进制字符串（供颜色选择器显示）
    const currentColor = `#${selectedCircle.mesh.material.color.getHexString()}`;
    // 2. 创建临时对象存储颜色（dat.GUI需要绑定对象属性）
    const colorObj = { color: currentColor };
    // 3. 添加颜色选择器控件
    colorGuiFolder
        .addColor(colorObj, "color")
        .name("色块颜色")
        .onChange((newColor) => {
            // 4. 实时更新色块材质颜色
            selectedCircle.mesh.material.color.set(newColor);
        });

    // 保持文件夹展开状态
    colorGuiFolder.open(wasOpen);

    // 修复2：给GUI输入框添加事件阻止冒泡
    setTimeout(() => {
        const guiInputs = colorGuiFolder.domElement.querySelectorAll("input");
        guiInputs.forEach((input) => {
            input.addEventListener("click", (e) => {
                e.stopPropagation(); // 阻止点击冒泡
            });
            input.addEventListener("keydown", (e) => {
                if (e.key === "Enter") e.stopPropagation(); // 阻止回车冒泡
            });
        });
    }, 0);
};

// ==================== 清空色块GUI面板 ====================
const clearColorGui = () => {
    // 如果色块文件夹存在，从GUI中移除
    if (colorGuiFolder && gui) {
        gui.removeFolder(colorGuiFolder);
        colorGuiFolder = null;
    }
    // 清空名称控制器
    colorNameController = null;
};

// ==================== 创建文字标签：色块中间的文字 ====================
const createTextLabel = (text) => {
    // 创建div元素：用于显示文字
    const div = document.createElement("div");
    // 文字颜色白色
    div.style.color = "#fff";
    // 基础字号8px（会跟随色块缩放）
    div.style.fontSize = "8px";
    // 文字加粗
    div.style.fontWeight = "bold";
    // 文字阴影：增强可读性
    div.style.textShadow = "1px 1px 2px #000";
    // 文字居中：向左/上移动50%（关键）
    div.style.transform = "translate(-50%, -50%)";
    // 文本水平居中
    div.style.textAlign = "center";
    // 不换行
    div.style.whiteSpace = "nowrap";
    // 不拦截鼠标事件
    div.style.pointerEvents = "none";
    // 设置文字内容
    div.textContent = text;

    // 创建CSS2DObject：将div转换为3D场景中的标签
    const label = new CSS2DObject(div);
    // 设置标签位置在色块中心（0,0,0）
    label.position.set(0, 0, 0);
    return label;
};

// ==================== 添加色块：创建圆形色块（支持自定义参数） ====================
// 添加色块：创建圆形色块（支持自定义参数）
const addIcon = (customData = null) => {
    // 非编辑模式：直接返回，禁止添加色块
    if (!isEditMode.value) return;

    // 默认参数
    const defaultParams = {
        radius: 0.5,
        color: new THREE.Color(Math.random(), Math.random(), Math.random()),
        position: new THREE.Vector3(-5, 3, 0),
        scale: new THREE.Vector3(1, 1, 1),
        rotation: new THREE.Euler(0, 0, 0),
        name: `图标_${circleBlocks.length + 1}`,
    };

    // 合并自定义参数（粘贴时使用）
    const params = {
        ...defaultParams,
        ...(customData || {}),
    };

    // 创建圆形几何体
    const geo = new THREE.CircleGeometry(params.radius, 32);
    // 创建基础材质
    const mat = new THREE.MeshBasicMaterial({
        color: params.color,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.8,
    });
    // 创建网格物体
    const mesh = new THREE.Mesh(geo, mat);

    // 设置色块属性（移除原有偏移逻辑）
    mesh.position.copy(params.position); // 直接使用目标位置
    mesh.scale.copy(params.scale);
    mesh.rotation.copy(params.rotation);
    mesh.name = params.name;

    // 创建文字标签
    const label = createTextLabel(params.name);
    // 将标签添加到色块mesh中
    mesh.add(label);
    // 将色块添加到场景
    scene.add(mesh);
    // 将色块和标签存入数组
    circleBlocks.push({ mesh, label });

    // 粘贴后自动选中新色块
    if (customData) {
        selectedCircle = circleBlocks[circleBlocks.length - 1];
        outlinePass.selectedObjects = [selectedCircle.mesh];
        updateColorGui();
    }
};

// ==================== 新增：复制选中的色块 ====================
const copySelectedCircle = () => {
    // 非编辑模式：直接返回
    if (!isEditMode.value) return;

    if (!selectedCircle) {
        alert("请先选中一个色块再复制");
        closeContextMenu();
        return;
    }

    // 深拷贝色块数据（避免引用问题）
    copiedCircleData = {
        radius: selectedCircle.mesh.geometry.parameters.radius,
        color: new THREE.Color().copy(selectedCircle.mesh.material.color),
        position: new THREE.Vector3().copy(selectedCircle.mesh.position),
        scale: new THREE.Vector3().copy(selectedCircle.mesh.scale),
        rotation: new THREE.Euler().copy(selectedCircle.mesh.rotation),
        name: `${selectedCircle.mesh.name}_副本`,
    };

    closeContextMenu();
    console.log("色块已复制：", copiedCircleData);
};

// ==================== 新增：粘贴色块 ====================
// 粘贴色块（粘贴到右键点击位置）
const pasteCircle = () => {
    // 非编辑模式：直接返回
    if (!isEditMode.value) return;

    if (!copiedCircleData) {
        alert("请先复制一个色块再粘贴");
        closeContextMenu();
        return;
    }

    // 1. 获取右键点击的3D位置（无点击记录则用默认位置）
    const targetPos = window.rightClickPosition?.normalized
        ? getWorldPositionFromScreen(window.rightClickPosition.normalized)
        : new THREE.Vector3(-5, 3, 0);

    // 2. 合并复制的数据和目标位置
    const pasteData = {
        ...copiedCircleData,
        position: targetPos, // 粘贴到右键点击的3D位置
    };

    // 3. 创建新色块
    addIcon(pasteData);
    closeContextMenu();
};

// ==================== 新增：键盘快捷键监听 ====================
// 键盘快捷键监听
const handleKeyDown = (e) => {
    // 修复3：判断是否在GUI输入框中，若是则跳过快捷键
    if (e.target.tagName === "INPUT" && e.target.closest(".dg")) return;

    // 非编辑模式：直接返回，禁止快捷键
    if (!isEditMode.value) return;

    // Ctrl+C 复制
    if (e.ctrlKey && e.key === "c") {
        e.preventDefault(); // 阻止浏览器默认复制行为
        copySelectedCircle();
    }
    // Ctrl+V 粘贴
    else if (e.ctrlKey && e.key === "v") {
        e.preventDefault(); // 阻止浏览器默认粘贴行为
        pasteCircle();
    }
    // 新增：Delete/Backspace 键删除选中的色块
    else if (e.key === "Delete" || e.key === "Backspace") {
        e.preventDefault(); // 阻止浏览器默认行为（如回退页面）
        if (selectedCircle) {
            deleteSelectedCircle();
        } else {
            alert("请先选中一个色块再删除");
        }
    }
};

// ==================== 3D拖拽：将鼠标坐标转换为3D空间坐标 ====================
const getMouseNormalized = (clientX, clientY) => {
    // 获取主容器位置和尺寸
    const rect = mainDom.getBoundingClientRect();
    // 返回归一化的鼠标坐标（-1到1）
    return new THREE.Vector2(
        ((clientX - rect.left) / rect.width) * 2 - 1,
        -((clientY - rect.top) / rect.height) * 2 + 1,
    );
};

// ==================== 初始化拖拽平面：开始拖拽时调用 ====================
const initDragPlane = (mesh, mp) => {
    // 设置拖拽平面法向量：沿相机视角的z轴
    dragNormal.set(0, 0, 1).applyQuaternion(camera.quaternion);
    // 创建拖拽平面：通过法向量和色块位置
    dragPlane.setFromNormalAndCoplanarPoint(dragNormal, mesh.position);
    // 设置拖拽射线：从相机到鼠标位置
    dragRay.setFromCamera(mp, camera);
    // 计算射线与拖拽平面的交点
    dragRay.ray.intersectPlane(dragPlane, dragIntersection);
};

// ==================== 鼠标按下事件：开始拖拽色块（新增编辑模式判断） ====================
const onMouseDown = (e) => {
    // 修复4：判断点击目标是否为GUI元素，若是则跳过
    if (e.target.closest(".dg")) return;

    // 非编辑模式：直接返回，禁止拖拽
    if (!isEditMode.value) return;

    // 只处理左键点击（e.button=0）
    if (e.button === 0) {
        // 获取归一化的鼠标坐标
        const mp = getMouseNormalized(e.clientX, e.clientY);
        // 设置射线检测器
        raycaster.setFromCamera(mp, camera);
        // 获取所有色块的mesh
        const arr = circleBlocks.map((i) => i.mesh);
        // 检测射线与色块的交点
        const intersects = raycaster.intersectObjects(arr, true);
        if (intersects.length > 0) {
            // 选中点击的色块
            selectedCircle = circleBlocks.find((i) => i.mesh === intersects[0].object);
            // 标记开始拖拽
            isDragging = true;
            // 初始化拖拽平面
            initDragPlane(selectedCircle.mesh, mp);
            // 禁用轨道控制器（避免同时拖拽场景）
            controls.enabled = false;
            // 描边选中的色块
            outlinePass.selectedObjects = [selectedCircle.mesh];
            // 更新色块GUI面板
            updateColorGui();
        } else {
            // 未点击色块，停止拖拽
            isDragging = false;
            // 启用轨道控制器
            controls.enabled = true;
        }
    }
};

// ==================== 鼠标移动事件：拖拽色块（新增编辑模式判断） ====================
const onMouseMove = (e) => {
    // 非编辑模式：直接返回，禁止拖拽
    if (!isEditMode.value) return;

    // 未拖拽或未选中色块则返回
    if (!isDragging || !selectedCircle) return;
    // 获取归一化的鼠标坐标
    const mp = getMouseNormalized(e.clientX, e.clientY);
    // 设置拖拽射线
    dragRay.setFromCamera(mp, camera);
    // 计算射线与拖拽平面的交点
    if (dragRay.ray.intersectPlane(dragPlane, dragIntersection)) {
        // 更新色块位置为交点位置
        selectedCircle.mesh.position.copy(dragIntersection);
        // 如果GUI面板存在，更新显示
        if (colorGuiFolder) colorGuiFolder.updateDisplay();
    }
};

// ==================== 鼠标松开事件：停止拖拽 ====================
const onMouseUp = () => {
    // 标记停止拖拽
    isDragging = false;
    // 启用轨道控制器
    controls.enabled = true;
};

// ==================== 右键菜单：打开自定义右键菜单（新增编辑模式判断） ====================
// 右键菜单：打开自定义右键菜单（支持整个场景）
const openContextMenu = (e) => {
    // 修复5：判断点击目标是否为GUI元素，若是则跳过
    if (e.target.closest(".dg")) return;

    // 非编辑模式：直接返回，禁止右键菜单
    if (!isEditMode.value) return;

    // 阻止原生右键菜单
    e.preventDefault();
    // 阻止事件冒泡
    e.stopPropagation();

    // 1. 检测是否点击了色块（保留原有选中逻辑）
    const mp = getMouseNormalized(e.clientX, e.clientY);
    raycaster.setFromCamera(mp, camera);
    const circleIntersects = raycaster.intersectObjects(
        circleBlocks.map((b) => b.mesh),
        true,
    );

    // 2. 如果点击了色块，选中该色块
    if (circleIntersects.length > 0) {
        selectedCircle = circleBlocks.find((b) => b.mesh === circleIntersects[0].object);
    }

    // 3. 记录右键点击的屏幕坐标（用于粘贴时定位）
    window.rightClickPosition = {
        clientX: e.clientX,
        clientY: e.clientY,
        normalized: mp, // 归一化坐标，用于3D空间计算
    };

    // 4. 显示自定义右键菜单（无论是否点击色块）
    contextMenuDom.style.display = "block";
    contextMenuDom.style.left = e.clientX + "px";
    contextMenuDom.style.top = e.clientY + "px";
};
// 工具函数：将屏幕坐标转换为3D场景中的世界坐标（粘贴位置）
const getWorldPositionFromScreen = (normalizedPos) => {
    // 创建一个平面（默认在z=0平面，可根据需要调整）
    const plane = new THREE.Plane(new THREE.Vector3(0, 0, 1), 0);
    const ray = new THREE.Raycaster();
    const intersectPoint = new THREE.Vector3();

    // 设置射线并计算与平面的交点
    ray.setFromCamera(normalizedPos, camera);
    ray.ray.intersectPlane(plane, intersectPoint);

    return intersectPoint;
};
// ==================== 关闭右键菜单 ====================
const closeContextMenu = (e) => {
    // 修复6：判断点击目标是否为GUI元素，若是则跳过
    if (e && e.target.closest(".dg")) return;

    contextMenuDom.style.display = "none";
};

// ==================== 删除选中的色块（新增编辑模式判断） ====================
// 删除选中的色块
const deleteSelectedCircle = () => {
    // 非编辑模式：直接返回
    if (!isEditMode.value) return;

    // 未选中色块则返回
    if (!selectedCircle) return;
    // 从场景中移除色块
    scene.remove(selectedCircle.mesh);
    // 从色块中移除标签
    selectedCircle.mesh.remove(selectedCircle.label);
    // 释放几何体资源（避免内存泄漏）
    selectedCircle.mesh.geometry.dispose();
    // 释放材质资源
    selectedCircle.mesh.material.dispose();
    // 从色块数组中移除该色块
    circleBlocks = circleBlocks.filter((b) => b !== selectedCircle);
    // 清空选中状态
    selectedCircle = null;
    // 清空描边
    outlinePass.selectedObjects = [];
    // 清空色块GUI面板
    clearColorGui();
    // 关闭右键菜单（快捷键删除时也会执行，无副作用）
    closeContextMenu();
};

// ==================== 组件挂载：初始化所有功能 ====================
onMounted(() => {
    // 初始化3D场景
    init();
    // 创建轨道控制器
    cretaeControls();
    // 绑定窗口适配事件
    resizeRender();
    // 启动渲染循环
    renderLoop();
    // 加载3D模型
    createModel();
    // 创建灯光
    createLight();
    // 初始化后处理（描边）
    initComposer();

    // 绑定鼠标事件监听
    renderer.domElement.addEventListener("mousedown", onMouseDown); // 鼠标按下
    window.addEventListener("mousemove", onMouseMove); // 鼠标移动
    window.addEventListener("mouseup", onMouseUp); // 鼠标松开
    renderer.domElement.addEventListener("click", selectMesh); // 点击检测
    window.addEventListener("mousemove", selectMesh); // 滑过检测
    window.addEventListener("click", closeContextMenu); // 点击空白处关闭右键菜单

    // 全局禁止原生右键菜单：画布区域
    renderer.domElement.addEventListener("contextmenu", (e) => {
        e.preventDefault();
        e.stopPropagation();
    });
    // 全局禁止原生右键菜单：主容器内
    window.addEventListener("contextmenu", (e) => {
        if (mainDom.contains(e.target)) {
            e.preventDefault();
            e.stopPropagation();
        }
    });

    // 绑定自定义右键菜单事件
    renderer.domElement.addEventListener("contextmenu", openContextMenu);

    // 新增：绑定键盘快捷键事件
    window.addEventListener("keydown", handleKeyDown);
});

// ==================== 组件卸载：清理资源 ====================
onUnmounted(() => {
    // 移除所有鼠标事件监听
    renderer.domElement.removeEventListener("mousedown", onMouseDown);
    window.removeEventListener("mousemove", onMouseMove);
    window.removeEventListener("mouseup", onMouseUp);
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

    // 新增：移除键盘快捷键监听
    window.removeEventListener("keydown", handleKeyDown);

    // 清理Three.js资源：释放渲染器
    if (renderer) renderer.dispose();
    // 遍历场景释放几何体和材质资源
    if (scene) {
        scene.traverse((obj) => {
            if (obj.geometry) obj.geometry.dispose();
            if (obj.material) obj.material.dispose();
        });
    }
});
</script>

<style scoped lang="scss">
// 根容器样式：全屏显示，垂直布局
.content {
    width: 100%;
    height: 100vh;
    display: flex;
    flex-direction: column;
}
// 头部操作栏样式：内边距16px，水平布局，间距12px
.header {
    padding: 16px;
    display: flex;
    gap: 12px;
}
// 3D场景主容器：占满剩余空间，相对定位
.main {
    flex: 1;
    position: relative;
}
// 模型名称标签样式：固定定位，半透明背景，白色文字，指针不事件
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
// 自定义右键菜单样式：固定定位，白色背景，阴影，高z-index
.context-menu {
    position: fixed;
    width: 120px;
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
    z-index: 99999;
    // 菜单项样式
    .menu-item {
        padding: 8px 12px;
        cursor: pointer;
        // 鼠标悬停样式
        &:hover {
            background: #f5f5f5;
        }
    }
}
</style>
