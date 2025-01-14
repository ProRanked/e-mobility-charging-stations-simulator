<template>
  <Container id="charging-stations-container">
    <Container
      v-show="Array.isArray(uiServerConfigurations) && uiServerConfigurations.length > 1"
      id="ui-server-container"
    >
      <select
        id="ui-server-selector"
        v-model="state.uiServerIndex"
        @change="
          () => {
            if (
              getFromLocalStorage<number>('uiServerConfigurationIndex', 0) !== state.uiServerIndex
            ) {
              app?.appContext.config.globalProperties.$uiClient.setConfiguration(
                app?.appContext.config.globalProperties.$configuration.uiServer[state.uiServerIndex]
              )
              initializeWSEventListeners()
              app?.appContext.config.globalProperties.$uiClient.registerWSEventListener(
                'open',
                () => {
                  setToLocalStorage<number>('uiServerConfigurationIndex', state.uiServerIndex)
                  $router.currentRoute.value.name !== 'charging-stations' &&
                    $router.push({ name: 'charging-stations' })
                },
                { once: true }
              )
              app?.appContext.config.globalProperties.$uiClient.registerWSEventListener(
                'error',
                () => {
                  state.uiServerIndex = getFromLocalStorage<number>('uiServerConfigurationIndex', 0)
                  app?.appContext.config.globalProperties.$uiClient.setConfiguration(
                    app?.appContext.config.globalProperties.$configuration.uiServer[
                      getFromLocalStorage<number>('uiServerConfigurationIndex', 0)
                    ]
                  )
                  initializeWSEventListeners()
                },
                { once: true }
              )
            }
          }
        "
      >
        <option
          v-for="uiServerConfiguration in uiServerConfigurations"
          :value="uiServerConfiguration.index"
        >
          {{ uiServerConfiguration.configuration.name ?? uiServerConfiguration.configuration.host }}
        </option>
      </select>
    </Container>
    <Container id="buttons-container">
      <Button @click="startSimulator()">Start Simulator</Button>
      <Button @click="stopSimulator()">Stop Simulator</Button>
      <Button @click="$router.push({ name: 'add-charging-stations' })">
        Add Charging Stations
      </Button>
      <ReloadButton
        id="reload-button"
        :loading="state.loading"
        @click="loadChargingStations(() => (state.renderChargingStationsList = randomUUID()))"
      />
    </Container>
    <CSTable
      v-show="
        Array.isArray(app?.appContext.config.globalProperties.$chargingStations) &&
        app?.appContext.config.globalProperties.$chargingStations.length > 0
      "
      :key="state.renderChargingStationsList"
      :charging-stations="app?.appContext.config.globalProperties.$chargingStations"
    />
  </Container>
</template>

<script setup lang="ts">
import { getCurrentInstance, onMounted, ref } from 'vue'
import { useToast } from 'vue-toast-notification'
import CSTable from '@/components/charging-stations/CSTable.vue'
import type { ResponsePayload, UIServerConfigurationSection } from '@/types'
import Container from '@/components/Container.vue'
import ReloadButton from '@/components/buttons/ReloadButton.vue'
import Button from '@/components/buttons/Button.vue'
import { getFromLocalStorage, setToLocalStorage } from '@/composables'

const randomUUID = (): `${string}-${string}-${string}-${string}-${string}` => {
  return crypto.randomUUID()
}

const app = getCurrentInstance()

const initializeWSEventListeners = () => {
  app?.appContext.config.globalProperties.$uiClient.registerWSEventListener('open', () => {
    uiClient
      .listTemplates()
      .then((response: ResponsePayload) => {
        if (app != null) {
          app.appContext.config.globalProperties.$templates = response.templates
        }
      })
      .catch((error: Error) => {
        if (app != null) {
          app.appContext.config.globalProperties.$templates = []
        }
        $toast.error('Error at fetching charging station templates')
        console.error('Error at fetching charging station templates:', error)
      })
    loadChargingStations(() => (state.value.renderChargingStationsList = randomUUID()))
  })
  app?.appContext.config.globalProperties.$uiClient.registerWSEventListener('error', () => {
    app.appContext.config.globalProperties.$chargingStations = []
    state.value.renderChargingStationsList = randomUUID()
  })
  app?.appContext.config.globalProperties.$uiClient.registerWSEventListener('close', () => {
    app.appContext.config.globalProperties.$chargingStations = []
    state.value.renderChargingStationsList = randomUUID()
  })
}

onMounted(() => {
  initializeWSEventListeners()
})

const state = ref({
  renderChargingStationsList: randomUUID(),
  loading: false,
  uiServerIndex: getFromLocalStorage<number>('uiServerConfigurationIndex', 0)
})

const uiClient = app?.appContext.config.globalProperties.$uiClient
const uiServerConfigurations: { configuration: UIServerConfigurationSection; index: number }[] =
  app?.appContext.config.globalProperties.$configuration.uiServer.map(
    (configuration: UIServerConfigurationSection, index: number) => ({
      configuration,
      index
    })
  )

const $toast = useToast()

const loadChargingStations = (renderCallback?: () => void): void => {
  if (state.value.loading === false) {
    state.value.loading = true
    uiClient
      .listChargingStations()
      .then((response: ResponsePayload) => {
        if (app != null) {
          app.appContext.config.globalProperties.$chargingStations = response.chargingStations
        }
      })
      .catch((error: Error) => {
        if (app != null) {
          app.appContext.config.globalProperties.$chargingStations = []
        }
        $toast.error('Error at fetching charging stations')
        console.error('Error at fetching charging stations:', error)
      })
      .finally(() => {
        if (renderCallback != null) {
          renderCallback()
        }
        state.value.loading = false
      })
  }
}

const startSimulator = (): void => {
  uiClient
    .startSimulator()
    .then(() => {
      $toast.success('Simulator successfully started')
    })
    .catch((error: Error) => {
      $toast.error('Error at starting simulator')
      console.error('Error at starting simulator:', error)
    })
}
const stopSimulator = (): void => {
  uiClient
    .stopSimulator()
    .then(() => {
      if (app != null) {
        app.appContext.config.globalProperties.$chargingStations = []
      }
      $toast.success('Simulator successfully stopped')
    })
    .catch((error: Error) => {
      $toast.error('Error at stopping simulator')
      console.error('Error at stopping simulator:', error)
    })
}
</script>

<style>
#charging-stations-container {
  height: fit-content;
  width: 100%;
  display: flex;
  flex-direction: column;
}

#ui-server-container {
  display: flex;
  flex-direction: row;
}

#ui-server-selector {
  width: 100%;
  text-align: center;
}

#buttons-container {
  display: flex;
  flex-direction: row;
}

#action-button {
  flex: none;
}

#reload-button {
  flex: auto;
  color: white;
  background-color: blue;
  font-size: 1.5rem;
  font-weight: bold;
  align-items: center;
  justify-content: center;
}

#reload-button:hover {
  background-color: rgb(0, 0, 225);
}

#reload-button:active {
  background-color: red;
}

#action {
  color: white;
  background-color: black;
  padding: 1%;
}
</style>
