<template>
  <div id="app">
    <div ref="editor"/>

    <div class="toolbar">
      <input
        @click="compile"
        type="button"
        value="compile"
      >
      <input
        @click="compileAndRun"
        type="button"
        value="compile & run"
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
        :disabled="compiled != null && !compiled.getValue().length"
      >
    </div>

    <div ref="all"/>
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

    this.editor.commands.on('exec', (e) => {
      const rowCol = this.editor.selection.getCursor()

      if (rowCol.row <= 6 && !['go', 'select', 'copy'].some(s => e.command.name.startsWith(s))) {
        e.preventDefault()
        e.stopPropagation()
        console.debug(`sorry ${e.command.name} is not allowd in this editor`)
      }
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
    this.all = this.setUpEditor(this.$refs.all, 'javascript', { readOnly: true })
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
      const text = d`
        with(this) {
          ${this.env.getValue()}

          ${this.compiled.getValue()}

          return {setup, step}
        }
      `
      this.all.setValue(text)
      this.all.clearSelection()

      // eslint-disable-next-line no-new-func
      const fn = new Function(text)

      Object.assign(this, fn())
      this.setup()
    },
    compileAndRun () {
      this.compile()
      this.run()
    }
  },
  data () {
    return {
      all: null,
      compiled: null,
      editor: null,
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
