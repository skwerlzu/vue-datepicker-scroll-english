<template>
  <div
    class="picker-item"
    @touchstart="_onTouchStart"
    @touchmove.prevent="_onTouchMove"
    @touchend="_onTouchEnd"
    @touchcancel="_onTouchEnd"
  >
    <div class="picker-mask" :style="pickerMask"></div>
    <ul class="picker-li" :style="transformStyle">
      <li
        v-for="(item, index) in data"
        v-text="timeFormat(item.name||item)"
        :key="index"
        :class="{'disabled':item.disabled}"
      ></li>
    </ul>
  </div>
</template>
<script type="text/ecmascript-6">
import dayjs from "dayjs";
import "dayjs/locale/zh-cn";

export default {
  name: "picker-item",
  data() {
    return {
      startY: 0, //touch时鼠标所有位置
      startOffset: 0, //touch前已移动的距离
      offset: 0 //当前移动的距离
    };
  },
  watch: {
    height() {
      //父组件mounted后更新了height的高度，这里将数据移动到指定位置
      this._moveTo();
    },
    data() {
      //在联动时，数据变化了，下级还会保持在上一次的移动位置
      this._moveTo();
    }
  },
  props: {
    height: Number, //移动单位的高度
    data: Array,
    change: Function,
    value: String, //选中的值
    index: Number //当前索引，多个选择时如联动时，指向的是第几个选择，在change时返回去区别哪项改变了
  },
  components: {},
  methods: {
    _getTouch(event) {
      return event.changedTouches[0] || event.touches[0];
    },
    _getVisibleCount() {
      //取显示条数的一半，因为选中的在中间，显示条数为奇数
      return Math.floor(this.$parent.visibleCount / 2);
    },
    _onTouchStart(event) {
      const touch = this._getTouch(event);
      this.startOffset = this.offset;
      this.startY = touch.clientY;
    },
    _onTouchMove(event) {
      const touch = this._getTouch(event);
      const currentY = touch.clientY;
      const distance = currentY - this.startY;
      this.offset = this.startOffset + distance;
    },
    _onTouchEnd() {
      let index = Math.round(this.offset / this.$parent.liHeight);
      const vc = this._getVisibleCount();
      // console.log("liHeight:" + this.$parent.liHeight);
      // console.log("this.offset:" + this.offset);
      // console.log("index:" + index);
      // index的有效范围
      const indexMax = vc - this.data.length;

      // console.log(index, this.data, this.data.length, indexMax, vc);

      if (index >= vc) {
        index = 0; // 选择第一个
      } else if (index < indexMax) {
        // 选择最后一个
        index = this.data.length - 1; //最后一个
      } else {
        index = vc - index;
      }
      /*if (index >= vc) {
          index = 0;// 选择第一个
        } else if (Math.abs(index) + vc > this.data.length - 1) {
          index = this.data.length - 1;//最后一个
        } else {
          if (index <= 0) {
            index = vc - index;
          }
        }*/
      // console.log("index2:" + index);
      this._setIndex(index, true);
    },
    _setIndex(index, bool) {
      //按显示5条计算，选择第3条时，偏移为0，选择第1条时，偏移为li的高*2
      //即偏移距离为(5/2取整－index)*liHeight
      //如果当前选中的为disabled状态，则往下选择，仅在滑动选择时判断，默认填值时不作判断
      //存在数据加载问题，有可能初始时数据是空的
      if (index > this.data.length - 1) {
        index = this.data.length - 1;
      }

      if (this.data.length > 0) {
        bool ? (index = this._isDisabled(index, index)) : "";
        this.offset = (this._getVisibleCount() - index) * this.height;
        //回调
        const value = this.data[index].value || this.data[index];
        this.change ? this.change(value, this.index, bool) : "";
      }
    },
    _isDisabled(index, index2) {
      if (this.data[index].disabled) {
        if (index == this.data.length - 1) {
          index = -1; //到最后一条时，再从第一条开始找
        }
        //防止死循环，全都是disabled时，原路返回
        if (index + 1 == index2) {
          return index2;
        }
        return this._isDisabled(index + 1);
      }
      return index;
    },
    _moveTo() {
      //根据value移动动相对应的位置，这个是组件加载完引用
      let index = 0;
      for (let i = 0; i < this.data.length; i++) {
        let v = this.data[i].value || this.data[i];
        if (this.value === v) {
          index = i;
          break;
        }
      }
      // index = this.data[i].findIndex(e=>e===this.value)
      this._setIndex(index, false);
      //没有默认时或是value不存在于数据数组中时index=0
    },

    // 时间格式化
    timeFormat(date) {
      let time = date;
      if (date.length > 5) {
        time = dayjs(date)
          .locale("zh-cn")
          .format("YYYY-MM-DD dddd");

        const today = dayjs()
          .locale("zh-cn")
          .format("YYYY-MM-DD dddd");
        const todayWeek = dayjs()
          .locale("zh-cn")
          .format("dddd");

        time = time === today ? time.replace(todayWeek, "今天") : time; // 把今天替换掉对应的周
        time = time.replace("week", "week"); // 把星期换成周
      }

      return time;
    }
  },
  computed: {
    pickerMask() {
      return {
        //设定过度遮罩的显示高度，即总显示个数减1（高亮）的一半
        backgroundSize: "100% " + this._getVisibleCount() * this.height + "px"
      };
    },
    transformStyle() {
      return {
        transition: "all 150ms ease",
        transform: `translate3d(0, ${this.offset}px, 0)`
      };
    }
  },
  mounted() {},
  filters: {}
};
</script>

<style lang="scss" scoped>
ul,
li {
  margin: 0;
  padding: 0;
  list-style: none;
  outline: none;
}

.picker-item {
  width: 100%;
  position: relative;
  overflow: hidden;
  // 对于三列，第一列要求更宽
  &:nth-child(2) {
    width: 250%;
  }
  .picker-mask {
    position: absolute;
    left: 0;
    top: 0;
    height: 100%;
    width: 100%;
    z-index: 3;
    background: linear-gradient(
        180deg,
        rgba(255, 255, 255, 0.95),
        rgba(255, 255, 255, 0.6)
      ),
      linear-gradient(0deg, rgba(255, 255, 255, 0.95), rgba(255, 255, 255, 0.6));
    background-repeat: no-repeat;
    background-position: top, bottom;
  }
  li {
    height: 50px;
    line-height: 50px;
    text-align: center;
    overflow: hidden;
    box-sizing: border-box;
    &.disabled {
      font-style: italic;
    }
  }
}
</style>