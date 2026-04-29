<template>
    <div class="cad-page">
        <aside class="cad-sidebar">
            <div class="title-block">
                <p class="eyebrow">CAD TO 3D</p>
                <h1>DXF 简化建模</h1>
                <p class="description">上传 DXF 后，前端会把二维线段和轮廓按墙体高度、厚度拉伸成可预览的 3D 模型。</p>
            </div>

            <section class="panel-section">
                <div class="section-title">图纸文件</div>
                <label class="upload-box" :class="{ disabled: isWorking }">
                    <input type="file" accept=".dxf,.dwg" :disabled="isWorking" @change="handleFileChange" />
                    <span class="upload-main">{{ fileName || "选择 DXF 文件" }}</span>
                    <span class="upload-sub">支持 ASCII DXF；DWG 需要先另存为 DXF</span>
                </label>
                <div class="action-row">
                    <el-button type="primary" :loading="isWorking" :disabled="!dxfText" @click="buildModel">
                        生成 3D
                    </el-button>
                    <el-button :disabled="isWorking" @click="loadSampleModel">示例模型</el-button>
                </div>
            </section>

            <section class="panel-section">
                <div class="section-title">建模参数</div>
                <div class="field">
                    <label>图纸单位</label>
                    <el-select v-model="unitScale" :disabled="isWorking" @change="rebuildIfReady">
                        <el-option label="毫米图纸" :value="0.001" />
                        <el-option label="厘米图纸" :value="0.01" />
                        <el-option label="米图纸" :value="1" />
                    </el-select>
                </div>
                <div class="field two-columns">
                    <label>墙体高度</label>
                    <el-input-number
                        v-model="wallHeight"
                        :min="0.2"
                        :max="100"
                        :step="0.2"
                        :disabled="isWorking"
                        @change="rebuildIfReady"
                    />
                </div>
                <div class="field two-columns">
                    <label>墙体厚度</label>
                    <el-input-number
                        v-model="wallThickness"
                        :min="0.02"
                        :max="20"
                        :step="0.02"
                        :disabled="isWorking"
                        @change="rebuildIfReady"
                    />
                </div>
                <div class="field two-columns">
                    <label>曲线精度</label>
                    <el-input-number
                        v-model="curveSegments"
                        :min="12"
                        :max="160"
                        :step="4"
                        :disabled="isWorking"
                        @change="rebuildIfReady"
                    />
                </div>
            </section>

            <section class="panel-section">
                <div class="section-title">结果</div>
                <div class="stats-grid">
                    <div>
                        <span>{{ stats.lines }}</span>
                        <small>线段</small>
                    </div>
                    <div>
                        <span>{{ stats.polylines }}</span>
                        <small>多段线</small>
                    </div>
                    <div>
                        <span>{{ stats.circles + stats.arcs }}</span>
                        <small>曲线</small>
                    </div>
                    <div>
                        <span>{{ stats.segments }}</span>
                        <small>墙段</small>
                    </div>
                </div>
                <p class="status-text">{{ statusText }}</p>
                <p v-if="ignoredText" class="status-warning">{{ ignoredText }}</p>
                <el-button class="export-button" type="success" :disabled="!currentModel" @click="exportGlb">
                    导出 GLB
                </el-button>
            </section>
        </aside>

        <main class="cad-viewer">
            <div ref="viewerRef" class="viewer-canvas"></div>
            <div class="viewer-hint">
                <span>拖拽旋转</span>
                <span>滚轮缩放</span>
                <span>右键平移</span>
            </div>
        </main>
    </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, shallowRef } from "vue";
import { ElMessage } from "element-plus";
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { GLTFExporter } from "three/examples/jsm/exporters/GLTFExporter.js";

const viewerRef = ref(null);
const fileName = ref("");
const dxfText = ref("");
const isWorking = ref(false);
const currentModel = shallowRef(null);

const unitScale = ref(0.001);
const wallHeight = ref(3);
const wallThickness = ref(0.2);
const curveSegments = ref(48);

const stats = ref({
    lines: 0,
    polylines: 0,
    circles: 0,
    arcs: 0,
    segments: 0,
});
const ignoredTypes = ref({});
const statusText = ref("请选择一个 DXF 文件，或先打开示例模型。");

const ignoredText = computed(() => {
    const entries = Object.entries(ignoredTypes.value);
    if (!entries.length) return "";
    return `已忽略暂不支持的实体：${entries.map(([type, count]) => `${type} ${count}`).join("，")}`;
});

let scene;
let camera;
let renderer;
let controls;
let resizeObserver;
let animationFrameId;
let gridHelper;
let groundMesh;

const wallMaterial = new THREE.MeshStandardMaterial({
    color: 0x4b7cba,
    roughness: 0.58,
    metalness: 0.06,
});
const curveMaterial = new THREE.MeshStandardMaterial({
    color: 0x6f9b72,
    roughness: 0.6,
    metalness: 0.04,
});
const floorMaterial = new THREE.MeshStandardMaterial({
    color: 0xd7dee8,
    roughness: 0.8,
    metalness: 0,
    transparent: true,
    opacity: 0.38,
    side: THREE.DoubleSide,
});
const outlineMaterial = new THREE.LineBasicMaterial({
    color: 0x172033,
    transparent: true,
    opacity: 0.44,
});

const sampleDxf = `0
SECTION
2
ENTITIES
0
LWPOLYLINE
8
WALL
90
4
70
1
10
0
20
0
10
8000
20
0
10
8000
20
5200
10
0
20
5200
0
LWPOLYLINE
8
ROOM
90
4
70
0
10
3200
20
0
10
3200
20
2200
10
8000
20
2200
10
8000
20
3000
0
LINE
8
DOOR
10
0
20
2600
11
1600
21
2600
0
CIRCLE
8
COLUMN
10
6400
20
3900
40
420
0
ARC
8
CURVE
10
1800
20
3900
40
900
50
20
51
160
0
ENDSEC
0
EOF`;

onMounted(() => {
    initScene();
    loadSampleModel();
});

onUnmounted(() => {
    if (animationFrameId) cancelAnimationFrame(animationFrameId);
    if (resizeObserver) resizeObserver.disconnect();
    if (controls) controls.dispose();
    if (renderer) {
        renderer.dispose();
        renderer.domElement.remove();
    }
    disposeObject(currentModel.value);
    wallMaterial.dispose();
    curveMaterial.dispose();
    floorMaterial.dispose();
    outlineMaterial.dispose();
});

const initScene = () => {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf4f7fb);

    camera = new THREE.PerspectiveCamera(45, 1, 0.01, 10000);
    camera.position.set(8, 7, 9);

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    viewerRef.value.appendChild(renderer.domElement);

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.08;
    controls.target.set(0, 1, 0);

    const ambient = new THREE.HemisphereLight(0xffffff, 0x9aa6b2, 2.2);
    scene.add(ambient);

    const keyLight = new THREE.DirectionalLight(0xffffff, 2.4);
    keyLight.position.set(12, 18, 10);
    keyLight.castShadow = true;
    scene.add(keyLight);

    gridHelper = new THREE.GridHelper(20, 20, 0x9daabc, 0xd1d8e2);
    gridHelper.position.y = -0.01;
    scene.add(gridHelper);

    const groundGeometry = new THREE.PlaneGeometry(200, 200);
    const groundMaterial = new THREE.ShadowMaterial({ color: 0x1f2937, opacity: 0.08 });
    groundMesh = new THREE.Mesh(groundGeometry, groundMaterial);
    groundMesh.rotation.x = -Math.PI / 2;
    groundMesh.position.y = -0.02;
    groundMesh.receiveShadow = true;
    scene.add(groundMesh);

    resizeObserver = new ResizeObserver(resizeRenderer);
    resizeObserver.observe(viewerRef.value);
    resizeRenderer();
    renderLoop();
};

const resizeRenderer = () => {
    if (!viewerRef.value || !renderer || !camera) return;
    const { clientWidth, clientHeight } = viewerRef.value;
    renderer.setSize(clientWidth, clientHeight);
    camera.aspect = clientWidth / Math.max(clientHeight, 1);
    camera.updateProjectionMatrix();
};

const renderLoop = () => {
    animationFrameId = requestAnimationFrame(renderLoop);
    controls.update();
    renderer.render(scene, camera);
};

const handleFileChange = async (event) => {
    const [file] = event.target.files || [];
    event.target.value = "";
    if (!file) return;

    const lowerName = file.name.toLowerCase();
    if (lowerName.endsWith(".dwg")) {
        ElMessage.warning("浏览器不能直接解析 DWG，请先在 CAD 软件中另存为 ASCII DXF 后再上传。");
        return;
    }
    if (!lowerName.endsWith(".dxf")) {
        ElMessage.warning("请上传 DXF 文件。");
        return;
    }

    fileName.value = file.name;
    isWorking.value = true;
    statusText.value = "正在读取 DXF 文件...";

    try {
        dxfText.value = await file.text();
        buildModel();
    } catch (error) {
        console.error(error);
        ElMessage.error("文件读取失败。");
        statusText.value = "文件读取失败，请重新选择。";
    } finally {
        isWorking.value = false;
    }
};

const loadSampleModel = () => {
    fileName.value = "示例平面.dxf";
    dxfText.value = sampleDxf;
    buildModel();
};

const rebuildIfReady = () => {
    if (dxfText.value && currentModel.value) buildModel();
};

const buildModel = () => {
    if (!dxfText.value) return;

    isWorking.value = true;
    statusText.value = "正在解析并生成 3D 模型...";

    try {
        const parsed = parseDxfEntities(dxfText.value);
        const buildResult = createModelFromEntities(parsed);

        if (!buildResult.group.children.length) {
            throw new Error("没有找到可生成模型的 DXF 实体。");
        }

        setCurrentModel(buildResult.group);
        fitCameraToObject(buildResult.group);

        stats.value = {
            lines: parsed.lines.length,
            polylines: parsed.polylines.length,
            circles: parsed.circles.length,
            arcs: parsed.arcs.length,
            segments: buildResult.segmentCount,
        };
        ignoredTypes.value = parsed.ignoredTypes;
        statusText.value = `已生成 ${buildResult.segmentCount} 个三维墙段，可在右侧查看和导出。`;
        ElMessage.success("3D 模型已生成。");
    } catch (error) {
        console.error(error);
        statusText.value = error.message || "生成失败，请检查 DXF 内容。";
        ElMessage.error(statusText.value);
    } finally {
        isWorking.value = false;
    }
};

const setCurrentModel = (group) => {
    if (currentModel.value) {
        scene.remove(currentModel.value);
        disposeObject(currentModel.value);
    }
    currentModel.value = group;
    scene.add(group);
};

const exportGlb = () => {
    if (!currentModel.value) return;

    const exporter = new GLTFExporter();
    exporter.parse(
        currentModel.value,
        (result) => {
            const blob = new Blob([result], { type: "model/gltf-binary" });
            const url = URL.createObjectURL(blob);
            const link = document.createElement("a");
            link.href = url;
            link.download = `${fileName.value.replace(/\.[^.]+$/, "") || "cad-model"}.glb`;
            document.body.appendChild(link);
            link.click();
            link.remove();
            URL.revokeObjectURL(url);
        },
        (error) => {
            console.error(error);
            ElMessage.error("GLB 导出失败。");
        },
        { binary: true },
    );
};

const parseDxfEntities = (text) => {
    const pairs = createDxfPairs(text);
    const rawEntities = extractRawEntities(pairs);
    const result = {
        lines: [],
        polylines: [],
        circles: [],
        arcs: [],
        ignoredTypes: {},
    };
    let activePolyline = null;

    rawEntities.forEach((raw) => {
        switch (raw.type) {
            case "LINE":
                result.lines.push({
                    start: {
                        x: getNumber(raw, 10, 0),
                        y: getNumber(raw, 20, 0),
                    },
                    end: {
                        x: getNumber(raw, 11, 0),
                        y: getNumber(raw, 21, 0),
                    },
                });
                break;
            case "LWPOLYLINE": {
                const points = getPolylinePoints(raw);
                if (points.length >= 2) {
                    result.polylines.push({
                        points,
                        closed: Boolean(getNumber(raw, 70, 0) & 1) || isSamePoint(points[0], points[points.length - 1]),
                    });
                }
                break;
            }
            case "POLYLINE":
                activePolyline = {
                    points: [],
                    closed: Boolean(getNumber(raw, 70, 0) & 1),
                };
                break;
            case "VERTEX":
                if (activePolyline) {
                    activePolyline.points.push({
                        x: getNumber(raw, 10, 0),
                        y: getNumber(raw, 20, 0),
                    });
                }
                break;
            case "SEQEND":
                if (activePolyline && activePolyline.points.length >= 2) {
                    result.polylines.push(activePolyline);
                }
                activePolyline = null;
                break;
            case "CIRCLE":
                result.circles.push({
                    center: {
                        x: getNumber(raw, 10, 0),
                        y: getNumber(raw, 20, 0),
                    },
                    radius: Math.abs(getNumber(raw, 40, 0)),
                });
                break;
            case "ARC":
                result.arcs.push({
                    center: {
                        x: getNumber(raw, 10, 0),
                        y: getNumber(raw, 20, 0),
                    },
                    radius: Math.abs(getNumber(raw, 40, 0)),
                    startAngle: getNumber(raw, 50, 0),
                    endAngle: getNumber(raw, 51, 0),
                });
                break;
            default:
                if (raw.type && !["SECTION", "ENDSEC", "EOF"].includes(raw.type)) {
                    result.ignoredTypes[raw.type] = (result.ignoredTypes[raw.type] || 0) + 1;
                }
        }
    });

    if (activePolyline && activePolyline.points.length >= 2) {
        result.polylines.push(activePolyline);
    }

    return result;
};

const createDxfPairs = (text) => {
    const lines = text.replace(/\r/g, "").split("\n");
    const pairs = [];
    for (let index = 0; index < lines.length - 1; index += 2) {
        const code = lines[index].trim();
        const value = lines[index + 1].trim();
        if (code) pairs.push({ code, value });
    }
    return pairs;
};

const extractRawEntities = (pairs) => {
    const rawEntities = [];
    let inEntitiesSection = false;
    let current = null;

    for (let index = 0; index < pairs.length; index += 1) {
        const pair = pairs[index];
        const value = pair.value.toUpperCase();

        if (pair.code === "0" && value === "SECTION") {
            const next = pairs[index + 1];
            if (next?.code === "2" && next.value.toUpperCase() === "ENTITIES") {
                inEntitiesSection = true;
                index += 1;
                continue;
            }
        }

        if (!inEntitiesSection) continue;

        if (pair.code === "0" && value === "ENDSEC") {
            if (current) rawEntities.push(current);
            current = null;
            inEntitiesSection = false;
            continue;
        }

        if (pair.code === "0") {
            if (current) rawEntities.push(current);
            current = {
                type: value,
                pairs: [],
            };
        } else if (current) {
            current.pairs.push(pair);
        }
    }

    return rawEntities;
};

const getNumber = (raw, code, fallback) => {
    const item = raw.pairs.find((pair) => pair.code === String(code));
    if (!item) return fallback;
    const number = Number.parseFloat(item.value);
    return Number.isFinite(number) ? number : fallback;
};

const getPolylinePoints = (raw) => {
    const points = [];
    let currentPoint = null;

    raw.pairs.forEach((pair) => {
        if (pair.code === "10") {
            if (currentPoint && Number.isFinite(currentPoint.x) && Number.isFinite(currentPoint.y)) {
                points.push(currentPoint);
            }
            currentPoint = {
                x: Number.parseFloat(pair.value),
                y: Number.NaN,
            };
        }
        if (pair.code === "20" && currentPoint) {
            currentPoint.y = Number.parseFloat(pair.value);
        }
    });

    if (currentPoint && Number.isFinite(currentPoint.x) && Number.isFinite(currentPoint.y)) {
        points.push(currentPoint);
    }

    return points;
};

const createModelFromEntities = (entities) => {
    const bounds = getEntitiesBounds(entities);
    if (!bounds) {
        return { group: new THREE.Group(), segmentCount: 0 };
    }

    const origin = {
        x: (bounds.minX + bounds.maxX) / 2,
        y: (bounds.minY + bounds.maxY) / 2,
    };
    const group = new THREE.Group();
    group.name = "CAD_DXF_3D_MODEL";

    const outlinePositions = [];
    let segmentCount = 0;

    const addWall = (start, end, material = wallMaterial) => {
        const mesh = createWallSegment(transformPoint(start, origin), transformPoint(end, origin), material);
        if (!mesh) return;
        mesh.castShadow = true;
        mesh.receiveShadow = true;
        group.add(mesh);
        outlinePositions.push(
            mesh.userData.start.x,
            wallHeight.value + 0.02,
            mesh.userData.start.y,
            mesh.userData.end.x,
            wallHeight.value + 0.02,
            mesh.userData.end.y,
        );
        segmentCount += 1;
    };

    entities.lines.forEach((line) => addWall(line.start, line.end));

    entities.polylines.forEach((polyline) => {
        const points = normalizeClosingPoint(polyline.points);
        for (let index = 0; index < points.length - 1; index += 1) {
            addWall(points[index], points[index + 1]);
        }
        if (polyline.closed && points.length > 2) {
            addWall(points[points.length - 1], points[0]);
            addFloor(points.map((point) => transformPoint(point, origin)), group);
        }
    });

    entities.circles.forEach((circle) => {
        const points = sampleCircle(circle);
        for (let index = 0; index < points.length; index += 1) {
            addWall(points[index], points[(index + 1) % points.length], curveMaterial);
        }
    });

    entities.arcs.forEach((arc) => {
        const points = sampleArc(arc);
        for (let index = 0; index < points.length - 1; index += 1) {
            addWall(points[index], points[index + 1], curveMaterial);
        }
    });

    if (outlinePositions.length) {
        const outlineGeometry = new THREE.BufferGeometry();
        outlineGeometry.setAttribute("position", new THREE.Float32BufferAttribute(outlinePositions, 3));
        const outlines = new THREE.LineSegments(outlineGeometry, outlineMaterial);
        outlines.name = "CAD_TOP_OUTLINES";
        group.add(outlines);
    }

    return { group, segmentCount };
};

const transformPoint = (point, origin) => ({
    x: (point.x - origin.x) * unitScale.value,
    y: (point.y - origin.y) * unitScale.value,
});

const createWallSegment = (start, end, material) => {
    const dx = end.x - start.x;
    const dz = end.y - start.y;
    const length = Math.hypot(dx, dz);
    if (length < 0.0001) return null;

    const geometry = new THREE.BoxGeometry(length, wallHeight.value, wallThickness.value);
    const mesh = new THREE.Mesh(geometry, material);
    mesh.position.set((start.x + end.x) / 2, wallHeight.value / 2, (start.y + end.y) / 2);
    mesh.rotation.y = -Math.atan2(dz, dx);
    mesh.userData.start = start;
    mesh.userData.end = end;
    return mesh;
};

const addFloor = (points, group) => {
    if (points.length < 3) return;

    const shape = new THREE.Shape();
    shape.moveTo(points[0].x, points[0].y);
    points.slice(1).forEach((point) => shape.lineTo(point.x, point.y));
    shape.closePath();

    const geometry = new THREE.ShapeGeometry(shape);
    const position = geometry.attributes.position;
    for (let index = 0; index < position.count; index += 1) {
        position.setXYZ(index, position.getX(index), 0.015, position.getY(index));
    }
    geometry.computeVertexNormals();

    const mesh = new THREE.Mesh(geometry, floorMaterial);
    mesh.name = "CAD_CLOSED_AREA";
    mesh.receiveShadow = true;
    group.add(mesh);
};

const getEntitiesBounds = (entities) => {
    const bounds = {
        minX: Infinity,
        minY: Infinity,
        maxX: -Infinity,
        maxY: -Infinity,
    };
    const includePoint = (point) => {
        bounds.minX = Math.min(bounds.minX, point.x);
        bounds.minY = Math.min(bounds.minY, point.y);
        bounds.maxX = Math.max(bounds.maxX, point.x);
        bounds.maxY = Math.max(bounds.maxY, point.y);
    };

    entities.lines.forEach((line) => {
        includePoint(line.start);
        includePoint(line.end);
    });
    entities.polylines.forEach((polyline) => polyline.points.forEach(includePoint));
    entities.circles.forEach((circle) => {
        includePoint({ x: circle.center.x - circle.radius, y: circle.center.y - circle.radius });
        includePoint({ x: circle.center.x + circle.radius, y: circle.center.y + circle.radius });
    });
    entities.arcs.forEach((arc) => {
        sampleArc(arc).forEach(includePoint);
    });

    if (!Number.isFinite(bounds.minX)) return null;
    return bounds;
};

const normalizeClosingPoint = (points) => {
    const normalized = [...points];
    if (normalized.length > 2 && isSamePoint(normalized[0], normalized[normalized.length - 1])) {
        normalized.pop();
    }
    return normalized;
};

const isSamePoint = (first, second) => {
    if (!first || !second) return false;
    return Math.hypot(first.x - second.x, first.y - second.y) < 0.000001;
};

const sampleCircle = (circle) => {
    const count = Math.max(12, curveSegments.value);
    return Array.from({ length: count }, (_, index) => {
        const angle = (index / count) * Math.PI * 2;
        return {
            x: circle.center.x + Math.cos(angle) * circle.radius,
            y: circle.center.y + Math.sin(angle) * circle.radius,
        };
    });
};

const sampleArc = (arc) => {
    let start = THREE.MathUtils.degToRad(arc.startAngle);
    let end = THREE.MathUtils.degToRad(arc.endAngle);
    if (end < start) end += Math.PI * 2;

    const arcRatio = Math.max((end - start) / (Math.PI * 2), 0.02);
    const count = Math.max(4, Math.ceil(curveSegments.value * arcRatio));

    return Array.from({ length: count + 1 }, (_, index) => {
        const angle = start + ((end - start) * index) / count;
        return {
            x: arc.center.x + Math.cos(angle) * arc.radius,
            y: arc.center.y + Math.sin(angle) * arc.radius,
        };
    });
};

const fitCameraToObject = (object) => {
    const box = new THREE.Box3().setFromObject(object);
    const size = box.getSize(new THREE.Vector3());
    const center = box.getCenter(new THREE.Vector3());
    const maxSize = Math.max(size.x, size.y, size.z, 1);
    const distance = maxSize / (2 * Math.tan(THREE.MathUtils.degToRad(camera.fov / 2)));

    controls.target.copy(center);
    camera.position.set(center.x + distance * 0.75, center.y + distance * 0.65, center.z + distance * 0.95);
    camera.near = Math.max(distance / 1000, 0.01);
    camera.far = distance * 1000;
    camera.updateProjectionMatrix();
    controls.update();

    const gridSize = Math.max(Math.ceil(maxSize * 1.8), 10);
    scene.remove(gridHelper);
    gridHelper.geometry.dispose();
    disposeMaterial(gridHelper.material);
    gridHelper = new THREE.GridHelper(gridSize, Math.min(gridSize, 80), 0x9daabc, 0xd1d8e2);
    gridHelper.position.y = -0.01;
    scene.add(gridHelper);
};

const disposeObject = (object) => {
    if (!object) return;
    object.traverse((child) => {
        if (child.geometry) child.geometry.dispose();
        if (Array.isArray(child.material)) {
            child.material.forEach((material) => {
                if (!isSharedMaterial(material)) material.dispose();
            });
        } else if (child.material && !isSharedMaterial(child.material)) {
            child.material.dispose();
        }
    });
};

const disposeMaterial = (material) => {
    if (Array.isArray(material)) {
        material.forEach((item) => item.dispose());
        return;
    }
    if (material) material.dispose();
};

const isSharedMaterial = (material) => {
    return [wallMaterial, curveMaterial, floorMaterial, outlineMaterial].includes(material);
};
</script>

<style scoped lang="scss">
.cad-page {
    width: 100%;
    min-height: 100vh;
    display: grid;
    grid-template-columns: 360px minmax(0, 1fr);
    background: #eef2f7;
    color: #172033;
}

.cad-sidebar {
    min-height: 100vh;
    padding: 24px;
    background: #ffffff;
    border-right: 1px solid #dbe3ee;
    display: flex;
    flex-direction: column;
    gap: 18px;
    overflow-y: auto;
}

.title-block {
    display: grid;
    gap: 8px;

    h1 {
        font-size: 24px;
        font-weight: 700;
        line-height: 1.2;
        margin: 0;
    }
}

.eyebrow {
    font-size: 12px;
    color: #4b7cba;
    font-weight: 700;
    letter-spacing: 0;
}

.description {
    font-size: 14px;
    line-height: 1.6;
    color: #607086;
}

.panel-section {
    display: grid;
    gap: 14px;
    padding-top: 18px;
    border-top: 1px solid #e4eaf2;
}

.section-title {
    font-size: 15px;
    font-weight: 700;
}

.upload-box {
    display: grid;
    gap: 6px;
    padding: 18px;
    border: 1px dashed #96a8bf;
    border-radius: 8px;
    background: #f8fafc;
    cursor: pointer;
    transition:
        border-color 0.2s ease,
        background 0.2s ease;

    input {
        display: none;
    }

    &:hover {
        border-color: #4b7cba;
        background: #f3f7fc;
    }

    &.disabled {
        cursor: not-allowed;
        opacity: 0.65;
    }
}

.upload-main {
    font-size: 15px;
    font-weight: 700;
    word-break: break-all;
}

.upload-sub {
    font-size: 12px;
    color: #738399;
    line-height: 1.5;
}

.action-row {
    display: flex;
    gap: 10px;

    .el-button {
        flex: 1;
    }
}

.field {
    display: grid;
    gap: 8px;

    label {
        font-size: 13px;
        color: #506176;
    }

    :deep(.el-select),
    :deep(.el-input-number) {
        width: 100%;
    }
}

.two-columns {
    grid-template-columns: 88px minmax(0, 1fr);
    align-items: center;

    label {
        line-height: 32px;
    }
}

.stats-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 8px;

    div {
        display: grid;
        gap: 4px;
        padding: 10px 8px;
        border-radius: 8px;
        background: #f3f7fc;
        text-align: center;
    }

    span {
        font-size: 18px;
        font-weight: 700;
        color: #26384d;
    }

    small {
        font-size: 12px;
        color: #738399;
    }
}

.status-text,
.status-warning {
    font-size: 13px;
    line-height: 1.55;
}

.status-text {
    color: #52677f;
}

.status-warning {
    color: #a45b11;
}

.export-button {
    width: 100%;
}

.cad-viewer {
    position: relative;
    min-height: 100vh;
    overflow: hidden;
}

.viewer-canvas {
    width: 100%;
    height: 100%;
    min-height: 100vh;
}

.viewer-hint {
    position: absolute;
    left: 24px;
    bottom: 24px;
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    pointer-events: none;

    span {
        padding: 7px 10px;
        border-radius: 8px;
        background: rgba(255, 255, 255, 0.86);
        border: 1px solid rgba(151, 163, 182, 0.36);
        color: #526176;
        font-size: 12px;
        backdrop-filter: blur(8px);
    }
}

@media (max-width: 900px) {
    .cad-page {
        grid-template-columns: 1fr;
    }

    .cad-sidebar {
        min-height: auto;
        border-right: none;
        border-bottom: 1px solid #dbe3ee;
    }

    .cad-viewer,
    .viewer-canvas {
        min-height: 62vh;
    }
}
</style>
