<template>
  <div class="container" v-if="visible">
    <h1>Clipping</h1>
    <div class="controls">
      <p class="svg-list">
        <span>选择形状：</span>
        <span
          v-for="svg in getSvgs()"
          :key="svg.name"
          @click="changeSvg(svg.path)"
          v-bind:class="{ selected: svg.path === svgPath }"
        >
          {{ svg.name }}
          <svg viewBox="0 0 24 24" width="16" height="16">
            <path :d="svg.path" fill="none" :stroke="svg.path === svgPath ? 'red' : 'black'" stroke-width="2" />
          </svg>
        </span>
      </p>
      <p class="prop-list">
        <span>调整参数：</span>
        <span
          >宽度
          <input
            :max="canvasWidth"
            :min="1"
            v-model.number="shellWidth"
            type="number"
            step="1"
            @change="changeShell('width', $event)"
        /></span>
        <span
          >高度
          <input
            :max="canvasHeight"
            :min="1"
            v-model.number="shellHeight"
            type="number"
            step="1"
            @change="changeShell('height', $event)"
        /></span>
        <!-- <span>scaleX <input v-model.number="shellScaleX" type="number" step="0.01" @change="changeShell('scaleX', $event)" /></span>
        <span>scaleY <input v-model.number="shellScaleY" type="number" step="0.01" @change="changeShell('scaleY', $event)" /></span> -->
        <span
          >旋转
          <input
            :max="360"
            :min="-360"
            v-model.number="shellAngle"
            type="number"
            @change="changeShell('angle', $event)"
        /></span>
        <span
          >背景缩放
          <select v-model.number="imgBgScale" @change="changeImgBgScale">
            <option value="0.25">0.25</option>
            <option value="0.5">0.5</option>
            <option value="0.75">0.75</option>
            <option value="1">1</option>
            <option value="1.25">1.25</option>
            <option value="1.5">1.5</option>
            <option value="2">2</option>
            <option value="3">3</option>
            <option value="5">5</option>
            <option value="10">10</option>
          </select>
          {{ toFixed2(imgBgScale) }}
        </span>
      </p>
    </div>
    <div
      ref="wheelContainerRef"
      class="clipping-container"
      :style="{
        width: canvasWidth / 2 + 'px',
        height: canvasHeight / 2 + 'px',
      }"
    >
      <div
        class="clipping-box"
        :style="{ width: canvasWidth + 'px', height: canvasHeight + 'px' }"
      >
        <img ref="imgBgRef" :src="imgBgUrl" alt="" />
        <canvas
          ref="canvasElementRef"
          :width="canvasWidth"
          :height="canvasHeight"
        ></canvas>
      </div>
    </div>
    <div class="submit-box">
      <button @click="clipAndSave">剪裁并保存为dataURL</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref, defineProps, defineExpose, watch } from "vue";
import { fabric } from "fabric";

const canvasElementRef = ref<HTMLCanvasElement | null>(null);
const imgBgRef = ref<HTMLImageElement | null>(null);

const wheelContainerRef = ref<HTMLDivElement | null>(null);
const imgBgUrl = ref("");
const imgBgScale = ref(1);
const canvasWidth = ref(1280 * 2);
const canvasHeight = ref(720 * 2);
const shellWidth = ref(0);
const shellHeight = ref(0);
const shellScaleX = ref(0);
const shellScaleY = ref(0);
const shellAngle = ref(0);
const visible = ref(true);
const svgPath = ref("");

let fabricCanvas: fabric.Canvas | null = null;
let fabricImage: fabric.Image | null = null;
let shellPath: fabric.Path | null = null;
let clipPath: fabric.Path | null = null;
let download = false;
let ready = false;
let cacheShellConfig = null;
let isChangingSvg = false;

onMounted(async () => {});

function toFixed2(num: number): number {
  const rounded = Math.round(num * 100);
  return Number(`${rounded / 100}`);
}

function getSvgs() {
  return [
    {
      name: "方形",
      path: "M 4 4 L 20 4 L 20 20 L 4 20 Z",
    },
    {
      name: "心形",
      path: "M20.808,11.079C19.829,16.132,12,20.5,12,20.5s-7.829-4.368-8.808-9.421C2.227,6.1,5.066,3.5,8,3.5a4.444,4.444,0,0,1,4,2,4.444,4.444,0,0,1,4-2C18.934,3.5,21.773,6.1,20.808,11.079Z",
    },
    {
      name: "五角星形",
      path: "M18.25,15.52l1.36,7.92L12.5,19.7,5.39,23.44l1.36-7.92L1,9.92,8.95,8.76l3.55-7.2,3.55,7.2L24,9.92Z",
    },
  ];
}

function init(shellConfig) {
  cacheShellConfig = shellConfig;
  svgPath.value = shellConfig?.svgPath || getSvgs()[0].path;
  download = shellConfig?.download || false;
  imgBgUrl.value = shellConfig?.imgBgUrl || "";
  imgBgScale.value = toFixed2(shellConfig?.imgBgScale || 1);
  initFabricCanvas({ shellConfig });
  initWheel(wheelContainerRef.value);
  ready = true;
}

function dispose() {
  if (fabricCanvas) {
    // 移除所有事件监听器
    fabricCanvas.off();
    // 销毁旧的canvas实例，释放资源
    fabricCanvas.dispose();
  }
  // 重置全局变量
  fabricCanvas = null;
  fabricImage = null;
  shellPath = null;
  clipPath = null;
  ready = false;
}

function getShellConfig() {
  return {
    svgPath: svgPath.value,
    imgBgUrl: imgBgUrl.value,
    imgBgScale: toFixed2(imgBgScale.value),
    top: toFixed2(shellPath.top),
    left: toFixed2(shellPath.left),
    scaleX: toFixed2(shellPath.scaleX),
    scaleY: toFixed2(shellPath.scaleY),
    angle: toFixed2(shellPath.angle),
    originX: shellPath.originX,
    originY: shellPath.originY,
  };
}

function getDefaultClipPathOptions(scale = 10) {
  return {
    originX: "center",
    originY: "center",
    left: canvasWidth.value / 2,
    top: canvasHeight.value / 2,
    scaleX: scale,
    scaleY: scale,
  };
}

function changeSvg(path: string) {
  if (isChangingSvg) return;
  isChangingSvg = true;
  dispose();
  setTimeout(() => {
    const newParams = {
      ...cacheShellConfig,
      ...getDefaultClipPathOptions(),
      angle: 0,
      svgPath: path,
    };
    init(newParams);
    isChangingSvg = false;
  }, 100);
}

async function clipAndSave() {
  if (!fabricCanvas || !fabricImage) {
    console.error("Canvas or image not initialized");
    return;
  }

  if (!fabricImage.clipPath) {
    console.error("No clipPath found");
    return;
  }

  if (!shellPath) {
    console.error("Shell not found");
  }

  return new Promise((resolve, reject) => {
    if (fabricCanvas.getActiveObject()) {
      fabricCanvas.discardActiveObject();
      fabricCanvas.renderAll();
    }

    try {
      // 直接获取整个canvas的dataURL，这是最可靠的方法
      const fullDataURL = fabricCanvas.toDataURL("image/png");

      if (shellPath) {
        const shellBounds = shellPath.getBoundingRect();
        cropImage(
          fullDataURL,
          shellBounds.left,
          shellBounds.top,
          shellBounds.width,
          shellBounds.height
        )
          .then((croppedDataURL) => {
            download && downloadImage(croppedDataURL);
            const params = {
              dataURL: croppedDataURL,
              shellConfig: getShellConfig(),
            };
            console.log("剪裁成功！dataURL:", params);
            resolve(params);
          })
          .catch((error) => {
            console.error("剪裁失败1:", error);
            reject(error);
          });
      } else {
        download && downloadImage(fullDataURL);
        const params = { dataURL: fullDataURL, shellConfig: getShellConfig() };
        console.log(
          "剪裁成功！完整canvas图像已下载，dataURL已输出到控制台",
          params
        );
        resolve(params);
      }
    } catch (error) {
      console.error("剪裁失败2:", error);
      reject(error);
    }
  });
}

function downloadImage(dataURL) {
  const link = document.createElement("a");
  link.href = dataURL;
  link.download = "clipped-image.png";
  link.click();
}

function cropImage(
  dataURL: string,
  x: number,
  y: number,
  width: number,
  height: number
): Promise<string> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.crossOrigin = "anonymous";
    img.onload = () => {
      const tempCanvas = document.createElement("canvas");
      tempCanvas.width = width;
      tempCanvas.height = height;
      const tempCtx = tempCanvas.getContext("2d");

      if (tempCtx) {
        tempCtx.drawImage(img, x, y, width, height, 0, 0, width, height);
        const croppedDataURL = tempCanvas.toDataURL("image/png");
        resolve(croppedDataURL);
      } else {
        reject(new Error("Could not get context for temp canvas"));
      }
    };
    img.onerror = () => {
      reject(new Error("Failed to load image from dataURL"));
    };
    img.src = dataURL;
  });
}

function roundShellProperties({ fabricCanvas, shellPath, clipPath }) {
  const roundedLeft = Math.round(shellPath.left);
  const roundedTop = Math.round(shellPath.top);
  shellPath.set({
    left: roundedLeft,
    top: roundedTop,
  });
  clipPath.setPositionByOrigin(
    { x: roundedLeft, y: roundedTop },
    "center",
    "center"
  );

  fabricCanvas.renderAll();
}

function initWheel(container) {
  if (!container) return;
  // 移除已有的wheel事件监听器，避免重复添加
  container.removeEventListener("wheel", handleWheelEvent);
  // 添加wheel事件监听器
  container.addEventListener("wheel", handleWheelEvent, { passive: false });
}

function changeShell(property: string, e: any) {
  if (!shellPath || !fabricCanvas || !clipPath) return;

  switch (property) {
    case "width":
      // 修改shellPath的宽度
      if (typeof shellWidth.value === "number" && !isNaN(shellWidth.value)) {
        if (shellWidth.value > canvasWidth.value) {
          shellWidth.value = canvasWidth.value;
        }
        if (shellWidth.value <= 10) {
          shellWidth.value = 10;
        }
        // 计算原始路径宽度 = 当前边界框宽度 / 当前scaleX
        const originalWidth =
          shellPath.getBoundingRect().width / shellPath.scaleX;

        // 计算新的scaleX = 目标宽度 / 原始路径宽度
        const newScaleX = shellWidth.value / originalWidth;

        // 更新shellPath的scaleX，只修改scaleX，不修改scaleY
        shellPath.set({
          scaleX: newScaleX,
        });

        // 同步更新clipPath
        clipPath.set({
          scaleX: newScaleX,
        });
      }
      break;

    case "height":
      // 修改shellPath的高度
      if (typeof shellHeight.value === "number" && !isNaN(shellHeight.value)) {
        if (shellHeight.value > canvasHeight.value) {
          shellHeight.value = canvasHeight.value;
        }
        if (shellHeight.value <= 10) {
          shellHeight.value = 10;
        }
        // 计算原始路径高度 = 当前边界框高度 / 当前scaleY
        const originalHeight =
          shellPath.getBoundingRect().height / shellPath.scaleY;

        // 计算新的scaleY = 目标高度 / 原始路径高度
        const newScaleY = shellHeight.value / originalHeight;

        // 更新shellPath的scaleY，只修改scaleY，不修改scaleX
        shellPath.set({
          scaleY: newScaleY,
        });

        // 同步更新clipPath
        clipPath.set({
          scaleY: newScaleY,
        });
      }
      break;

    case "angle":
      // 修改shellPath的旋转角度
      // 检查值是否为数字且不是NaN，允许0值
      if (typeof shellAngle.value === "number" && !isNaN(shellAngle.value)) {
        shellPath.set({
          angle: shellAngle.value,
        });

        // 同步更新clipPath
        clipPath.set({
          angle: shellAngle.value,
        });
      }
      break;

    default:
      break;
  }
  fabricCanvas.renderAll();
}

function changeImgBgScale() {
  if (fabricImage && fabricCanvas) {
    fabricImage.scale(imgBgScale.value);
    restrictShellPathWithinImage();
    fabricCanvas.renderAll();
    if (imgBgRef.value) {
      imgBgRef.value.style.transform = `translate(-50%, -50%) scale(${imgBgScale.value})`;
    }
  }
}

// 分离wheel事件处理函数，便于移除事件监听器
function handleWheelEvent(e: any) {
  if (e.ctrlKey || e.metaKey) {
    e.preventDefault();
    // 计算缩放比例，每次缩放10%
    const scaleFactor = e.deltaY > 0 ? 0.9 : 1.1;
    if (fabricImage && fabricCanvas) {
      const currentScale = fabricImage.scaleX;
      imgBgScale.value = toFixed2(currentScale * scaleFactor);
      changeImgBgScale();
    }
  }
}

function initImgScale({ fabricImage, clipPath, imgWidth, imgHeight }) {
  // 设置图片缩放并居中
  fabricImage.scale(imgBgScale.value).set({
    ...getDefaultClipPathOptions(imgBgScale.value),
    clipPath: clipPath,
    selectable: false, // 禁止选中img
    evented: false, // 禁止响应事件
    hasControls: false, // 隐藏控制点
    hasBorders: false, // 隐藏边框
    lockMovementX: true, // 锁定X轴移动
    lockMovementY: true, // 锁定Y轴移动
    lockScalingX: true, // 锁定X轴缩放
    lockScalingY: true, // 锁定Y轴缩放
    lockRotation: true, // 锁定旋转
    lockSkewingX: true, // 锁定X轴倾斜
    lockSkewingY: true, // 锁定Y轴倾斜
    lockUniScaling: true, // 锁定等比缩放
    lockFlipX: true, // 锁定X轴翻转
    lockFlipY: true, // 锁定Y轴翻转
  });

  if (imgBgRef.value) {
    imgBgRef.value.width = imgWidth;
    imgBgRef.value.height = imgHeight;

    // 应用复合变换：先居中，再缩放
    imgBgRef.value.style.transform = `translate(-50%, -50%) scale(${imgBgScale.value})`;
  }
}

function clipPathSetPosition({ clipPath, shellPath }) {
  const shellRect = shellPath.getBoundingRect();
  shellWidth.value = toFixed2(shellRect.width);
  shellHeight.value = toFixed2(shellRect.height);
  shellScaleX.value = toFixed2(shellPath.scaleX);
  shellScaleY.value = toFixed2(shellPath.scaleY);
  shellAngle.value = toFixed2(shellPath.angle);

  return clipPath.setPositionByOrigin(
    shellPath.getCenterPoint(),
    "center",
    "center"
  );
}

// 添加边界检查和限制函数，确保shellPath不超出fabricCanvas区域
function restrictShellPathWithinImage() {
  if (!shellPath || !fabricCanvas) return;

  // 获取fabricCanvas的边界（0, 0, canvasWidth, canvasHeight）
  const canvasWidth = fabricCanvas.width;
  const canvasHeight = fabricCanvas.height;

  // 获取shellPath的边界框
  let shellBounds = shellPath.getBoundingRect();

  // 计算shellPath的中心点
  const shellCenterX = shellPath.left;
  const shellCenterY = shellPath.top;

  // 检查并限制shellPath的位置
  // 计算shellPath的一半宽高
  const halfShellWidth = shellBounds.width / 2;
  const halfShellHeight = shellBounds.height / 2;

  // 计算允许的最小和最大坐标，只考虑fabricCanvas的边界
  const minX = 0 + halfShellWidth;
  const maxX = canvasWidth - halfShellWidth;
  const minY = 0 + halfShellHeight;
  const maxY = canvasHeight - halfShellHeight;

  // 限制shellPath的位置
  let newLeft = shellCenterX;
  let newTop = shellCenterY;

  if (shellCenterX < minX) newLeft = minX;
  if (shellCenterX > maxX) newLeft = maxX;
  if (shellCenterY < minY) newTop = minY;
  if (shellCenterY > maxY) newTop = maxY;

  // 更新shellPath的位置
  shellPath.set({
    left: newLeft,
    top: newTop,
  });

  // 同步更新clipPath
  clipPathSetPosition({ clipPath, shellPath });

  // 检查并限制shellPath的大小，确保不超出fabricCanvas区域
  // 获取更新后的shellPath边界框
  shellBounds = shellPath.getBoundingRect();

  // 计算允许的最大宽高，只考虑fabricCanvas的边界
  const maxAllowedWidth = canvasWidth;
  const maxAllowedHeight = canvasHeight;

  // 如果shellPath超出了允许的区域，缩小它
  if (
    shellBounds.width > maxAllowedWidth ||
    shellBounds.height > maxAllowedHeight
  ) {
    // 计算缩小比例
    const scaleFactorX = maxAllowedWidth / shellBounds.width;
    const scaleFactorY = maxAllowedHeight / shellBounds.height;
    const scaleFactor = Math.min(scaleFactorX, scaleFactorY);

    // 更新shellPath的缩放比例
    shellPath.set({
      scaleX: shellPath.scaleX * scaleFactor,
      scaleY: shellPath.scaleY * scaleFactor,
    });

    // 同步更新clipPath
    clipPath.set({
      scaleX: shellPath.scaleX,
      scaleY: shellPath.scaleY,
    });
  }
}

function initFabricCanvas({ shellConfig }) {
  if (!canvasElementRef.value) {
    console.error("Canvas element not found");
    return;
  }
  fabricCanvas = new fabric.Canvas(canvasElementRef.value);
  fabricCanvas.preserveObjectStacking = true;
  fabric.Object.prototype.transparentCorners = false;

  fabric.Image.fromURL(imgBgUrl.value, (img) => {
    fabricImage = img;
    const options = getDefaultClipPathOptions();
    const newOption = {
      originX: shellConfig.originX || options.originX,
      originY: shellConfig.originY || options.originY,
      left: shellConfig.left || options.left,
      top: shellConfig.top || options.top,
      scaleX: shellConfig.scaleX || options.scaleX,
      scaleY: shellConfig.scaleY || options.scaleY,
      angle: shellConfig.angle || 0,
    };
    shellPath = new fabric.Path(svgPath.value, {
      ...newOption,
      fill: "",
      stroke: "red",
      strokeWidth: 0,
      cornerSize: 15, // 拖拽点大小
      transparentCorners: false, // 拖拽点不透明
    });
    clipPath = new fabric.Path(svgPath.value, {
      ...newOption,
      absolutePositioned: true,
    });

    // 计算图片缩放比例，按最小边适配canvas
    const imgWidth = fabricImage.width || 1;
    const imgHeight = fabricImage.height || 1;
    const scaleX = canvasWidth.value / imgWidth;
    const scaleY = canvasHeight.value / imgHeight;
    imgBgScale.value = Math.min(scaleX, scaleY);

    initImgScale({ fabricImage, clipPath, imgWidth, imgHeight });
    clipPathSetPosition({ clipPath, shellPath });

    shellPath.on("moving", () => {
      clipPathSetPosition({ clipPath, shellPath });
      restrictShellPathWithinImage();
    });
    shellPath.on("scaling", () => {
      clipPathSetPosition({ clipPath, shellPath });
      clipPath.set({ scaleX: shellPath.scaleX, scaleY: shellPath.scaleY });
      restrictShellPathWithinImage();
    });
    shellPath.on("rotating", () => {
      clipPath.set({ angle: shellPath.angle });
      restrictShellPathWithinImage();
      shellAngle.value = toFixed2(shellPath.angle);
    });
    shellPath.on("selected", () => {});
    shellPath.on("deselected", () => {});

    fabricCanvas.on("object:modified", (e) => {
      // 检查修改的对象是否是shell
      if (e.target === shellPath) {
        // 限制shellPath不超出fabricImage和fabricCanvas区域
        restrictShellPathWithinImage();

        // 计算并更新shell的宽高
        clipPathSetPosition({ clipPath, shellPath });
        roundShellProperties({ fabricCanvas, shellPath, clipPath });
      }
    });
    fabricCanvas.add(fabricImage, shellPath);
    fabricCanvas.setActiveObject(fabricImage);
  });
}

defineExpose({
  getSvgs,
  init,
  dispose,
});
</script>

<style>
.container {
  background: #fff;
  padding: 20px;
  color: #000;
  text-align: left;
}
.container h1 {
  text-align: center;
}
.controls {
  margin-bottom: 20px;
  -webkit-user-select: none;
  user-select: none;
}
.svg-list {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}
.svg-list .selected {
  color: red;
}
.prop-list {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}
.prop-list input {
  display: inline-block;
  width: 7em;
}
.controls button:hover {
  background-color: #45a049;
}
.clipping-container {
  overflow: hidden;
}
.clipping-box {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  transform: scale(0.5);
  transform-origin: top left;
  background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAwAAAAMCAIAAADZF8uwAAAAGUlEQVQYV2M4gwH+YwCGIasIUwhT25BVBADtzYNYrHvv4gAAAABJRU5ErkJggg==);
  background-size: 30px 30px;
  background-repeat: repeat;
}
.clipping-box::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: #000;
  opacity: 0.5; /* 设置背景图片的透明度，0.5表示50%透明 */
  z-index: 1; /* 确保背景图片在底层 */
}
.clipping-box img {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  /* 使用filter实现黑色半透明效果，代替opacity */
  filter: grayscale(0%) brightness(50%);
}
.clipping-box canvas {
  position: relative;
  z-index: 3;
  width: 100%;
  height: 100%;
}
canvas {
  border: 1px solid #000;
  box-sizing: border-box;
}

.submit-box {
  margin-top: 20px;
  text-align: right;
}
.submit-box button {
  display: inline-block;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>
