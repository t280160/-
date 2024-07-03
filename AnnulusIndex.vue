<template>
  <div class="box" ref="container">
    <div class="orbit">
      <div
        class="pos"
        ref="pos"
        :class="getBgImage(item.projectCount)"
        v-for="(item, key) in props.data"
        :key="item"
        @click="handelClick(item, key)"
      >
        <div class="product-panel">
          <div class="panel-title">{{ item.keyDirection }}</div>
          <div class="panel-item text-ellipsis">
            <el-icon class="panel-title"><CaretRight /></el-icon>
            <span class="panel-item-title"> 项目</span>
            <span class="panel-title text-ellipsis">{{ item.projectCount }}</span>
          </div>
        </div>
      </div>
    </div>
    <div class="earth"></div>
  </div>
</template>
<script setup>
import { CaretRight } from '@element-plus/icons-vue'
import { useWindowResize } from '@/utils/use'
import { ref, nextTick, onMounted, markRaw, computed, defineEmits, watch } from 'vue'

const props = defineProps({
  data: {
    type: Array,
    default: () => []
  },
  speed: {
    type: Number,
    default: 30
  },
  activeCube: {
    type: String,
    default: ''
  }
})
const emit = defineEmits(['item-click'])
const pos = ref(null)
/* 缓存数组 */
const translateArr = ref([])
/* 上一个点击的坐标 */
const activeIndex = ref(0)
/* 是否在动画过程中 */
const boolClick = ref(false)

/* 计算属性 数据长度 */
const dataL = computed(() => props.data.length)

/* 点击切换数据 */
const handelClick = (item, key) => {
  /* 禁止连点 */
  if (boolClick.value === true) {
    return
  }
  boolClick.value = true
  if (key === activeIndex.value) {
    emitItemClick(item)
    return
  }
  /* 偏移距离 */
  let differenceV = 0
  if (key > activeIndex.value) {
    differenceV = key - activeIndex.value
  } else {
    differenceV = dataL.value - Math.abs(key - activeIndex.value)
  }
  if (differenceV === 0) return
  const setStyles = new Array(dataL.value)
    .fill('')
    .map((index, key) => setDomStyle(differenceV, key))
  Promise.all(setStyles).then(() => {
    /* 重新排列顺序 */
    const arr = [...translateArr.value]
    const arr2 = arr.splice(-differenceV)
    translateArr.value = arr2.concat(...arr)

    /* 当前中心 */
    activeIndex.value = key
    emitItemClick(item)
  })
}
/* 旋转结束后发送 事件 */
const emitItemClick = (item) => {
  emit('item-click', item)
  boolClick.value = false
}

/* 获取初始化数据 */
const getStyleS = (index) => {
  const newIndex = index + 1
  const m = parseInt(360 / dataL.value)
  // const speed = dataL.value % 2 === 0 ? m : m
  let arrStyle = new Array(m + 1).fill().map((e, key) => {
    //每一个BOX对应的角度;
    var avd = 360 / dataL.value
    const ad = key + index * m
    //每一个BOX对应的弧度;
    var ahd = ((avd + ad - props.speed) * Math.PI) / (180 * newIndex)

    return ahd
  })
  const container = document.querySelector('.box')
  const dot = document.querySelectorAll('.pos')[index]

  //中心点纵坐标
  var dotLeft = (container.clientWidth - dot.clientWidth) / 2
  var dotTop = (container.clientHeight - dot.clientHeight) / 2

  // 椭圆大小
  const earth = {
    width: container.clientWidth / 2,
    height: container.clientHeight / 2
  }

  //半径 偏移6像素 看着在圆环上面
  arrStyle = arrStyle.map((el) => {
    const translate = {
      left: Math.sin(el * newIndex) * earth.width + dotLeft + 'px',
      top: Math.cos(el * newIndex) * earth.height + dotTop - 6 + 'px'
    }
    return {
      transform: `translate(${translate.left}, ${translate.top}) rotateY(-4deg)`
    }
  })

  return arrStyle
}
/* 存储每个周长的距离 */
const getTranslateArr = () => {
  let arr = []
  for (let index = 0; index < dataL.value; index++) {
    arr.unshift({
      data: getStyleS(index),
      index: index + 1
    })
  }
  return arr
}
/* 更新dom初始位置 */
const initDomStyle = () => {
  const els = pos.value || []
  els.forEach((el, key) => {
    el.style.transform = translateArr.value[key].data[0].transform
  })
}

/* 判断差值获取每个元素的移动距离 */
const setDomStyle = (differenceV, clickIndex) => {
  /* 开始累加移动距离 */
  let count = 0
  let index = clickIndex
  let differenceArr = []
  while (count < differenceV) {
    index = index < 0 ? translateArr.value.length - 1 : index
    differenceArr.push(markRaw(translateArr.value[index].data))
    index--
    count++
  }
  const flatDiff = differenceArr.flat(1)
  const timeNum = 500 / flatDiff.length
  const el = pos.value[clickIndex]
  let cishu = 0
  return new Promise((resolve) => {
    const time = setInterval(() => {
      if (cishu === flatDiff.length) {
        clearInterval(time)
        resolve(true)
        return
      }
      el.style.transform = flatDiff[cishu].transform
      cishu++
    }, timeNum)
  })
}
/* 根据数量返回图片 */
const getBgImage = (num) => {
  if (num == '3') {
    return 'ellipse_mini'
  } else if (num == '2') {
    return 'ellipse_medium'
  } else {
    return ' ellipse_big'
  }
}

/*  重置数据*/
const reset = () => {
  nextTick(() => {
    translateArr.value = getTranslateArr()
    activeIndex.value = 0
    initDomStyle()
  })
}
watch(
  () => props.data,
  () => {
    reset()
  },
  {
    deep: true
  }
)
useWindowResize(() => {
  if (translateArr.value.length > 0) {
    const newArr = getTranslateArr()
    translateArr.value = translateArr.value.map((el) => {
      const data = newArr.filter((i) => el.index === i.index)[0]
      return {
        index: el.index,
        data: data.data
      }
    })
    initDomStyle()
  }
}, 60)
onMounted(() => {
  reset()
})
</script>
<style lang="scss" scoped>
.box {
  margin: auto;
  position: relative;
  width: 76.5vw;
  height: 33.3vh;
  top: 10vh;
  transform: rotateX(25deg) rotateZ(0deg);
  z-index: 5;
  pointer-events: none;
  animation-name: top;
  animation-timing-function: ease-out;
  animation-duration: 1s;
}
.earth {
  box-sizing: border-box;
  border-radius: 100%;
  width: 100%;
  height: 100%;
  z-index: 1;
  position: absolute;
  top: 0;
  left: 0;
  background-image: url('../assets/details/earth.png');
  background-size: 100% 100%;
  background-repeat: no-repeat;
  background-position: center;
}

.orbit {
  width: 100%;
  height: 100%;
  transition-duration: 0.8s;
  transition-timing-function: ease-in-out;
  transition-property: width, height, top, left, margin-left, margin-top, webkit-transform;
  transform-style: preserve-3d;
  animation-duration: 12s;
  transform: rotateZ(-360deg);
  /* animation-name: orbit; */
  animation-iteration-count: infinite;
  animation-timing-function: linear;
  z-index: 100;
  position: absolute;
  top: 0;
  left: 0;
}
/* 小球 */
.pos {
  width: 3.8vw;
  height: 3.8vw;
  position: absolute;
  z-index: 2;
  transform-style: preserve-3d;
  animation-iteration-count: infinite;
  animation-timing-function: linear;
  color: #000;
  background-repeat: no-repeat;
  background-size: 100% 100%;
  background-position: center;
  pointer-events: all;
  .product-panel {
    min-width: 9.4vw;
    max-width: 13vw;
    min-height: 7.2vh;
    padding-top: 1.1em;
    padding-left: 1.2em;
    padding-right: 1.2em;
    padding-bottom: 1.4em;
    box-sizing: border-box;
    background-image: url('../assets/details/product.png');
    background-size: 100% 100%;
    background-repeat: no-repeat;
    position: absolute;
    top: -100%;
    left: 100%;
    z-index: 4;
    display: flex;
    flex-direction: column;
    justify-content: center;
    opacity: 1;
    animation-name: opacity1;
    animation-duration: 2s;
    .panel-title {
      font-family: PingFangSC-Semibold;
      font-size: 1.1em;
      color: #ffffff;
      letter-spacing: 0;
    }
    .panel-item {
      margin-top: 0.5em;
      display: flex;
      align-items: center;

      .panel-item-title {
        opacity: 0.9;
        font-family: PingFangSC-Regular;
        font-size: 1em;
        color: #ffffff;
        letter-spacing: 0;
        font-weight: 400;
        margin-right: 12px;
      }
    }
  }
}
/* 小球大图 */
.ellipse_mini {
  background-image: url('../assets/details/ellipse_mini.png');
}
.ellipse_big {
  background-image: url('../assets/details/ellipse_big.png');
}

.ellipse_medium {
  background-image: url('../assets/details/ellipse_medium.png');
}

@keyframes top {
  0% {
    opacity: 0;
    transform: rotateX(25deg) rotateZ(0deg) scale(1);
  }
  1% {
    top: 30vh;
    opacity: 0.5;
    transform: rotateX(25deg) rotateZ(0deg) scale(0.5);
  }
  100% {
    top: 10vh;
    opacity: 1;
    transform: rotateX(25deg) rotateZ(0deg) scale(1);
  }
}
@keyframes opacity1 {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0;
  }

  100% {
    opacity: 1;
  }
}
</style>
