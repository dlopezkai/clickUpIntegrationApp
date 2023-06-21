<template>
  <div>
    <v-card>
      <v-card-title>
        <span class="text-h5">{{ formTitle }}</span>
      </v-card-title>

      <v-card-text>
        
      <v-tabs v-if="props.recordId" v-model="formTab" color="#428086">
        <v-tab value="one">Details</v-tab>
        <v-tab value="two">Comments</v-tab>
      </v-tabs>

      <v-window v-model="formTab">

        <v-window-item value="one">
          <v-form ref="form" @submit.prevent="submit">
            <v-row>
              <v-col cols="12" sm="12" md="12">
                <v-text-field v-model="editedItem.name" label="Name" 
                  :rules="[rules.required]"></v-text-field>
              </v-col>

              <v-col cols="12" sm="6" md="6">
                <v-select v-model="editedItem.folder" label="Project" :items="folders" item-title="name" item-value="id" @update:modelValue="loadLists()" :rules="[rules.select]"></v-select>
              </v-col>
              <v-col cols="12" sm="6" md="6">
                <v-select v-model="editedItem.list" label="Subtask" :items="lists" item-title="name" item-value="id" :rules="[rules.select]"></v-select>
              </v-col>

              <v-col cols="12" sm="12" md="12">
                <v-select v-model="editedItem.tags" label="Type" :items="tags" item-title="title" item-value="value" multiple chips clearable></v-select>
              </v-col>

              <v-col v-if="props.recordId" cols="12" sm="6" md="6">
                <v-select v-model="editedItem.status" label="Status" :items="statuses" item-title="title" item-value="value"></v-select>
              </v-col>
              <v-col cols="12" sm="6" md="6">
                <v-select v-model="editedItem.assignees" label="Assignee(s)" :items="members" item-title="title" item-value="value" multiple chips clearable></v-select>
              </v-col>

              <v-col cols="12" sm="6" md="6">
                <v-text-field v-model="editedItem.due_date" label="Due Date" type="date" :rules="[rules.due_date, rules.due_date_threshold]"></v-text-field>
              </v-col>
              <v-col cols="12" sm="6" md="6">
                <v-select v-model="editedItem.watchers" label="Notify Person" :items="members" item-title="title" item-value="value" multiple chips clearable></v-select>
              </v-col>

              <v-col cols="12" sm="6" md="6">
                <v-text-field v-model="editedItem.time_estimate" label="Budgeted Hours"></v-text-field>
              </v-col>
              <v-col cols="12" sm="6" md="6">
                <v-select v-model="editedItem.priority" label="Priority" :items="priorities" item-title="title" item-value="value" :hint="priorityMessages" persistent-hint></v-select>
              </v-col>

              <v-col cols="12" sm="12" md="12">
                <QuillEditor
                  v-model:content="editedItem.description" 
                  contentType="html" 
                  theme="snow" 
                  placeholder="Description"
                  toolbar="essential" 
                  style="height:200px; max-height:250px;"
                ></QuillEditor>
              </v-col>

              <v-col cols="12" sm="12" md="12" class="mt-5">
                <v-text-field v-model="editedItem.links" label="SharePoint File"></v-text-field>
                <v-btn href="https://kauffmaninc.sharepoint.com/" target="_blank" variant="tonal" class="rounded" color="#428086">Open SharePoint site</v-btn>
              </v-col>

              <!-- hide for production -->
              <v-col v-if="editedItem.url" cols="12" sm="12" md="12">
                <v-btn :href="editedItem.url" target="_blank" variant="text">ClickUp reference link</v-btn>
              </v-col>

              <v-col class="text-right">
                <v-btn variant="plain" @click="close">Cancel</v-btn>
                <v-btn :disabled="submitBtnDisabled" class="rounded" color="blue" @click="submit">{{ submitBtnText }}</v-btn>
              </v-col>
            </v-row>
          </v-form>
        </v-window-item>

        <v-window-item v-if="props.recordId" value="two">
          <!-- <comments-comp taskid=866a8z405 :clickUpUserInfo="clickUpUserInfo"></comments-comp> -->
          <comments-comp :taskid="props.recordId" :clickUpUserInfo="props.clickUpUserInfo"></comments-comp>
        </v-window-item>

      </v-window>

      </v-card-text>
    </v-card>

  </div>
</template>

<script setup>
import axios from 'axios'
import CommentsComp from './CommentsComp.vue';
import { convertToDate, dateToISOStr, hoursToMilliseconds } from '~/helpers/datetimeConversions.js';

const runtimeConfig = useRuntimeConfig()
const folders = ref([])
const lists = ref([])
const folderIDTemp = ref()
const formTab = ref(null)
const props = defineProps({
    recordId: String,
    clickUpUserInfo: Object,
})

const editedItem = ref([
  {
    assignees: '',
    creator: '',
    description: '',
    due_date: '',
    folder: '',
    id: '',
    links: '',
    list: '',
    name: '',
    priority: '',
    project: '',
    status: '',
    tags: '',
    time_estimate: '',
    watchers: ''
  },
])

const formTitle = computed(() => {
  return (!props.recordId) ? 'New Work Order Form' : 'Edit Work Order Form'
})

// form field validation rules
const rules =
{
  required: v => !!v || 'Field is required',
  length: v => v.length >= 3 || 'Minimum length is 3 characters',
  select: v => !!v || 'Select a valid option',
  due_date: v => !!v || 'Date must be selected',
  due_date_threshold: v => dateValidation(v) || 'Date must be 2 business days from today',
}

// checks for the 2 business days rule
function dateValidation(input) {
  // get day of week
  let selectedDateDay = new Date(input).getDay()

  // get the current date plus 2 days, the convert to ISO format
  let date = new Date();
  let twoDaysFromNow = date.setDate(date.getDate() + 2);
  twoDaysFromNow = new Date(twoDaysFromNow).toISOString();

  // convert to local time
  let twoDaysFromNowLocaleString = new Date(twoDaysFromNow).toLocaleDateString()
  let twoDaysFromNowDateObj = new Date(twoDaysFromNowLocaleString)

  // convert to yyyy-mm-dd to match format of calendar input
  let twoDaysFromNowFormatted = convertToYyyymmddFormat(twoDaysFromNowDateObj)

  // logic to determine if selected date is valid
  if(selectedDateDay !== 5 && selectedDateDay !== 6) {
    if(input >= twoDaysFromNowFormatted) {
      return true
    } else {
      return false
    }
  }
}

function convertToYyyymmddFormat(value) {
  return value.getFullYear() 
    + "-" 
    + ((value.getMonth()+1).length != 2 ? "0" + (value.getMonth() + 1) : (value.getMonth()+1)) 
    + "-" 
    + (value.getDate().length != 2 ? "0" + value.getDate() : value.getDate());
}

function millisecondsToHours(value) {
  if(value) {
    const hours = (value / 1000 / 60 / 60).toFixed(2)

    return hours
  }
}

onMounted(() => {
  loadItem()
  loadFolders()
})

async function loadItem() {
  // axios.get(`${runtimeConfig.public.API_URL}/task/` + props.recordId)
  //   .then((response) => {
  //     editedItem.value = Object.assign({}, response.data.data)
  //     loadLists(editedItem.value.folder.id)
  //   })
  //   .catch(err => console.log(err))

  // convert time estimate (milliseconds) to hours if not a new work order
  if (props.recordId) {
    await axios.get(`${runtimeConfig.public.API_URL}/task/` + props.recordId)
    .then((response) => {
      editedItem.value = Object.assign({}, response.data.data)
      editedItem.value.priority = (response.data.data.priority != null) ? capitalizeFirstLetter(response.data.data.priority.priority) : null
      editedItem.value.due_date = (response.data.data.due_date != null) ? convertToDate(response.data.data.due_date, "table") : null
      editedItem.value.time_estimate = millisecondsToHours(response.data.data.time_estimate)
      
    })
    .then(() => {
      loadLists(editedItem.value.folder.id)
    })
    .catch(err => console.log(err))


    loadFolders()
    folderIDTemp.value = editedItem.value.folder.id
    
    // loadLists(item.folder)
    // editedItem.value = Object.assign({}, item)
    // editedItem.value.priority = (item.priority != null) ? capitalizeFirstLetter(item.priority.priority) : null
    // editedItem.value.due_date = (item.due_date != null) ? convertToDate(item.due_date, "table") : null
    // editedItem.value.time_estimate = millisecondsToHours(item.time_estimate)

    // get list of watchers and assign it to the editedItem object
    // axios.get(`${runtimeConfig.public.API_URL}/task/` + item.id)
    // .then((response) => {
    //   editedItem.value.watchers = response.data.data.watchers
    // })
    // .catch(err => console.log(err))
  } else {
    editedItem.value = Object.assign({}, '')
  }

  // dialog.value = true
}

function loadFolders() {
  // load folder options
  axios.get(`${runtimeConfig.public.API_URL}/folders`)
  .then((response) => {
    folders.value = response.data.data.map((item) => {
      return {
        id: item.id,
        name: item.name
      }
    })
  })
  .catch(err => console.log(err))
}

function loadLists(presentFolderId) {
  // if the folder ID changes, then clear the selected list / subtask
  if ((folderIDTemp.value) && folderIDTemp.value !== presentFolderId) {
    // clear list/subtask
    editedItem.value.list = ''

    // clear list/subtask options
    lists.value = ''

    // reset temp ID
    folderIDTemp.value = editedItem.value.folder
  }

  // get selected folder ID
  let folderId = (presentFolderId) ? presentFolderId : editedItem.value.folder

  // load list/subtask options
  axios.get(`${runtimeConfig.public.API_URL}/folder/` + folderId + `/lists`)
  .then((response) => {
    lists.value = response.data.data.map((item) => {
      return {
        id: item.id,
        name: item.name
      }
    })
  })
  .catch(err => console.log(err))
}

</script>

<style scoped>

</style>