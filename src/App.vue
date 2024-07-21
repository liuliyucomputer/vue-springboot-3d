<template>
  <div id="app"><!-- 整个应用的基础容器 -->
    <div id="model-viewer" ref="modelViewer" class="model-viewer"></div>
    <!-- 用于承载 Three.js 渲染的3D模型视图 -->
    <div v-if="isLoading.value" class="loading">加载中...</div>
    <!-- 多余代码 -->
  </div>
</template>
<script>
import * as THREE from 'three';
// threeStlLoader.js需要从第三方下载，npm下载有错误。
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader';
import axios from 'axios';//接口库 用于发起HTTP请求
import { ref, onMounted, onUnmounted, reactive } from 'vue';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';//旋转手柄
export default {
  setup() {
    const modelViewer = ref(null);//定义DOM元素引用，用于访问模型视图容器DOM元素
    const isLoading = reactive({ value: true }); //响应式的对象
    const scene = new THREE.Scene();// 创建一个新的Three.js场景
    let camera;
    let renderer;
    let controls;// 声明相机、渲染器和控制器变量
    onMounted(async () => {// 组件挂载后执行的代码
      initThree();//设置3D环境，包括透视相机、渲染器尺寸、背景色、光源以及OrbitControls。
      await loadSTLModel(); //从服务器加载STL文件并将其转化为3D模型，添加到场景中。
      window.addEventListener('resize', handleWindowResize);// 监听窗口大小改变事件
      animate();// 启动渲染循环
      isLoading.value = false; //设置加载状态为false 
    });
    onUnmounted(() => {// 组件卸载前执行的代码
      window.removeEventListener('resize', handleWindowResize);// 移除窗口大小改变事件监听
      if (renderer) {
        renderer.dispose(); // 清理渲染器资源
      }
      if (controls) {
        controls.dispose(); // 清理控制器资源
      }
    });
    function initThree() {// 初始化Three.js设置的函数
      const width = modelViewer.value.clientWidth;// 获取模型视图容器的宽度
      const height = modelViewer.value.clientHeight;// 获取模型视图容器的高度
      camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);// 创建透视相机
      camera.position.z = 5;// 设置相机的初始位置
      renderer = new THREE.WebGLRenderer({ antialias: true });// 创建WebGL渲染器并开启抗锯齿
      renderer.setSize(width, height);// 设置渲染器的尺寸
      renderer.setPixelRatio(window.devicePixelRatio);// 设置渲染器的像素比，提高在高DPI设备上的清晰度
      renderer.setClearColor(0xffffff); // 设置背景色为白色
      modelViewer.value.appendChild(renderer.domElement); // 将渲染器的DOM元素添加到模型视图容器中
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);// 创建环境光源
      scene.add(ambientLight);// 将环境光源添加到场景中
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);// 创建方向光源
      directionalLight.position.set(5, 5, 5);// 设置方向光源的位置
      directionalLight.castShadow = true;// 允许方向光源产生阴影
      scene.add(directionalLight); // 将方向光源添加到场景中
      controls = new OrbitControls(camera, renderer.domElement);// 创建OrbitControls实例
      controls.enableDamping = true;// 启用阻尼（惯性），使控制更加平滑
      controls.dampingFactor = 0.05;// 设置阻尼系数
    }
    async function loadSTLModel() {// 从服务器加载STL模型的函数
      try {
        const response = await axios.get('http://localhost:8080/散热器.stl', { responseType: 'arraybuffer' });
        // 使用axios请求STL文件
        const loader = new STLLoader();// 创建STLLoader实例
        const geometry = loader.parse(response.data);// 解析STL文件数据
        const material = new THREE.MeshStandardMaterial({// 创建材质
          color: 0x777777,// 设置颜色
          metalness: 0.5, // 设置金属感
          roughness: 0.4,// 设置粗糙度
          transparent: true,// 设置材质透明
          opacity: 0.75// 设置透明度
        });
        const mesh = new THREE.Mesh(geometry, material); // 创建网格模型
        mesh.scale.setScalar(0.01);// 缩放模型
        mesh.castShadow = true;// 允许模型投射阴影
        mesh.receiveShadow = true;// 允许模型接收阴影
        scene.add(mesh);// 将模型添加到场景中
        mesh.position.sub(new THREE.Box3().setFromObject(mesh).getCenter(new THREE.Vector3())); // 将模型的中心移到原点
      } catch (error) {
        console.error('加载STL文件出错:', error);
      }
    }
    function handleWindowResize() {// 处理窗口大小改变的函数
      const width = modelViewer.value.clientWidth; // 获取新的宽度
      const height = modelViewer.value.clientHeight;// 获取新的高度
      camera.aspect = width / height;// 更新相机的宽高比
      camera.updateProjectionMatrix();// 更新相机的投影矩阵
      renderer.setSize(width, height);// 更新渲染器的尺寸
    }
    function animate() { // 渲染循环函数
      requestAnimationFrame(animate); // 请求下一帧动画
      controls.update(); // 更新控制器状态
      renderer.render(scene, camera);// 渲染场景
    }
    return {
      modelViewer,
      isLoading, // 返回模型视图容器和加载状态，以便在模板中使用
    };
  },
};
</script>
<style scoped>

#app {
  position: relative;
  width: 100%;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #ffffff;

}
.model-viewer {
  width: 60%;
  height: 80vh;
  background-color: transparent;
  border: 1px solid #e1e1e1;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
.loading {
  position: absolute;
  color: #333;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 10px 20px;
  border-radius: 5px;
  border: 1px solid #ddd;
  font-family: 'KaiTi', 'STKaiti', serif;
}</style>