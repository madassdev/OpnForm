<template>
  <div
    class="my-4 w-full mx-auto"
  >
    <h3 class="font-semibold mb-4 text-xl">
      Form Submissions
    </h3>

    <!--  Table columns modal  -->
    <modal :show="showColumnsModal" @close="showColumnsModal=false">
      <template #icon>
        <svg class="w-8 h-8" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M16 5H8C4.13401 5 1 8.13401 1 12C1 15.866 4.13401 19 8 19H16C19.866 19 23 15.866 23 12C23 8.13401 19.866 5 16 5Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
          <path d="M8 15C9.65685 15 11 13.6569 11 12C11 10.3431 9.65685 9 8 9C6.34315 9 5 10.3431 5 12C5 13.6569 6.34315 15 8 15Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </template>
      <template #title>
        Display columns
      </template>

      <div class="px-4">
        <template v-if="form.properties.length > 0">
          <h4 class="font-bold mb-2">
            Form Fields
          </h4>
          <div class="border border-gray-300 rounded-md">
            <div v-for="(field,index) in form.properties" :key="field.id" class="p-2 border-gray-300 flex items-center" :class="{'border-t':index!=0}">
              <p class="flex-grow truncate">
              {{ field.name }}
              </p>
              <v-switch v-model="displayColumns[field.id]" class="float-right" @update:model-value="onChangeDisplayColumns" />
            </div>
          </div>
        </template>
        <template v-if="removed_properties.length > 0">
          <h4 class="font-bold mb-2 mt-4">
            Removed Fields
          </h4>
          <div class="border border-gray-300 rounded-md">
            <div v-for="(field,index) in removed_properties" :key="field.id" class="p-2 border-gray-300 flex items-center" :class="{'border-t':index!=0}">
              <p class="flex-grow truncate">
                {{ field.name }}
              </p>
              <v-switch v-model="displayColumns[field.id]" class="float-right" @update:model-value="onChangeDisplayColumns" />
            </div>
          </div>
        </template>
      </div>
    </modal>

    <Loader v-if="!form || !formInitDone" class="h-6 w-6 text-nt-blue mx-auto" />
    <div v-else>
      <div v-if="form && tableData.length > 0" class="flex flex-wrap items-end">
        <div class="flex-grow">
          <text-input class="w-64" :form="searchForm" name="search" placeholder="Search..." />
        </div>
        <div class="font-semibold flex gap-4">
          <p class="float-right text-xs uppercase mb-2">
            <a
              href="javascript:void(0);" class="text-gray-500" @click="showColumnsModal=true"
            >Display columns</a>
          </p>
          <p class="text-right cursor-pointer text-xs uppercase">
            <a
              @click.prevent="downloadAsCsv" href="#"
            >Export as CSV</a>
          </p>
        </div>
      </div>

      <scroll-shadow
        ref="shadows"
        class="border max-h-full h-full notion-database-renderer"
        :shadow-top-offset="0"
        :hide-scrollbar="true"
      >
        <open-table
          ref="table"
          class="max-h-full"
          :columns="properties"
          :data="filteredData"
          :loading="isLoading"
          @resize="dataChanged()"
          @deleted="onDeleteRecord()"
          @update-columns="onColumnUpdated"
        />
      </scroll-shadow>
    </div>
  </div>
</template>

<script>
import Fuse from 'fuse.js'
import clonedeep from 'clone-deep'
import VSwitch from '../../../forms/components/VSwitch.vue'
import OpenTable from '../../tables/OpenTable.vue'

export default {
  name: 'FormSubmissions',
  components: { OpenTable, VSwitch },
  props: {},

  setup () {
    const workingFormStore = useWorkingFormStore()
    return {
      workingFormStore,
      runtimeConfig: useRuntimeConfig()
    }
  },

  data () {
    return {
      formInitDone: false,
      isLoading: false,
      tableData: [],
      currentPage: 1,
      fullyLoaded: false,
      showColumnsModal: false,
      properties: [],
      removed_properties: [],
      displayColumns: {},
      searchForm: useForm({
        search: ''
      })
    }
  },
  computed: {
    form: {
      get () {
        return this.workingFormStore.content
      },
      set (value) {
        this.workingFormStore.set(value)
      }
    },
    exportUrl () {
      if (!this.form) {
        return ''
      }
      return this.runtimeConfig.public.apiBase + '/open/forms/' + this.form.id + '/submissions/export'
    },
    filteredData () {
      if (!this.tableData) return []

      const filteredData = clonedeep(this.tableData)

      if (this.searchForm.search === '' || this.searchForm.search === null) {
        return filteredData
      }

      // Fuze search
      const fuzeOptions = {
        keys: this.properties.map((field) => field.id)
      }
      const fuse = new Fuse(filteredData, fuzeOptions)
      return fuse.search(this.searchForm.search).map((res) => {
        return res.item
      })
    }
  },
  watch: {
    'form.id' () {
      if (this.form === null) {
        return
      }
      this.initFormStructure()
      this.getSubmissionsData()
    }
  },
  mounted () {
    this.initFormStructure()
    this.getSubmissionsData()
  },
  methods: {
    initFormStructure () {
      if (!this.form || !this.form.properties || this.formInitDone) {
        return
      }

      // check if form properties already has a created_at column
      this.properties = clonedeep(this.form.properties)
      if (!this.properties.find((property) => {
        if (property.id === 'created_at') {
          return true
        }
      }) ) {
        // Add a "created at" column
        this.properties.push({
          name: 'Created at',
          id: 'created_at',
          type: 'date',
          width: 140
        })
      }
      this.formInitDone = true
      this.removed_properties = (this.form.removed_properties) ? clonedeep(this.form.removed_properties) : []

      // Get display columns from local storage
      const tmpColumns = window.localStorage.getItem('display-columns-formid-' + this.form.id)
      if (tmpColumns !== null && tmpColumns) {
        this.displayColumns = JSON.parse(tmpColumns)
        this.onChangeDisplayColumns()
      } else {
        this.properties.forEach((field) => {
          this.displayColumns[field.id] = true
        })
      }
    },
    getSubmissionsData () {
      if (!this.form || this.fullyLoaded) {
        return
      }
      this.isLoading = true
      opnFetch('/open/forms/' + this.form.id + '/submissions?page=' + this.currentPage).then((resData) => {
        this.tableData = this.tableData.concat(resData.data.map((record) => record.data))
        this.dataChanged()

        if (this.currentPage < resData.meta.last_page) {
          this.currentPage += 1
          this.getSubmissionsData()
        } else {
          this.isLoading = false
          this.fullyLoaded = true
        }
      }).catch((error) => {
        console.error(error)
        this.isLoading = false
      })
    },
    dataChanged () {
      if (this.$refs.shadows) {
        this.$refs.shadows.toggleShadow()
        this.$refs.shadows.calcDimensions()
      }
    },
    onColumnUpdated(columns) {
      this.properties = columns
    },
    onChangeDisplayColumns () {
      if (!process.client) return
      window.localStorage.setItem('display-columns-formid-' + this.form.id, JSON.stringify(this.displayColumns))
      this.properties = clonedeep(this.form.properties).concat(this.removed_properties).filter((field) => {
        return this.displayColumns[field.id] === true
      })
    },
    onDeleteRecord () {
      this.fullyLoaded = false
      this.tableData = []
      this.getSubmissionsData()
    },
    downloadAsCsv() {
      opnFetch(this.exportUrl, { responseType: "blob" })
        .then( blob => {
          var file = window.URL.createObjectURL(blob);
          window.location.assign(file);
        }).catch((error)=>{
        console.error(error)
      })
    }
  }
}
</script>
