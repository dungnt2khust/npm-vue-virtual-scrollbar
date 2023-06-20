<template>
  <div :style="styleScrollbar" class="virtual-scrollbar" v-if="haveScroll">
    <div
      class="virtual-scrollbar__main"
      @mousedown="mousedownMain($event)"
      ref="wrapper"
    >
      <div
        class="virtual-scrollbar__track"
        :style="{
          height: !horizontal ? trackSize + 'px' : 'calc(100% - 2px)',
          width: horizontal ? trackSize + 'px' : 'calc(100% - 2px)',
          left: horizontal ? trackLeft + 'px' : '50%',
          top: !horizontal ? trackTop + 'px' : '50%',
          transform: horizontal ? 'translateY(-50%)' : 'translateX(-50%)',
        }"
        ref="track"
        @mousedown="
          $event.stopPropagation();
          mousedownTrack($event);
        "
      ></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
declare const window: any;

const props =
  defineProps<{ horizontal: boolean; target?: HTMLElement | string }>();
const emit = defineEmits(['positionChanged'])
// Ref element
const wrapper = ref<HTMLElement>();
const track = ref<HTMLElement>();

// Properties
const positionScroll = ref<number>(0);
const movingTrack = ref<boolean>(false);
const startX = ref<number>(0);
const startY = ref<number>(0);
const trackTop = ref<number>(0);
const trackLeft = ref<number>(0);
const clientSize = ref<number>(0);
const scrollSize = ref<number>(0);

const thumbSize = ref<number>(0);
const trackSize = ref<number>(0);
const elementScrollable = ref<HTMLElement | undefined | null>();
const haveScroll = ref<boolean>(false);
const intervalScrollDisplay = ref<any>(null);
const elementScrollablePosition = ref<any>({});

watch(
  () => props.target,
  (val) => {
    console.log(val);
    removeEventScrollbar();
    setElementScrollable(val);
    initEventScrollbar();
  }
);

onMounted(() => {
  removeEventScrollbar();
  setElementScrollable(props.target);
  initEventScrollbar();

  window.addEventListener('mousemove', handleMousemove);
  window.addEventListener('mouseup', handleMouseup);
  window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
  removeEventScrollbar();
  window.removeEventListener('mousemove', handleMousemove);
  window.removeEventListener('mouseup', handleMouseup);
  window.removeEventListener('resize', handleResize);
});

/**
 * Gắn element scrollable
 * @createdby ntdung 23.05.2023
 **/
function setElementScrollable(val : HTMLElement | undefined | string) {
  if (val) {
    if (typeof val == 'string') {
      let element = document.querySelector(val);
      if (element) elementScrollable.value = element as HTMLElement;
    } else {
      elementScrollable.value = val;
    }
  }
}

/**
 * Kiểm tra element có đang hiển thị
 * @createdby ntdung 23.05.2023
 **/
function elementScrollableIsDisplay() {
  if (elementScrollable.value) {
    if (document.body.contains(elementScrollable.value)) {
      return true;
    } else {
      return false;
    }
  } else return false;
}

/**
 * Khởi tạo scrollbar
 * @createdby ntdung 23.05.2023
 **/
function initEventScrollbar() {
  if (elementScrollable.value) {
    elementScrollable.value.addEventListener('scroll', handleScrollable);
  }
  intervalScrollDisplay.value = setInterval(() => {
    if (!elementScrollableIsDisplay()) haveScroll.value = false;
    else {
      checkHaveScroll();
      setSizeScrollbar();
    }
    // Watch vị trí của element scrollable
    if (elementScrollable.value) {
      let rect = elementScrollable.value.getBoundingClientRect();
      let position = {left: rect.left, right: rect.right, top: rect.top, bottom: rect.bottom};
      if (JSON.stringify(position) != JSON.stringify(elementScrollablePosition.value)) {
        elementScrollablePosition.value = position;
        emit('positionChanged', position);
      }
    }
  }, 10);

  setTimeout(() => {
    checkHaveScroll();
    setTimeout(() => {
      setSizeScrollbar();
    }, 100);
  }, 100);
}

/**
 * Xóa event scrollbar
 * @createdby ntdung 23.05.2023
 **/
function removeEventScrollbar() {
  if (elementScrollable.value)
    elementScrollable.value.removeEventListener('scroll', handleScrollable);
  clearInterval(intervalScrollDisplay.value);
}

/**
 * Xử lý sự kiện scroll
 * @createdby ntdung 23.05.2023
 **/
function handleScrollable() {
  if (elementScrollable.value) {
    if (props.horizontal) {
      let scrollLeft = elementScrollable.value!.scrollLeft;
      trackLeft.value = (scrollLeft / scrollSize.value) * thumbSize.value;
    } else {
      let scrollTop = elementScrollable.value!.scrollTop;
      trackTop.value = (scrollTop / scrollSize.value) * thumbSize.value;
    }
  }
}

/**
 * Xử lý sự kiện mousemove
 * @createdby ntdung 23.05.2023
 **/
function handleMousemove(e: any) {
  if (movingTrack.value && elementScrollable.value) {
    clearSelection();
    var outWrapperVal = outWrapper(e);
    if (!outWrapperVal.out) {
      if (props.horizontal) {
        trackLeft.value = e.clientX - startX.value;
      } else {
        trackTop.value = e.clientY - startY.value;
      }
    } else {
      if (!props.horizontal) {
        if (
          trackTop.value <
          thumbSize.value - trackSize.value - trackTop.value
        ) {
          trackTop.value = 0;
        } else {
          trackTop.value = thumbSize.value - trackSize.value;
        }
      } else {
        if (
          trackLeft.value <
          thumbSize.value - trackSize.value - trackLeft.value
        ) {
          trackLeft.value = 0;
        } else {
          trackLeft.value = thumbSize.value - trackSize.value;
        }
      }
    }
    if (props.horizontal) {
      elementScrollable.value!.scrollLeft =
        (trackLeft.value / thumbSize.value) * scrollSize.value;
    } else {
      elementScrollable.value!.scrollTop =
        (trackTop.value / thumbSize.value) * scrollSize.value;
    }
  }
}

/**
 * Xử lý sự kiện mouseup
 * @createdby ntdung 23.05.2023
 **/
function handleMouseup() {
  if (movingTrack.value) {
    movingTrack.value = false;
  }
}

/**
 * Xử lý sự kiện resize
 * @createdby ntdung 23.05.2023
 **/
function handleResize() {
  setSizeScrollbar();
  checkHaveScroll();
}

/**
 * Xử lý khi click vào main scrollbar
 * @createdby ntdung 23.05.2023
 **/
function mousedownMain(e: any) {
  if (elementScrollable.value) {
    let size = track.value?.getBoundingClientRect() as DOMRect;
    if (props.horizontal) {
      if (e.clientX < size.left) {
        var newLeft = trackLeft.value - trackSize.value;
        trackLeft.value = newLeft ? newLeft : 0;
      }
      if (e.clientX > size.right) {
        var newLeft = trackLeft.value + trackSize.value;
        trackLeft.value =
          newLeft <= thumbSize.value - trackSize.value
            ? newLeft
            : thumbSize.value - trackSize.value;
      }
      elementScrollable.value!.scrollLeft =
        (trackLeft.value / thumbSize.value) * scrollSize.value;
    } else {
      if (e.clientY < size.top) {
        var newTop = trackTop.value - trackSize.value;
        trackTop.value = newTop ? newTop : 0;
      }
      if (e.clientY > size.bottom) {
        var newTop = trackTop.value + trackSize.value;
        trackTop.value =
          newTop <= thumbSize.value - trackSize.value
            ? newTop
            : thumbSize.value - trackSize.value;
      }
      elementScrollable.value!.scrollTop =
        (trackTop.value / thumbSize.value) * scrollSize.value;
    }
  }
}

/**
 * Xử lý khi nhấn xuống track
 * @createdby ntdung 23.05.2023
 **/
function mousedownTrack(e: any) {
  if (track.value) {
    movingTrack.value = true;
    startY.value = e.clientY - parseInt(getComputedStyle(track.value).top);
    startX.value = e.clientX - parseInt(getComputedStyle(track.value).left);
  }
}

/**
 * Đặt kicks thước cho scrollbar
 * createdby ntdung5 23.05.2023
 */
function setSizeScrollbar() {
  if (elementScrollable.value) {
    if (props.horizontal) {
      clientSize.value = elementScrollable.value.clientWidth;
      scrollSize.value = elementScrollable.value.scrollWidth;
      thumbSize.value = wrapper.value?.getBoundingClientRect().width || 0;
    } else {
      clientSize.value = elementScrollable.value.clientHeight;
      scrollSize.value = elementScrollable.value.scrollHeight;
      thumbSize.value = wrapper.value?.getBoundingClientRect().height || 0;
    }
    trackSize.value = (clientSize.value / scrollSize.value) * thumbSize.value;
  }
}

/**
 * Kiểm tra xem có scroll không
 * createdby ntdung5 23.05.2023
 */
function checkHaveScroll() {
  if (elementScrollable.value) {
    let scrollDimension = 0;
    let dimension = 0;
    if (props.horizontal) {
      scrollDimension = elementScrollable.value.scrollWidth;
      dimension = elementScrollable.value.getBoundingClientRect().width;
    } else {
      scrollDimension = elementScrollable.value.scrollHeight;
      dimension = elementScrollable.value.getBoundingClientRect().height;
    }
    if (scrollDimension > dimension) {
      haveScroll.value = true;
    } else {
      haveScroll.value = false;
    }
  } else {
    haveScroll.value = false;
  }
}

/**
 * Style scrollbar
 * @createdby ntdung 23.05.2023
 **/
const styleScrollbar = computed(() => {
  if (props.horizontal) {
    return {
      height: '18px',
      width: '100%',
    };
  } else {
    return {
      width: '18px',
      height: '100%',
    };
  }
});

/**
 * Xóa vùng đang chọn
 * createdby ntdung5 13.06.2022
 */
function clearSelection() {
  if (window.getSelection()) {
    if (window.getSelection().empty) {
      window.getSelection().empty();
    } else if (window.getSelection().removeAllRanges) {
      window.getSelection().removeAllRanges();
    }
  }
  // else if (document.selection) {
  //   document.selection.empty();
  // }
}

/**
 * Kiểm tra đã ra ngoài wrapper
 * @createdby ntdung 23.05.2023
 **/
function outWrapper(e: any) {
  let offset = props.horizontal
    ? e.clientX - startX.value
    : e.clientY - startY.value;
  if (offset >= 0 && offset <= thumbSize.value - trackSize.value) {
    return {
      out: false,
    };
  }
  return {
    out: true,
  };
}
</script>

<style scoped lang="scss">
.virtual-scrollbar {
  display: flex;
  flex-direction: column;
  align-items: center;
  &__main {
    background-color: #eee;
    position: relative;
    height: 100%;
    width: 100%;
  }
  &__track {
    position: absolute;
    background-color: #ccc;
    cursor: pointer;
    top: 10px;
    left: 50%;
    &:hover {
      background-color: #aaa;
    }
    &:active {
      background-color: #888;
    }
  }
}
</style>
