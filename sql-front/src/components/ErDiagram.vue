<!-- ER图显示组件 -->
<template>
  <el-card class="diagram-output" :body-style="{ padding: '0' }">
    <template #header>
      <div class="card-header">
        <div class="header-left">
          <i class="el-icon-connection"></i>
          <span>ER 图预览</span>
        </div>
        <el-button type="success" @click="openExportDialog" :icon="Download">
          导出图片
        </el-button>
      </div>
    </template>
    <div class="diagram-wrapper">
      <div ref="diagramDiv" class="diagram-container"></div>
    </div>
    
    <!-- 添加 ExportDialog 组件 -->
    <ExportDialog 
      v-model="isExportDialogVisible" 
      @confirm="handleExportConfirm"
    />
  </el-card>
</template>

<script>
import { ref, onMounted, nextTick } from 'vue'
// import { Download } from '@element-plus/icons-vue'
import * as go from 'gojs'
import ExportDialog from './ExportDialog.vue'  // 导入 ExportDialog 组件

export default {
  name: 'ErDiagram',
  components: {
    ExportDialog  // 注册组件
  },
  emits: ['export', 'open-export-dialog'],
  setup(props, { emit }) {
    const diagramDiv = ref(null)
    let myDiagram = null
    const isExportDialogVisible = ref(false)  // 添加对话框显示状态

    onMounted(async () => {
      await nextTick()
      initDiagram()
    })

    const initDiagram = () => {
      if (!diagramDiv.value) {
        console.error('图表容器未找到')
        return
      }

      const $ = go.GraphObject.make

      myDiagram = $(go.Diagram, diagramDiv.value, {
        initialContentAlignment: go.Spot.Center,
        "undoManager.isEnabled": true,
        layout: $(go.ForceDirectedLayout, {
          defaultSpringLength: 50,
          defaultElectricalCharge: 100
        }),
        "toolManager.mouseWheelBehavior": go.ToolManager.WheelZoom,
        "draggingTool.isEnabled": true,
        allowSelect: true,
        allowHorizontalScroll: true,
        allowVerticalScroll: true,
        padding: 40,
        "minScale": 0.25,
        "maxScale": 2,
        "animationManager.isEnabled": true,
        "grid.visible": true,
        "grid.gridCellSize": new go.Size(10, 10),
        "toolManager.dragSelectingTool": $(go.DragSelectingTool, {
          isEnabled: true,
          delay: 50,
          box: $(go.Part,
            { layerName: "Tool" },
            $(go.Shape,
              { strokeWidth: 2, stroke: "blue", fill: "rgba(0,0,255,0.1)" })
          )
        }),
        "toolManager.clickSelectingTool": $(go.ClickSelectingTool, {
          isEnabled: true
        })
      })

      // 启用多选工具
      myDiagram.toolManager.dragSelectingTool = new go.DragSelectingTool()
      myDiagram.toolManager.clickSelectingTool = new go.ClickSelectingTool()

      // 设置多选工具的选项
      myDiagram.toolManager.dragSelectingTool.box = $(go.Part,
        { layerName: "Tool" },
        $(go.Shape,
          { strokeWidth: 2, stroke: "blue", fill: "transparent" })
      )

      // 定义实体节点模板（矩形）
      myDiagram.nodeTemplateMap.add("Entity",
        $(go.Node, "Auto",
          {
            selectionAdornmentTemplate:
              $(go.Adornment, "Auto",
                $(go.Shape, { fill: null, stroke: "blue", strokeWidth: 2 }),
                $(go.Placeholder)
              )
          },
          $(go.Shape, "Rectangle",
            { fill: "white", stroke: "#1890ff", strokeWidth: 2, width: 120, height: 60 }),
          $(go.TextBlock,
            { 
              margin: 8, 
              font: "bold 16px 微软雅黑", 
              stroke: "#1890ff",
              editable: true
            },
            new go.Binding("text", "name").makeTwoWay())
        ))

      // 定义属性节点模板（椭圆形）
      myDiagram.nodeTemplateMap.add("Attribute",
        $(go.Node, "Auto",
          {
            minSize: new go.Size(60, 40),
            selectionAdornmentTemplate:
              $(go.Adornment, "Auto",
                $(go.Shape, { fill: null, stroke: "blue", strokeWidth: 2 }),
                $(go.Placeholder)
              )
          },
          $(go.Shape, "Ellipse",
            { 
              fill: "white", 
              stroke: "#1890ff", 
              strokeWidth: 2,
              margin: 4
            }),
          $(go.TextBlock,
            { 
              margin: 8, 
              font: "14px 微软雅黑", 
              stroke: "#333",
              editable: true,
              _class: new go.Binding("_class", "isPrimaryKey", pk => pk ? "primary-key" : "")
            },
            new go.Binding("text", "name").makeTwoWay())
        ))

      // 定义关系节点模板（菱形）
      myDiagram.nodeTemplateMap.add("Relationship",
        $(go.Node, "Auto",
          {
            selectionAdornmentTemplate:
              $(go.Adornment, "Auto",
                $(go.Shape, { fill: null, stroke: "blue", strokeWidth: 2 }),
                $(go.Placeholder)
              )
          },
          $(go.Shape, "Diamond",
            { fill: "white", stroke: "#1890ff", strokeWidth: 2, width: 60, height: 60 }),
          $(go.TextBlock,
            { 
              margin: 8, 
              font: "14px 微软雅黑", 
              stroke: "#333",
              editable: true
            },
            new go.Binding("text", "name").makeTwoWay())
        ))

      // 定义连接线模板
      myDiagram.linkTemplate =
        $(go.Link,
          { 
            routing: go.Link.Normal, 
            corner: 5,
            curve: go.Link.JumpOver,
            selectionAdornmentTemplate:
              $(go.Adornment,
                $(go.Shape,
                  { isPanelMain: true, stroke: "blue", strokeWidth: 2 })
              )
          },
          $(go.Shape,
            { 
              strokeWidth: 1.5, 
              stroke: "#1890ff"
            }),
          $(go.TextBlock,
              { 
                segmentOffset: new go.Point(0, -10), 
                font: "12px 微软雅黑",
                editable: true
              },
              new go.Binding("text", "text").makeTwoWay())
        )
    }

    const updateDiagram = (tables) => {
      if (!myDiagram) {
        console.error('图表未初始化')
        return
      }

      const nodeDataArray = []
      const linkDataArray = []

      // 首先收集所有的外键关系，使用 Map 来存储关系的方向
      const foreignKeyRelations = new Map()
      tables.forEach(table => {
        table.foreignKeys?.forEach(fk => {
          if (fk.referenceTable) {
            // 存储关系的方向：from -> to
            foreignKeyRelations.set(`${table.tableName}-${fk.referenceTable}`, true)
          }
        })
      })

      tables.forEach(table => {
        // 添加实体节点
        nodeDataArray.push({
          category: "Entity",
          key: table.tableName,
          name: removeQuotes(table.comment) || table.tableName
        })

        // 添加属性节点
        table.columns.forEach(col => {
          const attrKey = `${table.tableName}-${col.name}`
          nodeDataArray.push({
            category: "Attribute",
            key: attrKey,
            name: removeQuotes(col.comment) || col.name,
            isPrimaryKey: table.primaryKeys.includes(col.name)
          })
          // 添加属性与实体的连接
          linkDataArray.push({
            from: table.tableName,
            to: attrKey
          })
        })

        // 添加外键关系节点
        table.foreignKeys?.forEach(fk => {
          // 只有当外键关系存在且还未处理过时才添加关联
          const relationKey = `${table.tableName}-${fk.referenceTable}`
          if (fk.referenceTable && foreignKeyRelations.get(relationKey)) {
            const relationshipKey = `${relationKey}-relationship`
            nodeDataArray.push({
              category: "Relationship",
              key: relationshipKey,
              name: "关联"
            })
            // 添加关系与实体的连接（使用实线）
            linkDataArray.push({
              from: table.tableName,
              to: relationshipKey,
              text: "N"
            })
            linkDataArray.push({
              from: relationshipKey,
              to: fk.referenceTable,
              text: "1"
            })
            // 标记这个关系已经处理过
            foreignKeyRelations.delete(relationKey)
          }
        })
      })

      myDiagram.model = new go.GraphLinksModel(nodeDataArray, linkDataArray)
      console.log('节点数据:', nodeDataArray)
      console.log('连接数据:', linkDataArray)
      myDiagram.layoutDiagram(true)
      myDiagram.zoomToFit()
      myDiagram.contentAlignment = go.Spot.Center
    }

    // 去除字符串两端的单引号
    const removeQuotes = (str) => {
      if (!str) return str;
      return str.replace(/^'|'$/g, '');
    }

    const handleExport = () => {
      if (!myDiagram.nodes.count) {
        return
      }
      emit('export', myDiagram)
    }

    const openExportDialog = () => {
      isExportDialogVisible.value = true  // 显示对话框
    }

    const handleExportConfirm = (options) => {
      console.log('Export options:', options)
      if (!myDiagram.nodes.count) {
        return
      }

      // 设置背景颜色
      const originalBackground = myDiagram.div.style.background
      myDiagram.div.style.background = options.background === 'transparent' ? 'transparent' : 'white'

      // 创建临时画布
      const canvas = document.createElement('canvas')
      const context = canvas.getContext('2d')
      const bounds = myDiagram.documentBounds
      
      // 设置画布大小
      canvas.width = bounds.width
      canvas.height = bounds.height

      // 如果选择透明背景，确保上下文是透明的
      if (options.background === 'transparent') {
        context.clearRect(0, 0, canvas.width, canvas.height)
      } else {
        context.fillStyle = 'white'
        context.fillRect(0, 0, canvas.width, canvas.height)
      }

      // 将图表绘制到画布上
      const imageData = myDiagram.makeImageData({
        scale: 1,
        background: options.background === 'transparent' ? null : 'white',
        type: options.format === 'jpeg' ? 'image/jpeg' : 'image/png',
        maxSize: new go.Size(Infinity, Infinity)
      })

      // 生成时间戳文件名
      const now = new Date()
      const timestamp = `${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}`
      
      // 创建下载链接
      const link = document.createElement('a')
      link.download = `${timestamp}ER图.${options.format}`
      link.href = imageData
      
      // 触发下载
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)

      // 恢复原始背景
      myDiagram.div.style.background = originalBackground
    }

    return {
      diagramDiv,
      handleExport,
      updateDiagram,
      openExportDialog,
      isExportDialogVisible,  // 暴露对话框显示状态
      handleExportConfirm     // 暴露确认导出处理函数
    }
  }
}
</script>

<style lang="scss" scoped>
.diagram-output {
  height: calc(100vh - 140px);
  transition: all 0.3s ease;
  border: none;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  
  &:hover {
    box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.15);
  }

  :deep(.el-card__header) {
    padding: 16px 20px;
    border-bottom: 1px solid #ebeef5;
    background: #fff;
  }
  
  :deep(.el-card__body) {
    height: calc(100% - 60px);
    padding: 0;
    position: relative;
  }
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;

  .header-left {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 16px;
    font-weight: 600;
    color: #303133;

    i {
      font-size: 18px;
      color: #67c23a;
    }
  }
}

.diagram-wrapper {
  height: 100%;
  padding: 16px;
  background-color: #f8f9fa;
}

.diagram-container {
  width: 100%;
  height: 100%;
  background-color: white;
  position: relative;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  transition: all 0.3s ease;

  &:hover {
    border-color: #c0c4cc;
  }

  :deep(div[id^="diagram"]) {
    width: 100% !important;
    height: 100% !important;
    position: absolute !important;
    top: 0;
    left: 0;
    background: white;
  }

  :deep(canvas) {
    outline: none;
  }

  :deep(.primary-key) {
    text-decoration: underline !important;
  }
}
</style> 