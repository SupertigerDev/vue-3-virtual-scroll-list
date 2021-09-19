# vue-3-virtual-scroll-list
Ported [vue-virtual-scroll-list](https://github.com/tangbc/vue-virtual-scroll-list/tree/v1.4.7) v1.4.7 from [tangbc](https://github.com/tangbc).

Only tested with variable mode: 
```vue
<template>
  <div id="app">
    <VirtualList :size="50" :remain="20" :variable="true" >
      <div v-for="item of list" :key="item" :style="{ height: 15 + 'px' }">
        {{ item }}
      </div>
    </VirtualList>
  </div>
</template>


<script lang="ts">
import { defineComponent } from 'vue';
import VirtualList from '@/virtual-list.vue';

export default defineComponent({
  name: 'ServeDev',
  components: {
    VirtualList
  },
  data() {
    return {
      list: [] as number[]
    }
  },
  mounted() {
    for (let i = 0; i < 1000; i++) {
      this.list.push(i);
    }
  }
});
</script>

<style>
#app {
  display: flex;
  flex-direction: column;
  height: 300px;
  overflow-y: auto;
  overflow-x: hidden;
}
</style>
```