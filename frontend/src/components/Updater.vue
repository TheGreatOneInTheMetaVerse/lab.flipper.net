<template>
  <div class="updater column flex-center text-center">
    <template v-if="!flags.updateInProgress">
      <template v-if="flags.ableToUpdate && info.storage.sdcard.status">
        <template v-if="flags.outdated !== undefined">
          <p v-if="flags.outdated">Your firmware is out of date, newest release is {{ channels.release.version }}.</p>
          <p v-else-if="flags.aheadOfRelease">Your firmware is ahead of current release.</p>
          <p v-else-if="info.firmware.version !== 'unknown'">Your firmware is up to date.</p>
        </template>
        <p v-if="channels.custom">
          Detected custom firmware <b>"{{ channels.custom.channel }}"</b>
          <span v-if="!channels.custom.url.endsWith('tgz')"> with <b>unsupported</b> filetype</span>
        </p>
        <div class="flex q-mt-sm">
          <q-select
            v-model="fwModel"
            :options="fwOptions"
            label="Choose firmware"
            :suffix="fwOptions.find(({label}) => label === fwModel.label) ? fwOptions.find(({label}) => label === fwModel.label).version : ''"
            id="fw-select"
            :style="!$q.screen.xs ? 'width: 320px;' : 'width: 290px;'"
          >
            <template v-slot:option="scope">
              <q-item v-bind="scope.itemProps">
                <q-item-section class="items-start">
                  <q-item-label v-text="scope.opt.label" />
                </q-item-section>
                <q-item-section class="items-end">
                  <q-item-label v-text="scope.opt.version" :class="'fw-option-label ' + scope.opt.value"/>
                </q-item-section>
              </q-item>
            </template>
          </q-select>
          <q-btn
            v-if="fwModel"
            @click="update(false)"
            color="positive"
            padding="12px 30px"
            :class="!$q.screen.xs ? 'q-ml-lg' : 'q-mt-sm'"
          >Install</q-btn>
        </div>
        <q-btn
          v-if="mainFlags.installFromFile && flags.uploadEnabled"
          @click="flags.uploadPopup = true; uploadedFile = null"
          class="q-mt-lg"
          outline
          color="grey-8"
        >Install from file</q-btn>
      </template>
      <template v-else>
        <span v-if="info.storage.sdcard.status">Your firmware doesn't support self-update. Install latest release with <a href="https://update.flipperzero.one" target="_blank">qFlipper desktop tool</a>.</span>
        <span v-else>Self-update is impossible without an SD card.</span>
      </template>
    </template>
    <template v-else>
      <p>{{ updateStage }}</p>
      <q-btn
        v-if="flags.updateError"
        outline
        class="q-mt-md"
        @click="flags.updateInProgress = false; flags.updateError = false"
      >Cancel</q-btn>
      <ProgressBar
        v-if="write.filename.length > 0"
        :title="write.filename"
        :progress="write.progress"
      />
    </template>
    <q-dialog v-model="flags.uploadPopup">
      <q-card>
        <q-card-section class="q-pt-none">
          <q-file
            outlined
            v-model="uploadedFile"
            label="Drop or select files"
            class="q-pt-md"
            :style="$q.screen.width > 380 ? 'width: 300px;' : ''"
          >
            <template v-slot:prepend>
              <q-icon name="file_upload"></q-icon>
            </template>
          </q-file>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn
            flat
            label="Upload"
            v-close-popup
            @click="update(true)"
          ></q-btn>
          <q-btn
            flat
            label="Cancel"
            color="negative"
            v-close-popup
          ></q-btn>
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>

<script setup>
import { ref, defineEmits, watch, onMounted, computed } from 'vue'
import ProgressBar from './ProgressBar.vue'
import { unpack } from 'util/util'
import { fetchChannels, fetchFirmware, fetchRegions } from 'util/fetch'
import semver from 'semver'
import asyncSleep from 'simple-async-sleep'
import { PB } from '../flipper-js/protobufCompiled'
import { log } from 'composables/useLog'
import showNotif from 'composables/useShowNotif'
import { rpcErrorHandler } from 'composables/useRpcUtils'

import { useMainStore } from 'src/stores/main'
const mainStore = useMainStore()

const mainFlags = computed(() => mainStore.flags)
const flipper = computed(() => mainStore.flipper)

const info = computed(() => mainStore.info)

const emit = defineEmits(['update'])

const componentName = 'Updater'
const flags = ref({
  restarting: false,
  rpcActive: false,
  rpcToggling: false,
  outdated: undefined,
  aheadOfRelease: false,
  ableToUpdate: true,
  updateInProgress: false,
  updateError: false,
  uploadEnabled: true,
  uploadPopup: false,
  overrideDevRegion: false
})
const channels = ref({})
const fwOptions = ref([
  {
    label: 'Release', value: 'release', version: ''
  },
  {
    label: 'Release-candidate', value: 'rc', version: ''
  },
  {
    label: 'Dev (unstable)', value: 'dev', version: ''
  }
])
const fwModel = ref(fwOptions.value[0])
const updateStage = ref('')
const write = ref({
  filename: '',
  progress: 0
})
const uploadedFile = ref(null)

watch(fwModel, (newModel) => {
  localStorage.setItem('selectedFwChannel', newModel.value)
})

const update = async (fromFile) => {
  if (!info.value.storage.sdcard.status.isInstalled) {
    mainStore.toggleFlag('microSDcardMissingDialog', true)
    return
  }
  flags.value.updateInProgress = true
  emit('update', 'start')
  log({
    level: 'info',
    message: `${componentName}: Update started`
  })
  if (fromFile) {
    if (!uploadedFile.value) {
      flags.value.updateError = true
      emit('update', 'end')
      updateStage.value = 'No file selected'
      throw new Error('No file selected')
    } else if (!uploadedFile.value.name.endsWith('.tgz')) {
      flags.value.updateError = true
      emit('update', 'end')
      updateStage.value = 'Wrong file format'
      throw new Error('Wrong file format')
    }
    log({
      level: 'info',
      message: `${componentName}: Uploading firmware from file`
    })
  }
  await loadFirmware(fromFile)
    .catch(error => {
      flags.value.updateError = true
      updateStage.value = error
      showNotif({
        message: error.toString(),
        color: 'negative'
      })
      log({
        level: 'error',
        message: `${componentName}: ${error.toString()}`
      })
      throw error
    })
  // flags.value.updateInProgress = false
}

const loadFirmware = async (fromFile) => {
  updateStage.value = 'Loading firmware bundle...'
  if (info.value.hardware.region !== '0' || flags.value.overrideDevRegion) {
    const regions = await fetchRegions()
      .catch(error => {
        showNotif({
          message: 'Failed to fetch regional update information',
          color: 'negative'
        })
        log({
          level: 'error',
          message: `${componentName}: Failed to fetch regional update information: ${error.toString()}`
        })
        throw error
      })

    let bands
    if (regions.countries[regions.country]) {
      bands = regions.countries[regions.country].map(e => regions.bands[e])
    } else {
      bands = regions.default.map(e => regions.bands[e])
      regions.country = 'JP'
    }
    const options = {
      countryCode: regions.country,
      bands: []
    }

    for (const band of bands) {
      const bandOptions = {
        start: band.start,
        end: band.end,
        powerLimit: band.max_power,
        dutyCycle: band.duty_cycle
      }
      const message = PB.Region.Band.create(bandOptions)
      options.bands.push(message)
    }

    log({
      level: 'debug',
      message: `${componentName}: Region provisioning message: ${JSON.stringify(options)}`
    })

    options.countryCode = new TextEncoder().encode(regions.country)
    const message = PB.Region.create(options)
    const encoded = new Uint8Array(PB.Region.encodeDelimited(message).finish()).slice(1)

    await flipper.value.RPC('storageWrite', { path: '/int/.region_data', buffer: encoded })
      .catch(error => rpcErrorHandler(error, 'storageWrite'))

    log({
      level: 'info',
      message: `${componentName}: Set Sub-GHz region: ${regions.country}`
    })
  }

  if (fromFile || (channels.value[fwModel.value.value] && channels.value[fwModel.value.value].url)) {
    let files
    if (!fromFile) {
      files = await fetchFirmware(channels.value[fwModel.value.value].url)
        .catch(error => {
          flags.value.updateError = true
          updateStage.value = error
          showNotif({
            message: 'Failed to fetch firmware: ' + error.toString(),
            color: 'negative'
          })
          log({
            level: 'error',
            message: `${componentName}: Failed to fetch firmware: ${error.toString()}`
          })
          throw error
        })
        .finally(() => {
          log({
            level: 'debug',
            message: `${componentName}: Downloaded firmware from ${channels.value[fwModel.value.value].url}`
          })
        })
    } else {
      const buffer = await uploadedFile.value.arrayBuffer()
      files = await unpack(buffer)
        .finally(() => {
          log({
            level: 'debug',
            message: `${componentName}: Unpacked firmware`
          })
        })
    }

    updateStage.value = 'Loading firmware files'
    log({
      level: 'info',
      message: `${componentName}: Loading firmware files`
    })

    let path = '/ext/update/'
    await flipper.value.RPC('storageStat', { path: '/ext/update' })
      .catch(async error => {
        if (error.toString() !== 'ERROR_STORAGE_NOT_EXIST') {
          rpcErrorHandler(componentName, error, 'storageStat')
        } else {
          await flipper.value.RPC('storageMkdir', { path: '/ext/update' })
            .catch(error => rpcErrorHandler(componentName, error, 'storageMkdir'))
            .finally(() => {
              log({
                level: 'debug',
                message: `${componentName}: storageMkdir: /ext/update`
              })
            })
        }
      })

    for (const file of files) {
      if (file.size === 0) {
        path = '/ext/update/' + file.name
        if (file.name.endsWith('/')) {
          path = path.slice(0, -1)
        }
        await flipper.value.RPC('storageMkdir', { path })
          .catch(error => rpcErrorHandler(componentName, error, 'storageMkdir'))
          .finally(() => {
            log({
              level: 'debug',
              message: `${componentName}: storageMkdir: ${path}`
            })
          })
      } else {
        write.value.filename = file.name.slice(file.name.lastIndexOf('/') + 1)
        const unbind = flipper.value.emitter.on('storageWriteRequest/progress', e => {
          write.value.progress = e.progress / e.total
        })
        await flipper.value.RPC('storageWrite', { path: '/ext/update/' + file.name, buffer: file.buffer })
          .catch(error => rpcErrorHandler(componentName, error, 'storageWrite'))
          .finally(() => {
            log({
              level: 'debug',
              message: `${componentName}: storageWrite: /ext/update/${file.name}`
            })
          })
        unbind()
      }
      await asyncSleep(300)
    }
    write.value.filename = ''
    write.value.progress = 0

    updateStage.value = 'Loading manifest...'
    log({
      level: 'info',
      message: `${componentName}: Loading update manifest`
    })

    await flipper.value.RPC('systemUpdate', { path: path + '/update.fuf' })
      .catch(error => rpcErrorHandler(componentName, error, 'systemUpdate'))
      .finally(() => {
        log({
          level: 'debug',
          message: `${componentName}: systemUpdate: OK`
        })
      })

    updateStage.value = 'Update in progress, pay attention to your Flipper'
    log({
      level: 'info',
      message: `${componentName}: Rebooting Flipper`
    })

    await flipper.value.RPC('systemReboot', { mode: 'UPDATE' })
      .catch(error => rpcErrorHandler(componentName, error, 'systemReboot'))
  } else {
    flags.value.updateError = true
    updateStage.value = 'Failed to fetch channel'
    showNotif({
      message: 'Unable to load firmware channel from the build server. Reload the page and try again.',
      color: 'negative',
      reloadBtn: true
    })
    log({
      level: 'error',
      message: `${componentName}: Failed to fetch channel`
    })
  }
}

const compareVersions = () => {
  if (semver.lt((info.value.protobuf.version.major + '.' + info.value.protobuf.version.minor) + '.0', '0.6.0')) {
    flags.value.ableToUpdate = false
  }
  if (info.value.firmware.version) {
    if (info.value.firmware.version !== 'unknown' && semver.valid(info.value.firmware.version)) {
      if (semver.eq(info.value.firmware.version, channels.value.release.version)) {
        flags.value.outdated = false
      } else if (semver.gt(info.value.firmware.version, channels.value.release.version)) {
        flags.value.outdated = false
        flags.value.aheadOfRelease = true
      } else {
        flags.value.outdated = true
      }
    } else {
      flags.value.outdated = undefined
    }
  }
}

onMounted(async () => {
  channels.value = await fetchChannels(info.value.hardware.target)
    .catch(error => {
      showNotif({
        message: 'Unable to load firmware channels from the build server. Reload the page and try again.',
        color: 'negative',
        reloadBtn: true
      })
      log({
        level: 'error',
        message: `${componentName}: failed to fetch update channels`
      })
      throw error
    })
  compareVersions()
  fwOptions.value[0].version = channels.value.release.version
  fwOptions.value[1].version = channels.value.rc.version
  fwOptions.value[2].version = channels.value.dev.version
  if (channels.value.custom && channels.value.custom.url.endsWith('tgz')) {
    fwOptions.value.unshift({
      label: channels.value.custom.channel || 'Custom', value: 'custom', version: channels.value.custom.version || 'unknown'
    })
  }

  const selectedBefore = fwOptions.value.find(e => e.value === localStorage.getItem('selectedFwChannel'))
  if (selectedBefore && !channels.value.custom) {
    fwModel.value = selectedBefore
  } else {
    fwModel.value = fwOptions.value[0]
  }

  if (new URLSearchParams(location.search).get('overrideDevRegion') === 'true') {
    flags.value.overrideDevRegion = true
  }
})
</script>
