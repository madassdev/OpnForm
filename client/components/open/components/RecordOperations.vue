<template>
  <div class="flex items-center justify-center space-x-1">
    <button v-track.delete_record_click
            class="border rounded py-1 px-2 text-gray-500 dark:text-gray-400 hover:text-red-700"
            @click="onDeleteClick"
    >
      <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor"
           class="w-4 h-4"
      >
        <path stroke-linecap="round" stroke-linejoin="round"
              d="M14.74 9l-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 01-2.244 2.077H8.084a2.25 2.25 0 01-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 00-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 013.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 00-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 00-7.5 0"
        />
      </svg>
    </button>
  </div>
</template>

<script>

export default {
  components: { },
  props: {
    form: {
      type: Object,
      required: true
    },
    structure: {
      type: Array,
      default: () => []
    },
    rowid: {
      type: Number,
      default: () => {}
    }
  },
  setup () {
    return {
      useAlert: useAlert()
    }
  },
  data () {
    return {
    }
  },
  computed: {
  },
  mounted () {
  },
  methods: {
    onDeleteClick () {
      this.useAlert.confirm('Do you really want to delete this record?', this.deleteRecord)
    },
    async deleteRecord () {
      opnFetch('/open/forms/' + this.form.id + '/records/' + this.rowid + '/delete', {method:'DELETE'}).then(async (data) => {
        if (data.type === 'success') {
          this.$emit('deleted')
          this.useAlert.success(data.message)
        } else {
          this.useAlert.error('Something went wrong!')
        }
      }).catch((error) => {
        this.useAlert.error(error.response.data.message)
      })
    }
  }
}
</script>
