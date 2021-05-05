<template>
  <div id="app">
    <div ref="editor"/>

    <div class="toolbar">
      <input
        @click="compile"
        type="button"
        value="compile"
      >
    </div>

    <div ref="compiled"/>

    <div class="toolbar">
      <input
        @click="eval"
        type="button"
        value="eval"
        :disabled="compiled != null && !compiled.getValue().length"
      >
    </div>

    <div ref="env"/>

    <div class="toolbar">
      <input
        @click="run"
        type="button"
        value="run"
      >
    </div>

  </div>
</template>

<script>
import ace from 'ace-builds'
import d from 'dedent'
/* global Opal */

require('ace-builds/webpack-resolver')
require('ace-builds/src-noconflict/ext-language_tools')

export default {
  name: 'App',
  mounted () {
    const interval = setInterval(() => { this.step() }, 1000)
    this.$on('hook:beforeDestroy', () => {
      clearInterval(interval)
    })

    this.editor = this.setUpEditor(this.$refs.editor, 'ruby', {
      text: d`
        require 'native'

        def wrapper(fn, *vars)
          send(fn, *vars.map { |v| Native(\`v\`) })
        end

        # user code starts here

        def step(x)
          x.b
          puts x.a
        end

        def setup(x)
          puts "initial value is #{x.a}"
        end
      `
    })
    this.compiled = this.setUpEditor(this.$refs.compiled, 'javascript')
    this.env = this.setUpEditor(this.$refs.env, 'javascript', {
      text: d`
        o = {
          a: 3,
          b() { this.a++ }
        }

        function setup() {
          Opal.top.$wrapper('setup', o)
        }

        function step() {
          Opal.top.$wrapper('step', o)
        }
      `
    })
  },
  methods: {
    compile () {
      this.compiled.setValue(Opal.compile(this.editor.getValue()))
      this.compiled.clearSelection()
    },
    loadScript (lib, cb = () => {}) {
      const script = document.createElement('script')
      script.setAttribute('src', lib)
      script.addEventListener('load', cb)
      document.head.appendChild(script)
    },
    setUpEditor (element, lang, { text = '', readOnly = false } = {}) {
      const editor = ace.edit(element, {
        readOnly: readOnly,
        mode: `ace/mode/${lang}`,
        theme: 'ace/theme/monokai',
        enableBasicAutocompletion: true,
        maxLines: 20,
        minLines: 15,
        tabSize: 2,
        scrollPastEnd: true,
        value: text
      })

      editor.renderer.on('afterRender', () => {
        const worker = editor.getSession().$worker
        if (!worker) {
          return
        }
        worker.send('changeOptions', [{ asi: true }])
      })

      return editor
    },
    eval () {
      // eslint-disable-next-line no-eval
      eval(this.compiled.getValue())
    },
    run () {
      // eslint-disable-next-line no-new-func
      const fn = new Function('fn', `
        with(this) {
          ${this.compiled.getValue()}

          ${this.env.getValue()}

          return {setup, step}
        }
      `)

      Object.assign(this, fn())
      this.setup()
    }
  },
  data () {
    return {
      editor: null,
      compiled: null,
      setup: () => {},
      step: () => {}
    }
  }
}
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}
.toolbar {
  margin: 10px 0;
}
</style>
