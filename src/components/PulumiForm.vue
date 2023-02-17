<template>
  <div id="editZone">
    <div>
      <input type="radio" id="json" value="json" v-model="renderMode" />
      <label for="json">JSON</label>
      <input type="radio" id="form" value="form" v-model="renderMode" />
      <label for="json">Form</label>
    </div>
    <json-forms
      v-show="renderMode === 'form'"
      :data="data"
      :renderers="renderers"
      :schema="schema"
      :uischema="uischema"
      @change="onChange"
    />

    <textarea
      v-show="renderMode === 'json'"
      id="code"
      :value="dataAsJson"
      @input="onTextChange"
    ></textarea>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { JsonForms, JsonFormsChangeEvent } from "@jsonforms/vue";
import {
  defaultStyles,
  mergeStyles,
  vanillaRenderers,
} from "@jsonforms/vue-vanilla";

const myStyles = mergeStyles(defaultStyles, { control: { label: "mylabel" } });
const renderers = vanillaRenderers;
const schema = {
  properties: {
    cpu: {
      type: "integer",
      minimum: 10,
      maximum: 1024,
    },
    memory: {
      type: "integer",
      minimum: 32,
      maximum: 1024,
    },
  },
};
const uischema = {
  type: "HorizontalLayout",
  elements: [
    {
      type: "VerticalLayout",
      elements: [
        {
          type: "Control",
          scope: "#/properties/cpu",
        },
        {
          type: "Control",
          scope: "#/properties/memory",
        },
      ],
    },
  ],
};
export default defineComponent({
  name: "PulumiForm",
  components: {
    JsonForms,
  },
  emits: ["change"],
  data() {
    return {
      // freeze renderers for performance gains
      renderMode: "json",
      renderers: Object.freeze(renderers),
      data: {
        cpu: 1024,
        memory: 1024,
      },
      schema,
      uischema,
    };
  },
  methods: {
    onChange(event: JsonFormsChangeEvent) {
      this.data = event.data;
    },
    onTextChange(event) {
      this.data = JSON.parse(event.target.value);
    },
  },
  computed: {
    dataAsJson() {
      return JSON.stringify(this.data);
    },
  },
  provide() {
    return {
      styles: myStyles,
    };
  },
  mounted() {
    this.$emit("change", this.data);
  },
  watch: {
    data(newValue) {
      this.$emit("change", newValue);
    },
  },
});
</script>
<style>
#editZone {
  margin-top: 100px;
}
.mylabel {
  color: darkslategrey;
}
.vertical-layout {
  margin-left: 10px;
  margin-right: 10px;
}
.myform {
  width: 640px;
  margin: 0 auto;
}
.text-area {
  min-height: 80px;
}

#code {
  font-family: "Courier New", serif;
  width: 90%;
  height: 80%;
}
</style>
