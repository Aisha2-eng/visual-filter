<script>
import { h } from "vue"
import {
  FilterType,
  GroupType,
  DataType,
  deepCopy,
} from "@visual-filter/common"
import applyFilter from "@visual-filter/applyer"

import FilterGroup from "./FilterGroup.vue"
import FilterCondition from "./FilterCondition.vue"

export default {
  name: "VueVisualFilter",
  emits: ["filterUpdate", "update:modelValue"],
  props: {
    filteringOptions: {
      type: Object,
      required: true,
      validator(value) {
        try {
          return (
            value.data.length &&
            value.data.every(
              (field, index, fields) =>
                typeof field.name === "string" &&
                typeof field.type === "string" &&
                field.values.constructor === Array &&
                (index > 0
                  ? field.values.length === fields[index - 1].values.length
                  : true),
            ) &&
            Object.values(value.methods.numeric).every(
              (method) => typeof method === "function",
            ) &&
            Object.values(value.methods.nominal).every(
              (method) => typeof method === "function",
            )
          )
        } catch {
          return false
        }
      },
    },
    // New props to support v-model for binding filter state
    modelValue: {
      type: Object,
      default: null,
    },
    enableHistory: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      filter: this.modelValue
        ? deepCopy(this.modelValue)
        : { type: FilterType.GROUP, groupType: GroupType.AND, filters: [] },
      history: [],
      future: [],
      isRestoring: false,
    }
  },

  computed: {
    fieldNames() {
      return this.filteringOptions.data.map((field) => field.name)
    },
    numericMethodNames() {
      return Object.keys(this.filteringOptions.methods.numeric)
    },
    nominalMethodNames() {
      return Object.keys(this.filteringOptions.methods.nominal)
    },
  },

  watch: {
    // to sync 
    modelValue: {
      deep: true,
      handler(newVal) {
        if (newVal && JSON.stringify(newVal) !== JSON.stringify(this.filter)) {
          if (!this.isRestoring) {
            // When modelValue changes externally, clear history and future
            this.history = [];
            this.future = [];
          }
          this.filter = deepCopy(newVal);
          this.isRestoring = false; // Reset after potential external update
        }
      },
    },

    // when filter changes
    filter: {
      deep: true,
      handler(newVal, oldVal) {
        // Only record history if not currently restoring a state
        if (this.enableHistory && !this.isRestoring) {
          if (oldVal && Object.keys(oldVal).length > 0) {
            this.history.push(deepCopy(oldVal));
            this.future = []; 
          }
        }
        
        this.isRestoring = false;

        const updated = {
          filter: deepCopy(newVal),
          data: applyFilter(
            newVal,
            this.filteringOptions.methods,
            deepCopy(this.filteringOptions.data),
          ),
        }

        this.$emit("filterUpdate", updated)
        this.$emit("update:modelValue", deepCopy(newVal))//Every changes effect filter state in the parent.
      },
    },
  },

  methods: {
    updateConditionField(condition, newFieldName) {
      const {
        type: newType,
        values: [newSampleValue = ""],
      } = this.filteringOptions.data.find(
        (field) => field.name === newFieldName,
      )
      if (condition.dataType !== newType) {
        condition.method =
          (newType === DataType.NUMERIC
            ? this.numericMethodNames[0]
            : this.nominalMethodNames[0]) || ""
        condition.argument = newSampleValue
        condition.dataType = newType
      }
    },

    addFilter(filters, newFilterType) {
      if (newFilterType === FilterType.GROUP) {
        filters.push({
          type: FilterType.GROUP,
          groupType: GroupType.AND,
          filters: [],
        })
      } else {
        const {
          name,
          type,
          values: [sampleValue = ""],
        } = this.filteringOptions.data[0]

        filters.push({
          type: FilterType.CONDITION,
          fieldName: name,
          dataType: type,
          method:
            (type === DataType.NUMERIC
              ? this.numericMethodNames[0]
              : this.nominalMethodNames[0]) || "",
          argument: sampleValue,
        })
      }
    },

    deleteFilter(filterToDelete) {
      function recursiveDeletion(filter, index, filters) {
        if (filter === filterToDelete) {
          filters.splice(index, 1)
        } else if (filter.type === FilterType.GROUP) {
          filter.filters.map(recursiveDeletion)
        }
      }

      if (filterToDelete !== this.filter) {
        recursiveDeletion(this.filter)
      }
    },

    // New Methods for control
    //reset
    resetFilter() {
      this.filter = {
        type: FilterType.GROUP,
        groupType: GroupType.AND,
        filters: [],
      }
      this.history = []
      this.future = []
    },
    //undo
    undo() {
      if (!this.enableHistory || this.history.length === 0) return;
      this.isRestoring = true;
      this.future.unshift(deepCopy(this.filter)); // Add current state to future
      this.filter = this.history.pop(); // Restore previous state
    },
    // redo
    redo() {
      if (!this.enableHistory || this.future.length === 0) return;
      this.isRestoring = true;
      this.history.push(deepCopy(this.filter)); // Add current state to history
      this.filter = this.future.shift(); // Restore future state
    },
  },

  render() {
    const createVisualizer = (filter) => {
      if (filter.type === FilterType.GROUP) {
        return h(
          FilterGroup,
          {
            group: filter,
            filterTypes: Object.values(FilterType),
            groupTypes: Object.values(GroupType),
            removable: filter !== this.filter,
            onAddFilter: this.addFilter,
            onDeleteGroup: this.deleteFilter,
          },
          {
            groupTypes: this.$slots.groupTypes,
            filterAddition: this.$slots.filterAddition,
            groupDeletion: this.$slots.groupDeletion,
            groupChildren: () => filter.filters.map(createVisualizer),
          },
        )
      } else {
        return h(
          FilterCondition,
          {
            condition: filter,
            fieldNames: this.fieldNames,
            numericMethodNames: this.numericMethodNames,
            nominalMethodNames: this.nominalMethodNames,
            onUpdateField: this.updateConditionField,
            onDeleteCondition: this.deleteFilter,
          },
          {
            fieldUpdation: this.$slots.fieldUpdation,
            methodUpdation: this.$slots.methodUpdation,
            argumentUpdation: this.$slots.argumentUpdation,
            conditionDeletion: this.$slots.conditionDeletion,
          },
        )
      }
    }

    return createVisualizer(this.filter)
  },
}
</script>