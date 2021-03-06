<script>
import { mapActions, mapGetters } from 'vuex'

import Actions from '@/pages/TaskRun/Actions'
import BreadCrumbs from '@/components/BreadCrumbs'
import DetailsTile from '@/pages/TaskRun/Details-Tile'
import EditableTextField from '@/components/EditableTextField'
import ExternalLink from '@/components/ExternalLink'
import LogsCard from '@/components/LogsCard/LogsCard'
import DependenciesTile from '@/pages/TaskRun/Dependencies-Tile'
import MappedTaskRunsTile from '@/pages/TaskRun/MappedTaskRuns-Tile'
import SubPageNav from '@/layouts/SubPageNav'
import TaskRunHeartbeatTile from '@/pages/TaskRun/TaskRunHeartbeat-Tile'
import TileLayout from '@/layouts/TileLayout'
import TileLayoutFull from '@/layouts/TileLayout-Full'
import { parser } from '@/utils/markdownParser'

export default {
  metaInfo() {
    return {
      title: this.taskRun
        ? `Task Run | ${this.taskRun?.name ?? this.taskRun?.task?.name}`
        : null,
      link: [
        {
          rel: 'icon',
          type: 'image/svg',
          href: this.taskRun
            ? `/state-icons/${this.taskRun.state.toLowerCase()}.svg`
            : null,
          vmid: 'favicon'
        }
      ]
    }
  },
  components: {
    Actions,
    BreadCrumbs,
    DependenciesTile,
    DetailsTile,
    EditableTextField,
    ExternalLink,
    LogsCard,
    MappedTaskRunsTile,
    SubPageNav,
    TaskRunHeartbeatTile,
    TileLayout,
    TileLayoutFull
  },
  data() {
    return {
      loading: 0,
      tab: this.getTab(),
      taskRunNameLoading: false
    }
  },
  computed: {
    ...mapGetters('tenant', ['tenant']),
    hideOnMobile() {
      return { 'tabs-hidden': this.$vuetify.breakpoint.smAndDown }
    },
    taskRunId() {
      return this.$route.params.id
    },
    // Is this the correct definition? Can a mapped task run have siblings and children?
    mappedParent() {
      return this.taskRun?.task.mapped && this.taskRun?.map_index === -1
    },
    mappedChild() {
      return this.taskRun?.task.mapped && this.taskRun?.map_index > -1
    }
  },
  watch: {
    $route() {
      this.tab = this.getTab()
    },
    tab(val) {
      let query = {}
      switch (val) {
        case 'logs':
          query = { logId: '' }
          break
        case 'mapped-runs':
          query = { 'mapped-runs': '' }
          break
        case 'artifacts':
          query = { artifacts: '' }
          break
        default:
          break
      }
      this.$router
        .replace({
          query: query
        })
        .catch(e => e)
    },
    taskRun(val, prevVal) {
      if (!val || val?.id == prevVal?.id) return
      if (!this.$route.query || !this.$route.query.schematic) {
        this.$router
          .replace({
            query: {
              ...this.$route.query,
              schematic: this.taskRun.task.id
            }
          })
          .catch(e => e)
      }
    }
  },
  methods: {
    ...mapActions('alert', ['setAlert']),
    getTab() {
      if ('logId' in this.$route.query) return 'logs'
      if ('mapped-runs' in this.$route.query) return 'mapped-runs'
      if ('artifacts' in this.$route.query) return 'artifacts'
      return 'overview'
    },
    parseMarkdown(md) {
      return parser(md)
    },
    async saveTaskRunName(e) {
      const previousName = this.taskRun.name
      if (previousName === e) return

      try {
        this.taskRunNameLoading = true

        const { data } = await this.$apollo.mutate({
          mutation: require('@/graphql/Mutations/set-task-run-name.gql'),
          variables: {
            input: {
              task_run_id: this.taskRunId,
              name: e
            }
          }
        })

        await this.$apollo.queries.taskRun.refetch()

        if (!data.set_task_run_name.success) {
          throw new Error(data.set_task_run_name.error)
        }

        this.setAlert({
          alertShow: true,
          alertMessage: `<span class="font-weight-medium">${previousName ||
            'Your task run'}</span> has been renamed to <span class="font-weight-medium">${e}</span>`,
          alertType: 'success'
        })
      } catch {
        this.setAlert({
          alertShow: true,
          alertMessage:
            'Oops! Something went wrong while trying to update your task run name, please try again.',
          alertType: 'error'
        })
      } finally {
        this.taskRunNameLoading = false
      }
    }
  },
  apollo: {
    taskRun: {
      query: require('@/graphql/TaskRun/task-run.gql'),
      variables() {
        return {
          id: this.taskRunId
        }
      },
      loadingKey: 'loading',
      pollInterval: 5000,
      update: data => data.task_run_by_pk
    },
    parent: {
      query: require('@/graphql/TaskRun/parent.gql'),
      variables() {
        return {
          taskId: this.taskRun ? this.taskRun.task.id : null,
          flowRunId: this.taskRun ? this.taskRun.flow_run.id : null
        }
      },
      loadingKey: 'loading',
      pollInterval: 5000,
      update: data => (data.task_run ? data.task_run.length : null)
    }
  }
}
</script>

<template>
  <v-sheet v-if="taskRun" color="appBackground">
    <SubPageNav>
      <span slot="page-type">Task Run</span>
      <span
        slot="page-title"
        style="
          display: block;
          max-width: 100%;
          min-width: 300px;
          width: auto;
          "
      >
        <EditableTextField
          :content="
            taskRun.name ||
              `${taskRun.task.name} ${
                taskRun.map_index > -1
                  ? `(Mapped Child ${taskRun.map_index})`
                  : ''
              }`
          "
          label="Task run name"
          :loading="taskRunNameLoading"
          required
          @change="saveTaskRunName"
        />
      </span>
      <BreadCrumbs
        slot="breadcrumbs"
        :crumbs="[
          {
            route: {
              name: 'project',
              params: { id: taskRun.flow_run.flow.project.id }
            },
            text: taskRun.flow_run.flow.project.name
          },
          {
            route: {
              name: 'flow',
              params: { id: taskRun.flow_run.flow.flow_group_id }
            },
            text: taskRun.flow_run.flow.name
          },
          {
            route: {
              name: 'flow-run',
              params: { id: taskRun.flow_run.id }
            },
            text: taskRun.flow_run.name
          }
        ]"
      ></BreadCrumbs>

      <Actions slot="page-actions" :task-run="taskRun" />
    </SubPageNav>

    <v-tabs
      v-model="tab"
      class="px-6 mx-auto tabs-border-bottom"
      :class="hideOnMobile"
      style="max-width: 1440px;"
      light
    >
      <v-tabs-slider color="blue"></v-tabs-slider>

      <v-tab href="#overview" :style="hideOnMobile">
        <v-icon left>view_module</v-icon>
        Overview
      </v-tab>

      <v-tab href="#logs" :style="hideOnMobile">
        <v-icon left>format_align_left</v-icon>
        Logs
      </v-tab>

      <v-menu
        open-on-hover
        :close-on-click="false"
        :open-on-click="false"
        :close-on-content-click="false"
        offset-y
      >
        <template #activator="{on}">
          <div v-on="on">
            <!-- Height: 100% is required here since we're nesting the tab -->
            <v-tab
              href="#artifacts"
              :style="hideOnMobile"
              style="height: 100%;"
              disabled
            >
              <v-badge
                color="codePink"
                content="Coming Soon!"
                bottom
                bordered
                inline
              >
                <v-icon left>fas fa-fingerprint</v-icon>
                Artifacts
              </v-badge>
            </v-tab>
          </div>
        </template>
        <v-card tile class="pa-0" max-width="320">
          <v-card-title>
            <v-badge
              color="codePink"
              content="Coming Soon!"
              bottom
              bordered
              inline
            >
              <v-icon left>fas fa-fingerprint</v-icon>
              Artifacts
            </v-badge>
          </v-card-title>
          <v-card-text>
            <div>
              The Artifacts API is an experimental feature set currently under
              development. For a sneak preview, check out the
              <ExternalLink
                href="https://docs.prefect.io/api/latest/artifacts/artifacts.html#artifacts"
              >
                Artifacts API docs
              </ExternalLink>
              !
            </div>
          </v-card-text>
        </v-card>
      </v-menu>

      <v-tab
        v-if="mappedParent || mappedChild"
        href="#mapped-runs"
        :style="hideOnMobile"
      >
        <v-badge color="codePink" content="New!" bottom bordered inline>
          <v-icon left>device_hub</v-icon>
          Mapped Runs
        </v-badge>
      </v-tab>
    </v-tabs>

    <v-tabs-items
      v-model="tab"
      class="px-6 mx-auto tabs-border-bottom"
      style="max-width: 1440px;"
    >
      <v-tab-item
        class="tab-full-height pa-0"
        value="overview"
        transition="quick-fade"
        reverse-transition="quick-fade"
      >
        <TileLayout>
          <DetailsTile slot="row-2-col-1-row-1-tile-1" :task-run="taskRun" />

          <TaskRunHeartbeatTile
            slot="row-2-col-1-row-4-tile-1"
            :task-run-id="$route.params.id"
          />

          <DependenciesTile
            slot="row-2-col-2-row-3-tile-1"
            :task-run="taskRun"
            :loading="loading > 0"
          />
        </TileLayout>
      </v-tab-item>

      <v-tab-item
        class="tab-full-height"
        value="logs"
        transition="quick-fade"
        reverse-transition="quick-fade"
      >
        <TileLayoutFull>
          <LogsCard
            slot="row-2-tile"
            class="py-2 mt-4"
            entity="task"
            :query="require('@/graphql/Logs/task-run-logs.gql')"
            :query-for-scoping="
              require('@/graphql/Logs/task-run-logs-scoping.gql')
            "
            query-key="task_run_by_pk"
            :variables="{ id: $route.params.id }"
          />
        </TileLayoutFull>
      </v-tab-item>

      <v-tab-item
        class="tab-full-height"
        value="artifacts"
        transition="quick-fade"
        reverse-transition="quick-fade"
      >
        <!-- <div v-html="parseMarkdown('# hello')"></div> -->
      </v-tab-item>

      <v-tab-item
        class="tab-full-height"
        value="mapped-runs"
        transition="quick-fade"
        reverse-transition="quick-fade"
      >
        <TileLayoutFull>
          <v-skeleton-loader
            slot="row-2-tile"
            :loading="loading > 0"
            type="image"
            height="800"
            transition="quick-fade"
            class="my-2"
            tile
          >
            <MappedTaskRunsTile :task-run="taskRun" />
          </v-skeleton-loader>
        </TileLayoutFull>
      </v-tab-item>
    </v-tabs-items>

    <v-bottom-navigation v-if="$vuetify.breakpoint.smAndDown" fixed>
      <v-btn @click="tab = 'overview'">
        Overview
        <v-icon>view_module</v-icon>
      </v-btn>

      <v-btn @click="tab = 'logs'">
        Logs
        <v-icon>format_align_left</v-icon>
      </v-btn>

      <v-btn disabled @click="tab = 'artifacts'">
        Artifacts
        <v-icon>fas fa-fingerprint</v-icon>
      </v-btn>

      <v-btn @click="tab = 'mapped-runs'">
        Mapped Runs
        <v-icon>device_hub</v-icon>
      </v-btn>
    </v-bottom-navigation>
  </v-sheet>
</template>

<style lang="scss">
.custom-tab-active {
  background-color: #c8e1ff !important;
}

.tab-full-height {
  min-height: 80vh;
}

/* stylelint-disable */

.v-tab--disabled .v-badge__badge {
  pointer-events: none;
}

.v-badge--inline .v-badge__badge,
.v-badge--inline .v-badge__wrapper {
  margin: 5px;
}
</style>
