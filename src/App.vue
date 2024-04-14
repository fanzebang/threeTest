<script setup lang="ts">
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import {ref,onMounted} from "vue";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { OutlinePass } from 'three/examples/jsm/postprocessing/OutlinePass.js';
import { Sky } from "three/examples/jsm/objects/Sky";


let container = ref<HTMLElement | null>(null);



onMounted(() => {

  const scene = new THREE.Scene();

  const camera = new THREE.PerspectiveCamera(45, container.value!.clientWidth / container.value!.clientHeight, 0.1, 1000);

  const renderer = new THREE.WebGLRenderer();

  const composer = new EffectComposer(renderer);
  composer.setSize(container.value!.clientWidth ,container.value!.clientHeight);

  const renderPass = new RenderPass(scene, camera);
  composer.addPass(renderPass);

  const outlinePass = new OutlinePass(new THREE.Vector2(container.value!.clientWidth, container.value!.clientHeight), scene, camera);
  composer.addPass(outlinePass);

  camera.position.set(-176, 84, 88);



  outlinePass.edgeStrength = 3;    // 控制边缘的强度
  outlinePass.edgeGlow = 1;        // 发光的强度
  outlinePass.edgeThickness = 1;   // 边缘的厚度
  outlinePass.visibleEdgeColor.set('#5e94e7'); // 边缘的颜色
  outlinePass.hiddenEdgeColor.set('#190a05'); // 被遮挡部分的边缘颜色


  renderer.setSize(container.value!.clientWidth , container.value!.clientHeight);


  // 创建天空对象
  const sky = new Sky();
  sky.scale.setScalar(450000); // 控制天空盒的大小
  scene.add(sky);


  var uniforms = sky.material.uniforms;

// 天空的亮度，散射等
  uniforms['turbidity'].value = 10;  // 浑浊度
  uniforms['rayleigh'].value = 2;    // 瑞利散射，控制散射强度
  uniforms['mieCoefficient'].value = 0.005; // 米氏散射系数
  uniforms['mieDirectionalG'].value = 0.8;  // 米氏散射方向因子

// 设置太阳的位置
  var sun = new THREE.Vector3();
  function updateSunPosition() {
    var theta = Math.PI * (parameters.inclination - 0.5);
    var phi = 2 * Math.PI * (parameters.azimuth - 0.5);

    sun.x = Math.cos(phi);
    sun.y = Math.sin(phi) * Math.sin(theta);
    sun.z = Math.sin(phi) * Math.cos(theta);

    uniforms['sunPosition'].value.copy(sun);
  }

// 更新太阳位置的示例参数
  var parameters = {
    inclination: 0.49,  // 太阳高度
    azimuth: 0.205      // 太阳方位角
  };
  updateSunPosition();


  container.value!.appendChild(renderer.domElement);

  const ambientLight = new THREE.AmbientLight(0xffffff, 2); // 环境光
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5); // 方向光
  directionalLight.position.set(0, 1, 1);
  scene.add(ambientLight, directionalLight);

  // 加载 gltf 模型
  const loader = new GLTFLoader();
  camera.position.y = 100

  let wireframeMaterial = new THREE.MeshBasicMaterial({ color: 0xffffff, wireframe: true });


  loader.load('../public/gltf/test.gltf', (gltf:any) => {

    gltf.scene.traverse(function(child:any) {
      if (child.isMesh) {
        child.userData.originalMaterial = child.material;
        child.material = wireframeMaterial;
        child.scale.y = 0; // Initialize scale at 0 for Y-axis
      }
    });
    scene.add(gltf.scene);
    animateToRealMaterial(gltf.scene);
  });


  function animateToRealMaterial(object:any) {
    const duration = 1000; // Duration in milliseconds
    const startTime = Date.now();

    function animate() {
      const elapsedTime = Date.now() - startTime;
      const completionRatio = Math.min(elapsedTime / duration, 1);

      object.traverse(function (child:any) {
        if (child.isMesh && child.userData.originalMaterial) {
          if (completionRatio < 1) {
            // Continue the animation
            child.material = wireframeMaterial;
            wireframeMaterial.opacity = 1 - completionRatio; // Fade out wireframe
            child.scale.y = completionRatio;
          } else {
            // Set the original material
            child.material = child.userData.originalMaterial;
            child.scale.y = 1; // Ensure full scale on Y-axis
          }
        }
      });

      if (completionRatio < 1) {
        requestAnimationFrame(animate);
      } else {
        // Ensure the final material is applied
        object.traverse(function (child:any) {
          if (child.isMesh && child.userData.originalMaterial) {
            child.material = child.userData.originalMaterial;
            child.scale.y = 1; // Ensure full scale on Y-axis
          }
        });
      }
    }

    requestAnimationFrame(animate);
  }


  const orbitControls = new OrbitControls(camera, renderer.domElement);



  orbitControls.enableDamping = true;
  orbitControls.dampingFactor = 0.05;
  orbitControls.autoRotate = true;
  orbitControls.autoRotateSpeed = 0.1;

  let isDragging = false;


  let rayCast = new THREE.Raycaster();
  let mouse = new THREE.Vector2();


  function createLabelTexture(message:string, fontsize:number, fontface:string, borderColor:string, backgroundColor:string, textColor:string) {
    var canvas = document.createElement('canvas');
    var context:any = canvas.getContext('2d');
    context.font = fontsize + "px " + fontface;

    // 获取文字宽度并设置画布大小
    var metrics = context.measureText(message);
    var textWidth = metrics.width;
    canvas.width = textWidth;
    canvas.height = fontsize * 1.4;

    // 背景
    context.fillStyle = borderColor;
    context.fillRect(0, 0, canvas.width, canvas.height);
    context.fillStyle = backgroundColor;
    context.fillRect(2, 2, canvas.width - 4, canvas.height - 4);

    // 文字
    context.fillStyle = textColor;
    context.fillText(message, 10, fontsize);

    var texture = new THREE.Texture(canvas);
    texture.needsUpdate = true;

    return texture;
  }



  function focusOnObject(object:any) {
    const targetPosition = object.position;

    const newCameraPosition = targetPosition.clone().add(new THREE.Vector3(-30, 50, 30));
    const duration = 1000;

    const startTime = Date.now();

    function updateCamera() {
      const elapsedTime = Date.now() - startTime;
      const t = elapsedTime / duration;

      if (t < 1) {
        camera.position.lerpVectors(camera.position, newCameraPosition, t);
        orbitControls.target.lerpVectors(orbitControls.target, targetPosition, t);
        orbitControls.update();
        requestAnimationFrame(updateCamera);
      } else {
        camera.position.copy(newCameraPosition);
        orbitControls.target.copy(targetPosition);
        orbitControls.update();
      }
    }
    updateCamera();
  }



  const getMouse = (event:any)=>{

      mouse.x = (event.clientX / container.value!.clientWidth) * 2 - 1;
      mouse.y = -(event.clientY/ container.value!.clientHeight) *2 + 1

    rayCast.setFromCamera(mouse, camera);

    const intersects = rayCast.intersectObjects(scene.children);



    if (intersects.length > 0) {

      outlinePass.selectedObjects = [];


      function addSelectObject(selected:any) {
        if (selected instanceof THREE.Mesh) {
          outlinePass.selectedObjects.push(selected);
        }else if (selected instanceof THREE.Group) {
          selected.children.forEach((item:any) => {
            addSelectObject(item);
          });
        }
      }


      let selected = intersects[0].object;
      let name = ""
      while (selected.parent && !selected.parent.isScene) {
        if ( selected.name && selected.name.includes("area")){
          name = selected.name;

          // 遍历 selected.children 中的所有 Mesh
          selected.children.forEach((item:any) => {
            addSelectObject(item);
          });

        }
        selected = selected.parent;
      }


      scene.children.forEach((item:any) => {
        if (item instanceof THREE.Sprite) {
          scene.remove(item);
        }
      });

      var spriteMaterial = new THREE.SpriteMaterial({
        map: createLabelTexture("区域:"+name.split("_")[1], 48, "Arial", "black", "white", "black")
      });
      var sprite = new THREE.Sprite(spriteMaterial);
      sprite.position.copy(intersects[0].point);
      sprite.scale.set(20, 10, 10);
      sprite.position.y = intersects[0].point.y + 10;
      scene.add(sprite)

      focusOnObject(sprite);

    }

  }

  container.value?.addEventListener("mousedown", ()=>{
    isDragging = false;
  })

  container.value?.addEventListener("mousemove", ()=>{
    isDragging = true;
  })

  container.value?.addEventListener("mouseup", ()=>{
    if (!isDragging) {
      getMouse(event)
    }
  })

  const animate = () => {
    requestAnimationFrame(animate);
    orbitControls.update();
    composer.render();
  }
  animate();
  window.addEventListener('resize', () => {
    camera.aspect = container.value!.clientWidth / container.value!.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.value!.clientWidth, container.value!.clientHeight);
  });
});



</script>

<template>
  <div id="container" ref="container"></div>
</template>

<style scoped>
  #container {
    width: 100vw;
    height: 100vh;
  }
</style>
