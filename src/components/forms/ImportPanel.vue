<script>
import { parse, stringify } from 'yaml'
import composerize from 'composerize'
import lowerFirst from 'lodash/lowerFirst'

export default {
  props: {
    netWorks: Array,
    oriNetWorks: Array,
    deviceMemory: Number,
  },
  emits: ['update', 'close'],
  data() {
    return {
      activeTab: 0,
      file: {},
      dropFiles: {},
      dockerCliCommands: '',
      dockerComposeCommands: '',
      appFileLoaded: false,
      errors: '',
      dropText: this.$t('Drop your Docker Compose file here or click to upload'),
      uploadIcon: 'upload',
    }
  },
  computed: {
    yamlValid() {
      if (this.activeTab !== 0) {
        return true
      }
      const txt = String(this.dockerComposeCommands || '').trim()
      if (!txt) {
        return true
      }
      try {
        parse(txt)
        return true
      }
      catch {
        return false
      }
    },
    yamlError() {
      if (this.activeTab !== 0) {
        return ''
      }
      const txt = String(this.dockerComposeCommands || '').trim()
      if (!txt) {
        return ''
      }
      try {
        parse(txt)
        return ''
      }
      catch (e) {
        return e && e.message ? e.message : String(e)
      }
    },
  },
  methods: {
    /**
     * @description: Emit Event to tell parent Update
     * @return {*} void
     */
    emitSubmit() {
      switch (this.activeTab) {
        case 0:
          this.dockerComposeCommands = this.addTitleToYaml(this.dockerComposeCommands)
          this.errors = ''
          this.$emit('update', this.dockerComposeCommands)
          this.$emit('close')
          break
        case 1: {
          const cleanedCommand = this.dockerCliCommands.replace(/`#.*?`/g, '').replace(/#.*$/gm, '').trim()
          this.dockerComposeCommands = composerize(cleanedCommand)
          this.dockerComposeCommands = this.addTitleToYaml(this.dockerComposeCommands)
          this.$emit('update', this.dockerComposeCommands)
          this.$emit('close')
          break
        }
        case 2:
          if (this.appFileLoaded) {
            this.errors = ''
            this.$emit('close')
          }
          else {
            this.errors = this.$t('Please import a valid App file')
            this.parseError = true
          }
          break
      }
    },

    addTitleToYaml(yaml) {
      const yamlObj = parse(yaml)
      if (yamlObj['x-casaos'] !== undefined)
        return yaml

      const serviceName = Object.keys(yamlObj.services)[0]
      if (serviceName === undefined)
        return stringify(yamlObj)
      yamlObj['x-casaos'] = {}
      yamlObj['x-casaos'].title = {}
      yamlObj['x-casaos'].title.en_us = serviceName
      yamlObj['x-casaos'].icon = `https://icon.casaos.io/main/all/${serviceName}.png`
      return stringify(yamlObj)
    },

    volumeAutoCheck(containerPath, hostPath, appName) {
      let finalHostPath = hostPath
      const rootDir = '/DATA'
      const checkArray = [
        {
          keywords: ['config'],
          value: `/AppData/${appName}${containerPath}`,
        },
        {
          keywords: ['tvshows', 'TV', 'tv'],
          value: `/Media/TV Shows`,
        },
        {
          keywords: ['movies', 'Movie', 'movie'],
          value: `/Media/Movies`,
        },
        {
          keywords: ['Music', 'music'],
          value: `/Media/Music`,
        },
        {
          keywords: ['download'],
          value: `/Downloads`,
        },
        {
          keywords: ['pictures', 'photo'],
          value: `/Gallery`,
        },
        {
          keywords: ['media'],
          value: `/Media`,
        },
      ]
      checkArray.forEach((item) => {
        if (item.keywords.some((keywordsItem) => {
          return containerPath.includes(keywordsItem)
        })) {
          finalHostPath = rootDir + item.value
        }
      })
      return finalHostPath
    },

    /**
     * @description: Make String to Array
     * @param {*} foo
     * @return {Array}
     */
    makeArray(foo) {
      const newArray = (typeof (foo) == 'string') ? [foo] : foo
      return (newArray === undefined) ? [] : newArray
    },

    autoFillVolumePaths() {
      try {
        const yaml = parse(this.dockerComposeCommands)
        if (!yaml || !yaml.services) {
          this.errors = this.$t('Invalid YAML')
          return
        }
        const appNameRaw = yaml?.['x-casaos']?.title?.en_us || Object.keys(yaml.services)[0] || ''
        const appName = lowerFirst(appNameRaw)
        for (const serviceKey in yaml.services) {
          const service = yaml.services[serviceKey]
          if (!service || !service.volumes) {
            continue
          }
          service.volumes = this.makeArray(service.volumes).map((item) => {
            if (typeof item === 'string') {
              const parts = item.split(':')
              if (parts.length > 1) {
                const containerPath = parts[1]
                const hostPath = parts[0]
                const rest = parts.slice(2).join(':')
                const newHost = (!hostPath || hostPath === '' || hostPath === '/' || hostPath === containerPath)
                  ? this.volumeAutoCheck(containerPath, hostPath, appName)
                  : hostPath
                return rest ? `${newHost}:${containerPath}:${rest}` : `${newHost}:${containerPath}`
              }
              return item
            }
            if (item && typeof item === 'object') {
              if (item.target && item.source !== undefined) {
                const src = item.source
                if (!src || src === '' || src === '/' || src === item.target) {
                  item.source = this.volumeAutoCheck(item.target, src, appName)
                }
              }
              return item
            }
            return item
          })
        }
        this.dockerComposeCommands = stringify(yaml)
        this.errors = ''
      }
      catch (e) {
        this.errors = e && e.message ? e.message : String(e)
      }
    },

    checkYAML() {
      const yaml = parse(this.dockerComposeCommands)
      if (!(yaml?.name in yaml.services)) {
        this.errors = this.$t('Please select a service name in the "services" and add it as the value of the top-level attribute "name" to serve as the main application.')
        return false
      }
      return true
    },
    onSelect(val) {
      const _this = this
      const reader = new FileReader()
      if (typeof FileReader === 'undefined') {
        this.$buefy.toast.open({
          duration: 3000,
          message: this.$t('Your browser does not support file reading.'),
          type: 'is-danger',
        })
        return
      }
      reader.readAsText(val)
      reader.onload = function () {
        _this.dockerComposeCommands = this.result
      }
    },
    clearInput() {
      this.uploadIcon = 'upload'
      this.dropText = this.$t('Drop your Docker Compose file here or click to upload')
      this.appFileLoaded = false
      this.$refs.importUpload.clearInput()
      this.$buefy.toast.open({
        duration: 3000,
        message: this.$t('This is not a valid json file.'),
        type: 'is-danger',
      })
      this.appFileLoaded = false
    },

    getNetworkModel(netName) {
      const network = this.oriNetWorks.filter((net) => {
        return net.name === netName
      })
      return (network.length > 0) ? network[0].name : this.oriNetWorks[0].name
    },
  },
}
</script>

<template>
  <div class="modal-card">
    <!-- Modal-Card Header Start -->
    <header class="modal-card-head">
      <div class="is-flex-grow-1">
        <h3 class="title is-header">
          {{ $t('Import') }}
        </h3>
      </div>
    </header>
    <!-- Modal-Card Header End -->
    <!-- Modal-Card Body Start -->
    <section class="modal-card-body">
      <b-tabs v-model="activeTab" :animated="false">
        <b-tab-item label="Docker Compose">
          <b-field :message="yamlError || errors" :type="{ 'is-danger': !!(yamlError || errors) }">
            <b-input v-model="dockerComposeCommands" :placeholder="$t('Notice: If there are multiple services, only the first set can be analyzed correctly')" class="import-area" type="textarea" />
          </b-field>
          <div class="is-flex is-justify-content-flex-end mb-2">
            <b-button size="is-small" rounded type="is-light" @click="autoFillVolumePaths">
              {{ $t('Auto-fill volume paths') }}
            </b-button>
          </div>

          <b-upload ref="importUpload" v-model="dropFiles" accept=".yaml,.yml" drag-drop expanded @input="onSelect">
            <section class="section">
              <div class="content has-text-centered">
                <p>
                  <b-icon :icon="uploadIcon" custom-size="is-size-2" size="is-40" />
                </p>
                <p class="has-text-full-03">
                  {{ dropText }}
                </p>
              </div>
            </section>
          </b-upload>
        </b-tab-item>
        <b-tab-item label="Docker CLI">
          <b-field :message="errors" :type="{ 'is-danger': !!errors }" class="mb-0">
            <b-input v-model="dockerCliCommands" class="import-area-cli" type="textarea" />
          </b-field>
        </b-tab-item>

        <b-tab-item v-if="false" :label="$t('AppFile')">
          <b-field :message="errors" :type="{ 'is-danger': !!errors }">
            <b-upload ref="importUpload" v-model="dropFiles" accept="application/json" drag-drop expanded @input="onSelect">
              <section class="section">
                <div class="content has-text-centered">
                  <p>
                    <b-icon :icon="uploadIcon" size="is-large" />
                  </p>
                  <p>{{ dropText }}</p>
                </div>
              </section>
            </b-upload>
          </b-field>
        </b-tab-item>
      </b-tabs>
    </section>
    <!-- Modal-Card Body End -->
    <!-- Modal-Card Footer Start -->
    <footer class="modal-card-foot is-flex is-align-items-center">
      <div class="is-flex-grow-1 has-text-full-04" />
      <div>
        <b-button :label="$t('Cancel')" rounded @click="$emit('close')" />
        <b-button :label="$t('Submit')" rounded type="is-primary" :disabled="activeTab === 0 && !yamlValid" :title="activeTab === 0 && !yamlValid ? $t('Fix YAML errors before submitting') : ''" @click="emitSubmit" />
      </div>
    </footer>
    <!-- Modal-Card Footer End -->
  </div>
</template>

<style lang="scss" scoped>
.import-area {
  ::v-deep .textarea {
    height: 22rem;
  }
}

.import-area-cli {
  ::v-deep .textarea {
    height: 30rem;
  }
}

.control {
  min-height: 7.25rem;

  .section {
    padding: 0.84375rem;
  }
}
</style>
