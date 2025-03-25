<template>
  <!-- backend -->
  <div class="card">
    <div class="card-title px-4 pt-4">
      <div class="indicator">
        <span
          v-if="isCoreUpdateAvailable"
          class="indicator-item top-1 -right-1 flex"
        >
          <span class="bg-secondary absolute h-2 w-2 animate-ping rounded-full"></span>
          <span class="bg-secondary h-2 w-2 rounded-full"></span>
        </span>
        <a
          class="flex cursor-pointer items-center gap-2"
          :href="
            isSingBox
              ? 'https://github.com/sagernet/sing-box'
              : 'https://github.com/metacubex/mihomo'
          "
          target="_blank"
        >
          {{ $t('backend') }}
          <BackendVersion class="text-sm font-normal" />
        </a>
      </div>
    </div>
    <div class="card-body gap-4">
      <BackendSwitch />

      <template v-if="!isSingBox && configs">
        <div class="divider"></div>
        <div class="grid max-w-3xl grid-cols-2 gap-2 lg:grid-cols-3">
          <div
            class="flex items-center gap-2"
            v-for="portConfig in portList"
            :key="portConfig.key"
          >
            <span class="shrink-0"> {{ $t(portConfig.label) }} </span>
            <input
              class="input input-sm w-20 sm:w-24"
              type="number"
              v-model="configs[portConfig.key as keyof Config]"
              @change="
                updateConfigs({ [portConfig.key]: Number(configs[portConfig.key as keyof Config]) })
              "
            />
          </div>
        </div>
        <div class="grid max-w-3xl grid-cols-2 gap-2 lg:grid-cols-4">
          <div
            class="flex items-center gap-2"
            v-if="configs?.tun"
          >
            {{ $t('tunMode') }}
            <input
              class="toggle"
              type="checkbox"
              v-model="configs.tun.enable"
              @change="hanlderTunModeChange"
            />
          </div>
          <div class="flex items-center gap-2">
            {{ $t('allowLan') }}
            <input
              class="toggle"
              type="checkbox"
              v-model="configs['allow-lan']"
              @change="handlerAllowLanChange"
            />
          </div>
          <template v-if="!activeBackend?.disableUpgradeCore">
            <div class="flex items-center gap-2">
              {{ $t('checkUpgrade') }}
              <input
                class="toggle"
                type="checkbox"
                v-model="checkUpgradeCore"
                @change="handlerCheckUpgradeCoreChange"
              />
            </div>
            <div
              class="flex items-center gap-2"
              v-if="checkUpgradeCore"
            >
              {{ $t('autoUpgrade') }}
              <input
                class="toggle"
                type="checkbox"
                v-model="autoUpgradeCore"
              />
            </div>
          </template>
        </div>
      </template>

      <div
        class="grid max-w-4xl grid-cols-2 gap-2 md:grid-cols-3 lg:grid-cols-5"
        v-if="version"
      >
        <template v-if="!isSingBox">
          <button
            v-if="!activeBackend?.disableUpgradeCore"
            :class="twMerge('btn btn-primary btn-sm', isCoreUpgrading ? 'animate-pulse' : '')"
            @click="handlerClickUpgradeCore"
          >
            {{ $t('upgradeCore') }}
          </button>
          <button
            :class="twMerge('btn btn-sm', isCoreRestarting ? 'animate-pulse' : '')"
            @click="handlerClickRestartCore"
          >
            {{ $t('restartCore') }}
          </button>
          </template>
          <!-- 移除 v-if 条件，始终显示 reload 按钮 -->
          <button
            :class="twMerge('btn btn-sm', isConfigReloading ? 'animate-pulse' : '')"
            @click="handlerClickReloadConfigs"
          >
            {{ $t('reloadConfigs') }}
          </button>
          <template v-if="!isSingBox">
          <button
            :class="twMerge('btn btn-sm', isGeoUpdating ? 'animate-pulse' : '')"
            @click="handlerClickUpdateGeo"
          >
            {{ $t('updateGeoDatabase') }}
          </button>
        </template>
        <button
          class="btn btn-sm"
          @click="flushFakeIPAPI"
        >
          {{ $t('flushFakeIP') }}
        </button>
      </div>
      <div class="divider"></div>
      <DnsQuery />
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  flushFakeIPAPI,
  isCoreUpdateAvailable,
  isSingBox,
  reloadConfigsAPI,
  restartCoreAPI,
  updateGeoDataAPI,
  upgradeCoreAPI,
  version,
} from '@/api'
import BackendVersion from '@/components/common/BackendVersion.vue'
import BackendSwitch from '@/components/settings/BackendSwitch.vue'
import DnsQuery from '@/components/settings/DnsQuery.vue'
import { configs, fetchConfigs, updateConfigs } from '@/store/config'
import { fetchProxies } from '@/store/proxies'
import { fetchRules } from '@/store/rules'
import { autoUpgradeCore, checkUpgradeCore } from '@/store/settings'
import { activeBackend } from '@/store/setup'
import type { Config } from '@/types'
import { twMerge } from 'tailwind-merge'
import { ref } from 'vue'

const portList = [
  {
    label: 'mixedPort',
    key: 'mixed-port',
  },
  {
    label: 'httpPort',
    key: 'port',
  },
  {
    label: 'socksPort',
    key: 'socks-port',
  },
  {
    label: 'redirPort',
    key: 'redir-port',
  },
  {
    label: 'tproxyPort',
    key: 'tproxy-port',
  },
]

const reloadAll = () => {
  fetchConfigs()
  fetchRules()
  fetchProxies()
}

const isCoreRestarting = ref(false)
const handlerClickRestartCore = async () => {
  if (isCoreRestarting.value) return
  isCoreRestarting.value = true
  try {
    await restartCoreAPI()
    setTimeout(() => {
      reloadAll()
    }, 500)
    isCoreRestarting.value = false
  } catch {
    isCoreRestarting.value = false
  }
}

const isCoreUpgrading = ref(false)
const handlerClickUpgradeCore = async () => {
  if (isCoreUpgrading.value) return
  isCoreUpgrading.value = true
  try {
    await upgradeCoreAPI()
    reloadAll()
    isCoreUpgrading.value = false
  } catch {
    isCoreUpgrading.value = false
  }
}

const isConfigReloading = ref(false)
const handlerClickReloadConfigs = async () => {
  if (isConfigReloading.value) return
  isConfigReloading.value = true
  try {
    await reloadConfigsAPI()
    reloadAll()
    isConfigReloading.value = false
  } catch {
    isConfigReloading.value = false
  }
}

const isGeoUpdating = ref(false)
const handlerClickUpdateGeo = async () => {
  if (isGeoUpdating.value) return
  isGeoUpdating.value = true
  try {
    await updateGeoDataAPI()
    reloadAll()
    isGeoUpdating.value = false
  } catch {
    isGeoUpdating.value = false
  }
}

const handlerCheckUpgradeCoreChange = () => {
  if (!checkUpgradeCore.value) {
    autoUpgradeCore.value = false
    isCoreUpdateAvailable.value = false
  }
}

const hanlderTunModeChange = async () => {
  await updateConfigs({ tun: { enable: configs.value?.tun.enable } })
}
const handlerAllowLanChange = async () => {
  await updateConfigs({ ['allow-lan']: configs.value?.['allow-lan'] })
}
</script>
