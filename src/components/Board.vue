<template>
  <div class="wrapper">
    <canvas
      ref="canvas"
      @mousedown="startDrawing"
      @mousemove="draw"
      @mouseup="stopDrawing"
      @touchstart="startDrawing"
      @touchmove="draw"
      @touchend="stopDrawing"
    ></canvas>

    <canvas ref="cacheCanvas" style="display: none"></canvas>

    <div class="bar-top">
      <div class="color-picker">
        <span>画笔颜色：</span>
        <input
          id="colorPicker"
          type="color"
          @change="(e) => onChangeColor(e)"
        />
      </div>
      <div class="font-setup">
        <span>字体大小：</span>
        <select id="fontsizeSelect" @change="(e) => onChangeFontSize(e)">
          <option value="1">1</option>
          <option value="2">2</option>
          <option value="3">3</option>
          <option value="4">4</option>
          <option value="5">5</option>
        </select>
      </div>
    </div>
    <div class="bar">
      <div class="btn" id="eraser" @click="onBrush()">
        <img :src="brushSVG" title="画笔" />
      </div>
      <div class="btn" id="eraser" @click="onEraser()">
        <img :src="eraserSVG" title="橡皮擦" />
      </div>
      <div class="btn" id="reset" @click="onReset()">
        <img :src="resetSVG" title="重置" />
      </div>
      <div class="btn" id="revoke" @click="Undo()">
        <img :src="undoSVG" title="撤销" />
      </div>
      <div class="btn" id="recover" @click="onRedo()">
        <img :src="redoSVG" title="恢复" />
      </div>
    </div>
  </div>
</template>
<script setup>
import { ref, onMounted } from "vue";
import brushSVG from "../assets/img/brush.svg";
import eraserSVG from "../assets/img/eraser.svg";
import resetSVG from "../assets/img/zoom-reset.svg";
import redoSVG from "../assets/img/arrow-forward.svg";
import undoSVG from "../assets/img/arrow-back.svg";
import colorPickSVG from "../assets/img/color-picker.svg";

let isDrawing = false;
const canvas = ref(null);
const cacheCanvas = ref(null);
let context = null;
let cacheContext = null;

let penColor = "#000000";
let fontsize = "1";
let isErasing = false;
let isPecial = false;
let step = 0;
let histroyList = [];

const isPC = () => {
  const userAgent = navigator.userAgent;
  const pcKeywords = ["Windows", "Macintosh", "Linux"];
  for (const keyword of pcKeywords) {
    if (userAgent.includes(keyword)) {
      return true;
    }
  }
  return false;
};

const startDrawing = (event) => {
  isDrawing = true;
  context.beginPath();

  console.log(isPC());
  if (isPC()) {
    const { offsetX, offsetY } = event;

    context.moveTo(offsetX, offsetY);
  } else {
    let vertex = canvas.value.getBoundingClientRect();
    let t = vertex.top;
    let l = vertex.left;

    context.moveTo(
      parseInt(event.changedTouches[0].pageX - l),
      parseInt(event.touches[0].pageY - t)
    );
  }
};
const draw = (event) => {
  if (!isDrawing) return;
  if (!isPecial) return;

  let offsetX = 0;
  let offsetY = 0;
  if (isPC()) {
    offsetX = event.offsetX;
    offsetY = event.offsetY;
  } else {
    let vertex = canvas.value.getBoundingClientRect();
    let t = vertex.top;
    let l = vertex.left;

    offsetX = parseInt(event.changedTouches[0].pageX - l);
    offsetY = parseInt(event.changedTouches[0].pageY - t);
  }

  if (isErasing) {
    context.beginPath();
    context.arc(offsetX, offsetY, 10, 0, Math.PI * 2);
    context.fill();
  } else {
    context.lineTo(offsetX, offsetY);
    context.stroke();
  }
};
const stopDrawing = () => {
  console.log("stopDrawing");
  isDrawing = false;

  if (isErasing || isPecial) {
    step++;
    if (step < histroyList.length) {
      histroyList.length = step;
    }
    histroyList.push(canvas.value.toDataURL());
  }
};

const onBrush = () => {
  isErasing = false;
  isPecial = true;
  context.globalCompositeOperation = "source-over";
};

const onEraser = () => {
  isErasing = !isErasing;
  context.globalCompositeOperation = "destination-out";
};

const onReset = () => {
  context.clearRect(0, 0, canvas.value.width, canvas.value.height);
  histroyList = [];
};

const Undo = () => {
  if (step > 0) {
    onBrush(); // 撤销必须重新设置globalCompositeOperation，否则就出现空白

    step--;
    // context.clearRect(0, 0, canvas.value.width, canvas.value.height);
    let pic = new Image();
    pic.src = histroyList[step];
    pic.addEventListener("load", () => {
      setCache(pic);

      context.clearRect(0, 0, canvas.value.width, canvas.value.height);
      context.drawImage(cacheCanvas.value, 0, 0);
      //   context.drawImage(pic, 0, 0);
    });
  } else {
    console.log("不能继续撤销了");
  }
};

const onRedo = () => {
  if (step < histroyList.length - 1) {
    onBrush();
    step++;
    let pic = new Image();
    pic.src = histroyList[step];
    pic.addEventListener("load", () => {
      setCache(pic);

      context.clearRect(0, 0, canvas.value.width, canvas.value.height);
      context.drawImage(cacheCanvas.value, 0, 0);
      //   context.drawImage(pic, 0, 0);
    });
  } else {
    console.log("不能继续恢复了");
  }
};

const onChangeColor = (e) => {
  penColor = e.target.value;
  context.strokeStyle = penColor;

  console.log(e.target.value);
};

const onChangeFontSize = (e) => {
  fontsize = e.target.value;
  context.lineWidth = fontsize;
  console.log(e.target.value);
};

// 设置缓存，防止抖动
const setCache = (imgSrc) => {
  cacheCanvas.value.width = canvas.value.width;
  cacheCanvas.value.height = canvas.value.height;
  cacheContext = cacheCanvas.value.getContext("2d");
  cacheContext.drawImage(imgSrc, 0, 0);
};

onMounted(() => {
  // 适配canvas宽高
  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;

  context = canvas.value.getContext("2d");

  histroyList.push(canvas.value.toDataURL());

  window.addEventListener(
    "resize",
    () => {
      canvas.value.width = window.innerWidth;
      canvas.value.height = window.innerHeight;

      // 调整canvas宽高后重新绘制
      context.clearRect(0, 0, canvas.value.width, canvas.value.height);
      let pic = new Image();
      pic.src = histroyList[step];
      pic.addEventListener("load", () => {
        context.drawImage(pic, 0, 0);
      });
    },
    false
  );
});
</script>

<style scoped lang="scss">
.wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  overflow-x: hidden;
  overflow-y: hidden;
}
canvas {
  border: 1px solid #000;
  background-color: transparent;
}
.bar-top {
  width: 500px;
  height: 50px;
  background-color: rgb(87, 114, 231);
  border-radius: 25px;
  display: flex;
  justify-content: space-around;
  align-items: center;
  position: absolute;
  left: 20px;
  top: 100px;
  font-size: 25px;
  color: white;
  .color-picker {
    display: flex;
    align-items: center;
  }
  .font-setup {
    display: flex;
    align-items: center;
  }
}
.bar {
  width: 60px;
  height: 50vh;
  background-color: rgb(87, 114, 231);
  border-radius: 30px;
  position: absolute;
  left: 20px;
  top: 25vh;
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
  .btn {
    width: 50px;
    height: 50px;
    border-radius: 20px;
    background-color: #fff;
    color: #000;
    font-size: 16px;
    font-weight: bold;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    font-size: 22px;
  }
  .btn:active {
    background-color: #868383;
    color: #fff;
  }
}
.img {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
</style>
