模型文件：.GLTF和.OBJ

常规流程：创建场景、创建光源、创建网格模型（网格模型由几何体+材质组成）/加载GLTF模型、创建相机、创建渲染器、创建轨道控制器（用于鼠标拖动看模型的其他面）

```ts
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

// 创建场景
const scene = new THREE.Scene();

// 创建光源，光源分为点光源、平行光、聚光灯、环境光等
// 添加环境光 照亮所有的物体
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

// 创建网格模型/加载GLTF模型
// 1.网格模型
// 创建几何体
const geometry = new THREE.BoxGeometry(200, 100, 50);
// 创建材质
const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
// 创建网格模型（由几何体+材质组成，可以有多个）
const mesh = new THREE.Mesh(geometry, material);
// 将网格模型添加到场景中
scene.add(mesh);
// 2.加载gltf模型
const loader = new GLTFLoader();
loader.load("./Soldier.glb", (gltf) => {
  console.log(gltf);
  scene.add(gltf.scene);
});

// 创建相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000,
);
camera.position.set(0, 0, 500);
scene.add(camera);

// 创建渲染器
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);
renderer.render(scene, camera);

// 创建轨道控制器
const controls = new OrbitControls(camera, renderer.domElement);
const animate = () => {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
};
animate();

```

