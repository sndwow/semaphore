<template xmlns:v-slot="http://www.w3.org/1999/XSL/Transform">
  <div v-if="items != null">
    <v-toolbar flat>
      <v-app-bar-nav-icon @click="showDrawer()"></v-app-bar-nav-icon>
      <v-toolbar-title>
        {{ $t('dashboard2') }}
      </v-toolbar-title>
    </v-toolbar>

    <DashboardMenu
      :project-id="projectId"
      :project-type="projectType"
      :can-update-project="can(USER_PERMISSIONS.updateProject)"
    />

    <v-data-table
      :headers="headers"
      :items="items"
      :footer-props="{ itemsPerPageOptions: [20] }"
      class="mt-4 HistoryTable"
    >
      <template v-slot:item.tpl_alias="{ item }">
        <div class="d-flex align-center">
          <TaskLink
            :task-id="item.id"
            :label="'#' + item.id"
          />

          <v-icon small class="ml-1 mr-1">mdi-arrow-left</v-icon>

          <router-link :style="tplStyle(item.tpl_alias)" :to="
            '/project/' + item.project_id +
            '/templates/' + item.template_id"
          >{{ item.tpl_alias }}
          </router-link>
        </div>

        <div style="font-size: 14px;
            margin-left: 0 !important;
            white-space: normal;
            word-break: break-word;"
          class="ml-7">
            <span v-if="item.message">
              {{ item.message }}
            </span>
        </div>
      </template>

      <template v-slot:item.status="{ item }">
        <TaskStatus :status="item.status"/>
      </template>

      <template v-slot:item.start="{ item }">
        {{ item.start | formatDate }}
      </template>

      <template v-slot:item.end="{ item }">
        {{ [item.start, item.end] | formatMilliseconds }}
      </template>
    </v-data-table>
  </div>
</template>

<style lang="scss">
.HistoryTable td {
  height: 60px !important;
}
</style>

<script>
import ItemListPageBase from '@/components/ItemListPageBase';
import EventBus from '@/event-bus';
import TaskStatus from '@/components/TaskStatus.vue';
import TaskLink from '@/components/TaskLink.vue';
import socket from '@/socket';
import { TEMPLATE_TYPE_ICONS } from '@/lib/constants';
import AppsMixin from '@/components/AppsMixin';
import DashboardMenu from '@/components/DashboardMenu.vue';

export default {
  mixins: [ItemListPageBase, AppsMixin],

  data() {
    return { TEMPLATE_TYPE_ICONS };
  },

  components: { DashboardMenu, TaskStatus, TaskLink },

  watch: {
    async projectId() {
      await this.loadItems();
    },
  },

  created() {
    socket.addListener((data) => this.onWebsocketDataReceived(data));
  },

  methods: {
    showTaskLog(taskId) {
      EventBus.$emit('i-show-task', {
        taskId,
      });
    },
    tplStyle(taskName) {
      if (taskName.includes('prod')) {
        return { color: '#f32626' };
      }
      if (taskName.includes('pre')) {
        return { color: '#cf921d' };
      }
      if (taskName.includes('test')) {
        return { color: '#1c8f1c' };
      }
      return {};
    },

    async onWebsocketDataReceived(data) {
      if (data.project_id !== this.projectId || data.type !== 'update') {
        return;
      }

      if (!this.items.some((item) => item.id === data.task_id)) {
        await this.loadItems();
      }

      const task = this.items.find((item) => item.id === data.task_id);

      if (task) {
        Object.assign(task, {
          ...data,
          type: undefined,
        });
      }
    },

    getHeaders() {
      return [
        {
          text: this.$i18n.t('task2'),
          value: 'tpl_alias',
          sortable: false,
        },
        {
          text: this.$i18n.t('playbook'),
          value: 'tpl_playbook',
          sortable: false,
        },
        {
          text: this.$i18n.t('start'),
          value: 'start',
          sortable: false,
        },
        {
          text: this.$i18n.t('duration'),
          value: 'end',
          sortable: false,
        },
        {
          text: this.$i18n.t('user'),
          value: 'user_name',
          sortable: false,
        },
        {
          text: this.$i18n.t('status'),
          value: 'status',
          sortable: false,
        },
      ];
    },

    getItemsUrl() {
      return `/api/project/${this.projectId}/tasks/last`;
    },
  },
};
</script>
