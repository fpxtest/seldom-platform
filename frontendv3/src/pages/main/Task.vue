<script setup lang="ts">
import { reactive, onMounted, h, ref } from "vue";
import {
  NButton,
  NIcon,
  useMessage,
  useDialog,
  SelectOption,
  FormInst,
  NSpace,
} from "naive-ui";
import type { DataTableColumns } from "naive-ui";
import { SearchOutline } from "@vicons/ionicons5";
import TaskApi from "~/request/task";
import TeamApi from "~/request/team";
import TaskReport from "@/TaskReport.vue";
import TaskModal from "@/TaskModal.vue";

type RowData = {
  id: number;
  name: string;
  env: string;
  team: string;
  execute_count: number;
  status: number;
  create_time: string;
};

const createColumns = ({
  play,
}: {
  play: (row: RowData, action: String) => void;
}): DataTableColumns<RowData> => {
  return [
    {
      title: "ID",
      key: "id",
    },
    {
      title: "任务名称",
      key: "name",
      render(row) {
        return h(
          NButton,
          {
            type: "info",
            quaternary: true,
            onClick: () => play(row, "clickTaskName"),
          },
          { default: () => row.name }
        );
      },
    },
    {
      title: "环境",
      key: "env",
    },
    {
      title: "团队",
      key: "team",
    },
    {
      title: "执行次数",
      key: "execute_count",
    },
    {
      title: "状态",
      key: "status",
      render(row) {
        if (row.status === 0) {
          return "未执行";
        } else if (row.status === 1) {
          return "执行中";
        } else if (row.status === 2) {
          return "已执行";
        } else {
          return "未知";
        }
      },
    },
    {
      title: "创建",
      key: "create_time",
    },
    {
      title: "操作",
      key: "actions",
      render(row) {
        return h(
          NSpace,
          {},
          {
            default: () => [
              h(
                NButton,
                {
                  type: "info",
                  strong: true,
                  secondary: true,
                  size: "small",
                  onClick: () => play(row, "run"),
                },
                { default: () => "运行" }
              ),
              // h(
              //   NButton,
              //   {
              //     type: "info",
              //     strong: true,
              //     secondary: true,
              //     onClick: () => play(row, "set"),
              //   },
              //   { default: () => "定时" }
              // ),
              h(
                NButton,
                {
                  type: "primary",
                  strong: true,
                  secondary: true,
                  size: "small",
                  onClick: () => play(row, "edit"),
                },
                { default: () => "编辑" }
              ),
              h(
                NButton,
                {
                  type: "error",
                  strong: true,
                  secondary: true,
                  size: "small",
                  onClick: () => play(row, "delete"),
                },
                { default: () => "删除" }
              ),
            ],
          }
        );
      },
    },
  ];
};

type Tdatas = {
  loading: boolean;

  projectId: string;
  caseNumber: number;
  tableData: [];
  teamIdSelected: string;
  tid?: number;
  query: {
    project_id: null;
    team_id: null;
    name: null;
  };
  taskFlag: boolean;
};
const datas = reactive<Tdatas>({
  loading: true,
  projectId: "",
  caseNumber: 0,
  tableData: [],
  teamIdSelected: "",
  tid: 0,
  query: {
    project_id: null,
    team_id: null,
    name: null,
  },
  taskFlag: true,
});

type ModalDatas = {
  type?: number;
  taskId?: number;
  title: string;
};

const modalDatas = reactive<ModalDatas>({
  title: "",
  type: 1,
});

const message = useMessage();
const dialog = useDialog();

type TteamOptions = {
  value: string;
  label: string;
};
type TteamModel = {
  teamOptions: TteamOptions[];
  envOptions: [];
};

const model = ref<TteamModel>({
  teamOptions: [],
  envOptions: [],
});

// 初始化获取团队列表
const initTeamList = async () => {
  const resp = await TeamApi.getTeamAll();
  if (resp.success === true) {
    for (let i = 0; i < resp.result.length; i++) {
      model.value.teamOptions.push({
        value: resp.result[i].id,
        label: resp.result[i].name,
      });
    }
  } else {
    message.error("查询失败!");
  }
};

const changeTeam = (value: string, option: SelectOption) => {
  datas.teamIdSelected = value;
};

const initTaskList = async () => {
  datas.query.project_id = sessionStorage.projectId;
  if (datas.query.project_id == null || datas.query.project_id == undefined) {
    message.warning("请选择项目");
  } else {
    datas.loading = true;
    const resp = await TaskApi.getTaskAll(datas.query);
    if (resp.success === true) {
      datas.tableData = resp.result;
    } else {
      message.error("获得任务列表失败！");
    }
    datas.loading = false;
  }
};

// 新增编辑任务modal
const showModalTask = ref(false);

const openModalTask = (type: number) => {
  if (sessionStorage.projectId == "") {
    message.warning("请选择项目");
  } else {
    switch (type) {
      case 1:
        modalDatas.title = "新增任务";
        modalDatas.type = 1;
        datas.tid = undefined;
        break;
      case 2:
        modalDatas.title = "编辑任务";
        modalDatas.type = 2;
        break;
    }
    showModalTask.value = true;
  }
};

const segmented = {
  content: "soft",
  footer: "soft",
};

const formRef = ref<FormInst | null>(null);

const options = [{}];

// 删除任务
const deleteTask = (row: RowData) => {
  dialog.warning({
    title: "警告",
    content: "检查是否有正在运行的定时任务，确定删除?",
    positiveText: "确定",
    negativeText: "不确定",
    onPositiveClick: () => {
      TaskApi.deleteTask(row.id.toString()).then((resp: any) => {
        if (resp.success === true) {
          initTaskList();
          message.success("删除任务成功！");
        } else {
          message.error("删除任务失败！");
        }
      });
    },
    onNegativeClick: () => {
      message.error("不确定");
    },
  });
};

// 运行任务
const runTask = async (row: RowData) => {
  const resp = await TaskApi.runningTask(row.id.toString());
  if (resp.success === true) {
    message.success("开始运行！");
    initTaskList();
  } else {
    message.error("运行失败！");
  }
};

// 显示任务报告列表
const clickTaskName = (row: RowData) => {
  datas.tid = row.id;
  datas.taskFlag = false;
};
// 返回任务列表
const goBack = () => {
  datas.taskFlag = true;
};

// 定时设置
const showModalTimer = ref(false);

const formRefTimer = ref<FormInst | null>(null);

const formValueTimer = ref<{
  minute: string | null;
  hour: string | null;
  day_of_week: string | null;
  day: string | null;
  month: string | null;
}>({
  minute: "*",
  hour: "*",
  day_of_week: "*",
  day: "*",
  month: "*",
});

const closeModal = () => {
  showModalTask.value = false;
  initTaskList();
};

const columns = createColumns({
  play(row: RowData, action: String) {
    switch (action) {
      case "run":
        runTask(row);
        break;
      case "set":
        showModalTimer.value = true;
        break;
      case "edit":
        openModalTask(2);
        datas.tid = row.id;
        break;
      case "delete":
        deleteTask(row);
        break;
      case "clickTaskName":
        clickTaskName(row);
        break;
    }
  },
});
const pagination = false;

const rulesTimer = {
  name: {
    required: true,
    message: "请输入任务名称",
    trigger: ["input"],
  },
  env: {
    type: "number",
    required: true,
    trigger: ["blur", "change"],
    message: "请选择运行环境",
  },
  email: {
    required: true,
    message: "请输入邮箱地址",
    trigger: ["input"],
  },
};

onMounted(() => {
  initTeamList();
  initTaskList();
});
</script>

<template>
  <div class="body">
    <div class="task-list" v-if="datas.taskFlag">
      <div class="pageheader">
        <n-space justify="space-between" class="breadcrumb-navigation">
          <span>任务管理</span>
          <n-breadcrumb separator=">">
            <n-breadcrumb-item>首页</n-breadcrumb-item>
            <n-breadcrumb-item>任务管理</n-breadcrumb-item>
          </n-breadcrumb>
        </n-space>
      </div>
      <n-card class="main-card">
        <div>
          <n-space justify="space-between">
            <n-button type="primary" @click="openModalTask(1)">创建</n-button>
            <n-form inline :model="model" label-placement="left">
              <n-form-item label="团队">
                <n-select
                  style="width: 200px"
                  v-model:value="datas.query.team_id"
                  placeholder="选择团队"
                  :options="model.teamOptions"
                  @update:value="changeTeam"
                  clearable
                >
                </n-select>
              </n-form-item>
              <n-form-item label="名称" label-placement="left">
                <n-input
                  v-model:value="datas.query.name"
                  placeholder="请输入任务名称"
                ></n-input>
              </n-form-item>
              <n-form-item label-placement="left">
                <n-button type="primary" @click="initTaskList">
                  <template #icon>
                    <n-icon>
                      <SearchOutline />
                    </n-icon>
                  </template>
                  搜索
                </n-button>
              </n-form-item>
            </n-form>
          </n-space>
        </div>
        <n-data-table
          :columns="columns"
          :data="datas.tableData"
          :pagination="pagination"
          :bordered="false"
        />
      </n-card>

      <n-modal
        v-model:show="showModalTask"
        class="custom-card"
        preset="card"
        style="width: 1200px"
        :title="modalDatas.title"
        size="huge"
        :bordered="false"
        :segmented="segmented"
      >
        <TaskModal
          :type="modalDatas.type"
          :tid="datas.tid"
          @close="closeModal"
        />
      </n-modal>

      <!-- <n-modal
      v-model:show="showModalTimer"
      class="custom-card"
      preset="card"
      style="width: 80%"
      title="定时任务"
      size="huge"
      :bordered="false"
      :segmented="segmented"
    >
      <n-form
        ref="formRef"
        :model="formValue"
        :rules="rulesTimer"
        label-placement="left"
        inline
        :label-width="80"
        size="medium"
        show-require-mark
      >
        <n-form-item label="秒" path="name">
          <n-input
            v-model:value="formValue.name"
            placeholder="请输入任务名称"
          />
        </n-form-item>
        <n-form-item label="分" path="env">
          <n-select
            v-model:value="formValue.env_id"
            style="width: 200px"
            :options="model.envOptions"
            placeholder="请选择运行环境"
          >
          </n-select>
        </n-form-item>
        <n-form-item label="时" path="team_id">
          <n-input
            v-model:value="formValue.team_id"
            placeholder="请输入邮箱地址"
          />
        </n-form-item>
        <n-form-item>
          <n-button type="primary" @click="handleSave"> 保存 </n-button>
        </n-form-item>
      </n-form>
      <n-divider title-placement="left"> 选择用例 </n-divider>
      <div></div>
    </n-modal> -->
    </div>

    <div class="task-report" v-else>
      <div class="pageheader">
        <n-space justify="space-between" class="breadcrumb-navigation">
          <div>
            <n-button quaternary @click="goBack">返回</n-button>
            <span>任务报告</span>
          </div>

          <n-breadcrumb separator=">">
            <n-breadcrumb-item>首页</n-breadcrumb-item>
            <n-breadcrumb-item>任务管理</n-breadcrumb-item>
            <n-breadcrumb-item>任务报告</n-breadcrumb-item>
          </n-breadcrumb>
        </n-space>
      </div>
      <n-card class="main-card">
        <TaskReport :tid="datas.tid" />
      </n-card>
    </div>
  </div>
</template>
