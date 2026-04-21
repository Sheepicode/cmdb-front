<template>
  <div class="ci-sync-page">
    <!-- 查询 & 操作区 -->
    <a-row justify="space-between" align="middle" class="toolbar">
      <!-- 左侧：查询条件 -->
      <a-space wrap>
        <a-select
          v-model="query.type_id"
          allowClear
          placeholder="源模型类型"
          style="width:160px"
          show-search
          option-filter-prop="children"
          @change="handleChange"
        >
          <a-select-option
            v-for="item in ciTypeList"
            :key="item.type_id"
            :value="item.type_id"
          >
            {{ item.alias }}
          </a-select-option>
        </a-select>

        <a-select
          v-model="query.sync_status"
          allowClear
          placeholder="入库状态"
          style="width:120px"
        >
          <a-select-option :value="1">已入库</a-select-option>
          <a-select-option :value="0">未入库</a-select-option>
        </a-select>

        <a-range-picker
          v-model="query.sync_time"
          style="width:260px"
          :show-time="{ format: 'HH:mm:ss' }"
          format="YYYY-MM-DD HH:mm:ss"
          value-format="YYYY-MM-DD HH:mm:ss"
        />

        <a-button icon="search" @click="fetchData">
          查询
        </a-button>
      </a-space>

      <!-- 右侧：操作按钮 -->
      <a-space >
        <a-popconfirm
          :title="`最近检查时间:【${lastCheckTime}】，是否重新拉取自动发现配置项并检查差异？`"
          ok-text="确认"
          cancel-text="取消"
          @confirm="calculateAdc"
        >
          <a-button icon="reload">检查</a-button>
        </a-popconfirm>

        <a-popconfirm
          title="确认将选中项入库？"
          ok-text="确认"
          cancel-text="取消"
          :disabled="selectedRows.length === 0"
          @confirm="batchSync"
        >
          <a-button
            type="primary"
            :disabled="selectedRows.length === 0"
          >
            入库
          </a-button>
        </a-popconfirm>
      </a-space>
    </a-row>

    <!-- 表格 -->
    <div class="table-wrapper">
      <vxe-table
        border
        stripe
        show-overflow
        :loading="loading"
        :data="pagedData"
        :height="tableHeight"
        auto-resize
        :checkbox-config="{ reserve: true }"
        @checkbox-change="onSelectChange"
        @checkbox-all="onSelectChange"
      >
        <vxe-column type="checkbox" width="50" />
        <vxe-column type="seq" title="序号" width="60" />
        <vxe-column field="ad_type_alias" title="源模型" width="120" />
        <vxe-column field="peer_type_alias" title="关联模型" width="120" />

        <vxe-column title="源配置ID" min-width="160" show-overflow="tooltip">
          <template #default="{ row }">
            <a class="json-link" @click="openJsonModal(row.first_ci, 0)" :title="'点击查看配置详情'">
              {{ row.first_ci_id }}
            </a>
          </template>
        </vxe-column>

        <vxe-column title="关联配置ID" min-width="160" show-overflow="tooltip">
          <template #default="{ row }">
            <a class="json-link" @click="openJsonModal(row.second_ci, 1)" :title="'点击查看配置详情'">
              {{ row.second_ci_id }}
            </a>
          </template>
        </vxe-column>
        <!-- <vxe-column field="match_value" title="匹配值" width="150" show-overflow="tooltip" /> -->
        <vxe-column title="匹配值" min-width="200" show-overflow="tooltip">
          <template #default="{ row }">
            <a class="json-link" @click="openJsonModal(row.rule, 3)" :title="'点击查看发现规则'">
              {{ row.match_value }}
            </a>
          </template>
        </vxe-column>
        <vxe-column field="sync_by" title="入库人" width="120" />
        <vxe-column field="sync_at" title="入库时间" width="160" />

        <!-- 状态（可筛选） -->
        <vxe-column
          title="状态"
          width="100"
          :filters="syncStatusFilters"
          :filter-method="filterSyncStatus"
        >
          <template #default="{ row }">
            <a-tag v-if="row.is_sync" color="green">已入库</a-tag>
            <a-tag v-else color="orange">未入库</a-tag>
          </template>
        </vxe-column>

        <vxe-column title="操作" width="100" fixed="right" align="center">
          <template #default="{ row }">
            <a-space>
              <!-- <a @click="openDiff(row)">差异</a> -->
              <a-popconfirm title="确认是否入库" @confirm="batchSync(row)">
                <a>入库</a>
              </a-popconfirm>
            </a-space>
          </template>
        </vxe-column>
      </vxe-table>

      <div class="pager-wrapper">
        <vxe-pager
          background
          size="small"
          :current-page="page.currentPage"
          :page-size="page.pageSize"
          :total="page.total"
          :page-sizes="[10, 20, 50, 100]"
          @page-change="onPageChange"
        />
      </div>
    </div>

    <!-- JSON 查看弹窗 -->
    <a-modal
      v-model="editVisible"
      :title="modalTitle"
      ok-text="关闭"
      @ok="editVisible = false"
    >
      <a-textarea :value="editJson" :rows="14" readonly />
    </a-modal>
  </div>
</template>

<script>
import {
  getAdcRelationsCandidates,
  calculateAdcRelations,
  batchCreateAdcRelations
} from '../../api/discovery'

export default {
  name: 'CiRelationSyncTable',
  data() {
    return {
      tableHeight: 0,
      query: {
        type_id: undefined,
        sync_status: undefined,
        sync_time: []
      },

      syncStatusFilters: [
        { label: '已入库', value: 1 },
        { label: '未入库', value: 0 }
      ],
      loading: false,
      ciTypeList: [],

      tableData: [],
      selectedRows: [],

      page: {
        total: 0,
        currentPage: 1,
        pageSize: 10
      },
      editVisible: false,
      editJson: '',
      currentRow: null,
      modalTitle: '',
      lastCheckTime: ''
    }
  },
  created() {
    this.fetchData()
  },
  mounted() {
    this.calcTableHeight()
    window.addEventListener('resize', this.calcTableHeight)
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.calcTableHeight)
  },
  computed: {
    pagedData() {
      return this.tableData
    }
  },
  methods: {
    calcTableHeight() {
      // ===== 可按你系统实际高度微调 =====
      const headerH = 64
      const toolbarH = 64
      const pagerH = 56
      const paddingH = 24

      const winH = window.innerHeight

      this.tableHeight =
        winH - headerH - toolbarH - pagerH - paddingH
    },

    handleChange(value) {
        console.log(value)
    },
    filterSyncStatus({ value, row }) {
      return value === 1 ? row.is_sync : !row.is_sync
    },

    async fetchData() {
      this.loading = true
      const { sync_time, ...restQuery } = this.query
      const [start, end] = sync_time || []
      const res = await getAdcRelationsCandidates({
        ...restQuery,
        ...this.page,
        start,
        end
      })
      this.tableData = res.result
      this.page.total = res.total
      this.ciTypeList = res.ciTypeList
      this.loading = false
      this.lastCheckTime = res.lastCheckTime
    },

    async calculateAdc() {
      this.loading = true
      const res = await calculateAdcRelations({ ad_type_id: this.query.type_id })
      this.$message.success(`发现关系：${res.numfound} 对`)
      this.fetchData()
    },

    formatJSON(val) {
      if (!val) return '-'
      return JSON.stringify(val, null, 2)
    },

    onSelectChange({ records }) {
      this.selectedRows = records
    },

    onPageChange({ currentPage, pageSize }) {
      this.page.currentPage = currentPage
      this.page.pageSize = pageSize
      this.fetchData()
    },

    openJsonModal(rowOrJson, value) {
        // 编辑最新配置
        this.currentRow = rowOrJson
        this.editJson = JSON.stringify(rowOrJson || {}, null, 2)
        this.modalTitle = value === 0 ? '源模型配置项详情' : (value === 1 ? '关联模型配置项详情' : '关系发现规则')
        this.editVisible = true
    },

    async batchSync(row) {
      this.loading = true
      const ids = row?.id != null ? [ row.id ] : this.selectedRows.map(item => item.id)
      try {
        const { created, failed } = await batchCreateAdcRelations({ ids }, true)
        if (failed === 0) {
          this.$message.success(`关系批量入库完成, 成功:${created}`)
        } else {
          this.$message.error(`成功:${created}, 失败:${failed}`)
        }

        this.selectedRows = []
        this.fetchData()
      } catch (e) {
        if (e.response) {
          const { status, data } = e.response
          this.$message.error(`更新失败 (${status}): ${data.message || '未知错误'}`)
        } else if (e.request) {
          this.$message.error('网络错误，请检查网络连接')
        }
      }
      this.loading = false
    }
  }
}
</script>

<style scoped>
.ci-sync-page {
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.toolbar {
  flex-shrink: 0;
  margin-bottom: 12px;
}

.table-wrapper {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.vxe-table {
  flex: none;
}

.pager-wrapper {
  flex-shrink: 0;
  padding: 8px 16px;
  border-top: 1px solid #ebeef5;
  background: #fff;
}

.json-view {
  max-height: 100px;
  overflow: auto;
  background: #f6f8fa;
  border: 2px solid #ebeef5;
  padding: 4px;
  border-radius: 4px;
  font-size: 12px;
  line-height: 1.4;
}

.json-view:hover {
  border-color: #c0c4cc;
}

.muted {
  color: #999;
}

.json-link {
  cursor: pointer;
}
</style>
