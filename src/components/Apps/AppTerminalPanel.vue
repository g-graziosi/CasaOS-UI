<!--
 * @Author: JerryK
 * @Date: 2021-10-29 15:38:35
 * @LastEditors: JerryK
 * @LastEditTime: 2022-03-08 20:56:10
 * @Description: 
 * @FilePath: \CasaOS-UI\src\components\Apps\AppTerminalPanel.vue
-->
<template>
  <div class="modal-card">

    <!-- Modal-Card Body Start -->
    <section class="modal-card-body ">
      <div class="close-container"><button type="button" class="delete" @click="$emit('close')" /></div>
      <h2 class="title is-4">{{appName}}</h2>
      <div class="flex1">
        <b-tabs  :animated="false" @input="onInput">
          <b-tab-item :label="$t('Terminal')" value="terminal">
            <terminal-card ref="terminal" :wsUrl="wsUrl"></terminal-card>
          </b-tab-item>
          <b-tab-item :label="$t('Logs')" value="logs">
            <logs-card ref="logs" :data="logData"></logs-card>
          </b-tab-item>
        </b-tabs>

      </div>

    </section>
    <!-- Modal-Card Body End -->

    <b-loading :is-full-page="false" v-model="isLoading"></b-loading>
  </div>
</template>

<script>
import TerminalCard from '@/components/logsAndTerminal/TerminalCard.vue';
import LogsCard from '@/components/logsAndTerminal/LogsCard.vue'
export default {
  name: 'app-terminal-panel',
  components: {
    TerminalCard,
    LogsCard
  },
  data() {
    return {
      isLoading: false,
      wsUrl: (process.env.NODE_ENV === "'dev'") ? `ws://${this.$store.state.devIp}:${this.$store.state.devPort}/v1/app/terminal/${this.appid}?token=${this.$store.state.token}` : `ws://${document.location.host}/v1/app/terminal/${this.appid}?token=${this.$store.state.token}`,
      logData: ""
    }
  },
  props: {
    appid: String,
    appName: String
  },
  mounted() {
    this.getLogs();
  },
  methods: {
    getLogs() {
      this.$api.app.getContainerLogs(this.appid).then((res) => {
        if (res.data.success == 200) {
          this.logData = res.data.data
        }
      })
    },
    onInput(e) {
      if (e == "terminal") {
        this.$refs.terminal.active(true)
        this.$refs.logs.active(false)
      } else {
        this.$refs.terminal.active(false)
        this.$refs.logs.active(true)
      }
    }
  },
}
</script>

<style>
</style>
