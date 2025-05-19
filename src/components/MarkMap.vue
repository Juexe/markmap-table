<template>
  <svg class="flex-1" ref="svgRef" />
</template>

<script>
import { ref, onMounted, watch } from 'vue';
import { Markmap } from 'markmap-view';
import { transformer } from '../markmap';

export default {
  name: 'MarkMap',
  props: {
    value: {
      type: String,
      required: true
    }
  },
  setup(props) {
    const svgRef = ref(null);
    let mm = null;

    const update = async () => {
      if (!props.value) return;
      
      try {
        const { root } = transformer.transform(props.value);
        if (mm) {
          await mm.setData(root);
          mm.fit();
        }
      } catch (error) {
        console.error('Error updating markmap:', error);
      }
    };

    onMounted(() => {
      mm = Markmap.create(svgRef.value);
      update();
    });

    watch(() => props.value, update);

    return {
      svgRef
    };
  }
};
</script>

<style scoped>
.flex-1 {
  width: 100%;
  height: 100%;
  min-height: 400px;
  background: #fff;
}
</style>
