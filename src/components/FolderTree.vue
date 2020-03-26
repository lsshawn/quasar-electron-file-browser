<template>
  <div>
    <div class="text-weight-bold">{{ isWin ? "Drives" : "File System" }}</div>
    <!-- ref for accessing this component with this.$refs.folders -->
    <q-tree
      ref="folders"
      :nodes="rootDir"
      label-key="label"
      node-key="nodeKey"
      accordion
      default-expand-all
      :selected.sync="selected"
      @lazy-load="lazyLoad"
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
  // Have QTree to open a node that wasn't even loaded yet with global event bus
  created: function() {
    this.$root.$on("expand-tree", this.expandTree);
  },
  // always remove global event bus when a component is unloaded
  beforeDestroy: function() {
    this.$root.$off("expand-tree", this.expandTree);
  },
  watch: {
    // this comes from the parent
    folder: {
      handler() {
        this.selected = this.folder;
        this.expandTree(this.folder);
      }
    },
    selected: {
      handler() {
        this.$emit("selected", this.selected);
      }
    }
  },
  methods: {
    // recursively expand tree
    expandTree(absolutePath) {
      // get parts for the path
      let parts = absolutePath.split(path.sep);
      let path2 = "";
      let lastNodeKey;

      // iterate through the path parts.
      // This code will get the node. If the node is not found,
      // it forces lazy-load by programmatically expanding
      // the parent node.
      for (let index = 0; index < parts.length; ++index) {
        if (parts[index].length === 0) {
          continue;
        }
        if (index === 0) {
          path2 += parts[index] + path.sep;
        } else {
          if (path2[path2.length - 1] !== path.sep) {
            path2 += path.sep;
          }
          path2 += parts[index];
        }
        if (index > -1) {
          if ("folders" in this.$refs) {
            const key = this.$refs.folders.getNodeByKey(path2);
            // if we get key, then this folder has already been loaded
            if (key) {
              lastNodeKey = key;
            }
            // handle folder not expanded
            if (!this.$refs.folders.isExpanded(lastNodeKey.nodeKey)) {
              this.$refs.folders.setExpanded(lastNodeKey.nodeKey, true);
              if (path2 === absolutePath) {
                this.selected = absolutePath;
              } else {
                this.$nextTick(() => {
                  this.$root.$emit("expand-tree", absolutePath);
                });
              }
            }
          }
        }
      }
    }
  }
};
</script>
