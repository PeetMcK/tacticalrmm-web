<template>
  <div v-if="!selectedAgent" class="q-pa-sm">No agent selected</div>
  <div v-else-if="!['windows', 'darwin'].includes(agentPlatform.toLowerCase())" class="q-pa-sm">
    Only supported for Windows and macOS agents at this time
  </div>
  <div v-else>
    <q-table
      :table-class="{
        'table-bgcolor': !$q.dark.isActive,
        'table-bgcolor-dark': $q.dark.isActive,
      }"
      class="tabs-tbl-sticky"
      dense
      :rows="software"
      :columns="columns"
      :filter="filter"
      :style="{ 'max-height': tabHeight }"
      v-model:pagination="pagination"
      binary-state-sort
      row-key="id"
      virtual-scroll
      :rows-per-page-options="[0]"
      :loading="loading"
    >
      <template v-slot:loading>
        <q-inner-loading showing color="primary" />
      </template>

      <template v-slot:top>
        <q-btn
          class="q-mr-sm"
          dense
          flat
          push
          @click="refreshSoftware"
          icon="refresh"
          :disable="agentPlatform.toLowerCase() === 'darwin'"
        >
          <q-tooltip v-if="agentPlatform.toLowerCase() === 'darwin'">
            Software refresh not available for macOS
          </q-tooltip>
        </q-btn>
        <q-btn
          icon="add"
          label="Install Software"
          no-caps
          dense
          flat
          push
          @click="showInstallSoftwareModal"
          :disable="agentPlatform.toLowerCase() === 'darwin'"
        >
          <q-tooltip v-if="agentPlatform.toLowerCase() === 'darwin'">
            Software installation not available for macOS
          </q-tooltip>
        </q-btn>

        <q-space />

        <q-input
          v-model="filter"
          outlined
          label="Search"
          dense
          clearable
          class="q-pr-sm"
        >
          <template v-slot:prepend>
            <q-icon name="search" color="primary" />
          </template>
        </q-input>
        <export-table-btn :data="software" :columns="columns" />
      </template>

      <template v-slot:body-cell-uninstall="props">
        <td>
          <q-btn
            v-if="props.row.uninstall"
            label="Uninstall"
            color="primary"
            dense
            size="sm"
            @click="openUninstallSoftware(props.row)"
            :disable="agentPlatform.toLowerCase() === 'darwin'"
          >
            <q-tooltip v-if="agentPlatform.toLowerCase() === 'darwin'">
              Software uninstall not available for macOS
            </q-tooltip>
          </q-btn>
        </td>
      </template>
    </q-table>
  </div>
</template>

<script>
// composition imports
import { ref, computed, watch, onMounted } from "vue";
import { useQuasar } from "quasar";
import { useStore } from "vuex";
import {
  fetchAgentSoftware,
  refreshAgentSoftware,
  uninstallAgentSoftware,
} from "@/api/software";

// ui imports
import InstallSoftware from "@/components/software/InstallSoftware.vue";
import UninstallSoftware from "@/components/software/UninstallSoftware.vue";
import ExportTableBtn from "@/components/ui/ExportTableBtn.vue";
import { notifySuccess } from "@/utils/notify";

export default {
  name: "SoftwareTab",
  components: {
    ExportTableBtn,
  },
  setup() {
    // setup quasar
    const $q = useQuasar();

    // setup vuex
    const store = useStore();
    const selectedAgent = computed(() => store.state.selectedRow);
    const tabHeight = computed(() => store.state.tabHeight);
    const agentPlatform = computed(() => store.state.agentPlatform);

    // platform-specific columns
    const columns = computed(() => {
      const isMacOS = agentPlatform.value?.toLowerCase() === 'darwin';

      if (isMacOS) {
        // macOS: Name, Path, Copyright Info, Version (all sortable, no Size)
        return [
          {
            name: "name",
            align: "left",
            label: "Name",
            field: "name",
            sortable: true,
          },
          {
            name: "publisher",
            align: "left",
            label: "Path",
            field: "publisher",
            sortable: true,
          },
          {
            name: "install_date",
            align: "left",
            label: "Copyright Info",
            field: "install_date",
            sortable: true,
            format: (val) => {
              return val === "01/01/1" || val === "01-1-01" ? "" : val;
            },
          },
          {
            name: "version",
            align: "left",
            label: "Version",
            field: "version",
            sortable: true,
          },
          {
            name: "uninstall",
            align: "left",
            label: "",
            field: "uninstall",
            sortable: false,
          },
        ];
      } else {
        // Windows/Linux: all columns sortable
        return [
          {
            name: "name",
            align: "left",
            label: "Name",
            field: "name",
            sortable: true,
          },
          {
            name: "publisher",
            align: "left",
            label: "Publisher",
            field: "publisher",
            sortable: true,
          },
          {
            name: "install_date",
            align: "left",
            label: "Installed On",
            field: "install_date",
            sortable: true,
            format: (val) => {
              return val === "01/01/1" || val === "01-1-01" ? "" : val;
            },
          },
          {
            name: "size",
            align: "left",
            label: "Size",
            field: "size",
            sortable: true,
          },
          {
            name: "version",
            align: "left",
            label: "Version",
            field: "version",
            sortable: true,
          },
          {
            name: "uninstall",
            align: "left",
            label: "",
            field: "uninstall",
            sortable: false,
          },
        ];
      }
    });

    // software tab logic
    const software = ref([]);
    const loading = ref(false);
    const filter = ref("");
    const pagination = ref({
      rowsPerPage: 0,
      sortBy: "name",
      descending: false,
    });

    async function getSoftware() {
      loading.value = true;
      software.value = (await fetchAgentSoftware(selectedAgent.value)) || [];
      loading.value = false;
    }

    async function refreshSoftware() {
      loading.value = true;
      await refreshAgentSoftware(selectedAgent.value);
      await getSoftware();
      loading.value = false;
    }

    function showInstallSoftwareModal() {
      $q.dialog({
        component: InstallSoftware,
        componentProps: {
          agent_id: selectedAgent.value,
        },
      });
    }

    function openUninstallSoftware(software) {
      $q.dialog({
        component: UninstallSoftware,

        componentProps: {
          softwareName: software.name,
          initialUninstallString:
            software.uninstall +
            (software.uninstall.toLowerCase().includes("msiexec")
              ? " /qn /norestart"
              : ""),
        },
      }).onOk(async (data) => {
        try {
          loading.value = true;
          const ret = await uninstallAgentSoftware(selectedAgent.value, {
            name: software.name,
            command: data.uninstallString, // use user supplied value, not the one from db. to prevent db injection attack
            run_as_user: data.run_as_user,
            timeout: data.timeout,
          });
          notifySuccess(ret);
        } catch (e) {
          console.error(e);
        } finally {
          loading.value = false;
        }
      });
    }

    watch(selectedAgent, (newValue) => {
      if (newValue) {
        getSoftware();
      }
    });

    onMounted(() => {
      if (selectedAgent.value) getSoftware();
    });

    return {
      // reactive data
      software,
      loading,
      filter,
      pagination,
      selectedAgent,
      tabHeight,
      agentPlatform,

      // non-reactive data
      columns,

      // methods
      refreshSoftware,
      showInstallSoftwareModal,
      openUninstallSoftware,
    };
  },
};
</script>
