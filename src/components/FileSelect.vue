<template>
  <input type="file" @change="onChange" />
</template>
<script>
export default {
  props: {
    change: Function,
  },
  methods: {
    onChange: function (event) {
      console.log("change");
      this.change(event);
    },
  },
};
</script>