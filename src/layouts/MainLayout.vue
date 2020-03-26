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
        <q-toolbar-title>
          <folder-breadcrumbs
            :absolutePath="selectedFolder"
            @selected="onSelectedFolder"
          />
        </q-toolbar-title>
        <q-btn
          flat
          dense
          round
          @click="toggleListType"
          :icon="listType === 'grid' ? 'format_list_bulleted' : 'border_all'"
        >
          <q-tooltip>toggle between grid and list mode</q-tooltip>
        </q-btn>
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
      <folder-contents
        :contents="contents"
        :listType="listType"
        @click="onClicked"
        @dblClick="onDblClicked"
      />
    </q-page-container>
  </q-layout>
</template>

<script>
import FolderTree from "components/FolderTree";
import FolderContents from "components/FolderContents";
import FolderBreadcrumbs from "components/FolderBreadcrumbs";

import walkFolders from "../utils/walkFolders";
const path = require("path");
const fs = require("fs");
const mime = require("mime-types");
const { ipcRenderer } = require("electron");
// file watcher
const chokidar = require("chokidar");

export default {
  name: "MainLayout",

  components: {
    FolderTree,
    FolderContents,
    FolderBreadcrumbs
  },

  data() {
    return {
      leftDrawerOpen: true,
      selectedFolder: null,
      drive: process.platform === "win32" ? "C:" : "",
      rootDir: [],
      contents: [],
      listType: "grid" // ['grid', 'list']
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
    },
    // if there's new file or folder added or removed, show it.
    folderWatcherHandler(newFolder, oldFolder) {
      if (oldFolder && this.watcher) {
        this.watcher.close();
      }
      if (newFolder) {
        // let backend know to statically serve files from this folder
        ipcRenderer.send("folder", newFolder);

        this.watcher = chokidar.watch(newFolder, {
          depth: 0,
          ignorePermissionErrors: true
        });
        if (this.watcher) {
          this.watcher.on("ready", () => {
            // initial scan done
            // watch for additions
            this.watcher.on("raw", (event, path, details) => {
              this.$root.$emit("rescan-current-folder");
            });
          });
          this.watcher.on("error", error => {
            // initial scan done
            console.error(error);
          });
        }
      }
    },
    clearAllContentItems() {
      this.contents.splice(0, this.contents.length);
    },
    rescanCurrentFolder() {
      this.clearAllContentItems();
      this.contents.push(...this.getFolderContents(this.selectedFolder));
    },
    getFolderContents(folder) {
      let contents = [];

      // check incoming arg
      if (!folder || typeof folder !== "string") {
        return contents;
      }

      for (const fileInfo of walkFolders(folder, 0)) {
        // all files and folders
        if ("error" in fileInfo) {
          console.error(`Error: ${fileInfo.rootDir} - ${fileInfo.error}`);
          continue;
        }
        const node = this.createNode(fileInfo);
        contents.push(node);
      }

      return contents;
    },
    toggleListType() {
      this.listType = this.listType === "grid" ? "list" : "grid";
    },
    onClicked(node) {
      // on single-clicks we don't do anything here
      // if we wanted to drill-down into folders, we
      // can call this.onDblClicked function.
    },

    onDblClicked(node) {
      // This causes a drill-down if it's a folder
      if (node.data.isDir) {
        this.setSelectedFolder(node.nodeKey);
      } else {
        this.onFileSelected(node);
      }
    },
    onFileSelected(node) {
      alert(`selected file: ${node.label}`);
    }
  },
  watch: {
    selectedFolder: {
      handler(newFolder, oldFolder) {
        // if user deselect a folder, set newFolder as root folder
        if (!newFolder) {
          newFolder = this.drive + path.sep;
        }

        // tell backend to serve files from this folder
        ipcRenderer.send("folder", newFolder);

        // folder watcher handler
        this.folderWatcherHandler(newFolder, oldFolder);

        this.clearAllContentItems();
        this.contents.push(...this.getFolderContents(newFolder));
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
