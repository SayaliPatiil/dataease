<template>
  <el-drawer
    v-closePress
    :title="$t('user.filter_method')"
    :visible.sync="userDrawer"
    custom-class="de-user-drawer"
    size="680px"
    direction="rtl"
  >
    <div class="el-drawer__body-cont">
      <div class="filter">
        <span>{{ $t("commons.status") }}</span>
        <div class="filter-item">
          <span
            v-for="ele in status"
            :key="ele.id"
            class="item"
            :class="[activeStatus.includes(ele.id) ? 'active' : '']"
            @click="statusChange(ele.id)"
          >{{ $t(ele.label) }}</span>
        </div>
      </div>
      <div class="filter">
        <span>{{ $t("commons.organization") }}</span>
        <div class="filter-item">
          <span
            v-for="ele in selectDeptsCache"
            :key="ele.id"
            class="item"
            :class="[activeDept.includes(ele.id) ? 'active' : '']"
            @click="activeDeptChange(ele.id)"
          >{{ ele.label }}</span>
          <el-popover
            placement="bottom"
            popper-class="user-popper"
            width="200"
            trigger="click"
          >
            <el-popover
              placement="bottom"
              popper-class="user-popper dept"
              width="200"
              trigger="click"
            >
              <el-tree
                :load="loadNode"
                :lazy="true"
                :expand-on-click-node="false"
                :data="deptsComputed"
                :props="defaultProps"
                @node-click="handleNodeClick"
              />

              <el-select
                ref="roleSelect"
                slot="reference"
                v-model="selectDepts"
                popper-class="tree-select"
                multiple
                :placeholder="$t('commons.please_select')"
                value-key="id"
              >
                <el-option
                  v-for="item in selectDepts"
                  :key="item.label"
                  :label="item.label"
                  :value="item"
                />
              </el-select>
            </el-popover>
            <span
              slot="reference"
              class="more"
            >+ {{ $t("panel.more") }}</span>
          </el-popover>
        </div>
      </div>
      <div class="filter">
        <span>{{ $t("panel.role") }}</span>
        <div class="filter-item">
          <span
            v-for="ele in rolesValueCopy"
            :key="ele.id"
            class="item"
            :class="[activeRole.includes(ele.id) ? 'active' : '']"
            @click="activeRoleChange(ele.id)"
          >{{ ele.name }}</span>
          <el-popover
            placement="bottom"
            popper-class="user-popper"
            width="200"
            trigger="click"
          >
            <el-select
              ref="roleSelect"
              v-model="rolesValue"
              multiple
              :placeholder="$t('commons.please_select')"
              value-key="id"
              @change="changeRole"
              @remove-tag="changeRole"
            >
              <el-option
                v-for="item in rolesComputed"
                :key="item.name"
                :label="item.name"
                :value="item"
              />
            </el-select>
            <span
              slot="reference"
              class="more"
            >+ {{ $t("panel.more") }}</span>
          </el-popover>
        </div>
      </div>
    </div>
    <div class="de-foot">
      <deBtn
        secondary
        @click="reset"
      >{{
        $t("commons.reset")
      }}</deBtn>
      <deBtn
        type="primary"
        @click="search"
      >{{
        $t("commons.adv_search.search")
      }}</deBtn>
    </div>
  </el-drawer>
</template>

<script>
import { allRoles } from '@/api/system/user'
import { getDeptTree, treeByDeptId } from '@/api/system/dept'

export default {
  data() {
    return {
      roleCache: [],
      deptCache: [],
      roles: [],
      filterTextMap: [],
      status: [
        {
          id: 1,
          label: 'commons.enable'
        },
        {
          id: 0,
          label: 'commons.disable'
        }
      ],
      activeStatus: [],
      rolesValue: [],
      activeRole: [],
      depts: [],
      selectDepts: [],
      selectDeptsCache: [],
      activeDept: [],
      defaultProps: {
        children: 'children',
        label: 'label',
        isLeaf: 'leaf'
      },
      userDrawer: false
    }
  },
  computed: {
    rolesComputed() {
      return this.roles.filter((ele) => !this.activeRole.includes(ele.id))
    },
    rolesValueCopy() {
      return this.roleCache.filter((ele) => this.activeRole.includes(ele.id))
    },
    deptsComputed() {
      return this.depts.filter((ele) => !this.activeDept.includes(ele.id))
    }
  },
  mounted() {
    this.initRoles()
  },
  methods: {
    clearFilter() {
      Array(3)
        .fill(1)
        .forEach((_, index) => {
          this.clearOneFilter(index)
        })
      this.$emit('search', [], [])
    },
    clearOneFilter(index) {
      (this.filterTextMap[index] || []).forEach((ele) => {
        this[ele] = []
      })
    },
    // 获取弹窗内部门数据
    treeByDeptId() {
      treeByDeptId(0).then((res) => {
        this.depts = (res.data || []).map((ele) => {
          return {
            ...ele,
            leaf: !ele.hasChildren
          }
        })
      })
    },
    changeRole() {
      if (
        this.roleCache.length >
        this.rolesValue.length + this.activeRole.length
      ) {
        this.roleCache = this.roleCache.filter((ele) =>
          this.rolesValue
            .map((ele) => ele.id)
            .concat(this.activeRole)
            .includes(ele.id)
        )
        return
      }
      const roleIdx = this.rolesValue.findIndex(
        (ele) =>
          !this.roleCache
            .map((ele) => ele.id)
            .concat(this.activeRole)
            .includes(ele.id)
      )
      if (roleIdx === -1) return
      this.activeRole.push(this.rolesValue[roleIdx].id)
      this.roleCache.push(this.rolesValue[roleIdx])
      this.rolesValue.splice(roleIdx, 1)
    },
    activeRoleChange(id) {
      const roleIndex = this.activeRole.findIndex((ele) => ele === id)
      if (roleIndex === -1) {
        this.activeRole.push(id)
        this.rolesValue = this.rolesValue.filter((ele) => ele.id !== id)
      } else {
        this.activeRole.splice(roleIndex, 1)
        const role = this.roleCache.find((ele) => ele.id === id)
        this.rolesValue.push(role)
      }
    },
    handleNodeClick({ id, label }) {
      const deptIdx = this.selectDepts.findIndex((ele) => ele.id === id)
      if (deptIdx !== -1) {
        this.selectDepts.splice(deptIdx, 1)
        this.selectDeptsCache = this.selectDeptsCache.filter(
          (ele) => ele.id !== id
        )
        this.deptCache = this.deptCache.filter((ele) => ele.id !== id)
        return
      }
      this.activeDept.push(id)
      this.selectDeptsCache.push({ id, label })
      this.deptCache.push({ id, label })
    },
    activeDeptChange(id) {
      const dept = this.deptCache.find((ele) => ele.id === id)
      this.selectDepts.push(dept)
      this.activeDept = this.activeDept.filter((ele) => ele !== id)
      this.selectDeptsCache = this.selectDeptsCache.filter(
        (ele) => ele.id !== id
      )
    },
    statusChange(id) {
      const statusIndex = this.activeStatus.findIndex((ele) => ele === id)
      if (statusIndex === -1) {
        this.activeStatus.push(id)
      } else {
        this.activeStatus.splice(statusIndex, 1)
      }
    },
    changeDepts() {
      const depts = this.selectDepts.map((item) => item.id)
      this.activeDept = this.activeDept.filter((ele) => depts.includes(ele))
    },
    loadNode(node, resolve) {
      if (!this.depts.length) {
        this.treeByDeptId()
        return
      }
      getDeptTree(node.data.id).then((res) => {
        const filterDept = (res.data || []).filter(
          (ele) => !this.activeDept.includes(ele.deptId)
        )
        resolve(
          filterDept.map((dept) => {
            return this.normalizer(dept)
          })
        )
      })
    },
    normalizer(node) {
      return {
        id: node.deptId,
        label: node.name,
        leaf: !node.hasChildren
      }
    },
    initRoles() {
      allRoles().then((res) => {
        this.roles = res.data
      })
    },
    search() {
      this.userDrawer = false
      this.$emit('search', this.formatCondition(), this.formatText())
    },
    formatText() {
      this.filterTextMap = []
      const params = []
      if (this.activeStatus.length) {
        const str = `${this.$t('kettle.status')}:${this.activeStatus.reduce(
          (pre, next) =>
            this.$t((this.status.find((ele) => ele.id === next) || {}).label) +
            '、' +
            pre,
          ''
        )}`
        params.push(str.slice(0, str.length - 1))
        this.filterTextMap.push(['activeStatus'])
      }
      if (this.activeDept.length) {
        params.push(
          `${this.$t('panel.org')}:${this.selectDeptsCache
            .map((ele) => ele.label)
            .join('、')}`
        )
        this.filterTextMap.push([
          'activeDept',
          'selectDepts',
          'selectDeptsCache',
          'deptCache'
        ])
      }
      if (this.activeRole.length) {
        params.push(
          `${this.$t('panel.role')}:${this.rolesValueCopy
            .map((ele) => ele.name)
            .join('、')}`
        )
        this.filterTextMap.push(['rolesValue', 'activeRole', 'roleCache'])
      }
      return params
    },
    formatCondition() {
      const fildMap = {
        'r.role_id': this.activeRole,
        'd.dept_id': this.activeDept,
        'u.enabled': this.activeStatus
      }
      const conditions = []
      Object.keys(fildMap).forEach((ele) => {
        if (fildMap[ele].length) {
          conditions.push({
            field: ele,
            operator: 'in',
            value: fildMap[ele]
          })
        }
      })
      return conditions
    },
    init() {
      this.userDrawer = true
    },
    reset() {
      this.userDrawer = false
      this.clearFilter()
    }
  }
}
</script>
