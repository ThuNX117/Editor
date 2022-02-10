<template>
  <div class="editor">
    <div class="editor-toolbar">
      <div class="style">
        <div class="italic">I</div>
        <div class="bold">B</div>
        <div class="underline">U</div>
      </div>
      <div class="font">
        <input v-model="fontSize" type="number" />
      </div>
      <div class="search">
        <input v-model="searchText" />
      </div>
    </div>

    <div class="editor-wrapper">
      <div class="list-search" v-show="isSearchTag">
        <ul>
          <li
            v-for="(result, indexResult) in searchResult"
            :key="indexResult"
            @click="addTag(result)"
          >
            {{ result }}
          </li>
        </ul>
      </div>
      <div
        class="editor-content-editable"
        :contenteditable="isEditable"
        id="editor"
      >
        <div><br /></div>
      </div>
    </div>
    <div>{{ status ? "writing" : "resting" }}</div>
  </div>
</template>
<script>
export default {
  name: "editor",
  data() {
    return {
      isEditable: true,
      searchText: null,
      fontSize: 16,
      searchReference: ["a", "b", "c", "ab"],
      status: false,
      tagSearchText: "",
      isSearchTag: false,
      tagTarget: {},
      tagList: [],
      trackSearch: null,
      currentNode: undefined,
      count: 0,
    };
  },
  computed: {
    searchResult() {
      const vm = this;
      const text = vm.searchText;
      let rs = vm.searchReference.filter((item) => item.search(text) != -1);
      return rs;
    },
  },
  created() {
    console.clear();
  },
  mounted() {
    const vm = this;
    this.$nextTick(() => {
      console.log("render, listen to change");
      const editor = document.getElementById("editor");
      editor.addEventListener("keyup", vm.listenChange);
    });
  },
  methods: {
    // node_walk: walk the element tree, stop when func(node) returns false
    node_walk(node, func) {
      const vm = this;
      var result = func(node);
      for (
        node = node.firstChild;
        result !== false && node;
        node = node.nextSibling
      )
        result = vm.node_walk(node, func);
      return result;
    },
    // getCaretPosition: return [start, end] as offsets to elem.textContent that
    //   correspond to the selected portion of text
    //   (if start == end, caret is at given position and no text is selected)
    getCaretPosition(elem) {
      const vm = this;
      var sel = window.getSelection();
      var cum_length = [0, 0];

      if (sel.anchorNode == elem)
        cum_length = [sel.anchorOffset, sel.extentOffset];
      else {
        var nodes_to_find = [sel.anchorNode, sel.extentNode];
        if (!elem.contains(sel.anchorNode) || !elem.contains(sel.extentNode))
          return undefined;
        else {
          var found = [0, 0];
          var i;
          vm.node_walk(elem, function (node) {
            for (i = 0; i < 2; i++) {
              if (node == nodes_to_find[i]) {
                found[i] = true;
                if (found[i == 0 ? 1 : 0]) return false; // all done
              }
            }

            if (node.textContent && !node.firstChild) {
              for (i = 0; i < 2; i++) {
                if (!found[i]) cum_length[i] += node.textContent.length;
              }
            }
          });
          cum_length[0] += sel.anchorOffset;
          cum_length[1] += sel.extentOffset;
        }
      }
      if (cum_length[0] <= cum_length[1]) return cum_length;
      return [cum_length[1], cum_length[0]];
    },
    // Move caret to a specific point in a DOM element
    SetCaretPosition(el, pos) {
      const vm = this;
      // Loop through all child nodes
      for (var node of el.childNodes) {
        if (node.nodeType == 3) {
          // we have a text node
          if (node.length >= pos) {
            // finally add our range
            var range = document.createRange(),
              sel = window.getSelection();
            range.setStart(node, pos);
            range.collapse(true);
            sel.removeAllRanges();
            sel.addRange(range);
            return -1; // we are done
          } else {
            pos -= node.length;
          }
        } else {
          pos = vm.SetCaretPosition(node, pos);
          if (pos == -1) {
            return -1; // no need to finish the for loop
          }
        }
      }
      return pos; // needed because of recursion stuff
    },
    activeStatus() {
      const vm = this;
      vm.status = true;
      setTimeout(() => {
        vm.status = false;
      }, 500);
    },
    insertTag() {},
    removeTag() {},
    activeSearch() {},
    listenChange(e) {
      console.log(e);
      this.activeStatus();
      const selection = window.getSelection();
      console.log(selection);

      this.trackSearch = selection.anchorNode.textContent;
    },
    onBold: function () {
      console.log("onBold");
    },
    onItalic: function () {
      console.log("onItalic");
    },
    onUnderline: function () {
      console.log("onUnderline");
    },
    sliceTextSearch() {
      const tar = window.getSelection();
      const el = tar.anchorNode;
      console.log(tar);
      tar.nodeType = 1;
      let pos = [];
      for (let index = 0; index < el.textContent.length; index++) {
        if (el.textContent[index] === "@") {
          pos.push(index);
        }
      }

      const start = pos[0] + 1;
      const end = pos[1] ? pos[1] - 1 : el.textContent.length;
      const textSearch = el.textContent.slice(start, end);
      this.searchText = textSearch;
      this.currentNode = el.parentNode;
      if (el.parentNode.at.contains("editor-content-editable")) {
        el.innerHTML = `<div>${el.innerHTML}</div>`;
      }
    },
    addTag(e) {
      const data = {
        name: e,
        id: "123abc-456def-789pgh",
        title: "this is a hastag",
      };
      const tag = this.createTag(data);
      console.log(tag);
      let hasTag = document.createElement("div");
      hasTag.appendChild(tag);
      console.log(hasTag);
      var breakSpace = document.createElement("div");
      breakSpace.textContent = " ";
      this.currentNode.appendChild(hasTag);
      this.currentNode.appendChild(breakSpace);
    },
    createTag(data) {
      const { name, id, title } = data;
      var tag = document.createElement("div");
      tag.setAttribute("id", id + "_" + this.count);
      tag.setAttribute("data-id", id);
      tag.setAttribute("title", title);
      tag.setAttribute("contenteditable", false);
      tag.textContent = "@" + name;

      tag.classList.add("mark");
      this.count++;
      return tag;
    },
  },
  watch: {
    trackSearch: {
      handler(e) {
        console.log(e);
        if (e.search("@") != -1) {
          this.isSearchTag = true;
          this.sliceTextSearch();
        } else {
          this.isSearchTag = false;
        }
      },
      deep: true,
    },
    isSearchTag() {},
    tagSearchText(val) {
      console.log(val);
    },
  },
};
</script>

<style lang="scss">
.mark {
  background: red;
  padding: 3px;
  border-radius: 3px;
  background: skyblue;
}
</style>
<style lang="scss" scoped>
* {
  box-sizing: border-box;
  font-size: 16px;
}
.editor-toolbar {
  display: flex;
  align-items: center;

  .style {
    display: flex;
    div {
      display: flex;
      height: 50px;
      width: 50px;
      align-items: center;
      justify-content: center;
      border: 1px solid #ccc;
      margin: 5px;
    }
  }

  .search {
    input {
      height: 50px;
      width: 200px;
      margin: 5px;
      border: 1px solid #ccc;
      padding: 15px;
    }
  }
  .font {
    input {
      height: 50px;
      width: 100px;
      margin: 5px;
      border: 1px solid #ccc;
      padding: 15px;
    }
  }
}
.editor-wrapper {
  border: 1px solid #ccc;
  padding: 10px;
  border-radius: 10px;
  min-height: 500px;
  position: relative;

  .editor-content-editable {
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 10px;
    min-height: 100%;
    display: flex;
    text-align: left;
    align-items: flex-start;
    justify-content: flex-start;
    flex-direction: column;
    div {
      min-width: 1px;
    }
  }
  .list-search {
    position: absolute;
    border: 1px solid #dcdcdc;
    width: 300px;
    height: auto;
    bottom: calc(100% - 10px);
    background: white;
    box-shadow: 0 0px 5px #ccc;
    ul {
      padding: 0;
      margin: 5px 0;
      overflow: auto;
    }
    li {
      padding: 5px 15px;
      list-style-type: none;
      height: 30px;
      display: flex;
      text-align: left;
      align-items: flex-start;
      justify-content: flex-start;
      &:hover {
        background: #d9d9d9;
      }
    }
  }
}
.italic {
  font-style: italic;
}

.bold {
  font-weight: bold;
}

.underline {
  text-decoration: underline;
}
</style>
