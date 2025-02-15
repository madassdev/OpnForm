<template>
  <div class="flex flex-wrap flex-col">
    <transition v-if="stateReady" name="fade" mode="out-in">
      <div key="2">
        <create-form-base-modal :show="showInitialFormModal" @form-generated="formGenerated"
                                @close="showInitialFormModal=false"
        />
        <form-editor v-if="!workspacesLoading" ref="editor"
                     class="w-full flex flex-grow"
                     :error="error"
                     :is-guest="isGuest"
                     @openRegister="registerModal=true"
        />
        <div v-else class="text-center mt-4 py-6">
          <Loader class="h-6 w-6 text-nt-blue mx-auto"/>
        </div>
      </div>
    </transition>

    <quick-register :show-register-modal="registerModal" @close="registerModal=false" @reopen="registerModal=true"
                    @afterLogin="afterLogin"
    />
  </div>
</template>

<script setup>
import FormEditor from "~/components/open/forms/components/FormEditor.vue"
import QuickRegister from '~/components/pages/auth/components/QuickRegister.vue'
import CreateFormBaseModal from '../../../components/pages/forms/create/CreateFormBaseModal.vue'
import {initForm} from "~/composables/forms/initForm.js"
import {fetchTemplate} from "~/stores/templates.js"
import {fetchAllWorkspaces} from "~/stores/workspaces.js";


const templatesStore = useTemplatesStore()
const workingFormStore = useWorkingFormStore()
const workspacesStore = useWorkspacesStore()
const route = useRoute()

// Fetch the template
if (route.query.template !== undefined && route.query.template) {
  const {data} = await fetchTemplate(route.query.template)
  templatesStore.save(data.value)
}

// Store values
const workspace = computed(() => workspacesStore.getCurrent)
const workspacesLoading = computed(() => workspacesStore.loading)
const form = storeToRefs(workingFormStore).content

useOpnSeoMeta({
  title: 'Create a new Form for free',
})
definePageMeta({
  middleware: "guest"
})

// Data
const stateReady = ref(false)
const loading = ref(false)
const error = ref('')
const registerModal = ref(false)
const isGuest = ref(true)
const showInitialFormModal = ref(false)

// Component ref
const editor = ref(null)

onMounted(() => {
  // Set as guest user
  workspacesStore.set([{
    id: null,
    name: 'Guest Workspace',
    is_enterprise: false,
    is_pro: false
  }])

  form.value = initForm()
  if (route.query.template !== undefined && route.query.template) {
    const template = templatesStore.getByKey(route.query.template)
    if (template && template.structure) {
      form.value = useForm({...form.value.data(), ...template.structure})
    }
  } else {
    // No template loaded, ask how to start
    showInitialFormModal.value = true
  }
  stateReady.value = true
})

const afterLogin = () => {
  registerModal.value = false
  isGuest.value = false
  fetchAllWorkspaces()
  setTimeout(() => {
    if (editor) {
      editor.value.saveFormCreate()
    }
  }, 500)
}

const formGenerated = (newForm) => {
  form.value = useForm({...form.value.data(), ...newForm})
}
</script>
