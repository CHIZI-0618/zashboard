<template>
  <div class="flex flex-col gap-3">
    <span>{{ $t('customCardLines') }}</span>
    <div class="flex gap-2">
      <button
        class="btn btn-sm"
        @click="((connectionCardLines = SIMPLE_CARD_STYLE), setRestOfColumns())"
      >
        {{ $t('simpleCardPreset') }}
      </button>
      <button
        class="btn btn-sm"
        @click="((connectionCardLines = DETAILED_CARD_STYLE), setRestOfColumns())"
      >
        {{ $t('detailedCardPreset') }}
      </button>
      <div class="flex-1"></div>
      <button
        class="btn btn-circle btn-neutral btn-sm"
        @click="addLine"
      >
        <PlusCircleIcon class="h-4 w-4" />
      </button>
    </div>
    <div class="relative flex flex-col rounded">
      <div
        v-for="(_, index) in connectionCardLines"
        :key="index"
        :class="`flex items-center gap-2 p-2 ${index % 2 === 0 ? 'bg-base-200' : 'bg-base-300'}`"
      >
        <button
          v-if="connectionCardLines.length > 1"
          class="btn btn-circle btn-xs"
          @click="removeLine(index)"
        >
          <MinusCircleIcon class="h-4 w-4" />
        </button>
        <Draggable
          class="flex flex-1 flex-wrap items-center gap-2"
          v-model="connectionCardLines[index]"
          :animation="150"
          group="list"
          ghostClass="ghost"
          :item-key="(id: string) => id"
        >
          <template #item="{ element }">
            <div
              class="flex h-8 cursor-move select-none items-center rounded bg-neutral px-2 text-neutral-content"
            >
              {{ $t(element) }}
            </div>
          </template>
        </Draggable>
      </div>

      <Draggable
        class="flex flex-1 flex-wrap gap-2 p-2"
        v-model="restOfColumns"
        :animation="150"
        group="list"
        ghostClass="ghost"
        :item-key="(id: string) => id"
      >
        <template #item="{ element }">
          <div
            class="flex h-8 cursor-move select-none items-center rounded bg-neutral px-2 text-neutral-content"
          >
            {{ $t(element) }}
          </div>
        </template>
      </Draggable>
    </div>
  </div>
</template>

<script setup lang="ts">
import { CONNECTIONS_TABLE_ACCESSOR_KEY, DETAILED_CARD_STYLE, SIMPLE_CARD_STYLE } from '@/config'
import { connectionCardLines } from '@/store/settings'
import { MinusCircleIcon, PlusCircleIcon } from '@heroicons/vue/24/outline'
import { ref } from 'vue'
import Draggable from 'vuedraggable'

const restOfColumns = ref<CONNECTIONS_TABLE_ACCESSOR_KEY[]>([])

const setRestOfColumns = () => {
  restOfColumns.value = Object.values(CONNECTIONS_TABLE_ACCESSOR_KEY).filter(
    (key) => !connectionCardLines.value.flat().includes(key),
  )
}

setRestOfColumns()

const addLine = () => {
  connectionCardLines.value = [
    ...connectionCardLines.value,
    restOfColumns.value[0] ? [restOfColumns.value[0]] : [],
  ]
  setRestOfColumns()
}

const removeLine = (index: number) => {
  connectionCardLines.value.splice(index, 1)
  setRestOfColumns()
}
</script>
