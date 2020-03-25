<template>
  <div>
    Selected: {{ folder }}
    <q-tree
      ref="folders"
      @lazy-load="lazyLoad"
      :nodes="rootDir"
      node-key="nodeKey"
      :selected.sync="selected"
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
  },
  mounted() {
    this.selected = this.folder;
  },
  watch: {
    /// this comes from the parent
    folder: {
      handler() {
        this.selected = this.folder;
      }
    },
    selected: {
      handler() {
        this.$emit("selected", this.selected);
      }
    }
  }
};
</script>
