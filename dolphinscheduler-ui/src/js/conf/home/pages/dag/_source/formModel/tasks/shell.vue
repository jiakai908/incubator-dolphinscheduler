/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
<template>
  <div class="shell-model">
    <m-list-box>
      <div slot="text">{{$t('Script')}}</div>
      <div slot="content">
        <div class="from-mirror">
          <textarea
            id="code-shell-mirror"
            name="code-shell-mirror"
            style="opacity: 0">
          </textarea>
          <a class="ans-modal-box-max">
            <em class="ans-icon-max" @click="setEditorVal"></em>
          </a>
        </div>
      </div>
    </m-list-box>
    <m-list-box>
      <div slot="text">{{$t('Resources')}}</div>
      <div slot="content">
        <treeselect v-model="resourceList" :multiple="true" :options="options" :normalizer="normalizer" :placeholder="$t('Please select resources')">
          <div slot="value-label" slot-scope="{ node }">{{ node.raw.fullName }}</div>
        </treeselect>
      </div>
    </m-list-box>
    <!-- <m-list-box>
      <div slot="text">{{$t('Resources')}}</div>
      <div slot="content">
        <m-resources
                ref="refResources"
                @on-resourcesData="_onResourcesData"
                @on-cache-resourcesData="_onCacheResourcesData"
                :resource-list="resourceList">
        </m-resources>
      </div>
    </m-list-box> -->
    <m-list-box>
      <div slot="text">{{$t('Custom Parameters')}}</div>
      <div slot="content">
        <m-local-params
                ref="refLocalParams"
                @on-local-params="_onLocalParams"
                :udp-list="localParams"
                :hide="false">
        </m-local-params>
      </div>
    </m-list-box>
  </div>
</template>
<script>
  import _ from 'lodash'
  import i18n from '@/module/i18n'
  import mListBox from './_source/listBox'
  import mScriptBox from './_source/scriptBox'
  import mResources from './_source/resources'
  import mLocalParams from './_source/localParams'
  import disabledState from '@/module/mixin/disabledState'
  import Treeselect from '@riophae/vue-treeselect'
  import '@riophae/vue-treeselect/dist/vue-treeselect.css'
  import codemirror from '@/conf/home/pages/resource/pages/file/pages/_source/codemirror'

  let editor

  export default {
    name: 'shell',
    data () {
      return {
        // script
        rawScript: '',
        // Custom parameter
        localParams: [],
        // resource(list)
        resourceList: [],
        // Cache ResourceList
        cacheResourceList: [],
        // define options
        options: [],
        normalizer(node) {
          return {
            label: node.name
          }
        }
      }
    },
    mixins: [disabledState],
    props: {
      backfillItem: Object
    },
    methods: {
      /**
       * return localParams
       */
      _onLocalParams (a) {
        this.localParams = a
      },
      setEditorVal() {
        let self = this
          let modal = self.$modal.dialog({
            className: 'scriptModal',
            closable: false,
            showMask: true,
            maskClosable: true,
            onClose: function() {

            },
            render (h) {
              return h(mScriptBox, {
                on: {
                  getSriptBoxValue (val) {
                    editor.setValue(val)
                  },
                  closeAble () {
                    // this.$modal.destroy()
                    modal.remove()
                  }
                },
                props: {
                  item: editor.getValue()
                }
              })
            }
          })
      },
      /**
       * return resourceList
       *
       */
      _onResourcesData (a) {
        this.resourceList = a
      },
      /**
       * cache resourceList
       */
      _onCacheResourcesData (a) {
        this.cacheResourceList = a
      },
      /**
       * verification
       */
      _verification () {
        // rawScript verification
        if (!editor.getValue()) {
          this.$message.warning(`${i18n.$t('Please enter script(required)')}`)
          return false
        }

        // localParams Subcomponent verification
        if (!this.$refs.refLocalParams._verifProp()) {
          return false
        }
        // Process resourcelist
        let dataProcessing= _.map(this.resourceList, v => {
          return {
            id: v
          }
        })
        // storage
        this.$emit('on-params', {
          resourceList: dataProcessing,
          localParams: this.localParams,
          rawScript: editor.getValue()
        })
        return true
      },
      /**
       * Processing code highlighting
       */
      _handlerEditor () {
        // editor
        editor = codemirror('code-shell-mirror', {
          mode: 'shell',
          readOnly: this.isDetails
        })

        this.keypress = () => {
          if (!editor.getOption('readOnly')) {
            editor.showHint({
              completeSingle: false
            })
          }
        }

        // Monitor keyboard
        editor.on('keypress', this.keypress)
        editor.setValue(this.rawScript)

        return editor
      },
      diGuiTree(item) {  // Recursive convenience tree structure
        item.forEach(item => {
          item.children === '' || item.children === undefined || item.children === null || item.children.length === 0?　　　　　　　　
            delete item.children : this.diGuiTree(item.children);
        })
      }
    },
    watch: {
      //Watch the cacheParams
      cacheParams (val) {
        this.$emit('on-cache-params', val);
      }
    },
    computed: {
      cacheParams () {
        return {
          resourceList: _.map(this.resourceList, v => {
            return {id: v}
          }),
          localParams: this.localParams,
          rawScript: editor ? editor.getValue() : ''
        }
      }
    },
    created () {
      let item = this.store.state.dag.resourcesListS
      this.diGuiTree(item)
      this.options = item
      let o = this.backfillItem
      // Non-null objects represent backfill
      if (!_.isEmpty(o)) {
        this.rawScript = o.params.rawScript || ''

        // backfill resourceList
        let resourceList = o.params.resourceList || []
        if (resourceList.length) {
           _.map(resourceList, v => {
            if(v.res) {
              this.store.dispatch('dag/getResourceId',{
                type: 'FILE',
                fullName: '/'+v.res
              }).then(res => {
                this.resourceList.push(res.id)
              }).catch(e => {
                this.$message.error(e.msg || '')
              })
            } else {
              this.resourceList.push(v.id)
            }
          })
          this.cacheResourceList = resourceList
        }
        // backfill localParams
        let localParams = o.params.localParams || []
        if (localParams.length) {
          this.localParams = localParams
        }
      }
    },
    mounted () {
      setTimeout(() => {
        this._handlerEditor()
      }, 200)
    },
    destroyed () {
      if (editor) {
        editor.toTextArea() // Uninstall
        editor.off($('.code-shell-mirror'), 'keypress', this.keypress)
      }
    },
    components: { mLocalParams, mListBox, mResources, mScriptBox, Treeselect }
  }
</script>
<style lang="scss" rel="stylesheet/scss" scope>
  .scriptModal {
    .ans-modal-box-content-wrapper {
      width: 90%;
      .ans-modal-box-close {
        right: -12px;
        top: -16px;
        color: #fff;
      }
    }
  }
  .ans-modal-box-close {
    z-index: 100;
  }
  .ans-modal-box-max {
    position: absolute;
    right: -12px;
    top: -16px;
  }

</style>
