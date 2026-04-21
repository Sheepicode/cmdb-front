<template>
  <div
    ref="ipGridRef"
    class="ip-grid"
    :style="{ gap: gridGap + 'px' }"
    @mousedown="onMouseDown"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
    @mouseleave="onMouseUp"
  >
    <!-- 框选矩形 -->
    <div
      v-if="isSelecting"
      class="ip-grid-selection-box"
      :style="selectionBoxStyle"
    ></div>

    <!-- IP格子 -->
    <div
      v-for="(item) in gridList"
      :key="item.ip"
      :ref="`grid-item-${item.ip}`"
      :data-ip="item.ip"
      :class="[
        'ip-grid-item',
        selectedIPs.includes(item.ip) ? 'ip-grid-item-selected' : ''
      ]"
      :style="gridItemStyle(item)"
      @click.stop="clickGridItem(item, $event)"
    >
      {{ item.gridTitle }}
    </div>

    <!-- 信息卡片 -->
    <div
      v-show="infoCardVisible"
      class="info-card"
      :style="{
        top: infoCardY + 'px',
        left: infoCardX + 'px',
        width: infoCardWidth + 'px',
        height: infoCardHeight + 'px'
      }"
    >
      <div class="info-card-header">
        <div class="info-card-ip">
          {{ infoCardData.ip }}
        </div>
        <div
          class="info-card-status-dot"
          :style="{
            backgroundColor: `${STATUS_COLOR[infoCardData._ip_status]}22`
          }"
        >
          <div
            class="info-card-status-dot-content"
            :style="{
              backgroundColor: STATUS_COLOR[infoCardData._ip_status]
            }"
          ></div>
        </div>

        <div class="info-card-status-text">
          {{ $t(STATUS_LABEL[infoCardData._ip_status]) }}
        </div>

        <a-button
          type="primary"
          class="ops-button-ghost info-card-recycle"
          ghost
          @click="clickRecycle(infoCardData)"
        >
          <ops-icon type="veops-recycle" />
          {{ $t('cmdb.ipam.recycle') }}
        </a-button>
      </div>

      <div class="info-card-main">
        <div
          v-for="(col) in filterColumns"
          :key="col.field"
          class="info-card-main-row"
        >
          <div class="info-card-main-title">
            <a-tooltip :title="col.title">
              {{ col.title }}
            </a-tooltip>
          </div>

          <div class="info-card-main-value">
            <a-tooltip :title="infoCardTip[col.field]" placement="topLeft">
              <template v-if="col.is_reference && infoCardData[col.field]">
                <a
                  v-for="(ciId) in (col.is_list ? infoCardData[col.field] : [infoCardData[col.field]])"
                  :key="ciId"
                  :href="`/cmdb/cidetail/${col.reference_type_id}/${ciId}`"
                  target="_blank"
                >
                  {{ getReferenceAttrValue(ciId, col) }}
                </a>
              </template>

              <template v-else-if="col.is_link && infoCardData[col.field]">
                <a
                  v-for="(linkItem, linkIndex) in (col.is_list ? infoCardData[col.field] : [infoCardData[col.field]])"
                  :key="linkIndex"
                  :href="
                    linkItem.startsWith('http') || linkItem.startsWith('https')
                      ? linkItem
                      : `http://${linkItem}`
                  "
                  target="_blank"
                >
                  {{ getChoiceValueLabel(col, linkItem) || linkItem }}
                </a>
              </template>

              <template v-else-if="col.is_choice && infoCardData[col.field]">
                <span
                  v-for="value in (col.is_list ? infoCardData[col.field] : [infoCardData[col.field]])"
                  :key="value"
                  class="column-default-choice"
                >
                  {{ getChoiceValueLabel(col, value) || value }}
                </span>
              </template>

              <template v-else>
                {{
                  infoCardData[col.field] !== undefined
                    ? Array.isArray(infoCardData[col.field])
                      ? infoCardData[col.field].join(', ')
                      : infoCardData[col.field]
                    : ''
                }}
              </template>
            </a-tooltip>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { STATUS_COLOR, STATUS_LABEL, ADDRESS_STATUS } from './constants.js'

export default {
  name: 'GridIP',

  props: {
    ipList: { type: Array, default: () => [] },
    columns: { type: Array, default: () => [] },
    referenceShowAttrNameMap: { type: Object, default: () => ({}) },
    referenceCIIdMap: { type: Object, default: () => ({}) }
  },

  data() {
    return {
      STATUS_COLOR,
      STATUS_LABEL,

      gridItemSize: 52,
      gridGap: 8,

      // ===== 框选 =====
      isSelecting: false,
      startX: 0,
      startY: 0,
      currentX: 0,
      currentY: 0,

      selectedIPs: [],
      selectedIPSet: new Set(), // 用set，可自动去重
      ctrlPressed: false,

      // ===== 信息卡片 =====
      infoCardX: 0,
      infoCardY: 0,
      infoCardWidth: 375,
      infoCardVisible: false,
      infoCardData: {},
      infoCardTip: {}
    }
  },

  computed: {
    gridList() {
      return this.ipList.map(item => {
        const last = item?.ip?.split('.').pop() || ''
        return { ...item, gridTitle: last }
      })
    },

    filterColumns() {
      return this.columns.filter(col => col.field !== '_ip_status') || []
    },

    infoCardHeight() {
      let h = 311
      if (this.filterColumns.length < 6) {
        h -= (6 - this.filterColumns.length) * 36
      }
      return h
    },

    selectionBoxStyle() {
      const x = Math.min(this.startX, this.currentX)
      const y = Math.min(this.startY, this.currentY)

      return {
        position: 'fixed',
        left: x + 'px',
        top: y + 'px',
        width: Math.abs(this.currentX - this.startX) + 'px',
        height: Math.abs(this.currentY - this.startY) + 'px',
        border: '1px dashed #1890ff',
        backgroundColor: 'rgba(24,144,255,0.15)',
        zIndex: 999
      }
    }
  },

  mounted() {
    window.addEventListener('click', this.handleClick)

    // 键盘监听
    window.addEventListener('keydown', this.handleKeyDown)
    window.addEventListener('keyup', this.handleKeyUp)
  },

  beforeDestroy() {
    window.removeEventListener('click', this.handleClick)
    window.removeEventListener('keydown', this.handleKeyDown)
    window.removeEventListener('keyup', this.handleKeyUp)
  },

  methods: {
    // 键盘
    handleKeyDown(e) {
      if (e.key === 'Control') {
        this.ctrlPressed = true
      }

      if (e.key === 'Escape') {
        this.clearSelection()
      }
    },

    handleKeyUp(e) {
      if (e.key === 'Control') {
        this.ctrlPressed = false
      }
    },

    clearSelection() {
      this.selectedIPSet.clear()
      this.selectedIPs = []
      this.$emit('selectChange', [])
    },

    // 样式
    gridItemStyle(item) {
      const selected = this.selectedIPSet.has(item.ip)

      return {
        width: this.gridItemSize + 'px',
        height: this.gridItemSize + 'px',
        backgroundColor: selected
          ? '#2F54EB22'
          : `${STATUS_COLOR[item._ip_status]}22`,
        color: selected
          ? '#2F54EB'
          : STATUS_COLOR[item._ip_status],
        border: `1px solid ${
          selected ? '#2F54EB' : STATUS_COLOR[item._ip_status]
        }`
      }
    },

    handleClick(event) {
      const classStr = event?.target?.classList?.value || ''
      if (
        classStr.indexOf('info-card') === -1 &&
        classStr.indexOf('ip-grid-item') === -1
      ) {
        this.infoCardVisible = false
      }
    },

    // 鼠标按下
    onMouseDown(e) {
      if (e.button !== 0) return

      this.isSelecting = true
      this.startX = e.clientX
      this.startY = e.clientY
      this.currentX = e.clientX
      this.currentY = e.clientY

      // 按下Ctrl 清空
      if (!this.ctrlPressed) {
        this.selectedIPSet.clear()
      }
    },

    onMouseMove(e) {
      if (!this.isSelecting) return

      this.currentX = e.clientX
      this.currentY = e.clientY

      this.updateSelection()
    },

    onMouseUp() {
      this.isSelecting = false
    },

    // ===== 核心框选 =====
    updateSelection() {
      const rect = {
        left: Math.min(this.startX, this.currentX),
        right: Math.max(this.startX, this.currentX),
        top: Math.min(this.startY, this.currentY),
        bottom: Math.max(this.startY, this.currentY)
      }

      const tempSet = new Set()

      this.gridList.forEach(item => {
        const ref = this.$refs[`grid-item-${item.ip}`]
        if (!ref) return

        const el = Array.isArray(ref) ? ref[0] : ref
        const r = el.getBoundingClientRect()

        const hit =
          r.left < rect.right &&
          r.right > rect.left &&
          r.top < rect.bottom &&
          r.bottom > rect.top

        if (hit) tempSet.add(item.ip)
      })

      // 按Ctrl可叠加选择
      if (this.ctrlPressed) {
        tempSet.forEach(ip => this.selectedIPSet.add(ip))
      } else {
        this.selectedIPSet = tempSet
      }

      this.selectedIPs = Array.from(this.selectedIPSet)
      this.$emit('selectChange', this.selectedIPs)
      // console.log(this.selectedIPs)
    },

    // 点击格子
    clickGridItem(item, event) {
      if (
        [ADDRESS_STATUS.OFFLINE_UNASSIGNED, ADDRESS_STATUS.ONLINE_UNASSIGNED].includes(
          item?._ip_status
        )
      ) {
        this.$emit('openAssign', item)
      } else {
        if (this.ctrlPressed) {
          // Ctrl 点选
          if (this.selectedIPSet.has(item.ip)) {
            this.selectedIPSet.delete(item.ip)
          } else {
            this.selectedIPSet.add(item.ip)
          }
        } else {
          // 普通点击
          this.selectedIPSet.clear()
          this.selectedIPSet.add(item.ip)
        }

        this.selectedIPs = Array.from(this.selectedIPSet)
        this.$emit('selectChange', this.selectedIPs)

        this.showInfoCard(item, event)
      }
    },

    //  信息卡片
    showInfoCard(item, event) {
      let x = event.clientX
      let y = event.clientY + this.gridItemSize

      if (x + this.infoCardWidth > window.innerWidth) {
        x = x - this.infoCardWidth
      }

      if (y + this.infoCardHeight > window.innerHeight) {
        y = y - this.infoCardHeight - this.gridItemSize
      }

      const infoCardTip = {}

      this.filterColumns.forEach(col => {
        const arr = Array.isArray(item[col.field])
          ? item[col.field]
          : [item[col.field]]

        infoCardTip[col.field] = arr
          .map(v => {
            if (v == null) return v
            if (col.is_reference) return this.getReferenceAttrValue(v, col)
            if (col.is_link || col.is_choice) return this.getChoiceValueLabel(col, v)
            return v
          })
          .join(', ')
      })

      this.infoCardX = x
      this.infoCardY = y
      this.infoCardVisible = true
      this.infoCardData = item
      this.infoCardTip = infoCardTip
    },

    clickRecycle(data) {
      this.$emit('recycle', data.ip)
    },

    getReferenceAttrValue(id, col) {
      const ci = this?.referenceCIIdMap?.[col?.reference_type_id]?.[id]
      if (!ci) return id

      const attrName =
        this.referenceShowAttrNameMap?.[col.reference_type_id]

      return ci?.[attrName] || id
    },

    getChoiceValueLabel(col, val) {
      const f = col?.choice_value?.find(
        i => String(i[0]) === String(val)
      )
      return f ? f?.[1]?.label || '' : ''
    }
  }
}
</script>

<style lang="less" scoped>
.ip-grid {
  display: flex;
  flex-wrap: wrap;
  position: relative;
  user-select: none;

  max-height: calc(100vh - 230px);
  overflow-y: auto;
}

.ip-grid-item {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.ip-grid-item-selected {
  box-shadow: 0 0 0 2px #2f54eb inset;
}

.ip-grid-selection-box {
  pointer-events: none;
}

.info-card {
  position: fixed;
  background: #fff;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}
</style>
