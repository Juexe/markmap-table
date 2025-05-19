<template>
  <div class="app-container">
    <div class="editor-pane" :style="{ width: editorWidth + 'px' }">
      <div class="mode-switcher">
        <label>
          <input type="radio" v-model="mode" value="mindmap" /> 
          转换思维导图
        </label>
        <label>
          <input type="radio" v-model="mode" value="table" /> 
          转换表格
        </label>
      </div>
      <textarea 
        class="markdown-input" 
        v-model="value" 
        placeholder="Enter Markdown here..." 
      />
    </div>
    <div 
      class="resizer"
      @mousedown="startResize"
    ></div>
    <div class="content-pane">
      <mark-map v-if="mode === 'mindmap'" :value="value" />
      <mark-table v-else :value="value" />
    </div>
  </div>
</template>

<script>
import MarkMap from './components/MarkMap.vue';
import MarkTable from './components/MarkTable.vue';

const initValue = `# 项目计划

## 前端开发
- 完成用户界面设计
  - 登录页面
  - 主页面
- 实现数据交互

## 后端开发
- 设计数据库结构
- 实现API接口
  - 用户认证
  - 数据处理
  - 报表生成

## 测试
- 单元测试
- 集成测试
- 用户测试`;

export default {
  name: 'App',
  components: {
    MarkMap,
    MarkTable
  },
  data() {
    return {
      value: initValue,
      mode: 'mindmap',  // 默认显示思维导图
      editorWidth: 500 // 初始宽度
    };
  },
  methods: {
    startResize(e) {
      e.preventDefault();
      
      const startX = e.clientX;
      const startWidth = this.editorWidth;
      
      const doDrag = (e) => {
        const newWidth = startWidth + e.clientX - startX;
        // 限制最小和最大宽度
        this.editorWidth = Math.min(Math.max(200, newWidth), window.innerWidth - 200);
      };
      
      const stopDrag = () => {
        document.removeEventListener('mousemove', doDrag);
        document.removeEventListener('mouseup', stopDrag);
      };
      
      document.addEventListener('mousemove', doDrag);
      document.addEventListener('mouseup', stopDrag);
    }
  }
};
</script>

<style scoped>
.app-container {
  display: flex;
  height: 100vh;
  font-family: sans-serif;
  position: relative;
}

.editor-pane, .content-pane {
  display: flex;
  flex-direction: column;
  padding: 10px;
  box-sizing: border-box;
  height: 100%;
}

.editor-pane {
  border-right: none;
}

.mode-switcher {
  margin-bottom: 10px;
}

.mode-switcher label {
  margin-right: 20px;
  cursor: pointer;
}

.mode-switcher input[type="radio"] {
  margin-right: 5px;
}

.markdown-input {
  flex: 1;
  width: 100%;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 8px;
  box-sizing: border-box;
  font-family: monospace;
  font-size: 14px;
  resize: none;
}

.content-pane {
  flex: 1;
  overflow: auto;
}

.resizer {
  width: 4px;
  height: 100%;
  background-color: #eee;
  cursor: col-resize;
  transition: background-color 0.3s;
}

.resizer:hover,
.resizer:active {
  background-color: #999;
}
</style>