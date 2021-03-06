<template>
  <div class="q-pa-md">
    <q-table
      title="Tasks"
      :data="mappedData"
      :columns="defaultColumns"
      row-key="id"
      :dense="$q.screen.lt.md"
      :grid="$q.screen.xs || !isAdmin"
      separator="cell"
      selection="single"
      :selected.sync="selected"
      :filter="filter"
      :filter-method="protectedDeepSearch"
      :pagination.sync="pagination">

      <template v-slot:top>
        <h5 class="q-ma-xs">Tasks</h5>
        <q-space />
        <q-input placeholder="Search" dense outlined class="q-ml-md" debounce="300" color="primary" v-model="filter">
          <template v-slot:append>
            <q-icon name="search" />
          </template>
        </q-input>
      </template>

      <template v-slot:item="props">
        <div class="q-pa-xs col-xs-12 col-sm-6 col-md-4">
          <q-card :class="props.selected ? 'bg-grey-2' : ''">
            <q-card-section>
              <q-checkbox v-if="isAdmin" dense v-model="props.selected" :label="props.row.title" />
              <span v-if="!isAdmin">
                <q-avatar v-if="isTaskJoined(props.row)" class="q-mr-md" icon="done" color="primary" text-color="white" />
                <LabelDiv :label="props.row.title" width="200px" xsWidth="200px" />
                <q-btn color="primary" label="Details" :to="'/task/details/'+props.row.id" />
              </span>
            </q-card-section>

            <q-separator inset />

            <q-list dense>
              <q-item v-for="col in defaultColumns" :key="col.name">
                <q-item-section>
                  <q-item-label caption>{{ col.label }}</q-item-label>
                </q-item-section>
                <q-item-section side>
                  <q-item-label>{{ props.row[col.name] }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </q-card>
        </div>
      </template>

    </q-table>

    <q-btn v-if="isAdmin" class="q-ma-md" color="primary" :disable="!selected.length" label="Edit" :to="'/task/edit/'+(selected[0] || {}).id" />

    <q-btn v-if="isAdmin" class="q-ma-md" color="primary" label="New Task" :to="'/task/new/0'" />

    <q-btn v-if="isAdmin" class="q-ma-md" color="primary" :disable="!selected.length" label="Details" :to="'/task/details/'+(selected[0] || {}).id" />

    <EditDialog :show="!!dialogState" :objGetter="getTaskForDialog" :label="dialogLabel" :columns="editColumns" @close="onCloseNewEditDialog" :readonly="details" :labelCancel="details ? 'Close' : 'Cancel'">
      <template v-slot:customItems="props">
        <span v-if="props.editing">
          <q-item key="managerName">
            <q-item-section>
              <q-select
                v-model="props.editing.manager"
                :options="Object.values(managers)"
                :readonly="details"
              >
                <template v-slot:before>
                  <LabelDiv label="Manager" />
                </template>
              </q-select>
            </q-item-section>
          </q-item>

          <q-item key="statusStr">
            <q-item-section>
              <q-select
                v-model="props.editing.statusObj"
                :options="Object.keys(taskStatusEnum).map(n => ({label: taskStatusEnum[n], value: n}))"
                :readonly="details"
              >
                <template v-slot:before>
                  <LabelDiv label="Status" />
                </template>
              </q-select>
            </q-item-section>
          </q-item>

          <q-item key="description">
            <q-item-section>
              <q-input dense outlined autogrow :readonly="details" v-model="props.editing.description">
                <template v-slot:before>
                  <LabelDiv label="Description" />
                </template>
              </q-input>
            </q-item-section>
          </q-item>

          <q-item key="volunteers" v-if="!newTask && isAdmin">
            <q-item-section>
              <q-table
                title="Volunteers"
                :data="props.editing.volunteers"
                :columns="volunteersColumns"
                row-key="id"
                dense
                separator="cell"
                :filter="filterVolunteer"
                :filter-method="deepSearch">

                <template v-slot:top>
                  <h5 class="q-ma-xs">Volunteers</h5>
                  <q-space />
                  <span v-if="edit" style="color:red">click on a status to edit</span>
                  <q-space />
                  <q-input placeholder="Search" dense outlined class="q-ml-md" debounce="300" color="primary" v-model="filterVolunteer">
                    <template v-slot:append>
                      <q-icon name="search" />
                    </template>
                  </q-input>
                </template>

                <template v-slot:body-cell-status="props">
                  <q-td :props="props">
                    {{ props.row.statusObj.label }}
                    <q-popup-edit v-if="!details" v-model="props.row.statusObj" buttons persistent title="Edit the Status">
                      <q-select
                        dense
                        v-model="props.row.statusObj"
                        :options="Object.keys(volunteerStatusEnum).map(n => ({label: volunteerStatusEnum[n], value: n}))"
                      />
                    </q-popup-edit>
                  </q-td>
                </template>

              </q-table>
            </q-item-section>
          </q-item>
        </span>
      </template>

      <template v-slot:buttons="props" v-if="details && !isAdmin">
        <q-btn class="q-ma-md" color="primary" label="Join" :disable="!!isTaskJoined(props.editing)" :to="'/task/join/'+props.editing.id" />

        <q-dialog v-model="join" persistent>
          <q-card>
            <q-card-section class="items-center">
              <q-avatar icon="done" color="primary" text-color="white" />
              <span class="q-ml-sm">Are you sure you want to join this task?</span>
              <br/>
              <span v-if="props.editing">{{ props.editing.title }}</span>
            </q-card-section>

            <q-card-actions align="right">
              <q-btn flat label="Cancel" color="primary" @click="onJoin()" />
              <q-btn flat label="Join" color="primary" @click="onJoin(props.editing)" />
            </q-card-actions>
          </q-card>
        </q-dialog>
      </template>

    </EditDialog>
  </div>
</template>

<script>
import cloneObject from '../utils/cloneObject'
import defaultColumns from '../utils/defaultColumns'
import taskStatusEnum from '../utils/taskStatusEnum'
import volunteerStatusEnum from '../utils/volunteerStatusEnum'
import deepSearch from '../utils/deepSearch'

import EditDialog from 'components/EditDialog'
import LabelDiv from 'components/LabelDiv'

export default {
  name: 'TasksTable',
  props: {
  },

  components: {
    EditDialog,
    LabelDiv
  },

  data () {
    return {
      selected: [],
      filter: '',
      filterVolunteer: '',
      taskStatusEnum,
      volunteerStatusEnum,
      pagination: {
        rowsPerPage: 25
      },
      volunteersColumns: [
        { name: 'full_name', required: true, label: 'full_name', field: 'full_name', align: 'left', sortable: true },
        { name: 'email', required: true, label: 'email', field: 'email', align: 'left', sortable: true },
        { name: 'status', required: true, label: 'status', field: 'status', align: 'left', sortable: true }
      ],
      columns: [
        { name: 'managerName', required: true, label: 'Manager', align: 'left', field: 'managerName', sortable: true, hasCustomEdit: true },
        { name: 'title', required: true, label: 'Task Title', align: 'left', field: 'title', sortable: true },
        { name: 'estimation', required: true, label: 'Estimation', align: 'left', field: 'estimation', sortable: true },
        { name: 'description', required: true, label: 'Description', align: 'left', field: 'description', sortable: true, hasCustomStyle: true, hasCustomEdit: true },
        { name: 'phone', required: true, label: 'Phone', align: 'left', field: 'phone', sortable: true },
        { name: 'email', required: true, label: 'Email', align: 'left', field: 'email', sortable: true },
        { name: 'wanted_volunteers', required: true, label: 'Wanted Volunteers', align: 'left', field: 'wanted_volunteers', sortable: true },
        { name: 'statusStr', required: true, label: 'Status', align: 'left', field: 'statusStr', sortable: true, hasCustomEdit: true }
      ]
    }
  },

  computed: {
    defaultColumns,
    editColumns () {
      return this.columns.filter(col => !col.hasCustomEdit)
    },
    userRole () {
      return this.$store.state.user.role
    },
    isAdmin () {
      return this.userRole === 'admin'
    },
    managers () {
      return this.$store.state.managers.data
        .map(obj => Object.assign({ label: obj.name, value: obj }, obj))
        .reduce((dict, obj) => {
          dict[obj.id] = obj
          return dict
        }, {})
    },
    tasks () {
      return this.$store.state.tasks.data
    },
    mappedData () {
      return this.tasks.map(this.mapRow)
    },
    loggedInVolunteer () {
      return this.$store.state.user
    },
    dialogTaskId () {
      return Number(this.$route.params?.taskId)
    },
    dialogState () {
      return this.$route.params?.dialogState
    },
    emptyTask () {
      const t = { volunteers: [] }
      this.columns.forEach(({ field }) => { t[field] = '' })
      return t
    },
    dialogLabel () {
      const state = this.dialogState
      switch (state) {
        case 'edit': return 'Edit Task'
        case 'new': return 'New Task'
        case 'details':
        case 'join':
          return 'Task Details'
      }
      return ''
    },
    details () {
      return this.dialogState === 'details' || this.join
    },
    newTask () {
      return this.dialogState === 'new'
    },
    edit () {
      return this.dialogState === 'edit'
    },
    join () {
      return this.dialogState === 'join'
    }
  },

  methods: {
    cloneObject,
    deepSearch,
    protectedDeepSearch (rows, terms, cols, getCellValue) {
      if (this.isAdmin) {
        return this.deepSearch(rows, terms, cols, getCellValue)
      }
      return this.deepSearch(rows.map(r => Object.assign({}, r, { volunteers: [] })), terms, cols, getCellValue)
    },
    mapRow (row) {
      const m = cloneObject(row)
      if (m.managerId) {
        m.managerName = this.managers[m.managerId]?.name
        m.manager = this.managers[m.managerId]
      }
      if (m.status) {
        m.statusStr = taskStatusEnum[m.status]
        m.statusObj = { label: this.taskStatusEnum[m.status], value: m.status }
      }
      m.volunteers = m.volunteers.map(v => {
        v.statusObj = { label: this.volunteerStatusEnum[v.status], value: v.status }
        return v
      })
      return m
    },
    getTaskForDialog () {
      const v = this.$store.getters['tasks/getTask'](this.dialogTaskId)
      const ret = v ? cloneObject(v) : this.emptyTask
      return this.mapRow(ret)
    },
    onCloseNewEditDialog (newValue) {
      this.$router.go(-1)

      if (newValue) {
        if (this.edit) {
          this.$store.commit('tasks/editTask', { newValue, columns: this.columns })
        } else if (this.newTask) {
          this.$store.commit('tasks/addTask', newValue)
        }
      }
    },
    onJoin (task) {
      this.$router.go(-1)
      if (task) {
        this.$store.commit('tasks/joinVolunteer', { task, volunteer: this.loggedInVolunteer })
      }
    },
    isTaskJoined (task) {
      return task.volunteers.filter(({ id }) => id === this.loggedInVolunteer.id).length
    }
  }
}
</script>
