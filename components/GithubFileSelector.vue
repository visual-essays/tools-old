<template>
  <b-modal :id="id">

    <template #modal-header="{ close }">
      <h5>File Selector</h5>
    </template>

    <template #default="{ hide }">
      
      <b-breadcrumb>
        <b-breadcrumb-item v-for="crumb, idx in breadCrumbs" :key="`crumb-${idx}`" v-html="crumb.text" @click="update(crumb.to)"></b-breadcrumb-item>
      </b-breadcrumb>

      <div class="dirList">

        <ul v-if="dirs.length > 0" size="sm" class="dirs">
          <li v-for="dir in dirs" :key="dir" class="dir" @click="update(`${curDir}/${dir}`)">
            <fa :icon="faFolder" title="Folder"></fa>{{dir.split('/').pop()}}
          </li>
        </ul>

        <ul class="files" v-if="!selectedFile && files.length > 0">
          <li class="file" v-for="file in files" :key="file.sha">
            <span v-html="file.name" @click="fileSelected(file)"></span>
            <span v-b-tooltip.hover title="Delete file" class="trashcan" @click="deleteFile(file)">
              <fa class="trash-icon" :icon="faTrashCan"></fa>
            </span>
          </li>
        </ul>

      </div>

      <b-modal :id="`${id}-new-file-path`" title="Create File" hide-backdrop>
        <b-form-input v-model="newFilePath" placeholder="File name or path"></b-form-input>
      </b-modal>

    </template>

    <template #modal-footer="{ cancel }">
      <div class="w-100">
        <b-button class="float-left" size="sm" variant="primary" @click="openDialog">New File</b-button>
        <b-button class="float-right" size="sm" variant="primary" @click="select">Select</b-button>
        <b-button class="float-right" size="sm" style="margin-right:12px;" @click="cancel()">Cancel</b-button>
      </div>
    </template>

  </b-modal>
</template>

<script lang="ts">

import Vue from 'vue'
import { faFolder, faTrashCan }  from '@fortawesome/free-regular-svg-icons'

export default Vue.extend({
  name: 'GithubFileSelector',
  props: {
    id: { type: String, default: 'github-file-selector' }
  },
  data: () => ({
    dirList: <any[]>[],
    curDir: <string>'',
    selectedFile: <string>'',
    newFilePath: '',
  }),
  computed: {
    acct(): string {return this.$store.state.acct},
    repo(): string {return this.$store.state.repo},
    path(): string {return this.$store.state.fileSelectorPath},
    githubClient() {return this.$store.state.githubClient},
    dirs() {return this.dirList.filter(item => item.type === 'dir').map(item => item.name)},
    files() {return this.dirList.filter(item => item.type === 'file')},
    breadCrumbs(): any[] {
      let breadCrumbs = [{text: 'root', to: ``}]
      let pathElems = [...this.curDir.split('/'), this.selectedFile].filter(pe => pe)
      for (let i = 0; i < pathElems.length; i++) {
        breadCrumbs.push({text: pathElems[i], to: `${pathElems.slice(0,i+1).join('/')}`})
      }
      return breadCrumbs
    },
    faFolder() { return faFolder },
    faTrashCan() { return faTrashCan }
  },
  async mounted() {
    this.$root.$on('bv::modal::show', this.onOpen)
    this.$root.$on('bv::modal::hide', this.onClose)
  },
  methods: {

    select() {
      (this as any).$bvModal.hide(this.id)
      let path = this.curDir + (this.selectedFile ? `/${this.selectedFile}`: '')
      this.$root.$emit('github-path-changed', path)
    },

    onOpen(evt:any, modalId:string) {
      if (modalId === this.id) {
        this.curDir = this.path
        this.update(this.path)
      }
    },

    onClose(evt:any, modalId:string) {
      if (modalId === this.id) {
        this.dirList = []
      } else if (modalId === `${this.id}-new-file-path`) {
        this.createFile()
      }
    },

    fileSelected(file:any) {
      this.selectedFile = file.name
    },

    openDialog() {
      (this as any).$bvModal.show(`${this.id}-new-file-path`)
    },

    async update(path:string) {
      path = path.split('/').filter(pe => pe).join('/')
      let pathElems = path.split('/').filter(pe => pe)
      let dirList: any[] = await this.githubClient.dirlist(this.acct, this.repo, pathElems.join('/'))
      if (dirList.length === 0) {
        dirList = await this.githubClient.dirlist(this.acct, this.repo, pathElems.slice(0,-1).join('/'))
        this.dirList = dirList
        this.curDir = pathElems.slice(0,-1).filter(pe => pe).join('/')
        if (dirList.find(item => item.type === 'file' && item.name === `${pathElems[pathElems.length-1]}.md`)) {
          this.selectedFile = `${pathElems[pathElems.length-1]}.md`
        } else if (dirList.find(item => item.type === 'file' && item.name === 'README.md')) {
          this.selectedFile = 'README.md'
        }
      } else {
        this.dirList = dirList
        this.curDir = path
        this.selectedFile = ''
      }
      this.dirList = dirList

      // this.githubClient.dirlist(this.acct, this.repo, this.curDir).then((dirList:any[]) => this.dirList = dirList)
    },

    async createFile() {
      if (this.newFilePath) {
        let path: string = [...this.curDir.split('/').filter(pe => pe), ...this.newFilePath.split('/')].join('/')
        let fname = (path.split('/').pop() as string)
        let content = fname.indexOf('.md') === fname.length-3 ? `# ${fname.slice(0,-3)}` : ''
        await this.githubClient.putFile(this.acct, this.repo, path, content)

        this.githubClient.dirlist(this.acct, this.repo, this.curDir).then((dirList:any[]) => this.dirList = dirList)
      }
    },


    async deleteFile(toDelete:any) {
      let path = `${this.curDir}/${toDelete.name}`
      await this.githubClient.deleteFile(this.acct, this.repo, path, toDelete.sha)
      this.dirList = this.dirList.filter(file => file.name !== toDelete.name)
      if (this.files.length === 0) {
        this.curDir = this.curDir.split('/').filter(pe => pe).slice(0,-1).join('/')
        this.githubClient.dirlist(this.acct, this.repo, this.curDir).then((dirList:any[]) => this.dirList = dirList)
      }
    }

  },

  watch: {}
})

</script>

<style scoped>

  .breadcrumb-item {
    cursor: pointer;
  }

  .dirList {
    height: 40vh;
    overflow-y: scroll;
  }

  .dirs, .files {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .dir svg {
    margin-right: 8px;
    width: 20px;
  }

  [data-icon="trash-can"] {
    color: red;
    cursor: pointer;
  }

  .dir, .file {
    border: 1px solid transparent;
    border-radius: 3px;
    cursor: pointer;
  }

  .file {
    display: flex;
    align-items: center;
    gap: 24px;
    padding: 0 12px 0 28px;
  }

  .file .trashcan {
    margin-left: auto;
  }

  .dir:hover, .file:hover {
    background-color: #ddd;
    border: 1px solid #999;
    border-radius: 3px;
  }

</style>