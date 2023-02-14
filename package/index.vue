<template>
  <div style="padding: 0" class="flow-containers">
          <div ref="canvas" class="canvas" ></div>
        </div>
</template>

<script>
// 汉化
import customTranslate from "./common/customTranslate";
import Modeler from "bpmn-js/lib/Modeler";
import panel from "./PropertyPanel";
import xml from "./xml.json";
// 引入flowable的节点文件
import flowableModdle from "./flowable/flowable.json";
export default {
  name: "WorkflowBpmnModeler",
  components: {
    panel,
  },
  props: {
    xmlObj: {
      type: Object,
      default:()=>{},
    },
    storeColor:{
      type:String,
      default:'#19aba8'
    }
  },
  data() {
    return {
      modeler: null,
      taskName: "",
      completeTask:[],
      approvalList: [],
      taskList: [],
      zoom: 1,
    };
  },
  watch: {
    xmlObj: function (val) {
      if (val) {
        this.createNewDiagram(val);
      }
    },
  },
  mounted() {
    // 生成实例
    this.modeler = new Modeler({
      container: this.$refs.canvas,
      additionalModules: [
        {
          paletteProvider: ["value", ""], //禁用/清空左侧工具栏
          translate: ["value", customTranslate],
          labelEditingProvider: ["value", ""], //禁用节点编辑
          contextPadProvider: ["value", ""], //禁用图形菜单
          bendpoints: ["value", {}], //禁用连线拖动
          zoomScroll: ["value", ""], //禁用滚动
          //move: ["value", ""], //禁用单个图形拖动
        },
      ],
      moddleExtensions: {
        flowable: flowableModdle,
      },
    });
    //document.getElementById('divtest').on('Click',this.hddcli())
    /*
    // 新增流程定义
    if (!this.xml) {
      this.newDiagram();
    } else {
      this.createNewDiagram(xml.result);
    }
    */
  },
  methods: {
    newDiagram() {
      this.createNewDiagram(xml.result);
    },
    // 让图能自适应屏幕
    fitViewport() {
      this.modeler.get("canvas").zoom("fit-viewport",'auto');
    },
    // 放大缩小
    zoomViewport(zoomIn = true) {
      this.zoom = this.modeler.get("canvas").zoom();
      this.zoom += zoomIn ? 0.1 : -0.1;
      this.modeler.get("canvas").zoom(this.zoom);
    },
    async createNewDiagram(data) {
      // 将字符串转换成图显示出来
      // data = data.replace(/<!\[CDATA\[(.+?)]]>/g, '&lt;![CDATA[$1]]&gt;')
      data.processXml = data.processXml && data.processXml.replace(
        /<!\[CDATA\[(.+?)]]>/g,
        function (match, str) {
          return str.replace(/</g, "&lt;");
        }
      );
      this.completeTask=data.completeTask;
      try {
        await this.modeler.importXML(data.processXml);

        this.fitViewport();
        this.fillColor2(data.completeTask, data.historyLine);
      } catch (err) {
        console.error(err.message, err.warnings);
      }
    },

    fillColor() {
      const canvas = this.modeler.get("canvas");
      this.modeler._definitions.rootElements[0].flowElements.forEach((n) => {
        if (n.$type === "bpmn:UserTask") {
          const completeTask = this.taskList.find((m) => m.taskId === n.id) || {
            completed: true,
          };
          const todoTask = this.taskList.find((m) => !m.completed);
          const endTask = this.taskList[this.taskList.length - 1];
          if (completeTask) {
            canvas.addMarker(
              n.id,
              completeTask.completed ? "highlight" : "highlight-todo"
            );
            n.outgoing?.forEach((nn) => {
              const targetTask = this.taskList.find(
                (m) => m.taskId === nn.targetRef.id
              );
              if (targetTask) {
                canvas.addMarker(
                  nn.id,
                  targetTask.completed ? "highlight" : "highlight-todo"
                );
              } else if (nn.targetRef.$type === "bpmn:ExclusiveGateway") {
                // canvas.addMarker(nn.id, 'highlight');
                canvas.addMarker(
                  nn.id,
                  completeTask.completed ? "highlight" : "highlight-todo"
                );
                canvas.addMarker(
                  nn.targetRef.id,
                  completeTask.completed ? "highlight" : "highlight-todo"
                );
              } else if (nn.targetRef.$type === "bpmn:EndEvent") {
                if (!todoTask && endTask.taskId === n.id) {
                  canvas.addMarker(nn.id, "highlight");
                  canvas.addMarker(nn.targetRef.id, "highlight");
                }
                if (!completeTask.completed) {
                  canvas.addMarker(nn.id, "highlight-todo");
                  canvas.addMarker(nn.targetRef.id, "highlight-todo");
                }
              }
            });
          }
        } else if (n.$type === "bpmn:ExclusiveGateway") {
          n.outgoing.forEach((nn) => {
            const targetTask = this.taskList.find(
              (m) => m.taskId === nn.targetRef.id
            );
            if (targetTask) {
              canvas.addMarker(
                nn.id,
                targetTask.completed ? "highlight" : "highlight-todo"
              );
            }
          });
        }
        if (n.$type === "bpmn:StartEvent") {
          n.outgoing.forEach((nn) => {
            const completeTask = this.taskList.find(
              (m) => m.taskId === nn.targetRef.id
            );
            if (completeTask) {
              canvas.addMarker(nn.id, "highlight");
              canvas.addMarker(n.id, "highlight");
              return;
            }
          });
        }
      });
    },
    handlerClickEvent(element) {
      console.log(element,'element')
      this.taskName = element.businessObject.name;
      if (Array.isArray(this.completeTask)) {
        this.approvalList = this.completeTask.filter(
          (item) => item.taskId == element.id
        );
      }
      this.$emit('elementClick',{taskName:this.taskName,approvalList:this.approvalList})
    },
    fillColor2(nodeCodes, historyLine) {
      if (Array.isArray(nodeCodes)) {
        const elementRegistry = this.modeler.get("elementRegistry");
        const modeling = this.modeler.get("modeling");
        const startEventNodeArr = elementRegistry.filter(
          (item) => item.type === "bpmn:StartEvent"
        );
        startEventNodeArr.map((item) => {
          this.setLineColor(item.id, modeling, elementRegistry);
        });
         const eventBus = this.modeler.get("eventBus");
         console.log(eventBus,'eventBus')
        eventBus.on("drag.start", (e)=>{
          /*
          const { element } = e;
          if (!element.parent) return;
          if (!e || element.type !== "bpmn:UserTask") return;
          */
         if(e.shape.type==='bpmn:UserTask'){
          this.handlerClickEvent(e.shape);
         }
         return false;
         
          //this.handlerClickEvent(e);
        });
        for (let i = 0; i < nodeCodes.length; i++) {
          this.setNodeInfo(nodeCodes[i], modeling, elementRegistry);
        }
        if (Array.isArray(historyLine)) {
          for (let i = 0; i < historyLine.length; i++) {
            this.setLineColor(historyLine[i], modeling, elementRegistry);
          }
        }
      }
    },
    setLineColor(lineId, modeling, elementRegistry) {
      let shape = elementRegistry.get(lineId);
      modeling.setColor(shape, { stroke: this.storeColor });
    },
    setNodeInfo(nodeInfo, modeling, elementRegistry) {
      let shape = elementRegistry.get(nodeInfo.taskId);
      if (nodeInfo.taskState === 3) {
        modeling.setColor(shape, { stroke: this.storeColor });
      }
    },
   
    // 对外 api
    getProcess() {
      const element = this.getProcessElement();
      return {
        id: element.id,
        name: element.name,
        category: element.$attrs["flowable:processCategory"],
      };
    },
    getProcessElement() {
      const rootElements = this.modeler.getDefinitions().rootElements;
      for (let i = 0; i < rootElements.length; i++) {
        if (rootElements[i].$type === "bpmn:Process") return rootElements[i];
      }
    },
  },
};
</script>

<style lang="scss">
/*左边工具栏以及编辑节点的样式*/
@import "~bpmn-js/dist/assets/diagram-js.css";
@import "~bpmn-js/dist/assets/bpmn-font/css/bpmn.css";
@import "~bpmn-js/dist/assets/bpmn-font/css/bpmn-codes.css";
@import "~bpmn-js/dist/assets/bpmn-font/css/bpmn-embedded.css";
.bjs-container {
  flex: 1;
  position: relative;
  background: url("data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNDAiIGhlaWdodD0iNDAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PGRlZnM+PHBhdHRlcm4gaWQ9ImEiIHdpZHRoPSI0MCIgaGVpZ2h0PSI0MCIgcGF0dGVyblVuaXRzPSJ1c2VyU3BhY2VPblVzZSI+PHBhdGggZD0iTTAgMTBoNDBNMTAgMHY0ME0wIDIwaDQwTTIwIDB2NDBNMCAzMGg0ME0zMCAwdjQwIiBmaWxsPSJub25lIiBzdHJva2U9IiNlMGUwZTAiIG9wYWNpdHk9Ii4yIi8+PHBhdGggZD0iTTQwIDBIMHY0MCIgZmlsbD0ibm9uZSIgc3Ryb2tlPSIjZTBlMGUwIi8+PC9wYXR0ZXJuPjwvZGVmcz48cmVjdCB3aWR0aD0iMTAwJSIgaGVpZ2h0PSIxMDAlIiBmaWxsPSJ1cmwoI2EpIi8+PC9zdmc+")
    repeat !important;
  div.toggle-mode {
    display: none;
  }
}
.view-mode {
  .el-header,
  .el-aside,
  .djs-palette,
  .bjs-powered-by {
    display: none;
  }
  .el-loading-mask {
    background-color: initial;
  }
  .el-loading-spinner {
    display: none;
  }
}
.flow-containers {
  // background-color: #ffffff;
  width: 100%;
  height: 100%;
  .canvas {
    width: 100%;
    height: 100%;
  }
  .panel {
    position: absolute;
    right: 0;
    top: 50px;
    width: 300px;
  }
  .load {
    margin-right: 10px;
  }
  .el-form-item__label {
    font-size: 13px;
  }

  .djs-palette {
    left: 0px !important;
    top: 0px;
    border-top: none;
  }

  .djs-container svg {
    min-height: 650px;
  }

  .highlight.djs-shape .djs-visual > :nth-child(1) {
    fill: #19aba8 !important;
    stroke: #19aba8 !important;
    fill-opacity: 0.2 !important;
  }
  .highlight.djs-shape .djs-visual > :nth-child(2) {
    fill: #19aba8 !important;
  }
  .highlight.djs-shape .djs-visual > path {
    fill: #19aba8 !important;
    fill-opacity: 0.2 !important;
    stroke: #19aba8 !important;
  }
  .highlight.djs-connection > .djs-visual > path {
    stroke: #19aba8 !important;
  }
  // .djs-connection > .djs-visual > path {
  //   stroke: orange !important;
  //   stroke-dasharray: 4px !important;
  //   fill-opacity: 0.2 !important;
  // }
  // .djs-shape .djs-visual > :nth-child(1) {
  //   fill: orange !important;
  //   stroke: orange !important;
  //   stroke-dasharray: 4px !important;
  //   fill-opacity: 0.2 !important;
  // }
  .highlight-todo.djs-connection > .djs-visual > path {
    stroke: orange !important;
    stroke-dasharray: 4px !important;
    fill-opacity: 0.2 !important;
  }
  .highlight-todo.djs-shape .djs-visual > :nth-child(1) {
    fill: orange !important;
    stroke: orange !important;
    stroke-dasharray: 4px !important;
    fill-opacity: 0.2 !important;
  }
  .overlays-div {
    font-size: 10px;
    color: red;
    width: 100px;
    top: -20px !important;
  }
}
</style>
