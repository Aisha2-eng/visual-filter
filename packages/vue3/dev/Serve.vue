<script>
import { FilterType, GroupType, DataType } from "@visual-filter/common";

export default {
  name: "Serve",
  data() {
    return {
      filteringOptions: {
        data: [
          {
            name: "First Name",
            type: DataType.NOMINAL,  // Using constant for consistency
            values: ["Obada", "Ahmad", "Omar"],
          },
          {
            name: "Last Name",
            type: DataType.NOMINAL,  // Using constant for consistency
            values: ["Khalili", "Drhili", "Hala hili"],
          },
          {
            name: "Grade",
            type: DataType.NUMERIC,  // Using constant for consistency
            values: [3.72, 3.52, 3.4],
          },
          {
            name: "Enrollment Date",
            type: DataType.DATE,
            values: ["2023-01-15", "2023-03-20", "2022-11-01", "2024-02-28"],
          },
        ],
        methods: {
          numeric: {
            "="(cellValue, argument) {
              return cellValue == argument;
            },
            ">"(cellValue, argument) {
              return cellValue > argument;
            },
            "<"(cellValue, argument) {
              return cellValue < argument;
            },
          },
          nominal: {
            contains(cellValue, argument) {
              return cellValue.includes(argument);
            },
            startsWith(cellValue, argument) {
              return cellValue.startsWith(argument);
            },
            endsWith(cellValue, argument) {
              return cellValue.endsWith(argument);
            },
          },
          date: {
            before(cellValue, argument) {
              const cellDate = new Date(cellValue);
              const argDate = new Date(argument);
              return cellDate < argDate;
            },
            after(cellValue, argument) {
              const cellDate = new Date(cellValue);
              const argDate = new Date(argument);
              return cellDate > argDate;
            },
            on(cellValue, argument) {
              const cellDate = new Date(cellValue);
              const argDate = new Date(argument);
              return (
                cellDate.getFullYear() === argDate.getFullYear() &&
                cellDate.getMonth() === argDate.getMonth() &&
                cellDate.getDate() === argDate.getDate()
              );
            },
          },
        },
      },
  filterState: null, // current filter (used as v-model payload)
  filteredResult: null, // last { filter, data } emitted by filter-update
      savedFilters: {}, // stored filters in localStorage
      filterName: "", // name the filter for saving
      selectedFilter: "", // for loading
      preConfiguredFilters: {
        grade_a: {
          type: FilterType.GROUP,
          groupType: GroupType.AND,
          filters: [
            {
              type: FilterType.CONDITION,
              fieldName: "Grade",
              dataType: DataType.NUMERIC,
              method: ">",
              argument: 3.5,
            },
          ],
        },
        first_name_obada: {
          type: FilterType.GROUP,
          groupType: GroupType.AND,
          filters: [
            {
              type: FilterType.CONDITION,
              fieldName: "First Name",
              dataType: DataType.NOMINAL,
              method: "contains",
              argument: "Obada",
            },
          ],
        },
      },
    };
  },

  mounted() {
    this.loadSavedFilters();
    this.loadFilterFromUrl();
  },

  methods: {
    loadFilterFromUrl() {
      const urlParams = new URLSearchParams(window.location.search);
      const filterName = urlParams.get("filter");
      if (filterName && this.preConfiguredFilters[filterName]) {
        // Only set the filter config; component will compute result
        this.filterState = this.preConfiguredFilters[filterName];
        alert(`Loaded pre-configured filter: ${filterName}`);
      }
    },
    captureFilterUpdate(ctx) {
        // Do not override v-model payload; keep it as filter only
        // Store full payload separately for display purposes
        this.filteredResult = ctx;
    },

    //save current filter to localStorage
    saveFilter() {
      if (!this.filterName) {
        alert("Please enter a name for this filter");
        return;
      }
      if (!this.filterState) {
        alert("No filter selected yet");
        return;
      }

      this.savedFilters[this.filterName] = this.filterState;
      localStorage.setItem("saved_filters", JSON.stringify(this.savedFilters));
      alert("Filter saved successfully!");
      this.filterName = "";
    },

    // load all filters from localStorage
    loadSavedFilters() {
      const saved = localStorage.getItem("saved_filters");
      this.savedFilters = saved ? JSON.parse(saved) : {};
    },

    // load one filter into component
    loadFilter() {
      const filter = this.savedFilters[this.selectedFilter];
      if (filter) {
        // Load only the filter config
        this.filterState = filter;
        alert(`Loaded filter: ${this.selectedFilter}`);
      }
    },
    //undo
    undoFilter() {
      if (this.$refs.visualFilter) {
        this.$refs.visualFilter.undo();
      }
    },
    // redo
    redoFilter() {
      if (this.$refs.visualFilter) {
        this.$refs.visualFilter.redo();
      }
    },
    //reset
    resetFilter() {
      if (this.$refs.visualFilter) {
        this.$refs.visualFilter.resetFilter();
        alert("Filter state reset!");
      }
    },

    loadPreConfiguredFilter(filterName) {
      if (this.preConfiguredFilters[filterName]) {
        // Only set the filter config
        this.filterState = this.preConfiguredFilters[filterName];
        alert(`Loaded pre-configured filter: ${filterName}`);
      }
    },
  },
};
</script>


<template>
  <div style="padding: 20px;">
    <h2>Vue Visual Filter</h2>

    <div style="margin-bottom: 16px;">
      <input
        v-model="filterName"
        placeholder="Name your filter"
        style="padding:6px; border:1px solid #ccc; border-radius:4px;"
      />
      <button @click="saveFilter" style="margin-left:8px;">Save</button>

      <select
        v-model="selectedFilter"
        @change="loadFilter"
        style="margin-left:8px; padding: 8px 10px; border-radius: 8px; border: 1px solid #e2e8f0;"
      >
        <option value="">Load Saved Filter</option>
        <option v-for="(f, name) in savedFilters" :key="name" :value="name">
          {{ name }}
        </option>
      </select>
    </div>

    <div style="margin-top: 16px; display: flex; gap:8px; align-items: center;">
      <strong>Pre-configured Filters:</strong>
      <button @click="loadPreConfiguredFilter('grade_a')">Grade > 3.5</button>
      <button @click="loadPreConfiguredFilter('first_name_obada')">
        First Name contains 'Obada'
      </button>
    </div>

    <div style="margin-bottom: 8px; display: flex; gap:8px; align-items: center;">
      <button class="btn warn" @click="resetFilter" title="Reset">🔁</button>
      <button class="btn" @click="undoFilter" title="Undo">↩️</button>
      <button class="btn" @click="redoFilter" title="Redo">↪️</button>
    </div>

    <VueVisualFilter
      ref="visualFilter"
      v-model="filterState"
      :filtering-options="filteringOptions"
      @filter-update="captureFilterUpdate"
      :enable-history="true"
    />

    
  </div>
</template>


<style scoped>
button {
  background: #f3f6fb;
  color: #202020;
  padding: 7px 10px;
  border-radius: 8px;
  border: 1px solid transparent;
  cursor: pointer;
  font-weight: 600;
}
button:hover {
  background-color: #76aeea;
  filter: brightness(0.98);
  transform: translateY(-1px);
}
</style>
