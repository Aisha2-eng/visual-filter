<script>
import { DataType } from "@visual-filter/common"

export default {
  name: "FilterCondition",
  emits: ["updateField", "deleteCondition"],
  props: {
    dateMethodNames: {
      type: Array,
      required: false,
      default: () => [],
    },
    condition: {
      type: Object,
      required: true,
      validator(value) {
        return value.constructor === Object
      },
    },
    fieldNames: {
      type: Array,
      required: true,
    },
    numericMethodNames: {
      type: Array,
      required: true,
    },
    nominalMethodNames: {
      type: Array,
      required: true,
    },
  },
  computed: {
    isNumeric() {
      return this.condition.dataType === DataType.NUMERIC
    },
    isDate() {
      return this.condition.dataType === DataType.DATE
    },
    currentMethodNames() {
      if (this.isNumeric) {
        return this.numericMethodNames
      } else if (this.isDate) {
        return this.dateMethodNames
      } else {
        return this.nominalMethodNames
      }
    },
  },
  methods: {
    updateField(newFieldName) {
      if (this.fieldNames.includes(newFieldName)) {
        this.$emit("updateField", this.condition, newFieldName)
      }
    },
  },
}
</script>

<template>
  <div class="space-x-2">
    <slot name="fieldUpdation" v-bind="{ fieldNames, condition, updateField }">
      <select
        v-model="condition.fieldName"
        @change="updateField($event.target.value)"
        data-testId="field-name-select"
      >
        <option v-for="field in fieldNames" :key="field" :value="field">
          {{ field }}
        </option>
      </select>
    </slot>
    <slot
      name="methodUpdation"
      v-bind="{
        numericMethodNames: isNumeric && numericMethodNames,
        nominalMethodNames: !isNumeric && !isDate && nominalMethodNames,
        dateMethodNames: isDate && dateMethodNames,
        condition,
      }"
    >
      <select v-model="condition.method" data-testId="method-select">
        <option
          v-for="method in currentMethodNames"
          :key="method"
          :value="method"
        >
          {{ method }}
        </option>
      </select>
    </slot>
    <slot name="argumentUpdation" :condition="condition">
      <input
        v-if="isDate"
        type="date"
        v-model="condition.argument"
        data-testId="argument-input"
      />
      <input
        v-else
        type="text"
        v-model="condition.argument"
        data-testId="argument-input"
      />
    </slot>
    <slot
      name="conditionDeletion"
      :deleteCondition="() => $emit('deleteCondition', condition)"
    >
      <button @click="$emit('deleteCondition', condition)" data-testId="remove-condition-button">x</button>
    </slot>
  </div>
</template>
