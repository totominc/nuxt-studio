<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useStudio } from '../../composables/useStudio'
import { isImageFile, isVideoFile } from '../../utils/file'
import { Image } from '@unpic/vue'
import type { TreeItem as StudioTreeItem } from '../../types'
import { StudioFeature } from '../../types'

const { mediaTree, context } = useStudio()

const props = defineProps<{ open: boolean, type: 'image' | 'video' }>()

const emit = defineEmits<{
  select: [image: StudioTreeItem | null]
  cancel: []
}>()

interface MediaFolderTreeItem {
  label: string
  fsPath: string
  defaultExpanded?: boolean
  children?: MediaFolderTreeItem[]
}

const selectedFolderFsPath = ref(mediaTree.rootItem.value.fsPath)

const isValidFileType = (item: StudioTreeItem) => {
  if (props.type === 'image') {
    return isImageFile(item.fsPath)
  }
  if (props.type === 'video') {
    return isVideoFile(item.fsPath)
  }
  return false
}

/**
 * Builds a directory-only tree for the folder picker from the studio media tree.
 *
 * @param item The source media tree item.
 * @param depth The current nesting depth.
 * @returns The folder node shaped for `UTree`.
 */
function buildFolderTreeItem(item: StudioTreeItem, depth: number = 0): MediaFolderTreeItem {
  const directories = (item.children || []).filter(child => child.type === 'directory')

  return {
    label: item.name,
    fsPath: item.fsPath,
    defaultExpanded: depth < 2,
    children: directories.length > 0
      ? directories.map(child => buildFolderTreeItem(child, depth + 1))
      : undefined,
  }
}

/**
 * Finds a folder node in the picker tree by filesystem path.
 *
 * @param items The folder tree items to inspect.
 * @param fsPath The folder path to find.
 * @returns The matching folder node, if found.
 */
function findFolderTreeItem(items: MediaFolderTreeItem[], fsPath: string): MediaFolderTreeItem | undefined {
  for (const item of items) {
    if (item.fsPath === fsPath) {
      return item
    }

    if (!item.children?.length) {
      continue
    }

    const matchingChild = findFolderTreeItem(item.children, fsPath)
    if (matchingChild) {
      return matchingChild
    }
  }
}

/**
 * Finds a media directory in the studio tree by filesystem path.
 *
 * @param item The current media tree node.
 * @param fsPath The folder path to find.
 * @returns The matching directory node, if found.
 */
function findDirectoryItem(item: StudioTreeItem, fsPath: string): StudioTreeItem | undefined {
  if (item.type !== 'file' && item.fsPath === fsPath) {
    return item
  }

  for (const child of item.children || []) {
    if (child.type === 'file') {
      continue
    }

    const matchingChild = findDirectoryItem(child, fsPath)
    if (matchingChild) {
      return matchingChild
    }
  }
}

/**
 * Checks whether the current media tree contains any selectable assets.
 *
 * @param items The tree items to inspect.
 * @returns `true` when at least one matching asset exists.
 */
function hasSelectableMedia(items: StudioTreeItem[]): boolean {
  for (const item of items) {
    if (item.type === 'file' && !item.fsPath.endsWith('.gitkeep') && isValidFileType(item)) {
      return true
    }

    if (item.children?.length && hasSelectableMedia(item.children)) {
      return true
    }
  }

  return false
}

const folderTreeItems = computed<MediaFolderTreeItem[]>(() => [
  buildFolderTreeItem(mediaTree.rootItem.value),
])

const getFolderTreeItemKey = (item: MediaFolderTreeItem) => item.fsPath

const selectedFolderTreeItem = computed<MediaFolderTreeItem>({
  get() {
    return findFolderTreeItem(folderTreeItems.value, selectedFolderFsPath.value) || folderTreeItems.value[0]
  },
  set(item) {
    selectedFolderFsPath.value = item.fsPath
  },
})

const selectedFolder = computed(() => {
  return findDirectoryItem(mediaTree.rootItem.value, selectedFolderFsPath.value) || mediaTree.rootItem.value
})

const selectedFolderPath = computed(() => {
  if (selectedFolder.value.type === 'root') {
    return selectedFolder.value.name
  }

  return `${mediaTree.rootItem.value.name}/${selectedFolder.value.fsPath}`
})

const hasAnyMediaFiles = computed(() => {
  return hasSelectableMedia(mediaTree.root.value)
})

const mediaFiles = computed(() => {
  return (selectedFolder.value.children || []).filter((item): item is StudioTreeItem => {
    return item.type === 'file' && !item.fsPath.endsWith('.gitkeep') && isValidFileType(item)
  })
})

watch(() => props.open, (isOpen) => {
  if (isOpen) {
    selectedFolderFsPath.value = mediaTree.rootItem.value.fsPath
  }
})

watch(folderTreeItems, (items) => {
  if (!findFolderTreeItem(items, selectedFolderFsPath.value)) {
    selectedFolderFsPath.value = mediaTree.rootItem.value.fsPath
  }
}, { immediate: true })

const handleMediaSelect = (media: StudioTreeItem) => {
  emit('select', media)
}

const handleUpload = async () => {
  emit('cancel')
  await context.switchFeature(StudioFeature.Media)
}

const handleUseExternal = () => {
  // Emit select with null to trigger manual URL entry
  emit('select', null)
}
</script>

<template>
  <UModal
    :open="open"
    :title="$t(`studio.mediaPicker.${type}.title`)"
    :description="$t(`studio.mediaPicker.${type}.description`)"
    :ui="{ content: 'sm:max-w-5xl' }"
    @update:open="(value: boolean) => !value && emit('cancel')"
  >
    <template #body>
      <div
        v-if="!hasAnyMediaFiles"
        class="text-center py-4 text-muted"
      >
        <UIcon
          :name="type === 'image' ? 'i-lucide-image-off' : 'i-lucide-video-off'"
          class="size-8 mx-auto mb-2"
        />
        <p class="text-sm">
          {{ $t(`studio.mediaPicker.${type}.notAvailable`) }}
        </p>
      </div>

      <div
        v-else
        class="grid gap-4 lg:grid-cols-[240px_minmax(0,1fr)]"
      >
        <div class="rounded-lg border border-default bg-muted/20 p-3">
          <div class="mb-3 flex items-center gap-1">
            <UIcon
              name="i-lucide-folder-tree"
              class="size-4 text-muted"
            />
            <span class="text-xs uppercase tracking-wider text-muted">
              {{ $t('studio.headings.directories') }}
            </span>
          </div>

          <div class="max-h-[26rem] overflow-y-auto">
            <UTree
              v-model="selectedFolderTreeItem"
              :items="folderTreeItems"
              :get-key="getFolderTreeItemKey"
              color="neutral"
              size="sm"
              class="min-w-0"
            />
          </div>
        </div>

        <div class="min-w-0">
          <div class="mb-3 flex items-start justify-between gap-3">
            <div class="min-w-0">
              <p class="text-xs uppercase tracking-wider text-muted">
                {{ $t('studio.headings.media') }}
              </p>
              <p class="truncate text-sm text-highlighted">
                {{ selectedFolderPath }}
              </p>
            </div>

            <UBadge
              :label="mediaFiles.length.toString()"
              color="neutral"
              variant="soft"
              size="sm"
            />
          </div>

          <div
            v-if="mediaFiles.length === 0"
            class="flex min-h-72 flex-col items-center justify-center rounded-lg border border-dashed border-default px-6 text-center text-muted"
          >
            <UIcon
              :name="type === 'image' ? 'i-lucide-image-off' : 'i-lucide-video-off'"
              class="size-8"
            />
            <p class="mt-3 text-sm text-highlighted">
              {{ selectedFolderPath }}
            </p>
          </div>

          <div
            v-else
            class="grid grid-cols-2 gap-4 sm:grid-cols-3 xl:grid-cols-4"
          >
            <button
              v-for="media in mediaFiles"
              :key="media.fsPath"
              type="button"
              class="aspect-square rounded-lg cursor-pointer group relative"
              @click="handleMediaSelect(media)"
            >
              <!-- Image Preview -->
              <div
                v-if="type === 'image'"
                class="w-full h-full overflow-hidden rounded-lg border border-default hover:border-muted hover:ring-1 hover:ring-muted transition-all"
                style="background: repeating-linear-gradient(45deg, #d4d4d8 0 12px, #a1a1aa 0 24px), repeating-linear-gradient(-45deg, #a1a1aa 0 12px, #d4d4d8 0 24px); background-blend-mode: overlay; background-size: 24px 24px;"
              >
                <Image
                  :src="media.routePath || media.fsPath"
                  width="200"
                  height="200"
                  :alt="media.name"
                  class="w-full h-full object-cover rounded-lg group-hover:scale-105 transition-transform duration-300 ease-out"
                />
              </div>

              <!-- Video Preview -->
              <div
                v-else
                class="w-full h-full bg-linear-to-br from-neutral-900 via-neutral-800 to-neutral-900 flex flex-col items-center justify-center relative overflow-hidden rounded-lg"
              >
                <!-- Decorative film strip pattern -->
                <div class="absolute inset-y-0 left-0 w-3 bg-neutral-950 flex flex-col justify-around py-1">
                  <div
                    v-for="i in 6"
                    :key="i"
                    class="w-1.5 h-2 bg-neutral-700 mx-auto rounded-sm"
                  />
                </div>
                <div class="absolute inset-y-0 right-0 w-3 bg-neutral-950 flex flex-col justify-around py-1">
                  <div
                    v-for="i in 6"
                    :key="i"
                    class="w-1.5 h-2 bg-neutral-700 mx-auto rounded-sm"
                  />
                </div>

                <div class="size-14 rounded-full bg-white/10 backdrop-blur-sm flex items-center justify-center group-hover:bg-white/20 group-hover:scale-110 transition-all duration-300 shadow-lg">
                  <UIcon
                    name="i-lucide-video"
                    class="size-7 text-white ml-0.5"
                  />
                </div>

                <!-- Filename -->
                <p class="absolute bottom-0 inset-x-0 text-[10px] text-neutral-300 truncate px-4 py-2 bg-linear-to-t from-black/60 to-transparent text-center font-medium">
                  {{ media.name }}
                </p>
              </div>
            </button>
          </div>
        </div>
      </div>
    </template>

    <template #footer>
      <div class="flex gap-2">
        <UButton
          variant="solid"
          icon="i-lucide-upload"
          @click="handleUpload"
        >
          {{ $t(`studio.mediaPicker.${type}.upload`) }}
        </UButton>
        <UButton
          variant="outline"
          icon="i-lucide-link"
          @click="handleUseExternal"
        >
          {{ $t(`studio.mediaPicker.${type}.useExternal`) }}
        </UButton>
      </div>
    </template>
  </UModal>
</template>
