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
            ) &&
            // Add validation for date methods if they exist
            (!value.methods.date || Object.values(value.methods.date).every(
              (method) => typeof method === "function",
            ))
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
    dateMethodNames() {
      return this.filteringOptions.methods.date 
        ? Object.keys(this.filteringOptions.methods.date)
        : []
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
          // DEFENSIVE CHECK: Ensure oldVal is a valid object before deep copying
          if (oldVal && typeof oldVal === 'object' && Object.keys(oldVal).length > 0) {
            try {
              this.history.push(deepCopy(oldVal));
              this.future = [];
            } catch (error) {
              console.error("Error during deepCopy of oldVal:", error);
            }
          }
        }
        
        this.isRestoring = false;

        // DEFENSIVE CHECK: Ensure newVal is valid before processing
        if (!newVal || typeof newVal !== 'object') {
          console.warn("Invalid filter value:", newVal);
          return;
        }

        try {
          const updated = {
            filter: deepCopy(newVal),
            data: applyFilter(
              newVal,
              this.filteringOptions.methods,
              deepCopy(this.filteringOptions.data),
            ),
          }

          this.$emit("filterUpdate", updated)
          this.$emit("update:modelValue", deepCopy(newVal))
        } catch (error) {
          console.error("Error during filter update:", error);
        }
      },
    },
  },

  methods: {
    updateConditionField(condition, newFieldName) {
      const field = this.filteringOptions.data.find(
        (f) => f.name === newFieldName,
      );

      if (!field) {
        console.warn(`Field with name ${newFieldName} not found in filteringOptions.data.`);
        return; // Prevent further errors if field is not found
      }

      const { type: newType, values: [newSampleValue = ""] } = field;

      if (condition.dataType !== newType) {
        if (newType === DataType.NUMERIC) {
          condition.method = this.numericMethodNames[0] || ""
        } else if (newType === DataType.DATE) {
          condition.method = this.dateMethodNames[0] || ""
        } else {
          condition.method = this.nominalMethodNames[0] || ""
        }
        condition.argument = newSampleValue
        condition.dataType = newType
      }
      // Update the fieldName
      condition.fieldName = newFieldName;
    },

    addFilter(filters, newFilterType) {
      if (newFilterType === FilterType.GROUP) {
        filters.push({
          type: FilterType.GROUP,
          groupType: GroupType.AND,
          filters: [],
        })
      } else {
        // Ensure there's at least one field to pick from
        if (!this.filteringOptions.data || this.filteringOptions.data.length === 0) {
          console.warn("Cannot add filter condition: filteringOptions.data is empty.");
          return; // Prevent adding a condition if no data fields are available
        }

        const {
          name,
          type,
          values: [sampleValue = ""],
        } = this.filteringOptions.data[0]

        // Determine the appropriate method based on the data type
        let method = ""
        if (type === DataType.NUMERIC) {
          method = this.numericMethodNames[0] || ""
        } else if (type === DataType.DATE) {
          method = this.dateMethodNames[0] || ""
        } else {
          method = this.nominalMethodNames[0] || ""
        }

        filters.push({
          type: FilterType.CONDITION,
          fieldName: name,
          dataType: type,
          method: method,
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
      try {
        this.future.unshift(deepCopy(this.filter)); // Add current state to future
        this.filter = this.history.pop(); // Restore previous state
      } catch (error) {
        console.error("Error during undo:", error);
        this.isRestoring = false;
      }
    },
    // redo
    redo() {
      if (!this.enableHistory || this.future.length === 0) return;
      this.isRestoring = true;
      try {
        this.history.push(deepCopy(this.filter)); // Add current state to history
        this.filter = this.future.shift(); // Restore future state
      } catch (error) {
        console.error("Error during redo:", error);
        this.isRestoring = false;
      }
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
            dateMethodNames: this.dateMethodNames,
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