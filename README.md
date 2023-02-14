English | [简体中文](./README_CN.md)

## Preview 📟

![20200930030243](https://cdn.jsdelivr.net/gh/goldsubmarine/cdn@master/blog/20200930030243.png)


## Install ⏳

```bash
# Install
yarn add workflow-bpmn-modeler-maslong
```

## How to use 👣

```vue
<template>
  <div>
   <bpmn-modeler
      ref="refNode"
      :xmlObj="xml"
      storeColor="red"
      @elementClick="elementClick"
    />
  </div>
</template>

<script>
import bpmnModeler from "workflow-bpmn-modeler-maslong";
import xml from "../package/xml.json";
export default {
  components: {
    bpmnModeler,
  },
  data() {
    return {
      xml: undefined, // Query the xml xml={}
    };
  },
   mounted() {
    this.getModelDetail()
  },
  methods: {
    elementClick(e) {
      console.log(e); 
      
    },
    getModelDetail() {
      this.xml=xml.result
    },
  },
};
</script>
```

## License 📄

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2020-present, charles
