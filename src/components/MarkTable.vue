`<template>
  <div class="table-pane">
    <button v-if="tableHtml" @click="copyTableToClipboard" class="copy-button">
      复制表格到剪贴板
    </button>
    <div v-if="tableHtml" class="table-wrapper" ref="tableWrapper" v-html="tableHtml"></div>
    <div v-else class="placeholder-text">
      <p>Enter Markdown content to generate a table.</p>
      <p v-if="errorMessage" style="color: red;">{{ errorMessage }}</p>
    </div>
  </div>
</template>

<script>
import { transformer } from '../markmap';

// Helper for basic HTML sanitization
function sanitizeHTML(text) {
  if (text === null || typeof text === 'undefined') return '';
  return String(text)
  .replace(/&/g, '&') // Corrected: & should be &
  .replace(/</g, '<')  // Corrected: < should be <
  .replace(/>/g, '>')  // Corrected: > should be >
  .replace(/"/g, '"') // Corrected: " should be "
  .replace(/'/g, "'"); // Corrected: ' should be '
}

export default {
  name: 'MarkTable',
  props: {
    value: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      tableHtml: '',
      errorMessage: '',
    };
  },
  watch: {
    value: 'generateTable',
  },
  methods: {
    generateTable() {
      this.errorMessage = '';
      if (!this.value || !this.value.trim()) {
        this.tableHtml = '';
        return;
      }

      try {
        const { root } = transformer.transform(this.value);
        
        const flatRowItems = [];
        let maxDepthEncountered = 0;
        let totalTableRows = 0;

        const processNodeAndCalculateRowspan = (node, currentDepth) => {
          if (!node) return 0;

          let ownContentMakesARow = node.content && node.content.trim() !== "";
          let currentItemData = null;

          if (ownContentMakesARow) {
            currentItemData = { depth: currentDepth, content: node.content.trim(), rowspan: 1 };
            flatRowItems.push(currentItemData);
            maxDepthEncountered = Math.max(maxDepthEncountered, currentDepth);
          }

          let childrenRowCount = 0;
          if (node.children && node.children.length > 0) {
            for (const child of node.children) {
              childrenRowCount += processNodeAndCalculateRowspan(child, currentDepth + 1);
            }
          }

          if (currentItemData && childrenRowCount > 0) {
            currentItemData.rowspan = childrenRowCount;
          }
          
          if (ownContentMakesARow) {
            return Math.max(1, childrenRowCount);
          } else {
            return childrenRowCount;
          }
        };
        
        if (root) {
          if (root.children && root.children.length > 0) {
            for (const childNode of root.children) {
              totalTableRows += processNodeAndCalculateRowspan(childNode, 1);
            }
          } else if (root.content && root.content.trim() !== "") {
            totalTableRows += processNodeAndCalculateRowspan(root, 1);
          }
        }
        
        const numColumns = Math.max(1, maxDepthEncountered);

        if (flatRowItems.length === 0) {
          this.tableHtml = '<p>No structured content found to display in a table.</p>';
          return;
        }

        let tableRowsHtml = '';
        let itemPointer = 0; 
        const rowSpanState = {}; 
        for (let col = 1; col <= numColumns; col++) {
          rowSpanState[col] = 0;
        }

        for (let rowIndex = 0; rowIndex < totalTableRows; rowIndex++) {
          tableRowsHtml += '<tr>';
          for (let colIndex = 1; colIndex <= numColumns; colIndex++) {
            if (rowSpanState[colIndex] > 0) {
              rowSpanState[colIndex]--; 
            } else {
              if (itemPointer < flatRowItems.length && flatRowItems[itemPointer].depth === colIndex) {
                const currentItem = flatRowItems[itemPointer];
                const cellContent = sanitizeHTML(currentItem.content);
                const rowspanAttr = currentItem.rowspan > 1 ? ` rowspan="${currentItem.rowspan}"` : '';
                
                tableRowsHtml += `<td${rowspanAttr}>${cellContent}</td>`;
                
                if (currentItem.rowspan > 1) {
                  rowSpanState[colIndex] = currentItem.rowspan - 1;
                }
                itemPointer++;
              } else {
                tableRowsHtml += '<td></td>'; 
              }
            }
          }
          tableRowsHtml += '</tr>';
        }

        let headerCells = '';
        for (let i = 1; i <= numColumns; i++) {
          headerCells += `<th>标题 ${i}</th>`;
        }

        this.tableHtml = `
          <table class="generated-table">
            <thead>
              <tr>${headerCells}</tr>
            </thead>
            <tbody>
              ${tableRowsHtml}
            </tbody>
          </table>`;

      } catch (error) {
        console.error("Error transforming Markdown:", error);
        this.errorMessage = "Error processing Markdown. Check console for details.";
        this.tableHtml = "";
      }
    },
    async copyTableToClipboard() {
      if (!this.$refs.tableWrapper) {
        alert('没有表格可复制。');
        return;
      }
      const tableElement = this.$refs.tableWrapper.querySelector('table');
      if (!tableElement) {
        alert('找不到表格元素。');
        return;
      }

      const htmlContent = tableElement.outerHTML;

      try {
        await navigator.clipboard.write([
          new ClipboardItem({
            'text/html': new Blob([htmlContent], { type: 'text/html' })
          })
        ]);
        alert('表格已作为HTML复制到剪贴板！');
      } catch (err) {
        console.error('无法使用 Clipboard API 复制 HTML:', err);
        try {
          await navigator.clipboard.writeText(htmlContent);
          alert('表格已作为纯文本HTML复制到剪贴板 (可能丢失部分格式)。');
        } catch (fallbackErr) {
          console.error('无法复制表格:', fallbackErr);
          alert('复制失败。请尝试手动选择并复制表格内容，或检查浏览器权限。');
        }
      }
    },
  },
  mounted() {
    this.generateTable();
  },
};
</script>

<style scoped>
.table-pane {
  flex: 1;
  box-sizing: border-box;
  overflow: auto;
  display: flex;
  flex-direction: column;
}

.copy-button {
  margin-bottom: 10px;
  padding: 8px 15px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9em;
  align-self: flex-start;
}

.copy-button:hover {
  background-color: #0056b3;
}

.table-wrapper {
  width: 100%;
  flex-grow: 1;
  overflow: auto;
}

.placeholder-text {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #777;
  text-align: center;
}
</style>

<style>
.generated-table {
  border-collapse: collapse;
  width: 100%;
  table-layout: auto;
  font-size: 0.9em;
}

.generated-table th,
.generated-table td {
  border: 1px solid #ccc;
  padding: 8px;
  text-align: left;
  vertical-align: top;
  word-wrap: break-word;
}

.generated-table th {
  background-color: #f0f0f0;
  font-weight: bold;
}
</style>`
