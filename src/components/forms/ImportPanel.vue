<template>
  <div class="modal-card">
    <!-- Modal-Card Header Start -->
    <header class="modal-card-head">
      <div class="flex1">
        <h3 class="title is-4 has-text-weight-normal">{{ $t('Import') }}</h3>
      </div>
    </header>
    <!-- Modal-Card Header End -->
    <!-- Modal-Card Body Start -->
    <section class="modal-card-body">
      <b-tabs v-model="activeTab">
        <b-tab-item label="Docker CLI">
          <b-field :type="{ 'is-danger': parseError}" :message="errors">
            <b-input maxlength="800" type="textarea" class="import-area" v-model="dockerCliCommands"></b-input>
          </b-field>
        </b-tab-item>
        <b-tab-item label="Docker Compose">
          <b-field :type="{ 'is-danger': parseError}" :message="errors">
            <b-input maxlength="800" type="textarea" class="import-area" v-model="dockerComposeCommands" :placeholder="$t('Notice: If there are multiple services, only the first set can be analyzed correctly')"></b-input>
          </b-field>
        </b-tab-item>
        <b-tab-item :label="$t('AppFile')">
          <b-field :type="{ 'is-danger': parseError}" :message="errors">
            <b-upload v-model="dropFiles" drag-drop expanded accept="application/json" @input="onSelect">
              <section class="section">
                <div class="content has-text-centered">
                  <p>
                    <b-icon icon="upload" size="is-large"></b-icon>
                  </p>
                  <p>{{dropText}}</p>
                </div>
              </section>
            </b-upload>
          </b-field>
        </b-tab-item>

      </b-tabs>

    </section>
    <!-- Modal-Card Body End -->
    <!-- Modal-Card Footer Start-->
    <footer class="modal-card-foot is-flex is-align-items-center">
      <div class="flex1"></div>
      <div>
        <b-button :label="$t('Cancel')" @click="$emit('close')" rounded />
        <b-button :label="$t('Submit')" type="is-primary" @click="emitSubmit" rounded />
      </div>
    </footer>
    <!-- Modal-Card Footer End -->
  </div>
</template>

<script>
import upperFirst from 'lodash/upperFirst'
import lowerFirst from 'lodash/lowerFirst'
import parser from 'yargs-parser'
import concat from 'lodash/concat'
import YAML from 'yamljs'


export default {
  data() {
    return {
      activeTab: 0,
      file: {},
      dropFiles: {},
      dockerCliCommands: "",
      dockerComposeCommands: "",
      parseError: false,
      appFileLoaded: false,
      errors: "",
      dropText: this.$t('Drop your app file here or click to upload'),
      updateData: this.initData
    }
  },
  props: {
    initData: Object,
    netWorks: Array,
    oriNetWorks: Array,
    deviceMemory: Number
  },
  created() {
    //console.log(this.oriNetWorks);

  },
  methods: {
    /**
     * @description: Emit Event to tell parent Update
     * @param {*}
     * @return {*} void
     */
    emitSubmit() {
      if (this.activeTab == 0) {
        if (this.parseCli()) {
          this.errors = ""
          this.$emit('update', this.updateData)
          this.$emit('close')
        } else {
          this.errors = this.$t('Please fill correct command line')
          this.parseError = true;
        }
      } else if (this.activeTab == 1) {

        if (this.parseComposeYaml()) {
          this.errors = ""
          this.$emit('update', this.updateData)
          this.$emit('close')
        } else {
          this.errors = this.$t('Please fill correct compose YAML')
          this.parseError = true;
        }
      } else if (this.activeTab == 2) {
        if (this.appFileLoaded) {
          this.errors = ""
          this.$emit('update', this.updateData)
          this.$emit('close')
        } else {
          this.errors = this.$t('Please import a valid App file')
          this.parseError = true;
        }
      }
    },

    volumeAutoCheck(containerPath, hostPath, appName) {
      let finalHostPath = hostPath
      const rootDir = "/DATA"
      const checkArray = [
        {
          keywords: ["config"],
          value: `/AppData/${appName}${containerPath}`
        },
        {
          keywords: ["tvshows", "TV", "tv"],
          value: `/Media/TV Shows`
        },
        {
          keywords: ["movies", "Movie", "movie"],
          value: `/Media/Movies`
        },
        {
          keywords: ["Music", "music"],
          value: `/Media/Music`
        },
        {
          keywords: ["download"],
          value: `/Downloads`
        },
        {
          keywords: ["pictures", "photo"],
          value: `/Gallery`
        },
        {
          keywords: ["media"],
          value: `/Media`
        }

      ]

      checkArray.forEach(item => {
        if (item.keywords.some(keywordsItem => {
          return containerPath.includes(keywordsItem)
        })) {
          finalHostPath = rootDir + item.value
          return
        }
      })

      return finalHostPath
    },


    /**
     * @description: Parse Import Docker Cli Commands
     * @return {Boolean} 
     */
    parseCli() {
      const formattedInput = this.dockerCliCommands.replace(/<[^>]*>/g, 'Custom_data').replace(/[\r\n]/g, "").replace(/\\/g, "\\ ").replace("-d", "").replace(/[\\]/g, '').trim();
      const parsedInput = parser(formattedInput)
      const { _: command } = parsedInput;
      if (command[0] !== 'docker' || (command[1] !== 'run' && command[1] !== 'create')) {
        return false
      } else {

        //Image
        this.updateData.image = [...command].pop()
        //Label
        if (parsedInput.name != undefined) {
          this.updateData.label = upperFirst(parsedInput.name)
        } else {
          const imageArray = this.updateData.image.split("/")
          const lastNode = [...imageArray].pop()
          this.updateData.label = upperFirst(lastNode.split(":")[0])
        }

        //Envs
        let env = concat(this.makeArray(parsedInput.e), this.makeArray(parsedInput.env))
        this.updateData.envs = env.map(item => {
          let ii = item.split("=");
          return {
            host: ii[1].replace(/"/g, ""),
            container: ii[0]
          }
        })
        //Ports
        let ports = concat(this.makeArray(parsedInput.p), this.makeArray(parsedInput.publish))
        this.updateData.ports = ports.map(item => {
          let pArray = item.split(":")
          let endArray = pArray[1].split("/")
          let protocol = (endArray[1]) ? endArray[1] : 'tcp';
          return {
            container: endArray[0],
            host: pArray[0],
            protocol: protocol
          }
        })
        //Volume
        let volumes = concat(this.makeArray(parsedInput.v), this.makeArray(parsedInput.volume))
        this.updateData.volumes = volumes.map(item => {
          let ii = item.split(":");
          if (ii.length > 1) {
            // console.log(this.volumeAutoCheck(ii[1],ii[0], _.lowerFirst(this.updateData.label)));
            return {
              container: ii[1],
              host: this.volumeAutoCheck(ii[1], ii[0], lowerFirst(this.updateData.label))
            }
          } else {
            return {
              container: ii[0],
              host: this.volumeAutoCheck(ii[0], "", lowerFirst(this.updateData.label))
            }
          }

        })
        // Devices
        this.updateData.devices = this.makeArray(parsedInput.device).map(item => {
          let ii = item.split(":");
          return {
            container: ii[1],
            host: ii[0]
          }
        })

        //Network
        let pnetwork = (parsedInput.network != undefined) ? parsedInput.network : parsedInput.net
        if (pnetwork != undefined) {
          let network = (pnetwork == 'physical') ? 'macvlan' : pnetwork;
          let seletNetworks = this.netWorks.filter(item => {
            if (item.driver == network) {
              return true
            }
          })
          if (seletNetworks.length > 0) {
            this.updateData.network_model = seletNetworks[0].networks[0].name;
          }
        }

        //privileged
        this.updateData.privileged = parsedInput.privileged != undefined
        //cap-add
        this.updateData.cap_add = this.makeArray(parsedInput.capAdd)

        //hostname
        if (parsedInput.hostname != undefined) {
          this.updateData.host_name = parsedInput.hostname
        } else {
          if (parsedInput.h != undefined) {
            this.updateData.host_name = parsedInput.h
          } else {
            this.updateData.host_name = ""
          }
        }

        //Restart
        if (parsedInput.restart != undefined) {
          this.updateData.restart = parsedInput.restart
        }
        return true
      }
    },

    /**
     * @description: Make String to Array
     * @param {*}
     * @return {Array}
     */
    makeArray(foo) {
      const newArray = (typeof (foo) == "string") ? [foo] : foo
      return (newArray == undefined) ? [] : newArray
    },

    /**
     * @description: Parse Import Docker Compose Commands
     * @return {Boolean} 
     */
    parseComposeYaml() {

      try {
        const yaml = YAML.parse(this.dockerComposeCommands)
        if (yaml.version === undefined) {
          return false
        }
        const parsedInput = Object.values(yaml.services)[0]
        // Image
        this.updateData.image = parsedInput.image
        // Label
        if (parsedInput.container_name != undefined) {
          this.updateData.label = upperFirst(parsedInput.container_name)
        } else {
          const imageArray = parsedInput.image.split("/")
          const lastNode = [...imageArray].pop()
          this.updateData.label = upperFirst(lastNode.split(":")[0])
        }
        // Envs
        let envArray = Array.isArray(parsedInput.environment) ? parsedInput.environment : Object.entries(parsedInput.environment)
        this.updateData.envs = envArray.map(item => {
          let ii = typeof item === "object" ? Array.from(item) : item.split("=");
          return {
            host: ii[1].replace(/"/g, ""),
            container: ii[0]
          }
        })

        //Ports
        this.updateData.ports = this.makeArray(parsedInput.ports).map(item => {
          let pArray = item.split(":")
          let endArray = pArray[1].split("/")
          let protocol = (endArray[1]) ? endArray[1] : 'tcp';
          return {
            container: endArray[0],
            host: pArray[0],
            protocol: protocol
          }
        })

        //Volume
        this.updateData.volumes = this.makeArray(parsedInput.volumes).map(item => {
          let ii = item.split(":");
          if (ii.length > 1) {
            return {
              container: ii[1],
              host: this.volumeAutoCheck(ii[1], ii[0], lowerFirst(this.updateData.label))
            }
          } else {
            return {
              container: ii[0],
              host: this.volumeAutoCheck(ii[0], "", lowerFirst(this.updateData.label))
            }
          }
        })

        // Devices
        this.updateData.devices = this.makeArray(parsedInput.device).map(item => {
          let ii = item.split(":");
          return {
            container: ii[1],
            host: ii[0]
          }
        })

        //Network
        let pnetwork = (parsedInput.network_mode != undefined) ? parsedInput.network_mode : (parsedInput.network != undefined) ? parsedInput.network[0] : undefined
        if (pnetwork != undefined) {
          let network = (pnetwork == 'physical') ? 'macvlan' : pnetwork;
          let seletNetworks = this.netWorks.filter(item => {
            if (item.driver == network) {
              return true
            }
          })
          if (seletNetworks.length > 0) {
            this.updateData.network_model = seletNetworks[0].networks[0].name;
          }
        }

        //hostname
        this.updateData.host_name = parsedInput.hostname != undefined ? parsedInput.hostname : ""
        // privileged
        this.updateData.privileged = parsedInput.privileged != undefined

        //cap-add
        if (parsedInput.cap_add != undefined) {
          this.updateData.cap_add = parsedInput.cap_add
        }
        //Restart
        if (parsedInput.restart != undefined) {
          this.updateData.restart = parsedInput.restart
        }

        return true
      } catch (error) {
        console.log(error);
        return false
      }
    },
    // Delete Drop file
    deleteDropFile(index) {
      this.dropFiles.splice(index, 1);
    },
    onSelect(val) {
      const _this = this
      const reader = new FileReader();
      if (typeof FileReader === "undefined") {
        this.$buefy.toast.open({
          duration: 3000,
          message: this.$t('Your browser does not support file reading.'),
          type: 'is-danger'
        })
        return;
      }
      reader.readAsText(val)
      reader.onload = function () {
        try {
          _this.updateData = JSON.parse(this.result);
          if (_this.updateData.version === undefined) {
            _this.$buefy.toast.open({
              duration: 3000,
              message: _this.$t('This is not a valid App file.'),
              type: 'is-danger'
            })
            _this.appFileLoaded = false
            return false
          } else {
            delete _this.updateData.versison
            _this.updateData.network_model = _this.getNetworkModel(_this.updateData.network_model)
            _this.updateData.memory = _this.deviceMemory
            _this.dropText = val.name + " " + _this.$t('has been selected')
            _this.appFileLoaded = true
            return true
          }

        } catch (e) {
          _this.$buefy.toast.open({
            duration: 3000,
            message: _this.$t('This is not a valid json file.'),
            type: 'is-danger'
          })
          _this.appFileLoaded = false
          return false
        }
      }
    },

    getNetworkModel(netName) {
      const network = this.oriNetWorks.filter(net => {
        return net.name == netName
      })
      return (network.length > 0) ? network[0].name : this.oriNetWorks[0].name

    },
  },
}
</script>

<style>
</style>