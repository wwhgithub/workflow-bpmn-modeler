<template>
  <div id="app">
    <bpmn-modeler
      ref="refNode"
      :xmlObj="xml"
      storeColor="red"
      @elementClick="elementClick"
    />
  </div>
</template>

<script>
import bpmnModeler from '../package/index'
import xml from "../package/xml.json";
export default {
  components: {
    bpmnModeler
  },
  data() {
    return {
      xml:undefined, // 后端查询到的xml
    }
  },
  mounted() {
    this.getModelDetail()
  },
  methods: {
    elementClick(e){
      console.log(e)
    },
    getModelDetail() {
      this.xml=xml.result
    },
    saveModeler(data) {
      console.log(data)
    }
  }
}
</script>

<style lang="scss">
html, body, #app {
  // height: 650px;
  margin: 0;
}
</style>
