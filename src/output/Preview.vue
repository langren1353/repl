<script setup lang="ts">
import Message from '../Message.vue'
import {
  ref,
  onMounted,
  onUnmounted,
  watch,
  WatchStopHandle,
  inject,
  Ref
} from 'vue'
import srcdoc from './srcdoc.html?raw'
import { PreviewProxy } from './PreviewProxy'
import { compileModulesForPreview } from './moduleCompiler'
import { Store } from '../store'

const props = defineProps<{ show: boolean; ssr: boolean }>()

const store = inject('store') as Store
const clearConsole = inject('clear-console') as Ref<boolean>
const container = ref()
const runtimeError = ref()
const runtimeWarning = ref()

let sandbox: HTMLIFrameElement
let proxy: PreviewProxy
let stopUpdateWatcher: WatchStopHandle | undefined

// create sandbox on mount
onMounted(createSandbox)

// reset sandbox when import map changes
watch(
  () => store.state.files['import-map.json'].code,
  (raw) => {
    try {
      const map = JSON.parse(raw)
      if (!map.imports) {
        store.state.errors = [`import-map.json is missing "imports" field.`]
        return
      }
      createSandbox()
    } catch (e: any) {
      store.state.errors = [e as Error]
      return
    }
  }
)

// reset sandbox when version changes
watch(() => store.state.vueRuntimeURL, createSandbox)

onUnmounted(() => {
  proxy.destroy()
  stopUpdateWatcher && stopUpdateWatcher()
})

function createSandbox() {
  if (sandbox) {
    // clear prev sandbox
    proxy.destroy()
    stopUpdateWatcher && stopUpdateWatcher()
    container.value.removeChild(sandbox)
  }

  sandbox = document.createElement('iframe')
  sandbox.setAttribute(
    'sandbox',
    [
      'allow-forms',
      'allow-modals',
      'allow-pointer-lock',
      'allow-popups',
      'allow-same-origin',
      'allow-scripts',
      'allow-top-navigation-by-user-activation'
    ].join(' ')
  )

  const importMap = store.getImportMap()
  if (!importMap.imports) {
    importMap.imports = {}
  }
  if (!importMap.imports.vue) {
    importMap.imports.vue = store.state.vueRuntimeURL
  }
  const sandboxSrc = srcdoc.replace(
    /<!--IMPORT_MAP-->/,
    JSON.stringify(importMap)
  )
  sandbox.srcdoc = sandboxSrc
  container.value.appendChild(sandbox)

  proxy = new PreviewProxy(sandbox, {
    on_fetch_progress: (progress: any) => {
      // pending_imports = progress;
    },
    on_error: (event: any) => {
      const msg =
        event.value instanceof Error ? event.value.message : event.value
      if (
        msg.includes('Failed to resolve module specifier') ||
        msg.includes('Error resolving module specifier')
      ) {
        runtimeError.value =
          msg.replace(/\. Relative references must.*$/, '') +
          `.\nTip: add an "import-map.json" file to specify import paths for dependencies.`
      } else {
        runtimeError.value = event.value
      }
    },
    on_unhandled_rejection: (event: any) => {
      let error = event.value
      if (typeof error === 'string') {
        error = { message: error }
      }
      runtimeError.value = 'Uncaught (in promise): ' + error.message
    },
    on_console: (log: any) => {
      if (log.duplicate) {
        return
      }
      if (log.level === 'error') {
        if (log.args[0] instanceof Error) {
          runtimeError.value = log.args[0].message
        } else {
          runtimeError.value = log.args[0]
        }
      } else if (log.level === 'warn') {
        if (log.args[0].toString().includes('[Vue warn]')) {
          runtimeWarning.value = log.args
            .join('')
            .replace(/\[Vue warn\]:/, '')
            .trim()
        }
      }
    },
    on_console_group: (action: any) => {
      // group_logs(action.label, false);
    },
    on_console_group_end: () => {
      // ungroup_logs();
    },
    on_console_group_collapsed: (action: any) => {
      // group_logs(action.label, true);
    }
  })

  sandbox.addEventListener('load', () => {
    proxy.handle_links()
    stopUpdateWatcher = watch(() => store.state.activeFile, updatePreview, { immediate: true, deep: true })
  })
}

async function updatePreview() {
  if (import.meta.env.PROD && clearConsole.value) {
    console.clear()
  }
  runtimeError.value = null
  runtimeWarning.value = null

  let isSSR = props.ssr
  if (store.vueVersion) {
    const [_, minor, patch] = store.vueVersion.split('.')
    if (parseInt(minor, 10) < 2 || parseInt(patch, 10) < 27) {
      alert(
        `The selected version of Vue (${store.vueVersion}) does not support in-browser SSR.` +
        ` Rendering in client mode instead.`
      )
      isSSR = false
    }
  }

  try {
    const mainFile = store.state.mainFile

    // if SSR, generate the SSR bundle and eval it to render the HTML
    if (isSSR && mainFile.endsWith('.vue')) {
      const ssrModules = compileModulesForPreview(store, true)
      console.log(
        `[@vue/repl] successfully compiled ${ssrModules.length} modules for SSR.`
      )
      await proxy.eval([
          ...getScriptWithMode(`const __modules__ = {};`, 'module'),
          ...getScriptWithMode(ssrModules, 'module'),
          ...getScriptWithMode(`import { renderToString as _renderToString } from 'vue/server-renderer'
           import { createSSRApp as _createApp } from 'vue'
           new Promise((resolve) => {
              const int_id = setInterval(() => {
                const app = __modules__["${mainFile}"].default
                if(app) {
                  
                  clearInterval(int_id)
                  resolve(app)
                }
              }, 200)
            }).then(AppComponent => {
              AppComponent.name = 'Repl'
              const app = _createApp(AppComponent)
              app.config.unwrapInjectedRef = true
              app.config.warnHandler = () => {}
              window.__ssr_promise__ = _renderToString(app).then(html => {
                document.body.innerHTML = '<div id="app">' + html + '</div>'
              }).catch(err => {
                console.error("SSR Error", err)
              })
           })
          `, 'module')
      ])
    }

    // compile code to simulated module system
    const modules = compileModulesForPreview(store)
    console.log(`[@vue/repl] successfully compiled ${modules.length} modules.`)

    const codeToEval = [
      ...getScriptWithMode(`window.__modules__ = {};window.__css__ = '';` +
        `if (window.__app__) window.__app__.unmount();` +
        (isSSR ? `` : `document.body.innerHTML = '<div id="app"></div>'`), 'text/javascript'),
      ...getScriptWithMode(modules, 'module'),
      ...getScriptWithMode(`document.getElementById('__sfc-styles').innerHTML = window.__css__`, 'text/javascript')
    ]

    // if main file is a vue file, mount it.
    if (mainFile.endsWith('.vue')) {
      codeToEval.push(
        ...getScriptWithMode(`import { ${
          isSSR ? `createSSRApp` : `createApp`
        } as _createApp } from "vue"
        const _mount = async () => {
          const AppComponent = await new Promise((resolve) => {
            const int_id = setInterval(() => {
              const app = __modules__["${mainFile}"].default
              if(app) {
                
                clearInterval(int_id)
                resolve(app)
              }
            }, 200)
          })
          AppComponent.name = 'Repl'
          const app = window.__app__ = _createApp(AppComponent)
          app.config.unwrapInjectedRef = true
          app.config.errorHandler = e => console.error(e)
          app.mount('#app')
        }
        if (window.__ssr_promise__) {
          window.__ssr_promise__.then(_mount)
        } else {
          _mount()
        }`, 'module')
      )
    }

    // eval code in sandbox
    await proxy.eval(codeToEval)
  } catch (e: any) {
    runtimeError.value = (e as Error).message
  }
}
function getScriptWithMode(script: string | string[], mode = 'module') {
  if (typeof script === 'string') script = [script]
  return script.map(one => {
    return {
      code: one,
      mode,
    }
  })
}
</script>

<template>
  <div class="iframe-container" v-show="show" ref="container"></div>
  <Message :err="runtimeError" />
  <Message v-if="!runtimeError" :warn="runtimeWarning" />
</template>

<style scoped>
.iframe-container,
.iframe-container :deep(iframe) {
  width: 100%;
  height: 100%;
  border: none;
  background-color: #fff;
}
</style>
