<template>
  <div class="update">
    <el-dialog
      custom-class="update-dialog"
      :title="$t('update.updateTitle')"
      :visible.sync="showDialog"
      :width="dialogWidth"
      :close-on-click-modal="false"
      append-to-body
      center
    >
      <div class="scrollable-content" :style="{ height: dialogHeight }">
        <div ref="detail_display" class="text-content" v-html="detail"></div>
      </div>
      <div slot="footer" class="dialog-footer">
        <el-button size="mini" class="update-button--normal left-button" @click="ignoreUpdate">{{
          $t('update.ignoreVersion')
        }}</el-button>
        <div class="right-buttons">
          <el-button size="mini" class="update-button--normal" @click="nextUpdate">{{
            $t('update.nextRemind')
          }}</el-button>
          <el-button size="mini" type="primary" @click="toUpdate">{{ $t('update.update') }}</el-button>
        </div>
      </div>
    </el-dialog>
    <el-dialog
      custom-class="progress-dialog"
      :visible.sync="progressVisible"
      :title="$t('update.downloadProgress')"
      :close-on-click-modal="false"
      :show-close="false"
      center
    >
      <div class="progress-text">{{ !downloaded ? $t('update.downloading') : $t('update.downloaded') }}</div>
      <el-progress :percentage="progress" show-text :status="downloaded ? 'success' : null"></el-progress>
      <el-button size="mini" v-if="!downloaded" slot="footer" type="danger" @click="cancelDownload">{{
        $t('update.cancel')
      }}</el-button>
      <el-button size="mini" v-else slot="footer" type="primary" @click="toInstall">{{
        $t('update.install')
      }}</el-button>
    </el-dialog>
  </div>
</template>

<script lang="ts">
import { Component, Ref, Vue } from 'vue-property-decorator'
import { Getter } from 'vuex-class'
import { updateChecker } from '@/main/updateChecker'
import { ipcRenderer } from 'electron'

const Store = require('electron-store')
const electronStore = new Store()

@Component
export default class Update extends Vue {
  @Getter('currentTheme') private theme!: Theme
  @Getter('currentLang') private getterLang!: Language
  @Getter('autoCheck') private autoCheck!: boolean
  @Ref('detail_display') private detail_display!: HTMLDivElement
  private showDialog: boolean = false
  private progressVisible: boolean = false
  private version: string = ''
  private detail: string = ''
  private progress: number = 0
  private downloaded: boolean = false
  private dialogWidth: string = '960px'
  private dialogHeight: string = '450px'

  private goToLink(url: string) {
    const windowUrl = window.open(url)
    if (windowUrl) {
      windowUrl.opener = null
    }
  }

  private handleATags() {
    this.$nextTick(() => {
      const aTags = this.detail_display.getElementsByTagName('a')
      for (let a of aTags) {
        a.onclick = (e) => {
          e.preventDefault()
          this.goToLink(a.href)
          return false
        }
      }
    })
  }

  private setDialogSize() {
    const width = document.body.clientWidth
    const height = document.body.clientHeight
    const def_width = 900
    if (width < def_width) {
      this.dialogWidth = '100%'
    } else {
      this.dialogWidth = def_width + 'px'
    }

    this.dialogHeight = String(Math.floor(height * 0.5)) + 'px'
  }

  private ignoreUpdate() {
    electronStore.set('isIgnore', this.version)
    this.showDialog = false
  }

  private nextUpdate() {
    this.showDialog = false
  }

  private toUpdate() {
    this.showDialog = false
    ipcRenderer.send('startDownloadProgress', { version: this.version, detail: this.detail })
    ipcRenderer.on('downloadProgressPercent', (_, percent) => {
      let num = Math.trunc(percent)
      if (num > this.progress) {
        this.progress = num
      }
      if (num === 100) {
        this.downloaded = true
      }
    })

    this.progressVisible = true
  }

  private cancelDownload() {
    this.progressVisible = false
    ipcRenderer.send('cancelDownload')
  }

  private toInstall() {
    this.progressVisible = false
    ipcRenderer.send('toQuitAndInstall')
  }

  private async updateCheck(auto: boolean) {
    let res = await updateChecker(auto)
    if (res && typeof res !== 'boolean') {
      this.detail = res.detail
      this.version = res.version
      this.showDialog = true
      this.handleATags()
    } else if (!auto) {
      ipcRenderer.send('showMsg')
    }
  }

  private async created() {
    ipcRenderer.on('clickUpdate', (event) => {
      this.updateCheck(false)
    })
    if (this.autoCheck) {
      await this.updateCheck(true)
    }
    window.addEventListener('resize', () => {
      this.setDialogSize()
    })
  }

  private beforeDestroy() {
    window.removeEventListener('resize', () => {
      this.setDialogSize()
    })
  }
}
</script>

<style lang="scss">
@import '~@/assets/scss/variable.scss';

.update-dialog,
.progress-dialog {
  .el-dialog__header {
    padding: 0 20px;
    line-height: 56px;
    border-bottom: 1px solid var(--color-border-default);
    .el-dialog__title {
      color: var(--color-text-title);
      font-size: $font-size--subtitle;
    }
  }
  .el-dialog--center .el-dialog__body {
    padding: 32px 24px 0;
  }
  .el-dialog__footer {
    padding: 10px 24px;
    .el-button--text {
      font-size: 14px;
      color: var(--color-text-default);
      &:hover {
        color: var(--color-main-green);
      }
    }
  }
  .update-button--normal {
    background: transparent;
    border: 1px solid var(--color-border-default);
    color: var(--color-text-default);
    &:hover {
      color: var(--color-main-green);
      border-color: var(--color-main-green);
    }
  }
}

.update-dialog {
  overflow-y: auto;
  margin-top: 8vh !important;
  .el-dialog__footer {
    text-align: right;
  }
  .scrollable-content {
    overflow-y: auto;
    overflow-x: hidden;
  }
  .text-content {
    color: var(--color-text-default);
    padding-bottom: 0.3rem;
    .icon.icon-link {
      display: none;
    }
    h1 {
      font-size: 1.75rem;
      &:first-child {
        margin-top: 0;
        padding-top: 0;
      }
    }
    h2 {
      font-size: 1.5rem;
      padding-bottom: 0.8rem;
      border-bottom: 1px solid var(--color-border-default);
    }
    h3 {
      font-size: 1rem;
    }
    h1,
    h2,
    h3,
    h4 {
      color: var(--color-text-title);
      line-height: 1.25;
      font-weight: 600;
      margin-top: -6.1rem;
      padding-top: 7.6rem;
      margin-bottom: 0;
      a {
        color: var(--color-main-green);
      }
    }
    h4 {
      margin-top: 0;
      padding-top: 0;
    }
    ol,
    p,
    ul {
      line-height: 1.7;
      margin: 1rem 0;
      li {
        margin: 1rem 0;
      }
    }
    code {
      font-family: source-code-pro, Menlo, Monaco, Consolas, Courier New, monospace;
      color: var(--color-main-green);
      padding: 0.25rem 0.5rem;
      margin: 0;
      font-size: 12px;
      background-color: var(--color-bg-code);
      border-radius: 3px;
    }
    pre,
    pre[class*='language-'] {
      line-height: 1.4;
      padding: 1.25rem 1.5rem;
      margin: 0.85rem 0;
      background-color: #282c34;
      border-radius: 6px;
      overflow: auto;
      text-shadow: none;
      code {
        color: #fff;
        padding: 0;
        background-color: transparent;
        border-radius: 0;
      }
      .number {
        display: unset;
        background-color: initial;
        border-radius: initial;
        font-size: 14px;
        height: auto;
        margin-right: initial;
        min-width: initial;
        padding: initial;
        color: #cc99cd;
        text-align: unset;
        vertical-align: unset;
      }
    }
    a {
      color: var(--color-main-green);
      font-weight: 500;
      word-break: break-word;
    }
    ul {
      list-style: disc;
      padding-left: 1.2em;
    }
    ol {
      list-style: decimal;
      padding-left: 1.2em;
    }
    table {
      border-collapse: collapse;
      margin: 1rem 0;
      display: block;
      overflow-x: auto;
    }
    tr {
      border-top: 1px solid var(--color-border-default);
    }
    td,
    th {
      border: 1px solid var(--color-border-default);
      padding: 0.6em 1em;
    }
    tr:nth-child(2n) {
      background-color: #f6f8fa;
    }
  }
  article {
    margin: 0 40px;
  }
  blockquote {
    background-color: var(--color-bg-code);
    border-color: var(--color-main-green);
    padding: 0.1rem 1.5rem;
    border-left-width: 0.5rem;
    border-left-style: solid;
    margin: 1rem 0;
  }
  strong {
    color: var(--color-text-title);
  }
  .language-css .token.string,
  .style .token.string,
  .token.entity,
  .token.operator,
  .token.url {
    background-color: transparent;
  }
  .dialog-footer {
    display: flex;
    justify-content: space-between;

    .left-button {
      order: -1;
    }
    .right-buttons {
      display: flex;
      gap: 8px;
    }
  }
}
.progress-dialog {
  .progress-text {
    color: var(--color-text-title);
  }
  .el-dialog__footer {
    text-align: center;
  }
}
</style>
