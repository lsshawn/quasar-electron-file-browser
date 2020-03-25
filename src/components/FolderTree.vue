<template>
  <div>
    <q-tree
      ref="folders"
      @lazy-load="lazyLoad"
      :nodes="rootDir"
      node-key="nodeKey"
    />
  </div>
</template>

<script>
const path = require("path");

export default {
  props: {
    rootDir: {
      type: Array,
      required: true
    },
    folder: {
      type: String,
      required: false
    },
    lazyLoad: {
      type: Function,
      required: true
    }
  },
  data() {
    return {
      selected: null,
      isWin: process.platform === "win32"
    };
  }
};
</script>
