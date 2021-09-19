<script lang="ts">
import {
  defineComponent,
  getCurrentInstance,
  h,
  nextTick,
  onBeforeUpdate,
  ref,
  VNode,
  computed
} from "vue";

export default defineComponent({
  props: {
    size: {
      type: Number,
      required: true
    },
    remain: {
      type: Number,
      required: true
    },
    rtag: {
      type: String,
      default: "div"
    },
    wtag: {
      type: String,
      default: "div"
    },
    wclass: {
      type: String,
      default: ""
    },
    wstyle: {
      type: Object,
      default: () => ({})
    },
    start: {
      type: Number,
      default: 0
    },
    offset: {
      type: Number,
      default: 0
    },
    variable: {
      type: [Function, Boolean],
      default: false
    },
    bench: {
      type: Number,
      default: 0 // also equal to remain
    },
    totop: {
      type: [Function, Boolean],
      default: false
    },
    tobottom: {
      type: [Function, Boolean],
      default: false
    },
    onscroll: {
      type: [Function, Boolean], // Boolean disables default behavior
      default: false
    },
    istable: {
      type: Boolean,
      default: false
    },
    item: {
      type: [Function, Object],
      default: null
    },
    itemcount: {
      type: Number,
      default: 0
    },
    itemprops: {
      type: Function,
      /* istanbul ignore next */
      // eslint-disable-next-line @typescript-eslint/no-empty-function
      default: () => {}
    }
  },
  setup(props, context) {
    const instance = getCurrentInstance();

    const vsl = ref<any>(null);
    let changeProp = "";
    const delta = Object.create(null);
    const start = props.start >= props.remain ? props.start : 0;
    const keeps = props.remain + (props.bench || props.remain);

    delta.direction = ""; // current scroll direction, D: down, U: up.
    delta.scrollTop = 0; // current scroll top, use to direction.
    delta.start = start; // start index.
    delta.end = start + keeps - 1; // end index.
    delta.keeps = keeps; // nums keeping in real dom.
    delta.total = 0; // all items count, update in filter.
    delta.offsetAll = 0; // cache all the scrollable offset.
    delta.paddingTop = 0; // container wrapper real padding-top.
    delta.paddingBottom = 0; // container wrapper real padding-bottom.
    delta.varCache = {}; // object to cache variable index height and scroll offset.
    delta.varAverSize = 0; // average/estimate item height before variable be calculated.
    delta.varLastCalcIndex = 0; // last calculated variable height/offset index, always increase.

    // public method, force render ui list if needed.
    // call this before the next rerender to get better performance.
    const forceRender = () => {
      window.requestAnimationFrame(() => {
        instance?.proxy?.$forceUpdate();
      });
    };

    const defaultSlot = computed(() => {
      const slots = (context.slots as any)?.default?.() as VNode[];
      return slots.map(s => s.children).flat() as VNode[];
    });

    // return the right zone info based on `start/index`.
    const getZone = (index: any) => {
      let start, end;

      index = parseInt(index, 10);
      index = Math.max(0, index);

      const lastStart = delta.total - delta.keeps;
      const isLast =
        (index <= delta.total && index >= lastStart) || index > delta.total;

      if (isLast) {
        start = Math.max(0, lastStart);
      } else {
        start = index;
      }
      end = start + delta.keeps - 1;
      if (delta.total && end > delta.total) {
        end = delta.total - 1;
      }

      return {
        end,
        start,
        isLast
      };
    };
    // return a variable size (height) from given index.
    const getVarSize = (index: number, nocache?: boolean) => {
      const cache = delta.varCache[index];
      if (!nocache && cache) {
        return cache.size;
      }

      if (typeof props.variable === "function") {
        return props.variable(index) || 0;
      } else {
        const slot = defaultSlot.value[index];

        const style = slot && slot.props && slot.props.style;
        if (style && style.height) {
          const shm = style.height.match(/^(.*)px$/);
          return (shm && +shm[1]) || 0;
        }
      }
      return 0;
    };

    // return a variable scroll offset from given index.
    const getVarOffset = (index: number, nocache?: boolean) => {
      const cache = delta.varCache[index];

      if (!nocache && cache) {
        return cache.offset;
      }

      let offset = 0;
      for (let i = 0; i < index; i++) {
        const size = getVarSize(i, nocache);
        delta.varCache[i] = {
          size: size,
          offset: offset
        };
        offset += size;
      }

      delta.varLastCalcIndex = Math.max(delta.varLastCalcIndex, index - 1);
      delta.varLastCalcIndex = Math.min(
        delta.varLastCalcIndex,
        delta.total - 1
      );

      return offset;
    };
    // set manual scroll top.
    const setScrollTop = (scrollTop: number) => {
      if (vsl) {
        (vsl.$el || vsl).scrollTop = scrollTop;
      }
    };
    onBeforeUpdate(() => {
      delta.keeps = props.remain + (props.bench || props.remain);

      const calcstart = changeProp === "start" ? props.start : delta.start;
      const zone = getZone(calcstart);

      // if start, size or offset change, update scroll position.
      if (changeProp && ["start", "size", "offset"].includes(changeProp)) {
        const scrollTop =
          changeProp === "offset"
            ? props.offset
            : props.variable
            ? getVarOffset(zone.isLast ? delta.total : zone.start)
            : zone.isLast && delta.total - calcstart <= props.remain
            ? delta.total * props.size
            : calcstart * props.size;

        nextTick(setScrollTop.bind(this, scrollTop));
      }

      // if points out difference, force update once again.
      if (changeProp || delta.end !== zone.end || calcstart !== zone.start) {
        changeProp = "";
        delta.end = zone.end;
        delta.start = zone.start;
        forceRender();
      }
    });

    // return the variable paddingBottom based on the current zone.
    const getVarPaddingBottom = () => {
      const last = delta.total - 1;
      if (
        delta.total - delta.end <= delta.keeps ||
        delta.varLastCalcIndex === last
      ) {
        return getVarOffset(last) - getVarOffset(delta.end);
      } else {
        // if unreached last zone or uncalculated real behind offset
        // return the estimate paddingBottom and avoid too much calculations.
        return (delta.total - delta.end) * (delta.varAverSize || props.size);
      }
    };

    // return the variable paddingTop based on current zone.
    // @todo: if set a large `start` before variable was calculated,
    // here will also case too much offset calculate when list is very large,
    // consider use estimate paddingTop in this case just like `getVarPaddingBottom`.
    const getVarPaddingTop = () => {
      return getVarOffset(delta.start);
    };

    // return the variable all heights use to judge reach bottom.
    const getVarAllHeight = () => {
      if (
        delta.total - delta.end <= delta.keeps ||
        delta.varLastCalcIndex === delta.total - 1
      ) {
        return getVarOffset(delta.total);
      } else {
        return (
          getVarOffset(delta.start) +
          (delta.total - delta.end) * (delta.varAverSize || props.size)
        );
      }
    };

    const filter = () => {
      const slots = defaultSlot.value || [];

      if (!slots.length) {
        delta.start = 0;
      }
      delta.total = slots.length;

      let paddingTop, paddingBottom, allHeight;
      const hasPadding = delta.total > delta.keeps;

      if (props.variable) {
        allHeight = getVarAllHeight();
        paddingTop = hasPadding ? getVarPaddingTop() : 0;
        paddingBottom = hasPadding ? getVarPaddingBottom() : 0;
      } else {
        allHeight = props.size * delta.total;
        paddingTop = props.size * (hasPadding ? delta.start : 0);
        paddingBottom =
          props.size * (hasPadding ? delta.total - delta.keeps : 0) -
          paddingTop;
      }

      if (paddingBottom < props.size) {
        paddingBottom = 0;
      }

      delta.paddingTop = paddingTop;
      delta.paddingBottom = paddingBottom;
      delta.offsetAll = allHeight - props.size * props.remain;

      const renders: VNode[] = [];
      for (
        let i = delta.start;
        i < delta.total && i <= Math.ceil(delta.end);
        i++
      ) {
        let slot = slots[i];
        renders.push(slot);
      }

      return renders;
    };
    // return the scroll of passed items count in variable.
    const getVarOvers = (offset: number) => {
      let low = 0;
      let middle = 0;
      let middleOffset = 0;
      let high = delta.total;

      while (low <= high) {
        middle = low + Math.floor((high - low) / 2);
        middleOffset = getVarOffset(middle);

        // calculate the average variable height at first binary search.
        if (!delta.varAverSize) {
          delta.varAverSize = Math.floor(middleOffset / middle);
        }

        if (middleOffset === offset) {
          return middle;
        } else if (middleOffset < offset) {
          low = middle + 1;
        } else if (middleOffset > offset) {
          high = middle - 1;
        }
      }

      return low > 0 ? --low : 0;
    };

    // update render zone by scroll offset.
    const updateZone = (offset: number) => {
      let overs = props.variable
        ? getVarOvers(offset)
        : Math.floor(offset / props.size);

      // if scroll up, we'd better decrease it's numbers.
      if (delta.direction === "U") {
        overs = overs - props.remain + 1;
      }

      const zone = getZone(overs);
      const bench = props.bench || props.remain;

      // for better performance, if scroll passes items within the bench, do not update.
      // and if it's close to the last item, render next zone immediately.
      const shouldRenderNextZone = Math.abs(overs - delta.start - bench) === 1;
      if (
        !shouldRenderNextZone &&
        overs - delta.start <= bench &&
        !zone.isLast &&
        overs > delta.start
      ) {
        return;
      }

      // make sure forceRender calls as less as possible.
      if (
        shouldRenderNextZone ||
        zone.start !== delta.start ||
        zone.end !== delta.end
      ) {
        delta.end = zone.end;
        delta.start = zone.start;
        forceRender();
      }
    };

    const onScroll = () => {
      let offset;

      offset = vsl.value.scrollTop || 0;

      delta.direction = offset > delta.scrollTop ? "D" : "U";
      delta.scrollTop = offset;

      if (delta.total > delta.keeps) {
        updateZone(offset);
      } else {
        delta.end = delta.total - 1;
      }

      // const offsetAll = delta.offsetAll;
      // if (this.onscroll) {
      //   const param = Object.create(null);
      //   param.offset = offset;
      //   param.offsetAll = offsetAll;
      //   param.start = delta.start;
      //   param.end = delta.end;
      //   // props.onscroll(event, param);
      // }

      // if (!offset && delta.total) {
      //   this.fireEvent("totop");
      // }

      // if (offset >= offsetAll) {
      //   this.fireEvent("tobottom");
      // }
    };

    return () => {
      let list = filter();
      const { paddingTop, paddingBottom } = delta;

      const istable = props.istable;
      const wtag = istable ? "div" : props.wtag;
      const rtag = istable ? "div" : props.rtag;
      if (istable) {
        list = [h("table", [h("tbody", list)])];
      }
      const renderList = h(
        wtag,
        {
          style: Object.assign(
            {
              display: "block",
              "padding-top": paddingTop + "px",
              "padding-bottom": paddingBottom + "px"
            },
            props.wstyle
          ),
          class: props.wclass,
          role: "group"
        },
        list
      );

      return h(
        rtag,
        {
          ref: vsl,
          style: {
            display: "block",
            "overflow-y": props.size >= props.remain ? "auto" : "initial",
            height: props.size * props.remain + "px"
          },
          onScroll: onScroll
        },
        [renderList]
      );
    };
  }
});
</script>
