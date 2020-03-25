<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-btn
          flat
          dense
          round
          icon="menu"
          aria-label="Menu"
          @click="leftDrawerOpen = !leftDrawerOpen"
        />
      </q-toolbar>
    </q-header>

    <q-drawer
      v-model="leftDrawerOpen"
      show-if-above
      bordered
      content-class="bg-grey-1"
    >
      <folder-tree
        :rootDir="rootDir"
        :folder.sync="selectedFolder"
        :lazyLoad="onLazyLoad"
        @selected="onSelectedFolder"
      />
    </q-drawer>

    <q-page-container>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<script>
import FolderTree from "components/FolderTree";

import walkFolders from "../utils/walkFolders";
const path = require("path");
const fs = require("fs");
const mime = require("mime-types");

export default {
  name: "MainLayout",

  components: {
    FolderTree
  },

  data() {
    return {
      leftDrawerOpen: true,
      rootDir: [],
      selectedFolder: null,
      drive: process.platform === "win32" ? "C:" : ""
    };
  },
  methods: {
    getFolders(absolutePath) {
      let folders = [];

      if (!absolutePath || typeof absolutePath !== "string") {
        return folders;
      }

      for (const fileInfo of walkFolders(absolutePath, 0)) {
        // we only want folders
        if (!fileInfo.isDir) {
          continue;
        }

        // create new node
        const node = this.createNode(fileInfo);
        folders.push(node);
      }

      return folders;
    },
    onSelectedFolder(absolutePath) {
      this.setSelectedFolder(absolutePath);
    },
    setSelectedFolder(folder) {
      this.selectedFolder = folder;
      // handle windows drive
      if (process.platform === "win32") {
        if (
          this.selectedFolder.charAt(this.selectedFolder.length - 1) === ":"
        ) {
          this.selectedFolder += path.sep;
        }
      }
    },
    createNode(fileInfo) {
      let nodeKey = fileInfo.rootDir;
      if (nodeKey.charAt(nodeKey.length - 1) !== path.sep) {
        nodeKey += path.sep;
      }
      if (fileInfo.fileName === path.sep) {
        fileInfo.fileName = nodeKey;
      } else {
        nodeKey += fileInfo.fileName;
      }
      // get file mime type
      const mimeType = mime.lookup(nodeKey);
      // create object
      return {
        label: fileInfo.fileName,
        nodeKey: nodeKey,
        expandable: fileInfo.isDir,
        tickable: true,
        lazy: true,
        children: [],
        data: {
          rootDir: fileInfo.rootDir,
          isDir: fileInfo.isDir,
          mimeType: mimeType,
          stat: fileInfo.stat
        }
      };
    },
    // called by folderTree component
    onLazyLoad({ node, key, done, fail }) {
      this.loadChildren(node, key);
      done();
    },
    loadChildren(node, key) {
      try {
        if (node.children.length) {
          node.children.splice(0, node.children.length);
        }

        for (const fileInfo of walkFolders(key, 0)) {
          // only load folders
          if (!fileInfo.isDir) {
            continue;
          }
          // create new node
          const n = this.createNode(fileInfo);
          node.children.push(n);
        }
        return true;
      } catch (err) {
        // usually access error
        console.error(`Error: ${err}`);
      }
    }
  },
  watch: {
    selectedFolder: {
      handler(newFolder, oldFolder) {
        // if user deselect a folder, set newFolderas root folder
        if (!newFolder) {
          newFolder = this.drive + path.sep;
        }
      }
    }
  },
  created() {
    if (process.platform === "win32") {
    } else {
      // set and get root folder's folders
      this.setSelectedFolder(this.drive + path.sep);
      this.rootDir.push(...this.getFolders(this.selectedFolder));
    }
  }
};
</script>
