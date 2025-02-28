<template>
  <section class="dumi-previewer">
    <div class="dumi-previewer-demo">
      <template v-if="scope && demo">
        <component :is="demo" />
      </template>

      <template v-else>
        <slot name="demo"></slot>
      </template>
    </div>

    <div class="dumi-previewer-actions">
      <div></div>

      <div>
        <svg
          v-if="copied"
          class="dumi-previewer-actions__icon"
          style="fill: green;"
          xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"><path fill="none" d="M0 0h24v24H0z"/><path d="M10 15.172l9.192-9.193 1.415 1.414L10 18l-6.364-6.364 1.414-1.414z"/></svg>

        <svg
          v-else
          class="dumi-previewer-actions__icon"
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 24 24" width="24" height="24"
          @click="handleCopy"
        ><path fill="none" d="M0 0h24v24H0z"/><path d="M7 6V3a1 1 0 0 1 1-1h12a1 1 0 0 1 1 1v14a1 1 0 0 1-1 1h-3v3c0 .552-.45 1-1.007 1H4.007A1.001 1.001 0 0 1 3 21l.003-14c0-.552.45-1 1.007-1H7zM5.003 8L5 20h10V8H5.003zM9 6h8v10h2V4H9v2z"/></svg>

        <svg
          class="dumi-previewer-actions__icon"
          xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24"
          @click="handleCollapse"
        ><path fill="none" d="M0 0h24v24H0z"/><path d="M24 12l-5.657 5.657-1.414-1.414L21.172 12l-4.243-4.243 1.414-1.414L24 12zM2.828 12l4.243 4.243-1.414 1.414L0 12l5.657-5.657L7.07 7.757 2.828 12zm6.96 9H7.66l6.552-18h2.128L9.788 21z"/></svg>
      </div>
    </div>

    <div
      v-show="!collapsed"
      class="dumi-previewer-source">
      <div
        v-html="highlightCode"
        class="language-vue extra-class"
      />
    </div>
  </section>
</template>

<script>
import copy from 'copy-to-clipboard'
import highlight from './highlight'

const compiler = require('vue-template-compiler')
const Babel = require('@babel/standalone')
const jsx = require('babel-plugin-transform-vue-jsx')

const templateBlockReg = /<template>([\s\S]+)<\/template>/
const scriptBlockReg = /<script>([\s\S]+)<\/script>/

Babel.registerPlugin('jsx', jsx)

export default {
  name: 'dumi-previewer',

  props: {
    code: {
      type: String,
      default: ''
    },

    scope: {
      type: Boolean,
      default: false
    }
  },

  data () {
    return {
      collapsed: false,
      copied: false,
      timerId: null,

      /**
       * take over VuePress render
       * 接管VuePress渲染
      */
      demo: null
    }
  },

  computed: {
    decodedCode () {
      return decodeURIComponent(this.code)
    },

    highlightCode () {
      return highlight(this.decodedCode, 'vue')
    }
  },

  created () {
    if (this.decodedCode && this.scope) {
      const code = this.decodedCode
      const scriptBlock = code.match(scriptBlockReg)
      const templateBlock = code.match(templateBlockReg)
      /**
       * not match Vue SFC
       * 不符合Vue SFC定义
       */
      if (!scriptBlock || !templateBlock) return
      if (!scriptBlock[1] || !templateBlock[1]) return

      /**
       * generate data/methods etc Vue options
       * 提取data/methods等选项
       */
      let optionString = scriptBlock[1].trim()
      try {
        // compile jsx with @babel/standalone, 使用@babel/standalone编译jsx
        optionString = Babel.transform(optionString, { plugins: ['jsx'] }).code
      } catch (err) {
        console.error(err)
      }
      optionString = optionString.replace('export default ', '')

      // eslint-disable-next-line prefer-const
      let option = null
      // eslint-disable-next-line
      eval(`option = ${optionString}`)

      if (!this.$root.constructor) return

      /**
       * generate render function
       * 编译得到渲染函数
       */
      const renderFunc = compiler.compile(`<div>${templateBlock[1]}</div>`).render

      const demoComponent = this.$root.constructor.component('demo', {
        ...option,
        // eslint-disable-next-line
        render: new Function(renderFunc)
      })
      this.demo = demoComponent
      this.$options.components.demo = demoComponent
    }
  },

  beforeDestroy () {
    clearTimeout(this.timerId)
  },

  methods: {
    handleCollapse () {
      this.collapsed = !this.collapsed
    },

    handleCopy () {
      this.copied = true
      copy(this.decodedCode)

      clearTimeout(this.timer)
      this.timerId = setTimeout(() => {
        this.copied = false
      }, 2000)
    }
  }
}
</script>

<style lang="stylus">
.dumi-previewer {
  background-color: #fff;
  border: 1px solid #ebedf1;
  border-radius: 1px;

  .dumi-previewer-demo {
    padding: 40px 24px;
  }

  .dumi-previewer-actions {
    display: flex;
    align-items: center;
    justify-content: space-between;
    border-top: 1px dashed #ebedf1;
    height: 40px;
    padding: 0 1em;

    .dumi-previewer-actions__icon {
      width: 16px;
      height: 16px;
      padding: 8px 4px;
      opacity: 0.4;
      cursor: pointer;
      transition: opacity .3s;

      &:hover {
        opacity: 0.6;
      }
    }
  }

  .dumi-previewer-source {
    border-top: 1px dashed #ebedf1;

    div[class*="language-"] {
      background-color: #f9fafb;
      border-radius: 0;
    }

    pre[class*="language-"] {
      margin: 0 !important;
    }

    pre[class*="language-"] code {
      color: #000 !important;
    }

    .token.cdata,.token.comment,.token.doctype,.token.prolog {
      color: #708090
    }

    .token.punctuation {
      color: #999
    }

    .token.namespace {
      opacity: .7
    }

    .token.boolean,.token.constant,.token.deleted,.token.number,.token.property,.token.symbol,.token.tag {
      color: #905
    }

    .token.attr-name,.token.builtin,.token.char,.token.inserted,.token.selector,.token.string {
      color: #690
    }

    .language-css .token.string,.style .token.string,.token.entity,.token.operator,.token.url {
      color: #9a6e3a;
      background: hsla(0,0%,100%,.5)
    }

    .token.atrule,.token.attr-value,.token.keyword {
      color: #07a
    }

    .token.class-name,.token.function {
      color: #dd4a68
    }

    .token.important,.token.regex,.token.variable {
      color: #e90
    }
  }
}
</style>
