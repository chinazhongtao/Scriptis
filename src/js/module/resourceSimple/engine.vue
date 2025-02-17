<template>
  <div class="engine-box">
    <Spin
      v-if="loading"
      size="large"
      fix/>
    <div
      class="engine-content"
      v-if="ideEngineList.length > 0">
      <div class="engine-header-bar">
        <h3 class="data-type-title">意书</h3>
        <div class="classify">
          <span>分类</span>
          <Select v-model="ideSelectData">
            <Option
              v-for="item in typeList"
              :value="item.value"
              :key="item.value">{{ item.label }}</Option>
          </Select>
        </div>
      </div>
      <div
        class="engine-list"
        v-for="item in ideClassList"
        :key="item">
        <span class="engline-name">{{ item | calssifyName }}</span>
        <ul class="engine-ul">
          <template
            v-for="(subitem, index) in ideEngineList">
            <li
              class="engine-li"
              :class="[{'active': subitem.isActive}, supportColor(subitem.engineStatus)]"
              v-if="subitem.applicationName === item || subitem.engineStatus === item || (item === 'Idle' && (subitem.engineStatus === 'Error' || subitem.engineStatus === 'ShuttingDown' || subitem.engineStatus === 'Dead'))"
              :key="index"
              @click="subitem.isActive = !subitem.isActive">
              <Icon
                class="engine-icon"
                custom="job-content-icon"
                size="30"
                :class="supportIcon(subitem)"></Icon>
              <Icon
                v-show="subitem.isActive"
                class="engine-right"
                :class="supportColor(subitem.engineStatus)"
                type="md-checkmark"></Icon>
            </li>
          </template>
        </ul>
      </div>
    </div>
    <div
      class="engine-content"
      v-if="boardEngineList.length > 0">
      <div class="engine-header-bar">
        <h3 class="data-type-title">看板系统</h3>
        <div class="classify">
          <span>分类</span>
          <Select v-model="boardSelectData">
            <Option
              v-for="item in typeList"
              :value="item.value"
              :key="item.value">{{ item.label }}</Option>
          </Select>
        </div>
      </div>
      <div
        class="engine-list"
        v-for="item in boardClassList"
        :key="item">
        <span class="engline-name">{{ item | calssifyName }}</span>
        <ul class="engine-ul">
          <template
            v-for="(subitem, index) in boardEngineList">
            <li
              class="engine-li"
              :class="[{'active': subitem.isActive}, supportColor(subitem.engineStatus)]"
              v-if="subitem.applicationName === item || subitem.engineStatus === item || (item === 'Idle' && (subitem.engineStatus === 'Error' || subitem.engineStatus === 'ShuttingDown' || subitem.engineStatus === 'Dead'))"
              :key="index"
              @click="subitem.isActive = !subitem.isActive">
              <Icon
                class="engine-icon"
                custom="job-content-icon"
                size="30"
                :class="supportIcon(subitem)"></Icon>
              <Icon
                v-show="subitem.isActive"
                class="engine-right"
                :class="supportColor(subitem.engineStatus)"
                type="md-checkmark"></Icon>
            </li>
          </template>
        </ul>
      </div>
    </div>
    <span
      class="no-data"
      v-if="ideEngineList.length === 0 && boardEngineList.length === 0">暂无数据</span>
  </div>
</template>
<script>
import api from '@/js/service/api';
export default {
    name: 'Job',
    filters: {
        calssifyName(params) {
            switch (params) {
                case 'sparkEngine':
                    return 'Spark';
                case 'hiveEngine':
                    return 'Hive';
                case 'pythonEngine':
                    return 'Python';
                case 'pipeLineEngine':
                    return 'PipeLine';
                case 'pipelineEngine':
                    return 'PipeLine';
                case 'Idle':
                    return '空闲';
                case 'Error':
                    return '空闲';
                case 'Busy':
                    return '繁忙';
                case 'Starting':
                    return '启动';
                default:
            }
        },
    },
    data() {
        return {
            btnSize: 'small',
            loading: false,
            ideSelectData: 0, // 数据开发分类选择
            boardSelectData: 0,
            typeList: [
                {
                    value: 0,
                    label: '按类别',
                },
                {
                    value: 1,
                    label: '按状态',
                },
            ],
            ideEngineList: [],
            boardEngineList: [],
            ideClassList: [],
            boardClassList: [],
        };
    },
    computed: {
        activeList() {
            return this.ideEngineList.concat(this.boardEngineList).filter((item) => item.isActive);
        },
    },
    watch: {
        // 选择分类，分组数据
        ideSelectData() {
            this.ideClassList = [];
            this.getClassListAction(this.ideSelectData, this.ideEngineList, this.ideClassList);
        },
        boardSelectData() {
            this.boardClassList = [];
            this.getClassListAction(this.boardSelectData, this.boardEngineList, this.boardClassList);
        },
        loading(val) {
            this.$emit('change-loading', val);
        },
        activeList(val) {
            let params = !!val.length > 0;
            this.$emit('disabled', !params);
        },
    },
    methods: {
        killJob() {
            if (this.loading) return this.$Message.warning('请等待接口返回！');
            const params = [];
            let flage = false;
            this.activeList.map((item) => {
                if (item.engineStatus === 'Starting') flage = true;
                params.push(
                    {
                        ticketId: item.ticketId,
                        moduleName: item.moduleName,
                        engineManagerInstance: item.engineManagerInstance,
                        creator: item.creator,
                    }
                );
            });
            if (flage) return this.$Message.warning('启动状态引擎无法结束！');
            if (params.length === 0) return this.$Message.warning('未选中引擎！');
            this.loading = true;
            api.fetch(`/resourcemanager/enginekill`, params).then((rst) => {
                this.getEngineData();
                this.loading = false;
            }).catch((err) => {
                this.loading = false;
            });
        },
        getEngineData() {
            this.ideEngineList = [];
            this.boardEngineList = [];
            this.ideClassList = [];
            this.boardClassList = [];
            this.loading = true;
            api.fetch('/resourcemanager/engines').then((res) => {
                this.loading = false;
                res.engines.map((item) => {
                    item.isActive = false;
                    if (item.applicationName === 'pipelineEngine') {
                        item.applicationName = 'pipeLineEngine';
                    }
                    if (item.creator === 'IDE') {
                        this.ideEngineList.push(item);
                    } else {
                        this.boardEngineList.push(item);
                    }
                });
                // 根据状态改变数据
                this.getClassListAction(this.ideSelectData, this.ideEngineList, this.ideClassList);
                this.getClassListAction(this.boardSelectData, this.boardEngineList, this.boardClassList);
            }).catch(() => {
                this.loading = false;
            });
        },
        getClassListAction(selectData, engineList, classList) {
            if (selectData === 0) {
                engineList.map((item) => {
                    if (!classList.includes(item.applicationName)) {
                        classList.push(item.applicationName);
                    }
                });
            } else {
                engineList.map((item) => {
                    if (!classList.includes('Idle') && (item.engineStatus === 'Error' || item.engineStatus === 'ShuttingDown' || item.engineStatus === 'Dead')) {
                        classList.push('Idle');
                    }
                    if (!classList.includes(item.engineStatus) && (item.engineStatus !== 'Error' && item.engineStatus !== 'ShuttingDown' && item.engineStatus !== 'Dead')) {
                        classList.push(item.engineStatus);
                    }
                });
            }
        },
        // 颜色过滤
        supportColor(status) {
            switch (status) {
                case 'Busy':
                    return 'yellow';
                case 'Error':
                    return 'green';
                case 'ShuttingDown':
                    return 'green';
                case 'Dead':
                    return 'green';
                case 'Idle':
                    return 'green';
                case 'Starting':
                    return 'blue';
                default:
            }
        },
        // 图标过滤
        supportIcon(item) {
            const supportTypes = [
                { rule: 'hiveEngine', logo: 'fi-hive' },
                { rule: 'pythonEngine', logo: 'fi-python' },
                { rule: 'sparkEngine', logo: 'fi-spark' },
                { rule: 'pipeLineEngine', logo: 'fi-storage' },
                { rule: 'pipelineEngine', logo: 'fi-storage' },
            ];
            const color = this.supportColor(item.engineStatus);
            const support = supportTypes.filter((type) => type.rule === item.applicationName);
            if (support.length > 0) {
                return `is-leaf ${support[0].logo} ${color}`;
            } else {
                return `fi-file ${color}`;
            }
        },
    },
};
</script>
