<template>
  <div v-if="errorLogs.length > 0">
    <el-badge
      :is-dot="true"
      style="line-height: 25px; margin-top: -5px"
      @click.native="dialogTableVisible = true"
    >
      <el-button style="padding: 8px 10px" size="small" type="danger">
        <svg-icon icon-class="bug" />
      </el-button>
    </el-badge>

    <el-dialog :visible.sync="dialogTableVisible" width="80%" append-to-body>
      <template #title>
        <span style="padding-right: 10px">Error Log</span>
        <el-button size="mini" type="primary" icon="el-icon-delete" @click="clearAll"
          >Clear All</el-button
        >
      </template>
      <el-table :data="errorLogs" border>
        <el-table-column label="Message">
          <template slot-scope="{ row }">
            <div>
              <span class="message-title">Msg:</span>
              <el-tag type="danger">
                {{ row.err.message }}
              </el-tag>
            </div>
            <br />
            <div>
              <span class="message-title" style="padding-right: 10px">Info: </span>
              <el-tag type="warning"> {{ row.vm.$vnode.tag }} error in {{ row.info }} </el-tag>
            </div>
            <br />
            <div>
              <span class="message-title" style="padding-right: 16px">Url: </span>
              <el-tag type="success">
                {{ row.url }}
              </el-tag>
            </div>
          </template>
        </el-table-column>
        <el-table-column label="Stack">
          <template slot-scope="scope">
            {{ scope.row.err.stack }}
          </template>
        </el-table-column>
      </el-table>
    </el-dialog>
  </div>
</template>

<script>
import { useStore } from 'vuex'
import { reactive, computed, toRefs } from 'vue'

export default {
  name: 'ErrorLog',
  setup() {
    const state = reactive({
      dialogTableVisible: false
    })
    const store = useStore()
    const errorLogs = computed(() => store.getters.errorLogs)
    const clearAll = () => {
      state.dialogTableVisible = false
      store.dispatch('errorLog/clearErrorLog')
    }
    return {
      ...toRefs(state),
      errorLogs,
      clearAll
    }
  }
}
</script>

<style scoped>
.message-title {
  font-size: 16px;
  color: #333;
  font-weight: bold;
  padding-right: 8px;
}
</style>
