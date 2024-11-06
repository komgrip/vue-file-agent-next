<template>
  <div
    :class="[
      'file-preview-wrapper-' + fileRecord.ext(),
      fileRecord.isImage() ? 'file-preview-wrapper-image' : 'file-preview-wrapper-other',
      'file-category-' + fileRecord.icon().category,
      {'file-is-playing-av': fileRecord.isPlayingAv},
      {'is-deletable': deletable === true},
      {'is-editable': editable === true},
      {'is-edit-input-focused': isEditInputFocused},
      {'has-error': fileRecord.error},
    ]"
  >
    <div
      v-if="fileRecord.isPlayableAudio() || fileRecord.isPlayableVideo()"
      ref="wrapper"
      class="file-av-wrapper"
    >
      <div class="file-av-action" @click="playAv(fileRecord)">
        <span class="file-av-stop">
          <VueFileIcon name="system-close" />
        </span>
        <span class="file-av-play">
          <VueFileIcon name="system-file-av-play" />
        </span>
      </div>
    </div>
    <span
      class="file-preview"
      :class="{
        'image-preview': fileRecord.isImage(),
        'other-preview': !fileRecord.isImage(),
        'dark-content': fileRecord.isImage() && fileRecord.isDarkColor(),
      }"
      :style="{
        'background-color': fileRecord.color(),
      }"
    >
      <span v-if="fileRecord.error" class="file-error-wrapper" @click="dismissError()">
        <span class="file-error-message file-error-message-client">
          {{ fileRecord.getErrorMessage(errorText) }}
        </span>
      </span>
      <span class="file-preview-overlay" />
      <span
        v-if="fileRecord.isImage() || fileRecord.isPlayableVideo()"
        class="thumbnail"
        style="position: absolute; top: 0; right: 0; bottom: 0; left: 0; overflow: hidden"
      >
        <!-- <a
          v-if="hasLinkableUrl"
          :href="fileRecord.url()"
          target="_blank"
          :title="fileRecord.name()"
        > 
          <img class="file-preview-img" :src="fileRecord.src()" style="cursor: zoom-in" />
        </a>
        -->
        <a
          v-if="hasLinkableUrl"
          @click="showViewer(fileRecord.url() || '')"
          :title="fileRecord.name()"
        >
          <img class="file-preview-img" :src="fileRecord.src()" style="cursor: zoom-in" />
        </a>

        <img v-if="fileRecord?.videoThumbnail" class="file-preview-img" :src="fileRecord.src()" />
        <video
          v-else-if="fileRecord?.raw?.type?.includes('video')"
          style="width: 100%; height: 100%"
        >
          <source :src="fileRecord.urlValue || ''" :type="fileRecord.raw.type" />
        </video>
        <img
          v-else
          class="file-preview-img"
          :src="fileRecord.src()"
          @click="showViewer(fileRecord.url() || '')"
          :style="fileRecord.url() || '' ? `cursor: pointer;` : ''"
        />
      </span>
      <span class="file-ext">{{ fileRecord.ext() }}</span>
      <span class="file-size">{{ fileRecord.size() }}</span>
      <span
        v-if="deletable"
        class="file-delete"
        @click="removeFileRecord(fileRecord)"
        @touchstart="filenameClearPressed()"
        @mousedown="filenameClearPressed()"
      >
        <VueFileIcon name="system-close" />
      </span>
      <span class="file-name" @click="editFileName()">
        <input
          v-if="editable === true"
          ref="input"
          class="file-name-input"
          :disabled="disabled === true"
          type="text"
          :value="fileRecord.name(true)"
          @focus="editInputFocused()"
          @blur="editInputBlured()"
          @change="filenameChanged()"
          @input="filenameChanged()"
          @keyup.enter="filenameChanged(true)"
          @keyup.esc="filenameChanged(false)"
        />
        <span v-if="editable === true" class="file-name-edit-icon">
          <VueFileIcon name="system-file-name-edit" />
        </span>
        <span class="file-name-text">{{ fileRecord.name(true) }}</span>
      </span>
      <!-- <span
        v-if="fileRecord.dimensions.width && fileRecord.dimensions.height"
        class="image-dimension"
      >
        <span class="image-dimension-width">{{ fileRecord.dimensions.width }}</span
        ><span class="image-dimension-height">{{ fileRecord.dimensions.height }}</span>
      </span> -->
      <span
        v-if="fileRecord.hasProgress()"
        class="file-progress"
        :class="{
          'file-progress-full': fileRecord.progress() >= 99.9999,
          'file-progress-done': fileRecord.progress() >= 100,
          'has-file-progress': fileRecord.progress() > 0,
        }"
      >
        <span class="file-progress-bar" :style="{width: fileRecord.progress() + '%'}" />
      </span>
      <span class="file-icon">
        <a
          v-if="hasLinkableUrl"
          :href="fileRecord.url()"
          target="_blank"
          :title="fileRecord.name()"
        >
          <VueFileIcon :ext="fileRecord.ext()" />
        </a>
        <VueFileIcon v-else :ext="fileRecord.ext()" />
      </span>
    </span>
  </div>

  <div class="vue-file-image-viewer" :class="{'d-none': !isShowImageViewer}" @click="hideViewer">
    <span class="close" @click="hideViewer">&times;</span>
    <img
      class="vue-file-modal-content"
      :src="imageViewerSrc"
      :class="{'d-none': !isShowImageViewer}"
      @click="stopPropagation"
    />
  </div>
</template>

<script lang="ts">
import {defineComponent, PropType} from 'vue'
import VueFileIcon from './VueFileIcon.vue'
import FileRecord, {ErrorText} from '../lib/file-record'
import utils from '../lib/utils'

export default /* #__PURE__ */ defineComponent({
  name: 'VueFilePreview',
  components: {
    VueFileIcon,
  },
  props: {
    averageColor: Boolean,
    deletable: Boolean,
    disabled: Boolean,
    editable: Boolean,
    errorText: {
      type: Object as PropType<ErrorText>,
      required: false,
    },
    linkable: Boolean,
    thumbnailSize: Number,
    fileRecord: {
      type: Object as PropType<FileRecord>,
      required: true,
    },
    withCredentials: Boolean,
  },
  emits: ['dismisserror', 'remove', 'rename'],
  data() {
    return {
      isEditInputFocused: false,
      isEditCancelable: true,
      isShowImageViewer: false,
      imageViewerSrc: '',
    }
  },
  computed: {
    hasLinkableUrl(): boolean {
      if (!this.linkable) {
        return false
      }
      return (
        !!this.fileRecord.url() &&
        !this.fileRecord.isPlayableVideo() &&
        !this.fileRecord.isPlayableAudio()
      )
    },
  },
  watch: {
    fileRecord: {
      handler() {
        // fileRecord is updated, but the component not redrawn when used inside a loop
        // -> force update in the meantime
        this.$forceUpdate()
      },
      deep: true,
    },
  },
  methods: {
    createThumbnail(fileRecord: FileRecord, video: HTMLVideoElement) {
      if (fileRecord.videoThumbnail) {
        video.poster = fileRecord.src()
        return
      }
      const canvas = document.createElement('canvas')
      utils
        .createVideoThumbnail(
          video,
          canvas,
          this.fileRecord.thumbnailSize,
          this.averageColor !== false,
          this.withCredentials === true
        )
        .then((thumbnail) => {
          fileRecord.imageColor = thumbnail.color
          fileRecord.videoThumbnail = thumbnail.url
          fileRecord.dimensions.width = thumbnail.width
          fileRecord.dimensions.height = thumbnail.height
          video.poster = fileRecord.src()
        })
    },
    playAv(fileRecord: FileRecord) {
      if (fileRecord.stopAv) {
        fileRecord.stopAv()
        return
      }
      const {createObjectURL} = window.URL || window.webkitURL || {}
      const {revokeObjectURL} = window.URL || window.webkitURL || {}

      const wrapper = this.$refs.wrapper as HTMLElement
      const player = document.createElement(fileRecord.isAudio() ? 'audio' : 'video')
      if (player instanceof HTMLVideoElement && fileRecord.isPlayableVideo()) {
        this.createThumbnail(fileRecord, player)
      }
      player.controls = true
      // player.style.width = this.prvWidth + 'px';
      wrapper.appendChild(player)
      const url = (fileRecord.url() as string) || createObjectURL(fileRecord.file)
      player.src = url
      player.play()
      fileRecord.isPlayingAv = true
      fileRecord.stopAv = () => {
        // player.src = null;
        player.src = ''
        wrapper.removeChild(player)
        revokeObjectURL(url)
        fileRecord.isPlayingAv = false
        fileRecord.stopAv = null
      }
    },
    removeFileRecord(fileRecord: FileRecord) {
      if (this.clearFilename()) {
        return
      }
      if (this.disabled === true) {
        return
      }
      this.$emit('remove', fileRecord)
    },
    editFileName() {
      if (this.editable !== true) {
        return
      }
      if (!this.$refs.input) {
        return
      }
      ;(this.$refs.input as HTMLInputElement).focus()
    },
    editInputFocused() {
      this.isEditInputFocused = true
      this.isEditCancelable = true
    },
    editInputBlured() {
      const {fileRecord} = this
      fileRecord.oldFileName = this.fileRecord.name()
      const oldFilenameWithoutExt = this.fileRecord.name(true)
      const {value} = this.$refs.input as HTMLInputElement
      fileRecord.customName = value
      const newValueWithoutExt = this.fileRecord.name(true)
      if (newValueWithoutExt !== oldFilenameWithoutExt) {
        fileRecord.oldCustomName = oldFilenameWithoutExt
        this.$emit('rename', fileRecord)
      }
      const timeout = 100
      window.setTimeout(() => {
        this.$nextTick(() => {
          if (!this.isEditCancelable) {
            return
          }
          this.isEditInputFocused = false
        })
      }, timeout)
    },
    filenameChanged(completed?: boolean) {
      if (completed) {
        ;(this.$refs.input as HTMLInputElement).blur() // @see editInputBlured method
      }
      if (completed === false) {
        this.clearFilename()
      }
    },
    filenameClearPressed() {
      if (!(this.editable === true && this.isEditInputFocused)) {
        return
      }
      this.isEditCancelable = false
    },
    clearFilename() {
      if (!(this.editable === true && this.isEditInputFocused)) {
        return false
      }
      ;(this.$refs.input as HTMLInputElement).value = ''
      this.isEditCancelable = true
      this.editInputBlured()
      return true
    },
    dismissError() {
      if (this.fileRecord.error && (this.fileRecord.error.size || this.fileRecord.error.type)) {
        return
      }

      this.$emit('dismisserror', this.fileRecord)
    },
    showViewer(src: string): void {
      if (src === '') {
        return
      }

      this.isShowImageViewer = true
      this.imageViewerSrc = src

      document.body.classList.add('vue-file-image-viewer-on')
    },
    hideViewer(): void {
      this.isShowImageViewer = false
      document.body.classList.remove('vue-file-image-viewer-on')
    },
    stopPropagation(event: Event): void {
      event.stopPropagation()
    },
  },
})
</script>

<style scoped lang="scss">
.d-none {
  display: none;
}

.vue-file-image-viewer {
  display: block;
  position: fixed;
  z-index: 999;
  padding-top: 100px;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: auto;
  background-color: rgba(0, 0, 0, 0.9);
}
.vue-file-modal-content {
  margin: auto;
  display: block;
  width: 50%;
  max-width: 700px;
  border-radius: 5px;
  transition: 0.3s;
  &:hover {
    opacity: 1 !important;
  }
}
.vue-file-modal-content {
  animation-name: zoom;
  animation-duration: 0.6s;
}
@keyframes zoom {
  from {
    transform: scale(0);
  }
  to {
    transform: scale(1);
  }
}
.vue-file-image-viewer .close {
  position: absolute;
  top: 10px;
  right: 45px;
  color: #f1f1f1;
  font-size: 45px;
  font-weight: 300;
  transition: 0.3s;
}
.vue-file-image-viewer .close:hover,
.vue-file-image-viewer .close:focus {
  color: #bbb;
  text-decoration: none;
  cursor: pointer;
}

@media only screen and (max-width: 700px) {
  .vue-file-modal-content {
    width: 100%;
  }
}
</style>
