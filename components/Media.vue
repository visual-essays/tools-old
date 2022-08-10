<template>
  <div>
    <app-header></app-header>

    <b-breadcrumb :items="breadCrumbs"></b-breadcrumb>

    <b-button-group size="sm" class="dir">
      <b-button v-for="dir in dirs" :key="dir" v-html="dir.split('/').pop()" :to="dir" pill></b-button>
    </b-button-group>

    <ve-image-grid as-cards >
      <ul>
        <li v-for="manifest, idx in manifests" :key="`grid-${idx}`" v-html="manifest"></li>
      </ul>
    </ve-image-grid>

    <b-button v-if="isLoggedIn" pill class="fab" variant="primary" v-b-modal.add-image>+</b-button>
    <add-image-dialog></add-image-dialog>

  </div>
</template>

<script lang="ts">

import Vue from 'vue'

const api = 'https://api.juncture-digital.org'
const iiifServer = 'https://iiif.juncture-digital.org'

const fileExtensions = new Set(['yaml','jpg','jpeg','jp2','png','tif','tiff'])
const ignore = new Set(['iiif-props.yaml', 'iiif-props.template.yaml'])

export default Vue.extend({
  name: 'Media',
  data: () => ({
    baseRoute: '',
    root: '',
    baseUrl: '',
    dirs: <string[]>[],
    manifests: <any[]>[],
    viewerWindow: null
  }),
  computed: {
    acct(): string {return this.$store.state.acct},
    repo(): string {return this.$store.state.repo},
    path(): string {return this.$store.state.path},
    isLoggedIn() {return this.$store.state.authToken !== ''},
    breadCrumbs(): any[] {
      console.log(this.$route.path)
      let pathElems = this.$route.path.split('/').filter(pe => pe)
      let root = `/${pathElems.slice(0,3).join('/')}`
      let breadCrumbs = [{text: 'root', to: root}]
      for (let i = 3; i < pathElems.length; i++) {
        breadCrumbs.push({text: pathElems[i], to: `${root}/${pathElems.slice(i,i+1).join('/')}`})
      }
      return breadCrumbs
    }
  },
  async created() {},
  async mounted() {
    this.baseRoute = (this.$route.name || '').replace(/-all$/,'').split('/').filter(pe => pe).join('/')
    console.log(this.$route)
    let pathElems = (this.$route.params?.pathMatch || '').split('/').filter(pe => pe)
    this.$store.commit('setPath', pathElems.length > 2 ? pathElems.slice(2).join('/') : '')
    if (pathElems.length > 0) this.$store.commit('setAcct', pathElems[0])
    if (pathElems.length > 1) this.$store.commit('setRepo', pathElems[1])
    this.baseUrl = location.origin
    this.root = [this.acct, this.repo, ...this.path.split('/')].filter(pe => pe).join('/')
    // this.root = this.acct && this.repo ? `${this.acct}/${this.repo}` + (this.path ? `/${this.path}` : '') : ''
    console.log(`${this.$options.name}.mounted acct=${this.acct} repo=${this.repo} path=${this.path} baseUrl=${this.baseUrl} root=${this.root}`)
    if (this.acct && this.repo) {
      let path = [this.baseRoute, this.acct, this.repo, ...this.path.split('/')].filter(pe => pe).join('/')
      window.history.replaceState({}, '', `/${path}`)
    }
  },
  methods: {
    async listContents() {
      await fetch(`${api}/dir/${this.root}/`).then(resp => resp.json())
      .then(items => {
        let manifests = new Set()
        let dirs: string[] = []
        items.forEach((item:any) => {
          if (item.type === 'dir') {
            let path = [this.baseRoute, this.acct, this.repo, ...this.path.split('/')].filter(pe => pe).join('/')
            dirs.push(`/${path}/${item.name}`)
          } else {
            if (!ignore.has(item.name)) {
              let elems = item.name.split('.')
              let name = elems.slice(0,-1).join('.')
              let extension = elems.pop().toLowerCase()
              if (fileExtensions.has(extension)) {                
                manifests.add(`${iiifServer}/gh:${this.root}/${encodeURIComponent(name)}/manifest.json`)
              }
            }
          }
        })
        this.dirs = dirs
        this.manifests = Array.from(manifests).sort()
      })     
    }
  },
  watch: {
    root: {
      async handler(root) {
        if (root) this.listContents()
     },
      immediate: true
    },

    manifests(manifests) {
      console.log(`manifests=${manifests.length}`)
    }
  
  }
})

</script>

<style scoped>
  [v-cloak] {
    display: none
  }
  
  * {
    box-sizing: border-box;
  }
  
  body {
    margin: 12px;
    font-family: Roboto, sans-serif;
  }
  
  .name {
    margin-right: 12px;
    cursor: pointer;
  }    
  
  .name:hover {
    font-weight: bold;
  }
  
  .breadcrumb {
    margin-bottom: 0;
    padding: .5rem;
  }
  
  .dir.btn-group {
    width: 100%;
    background-color: #e9ecef;
  }
  .dir.btn-group .btn {
    color: black;
    background-color: white;
    margin: 0 0 .5rem .5rem;
    flex: none;
  }

  .fab {
    position: fixed;
    right: 10px;
    bottom: 10px;
    font-weight: bold;
    font-size: 1.2rem;
  }

</style>