<template>
  <div id="translate-man-app"
    :class="$style.app"
    :style="appBgColorStyle"
    ref="app"
    v-if="translateResult.keyword"
    @dblclick.stop
    @mousemove.stop
    @mousedown.stop
    @mouseup.stop
    @mouseover="outputBoxMouseOver">
    <OutputBox></OutputBox>
  </div>
</template>

<script>
  import Vuex from 'vuex'
  import OutputBox from './containers/outputBox'
  import { getWordFromPoint } from './utils'

  export default {
    components: {
      OutputBox,
    },

    data() {
      return {
        mouseX: 0,
        mouseY: 0,
        pageX: 0,
        pageY: 0,
        isPressKey: false,
        isHoverTranslate: false,
        selectStartTimer: null,
        hoverTimeHandler: null,
        hideTimer: null,
        moveTimer: null,
      }
    },

    computed: {
      ...Vuex.mapState([
        'userSetting',
        'translateResult',
      ]),
      appBgColorStyle() {
        return {
          'background-color': this.userSetting.bgColor,
        }
      },
    },

    mounted() {
      browser.runtime.onMessage.addListener(this.onMessage)
      document.addEventListener('dblclick', this.doubleClick)
      document.addEventListener('mousemove', this.mouseMove)
      document.addEventListener('mousedown', this.mouseDown)
      document.addEventListener('mouseup', this.mouseUp)
      window.addEventListener('keydown', this.keyDown)
      window.addEventListener('keyup', this.keyUp)
      window.addEventListener('scroll', this.scrollEvent)
    },

    destroyed() {
      document.removeEventListener('dblclick', this.doubleClick)
      document.removeEventListener('mousemove', this.mouseMove)
      document.removeEventListener('mousedown', this.mouseDown)
      document.removeEventListener('mouseup', this.mouseUp)
      window.removeEventListener('keydown', this.keyDown)
      window.removeEventListener('keyup', this.keyUp)
      window.removeEventListener('scroll', this.scrollEvent)
      clearTimeout(this.selectStartTimer)
      clearTimeout(this.hoverTimeHandler)
      clearTimeout(this.hideTimer)
      clearTimeout(this.moveTimer)
    },

    methods: {
      onMessage(request) {
        if (request.type === 'reload') {
          this.$store.dispatch('SYNC_USER_SETTING')
        }
      },
      hide() {
        if (this.translateResult.keyword) {
          this.$store.commit('reset')
          this.$store.commit('stopSound')
        }
      },
      resizePosition() {
        const containerWrap = this.$refs.app
        const containerWrapHeight = parseInt(window.getComputedStyle(containerWrap).height, 10)
        const containerWrapWidth = parseInt(window.getComputedStyle(containerWrap).width, 10)

        if (this.pageX > document.body.scrollWidth - containerWrapWidth - 50) {
          containerWrap.style.left = `${this.pageX - containerWrapWidth - 5}px`
        } else {
          containerWrap.style.left = `${this.pageX + 5}px`
        }

        if (this.pageY > document.body.scrollHeight - containerWrapHeight - 50) {
          containerWrap.style.top = `${this.pageY - containerWrapHeight - 15}px`
        } else {
          containerWrap.style.top = `${this.pageY + 15}px`
        }
      },
      outputBoxMouseOver() {
        clearTimeout(this.hideTimer)
      },
      doubleClick() {
        if (this.userSetting.doubleClick) {
          const word = window.getSelection().toString()
          if (word) {
            this.translate(word)
          }
        }
      },
      mouseMove(e) {
        clearTimeout(this.moveTimer)
        this.moveTimer = setTimeout(() => {
          this.mouseX = e.clientX
          this.mouseY = e.clientY
          this.pageX = e.pageX
          this.pageY = e.pageY
          clearTimeout(this.hoverTimeHandler)
          if (this.isHoverTranslate) {
            clearTimeout(this.hideTimer)
            this.hideTimer = setTimeout(() => {
              this.hide()
            }, 200)
          }
          this.hoverTimeHandler = setTimeout(() => {
            if (!this.userSetting.hover || this.translateResult.keyword) {
              return
            }
            const word = getWordFromPoint(this.mouseX, this.mouseY, this.$refs.app)
            this.translate(word, true)
          }, this.userSetting.hoverTime * 1000)
        }, 50)
      },
      mouseDown() {
        this.selectStartTimer = new Date().getTime()
        this.isPressKey = false
        this.hide()
      },
      mouseUp() {
        if (this.userSetting.stroke) {
          if (new Date().getTime() - this.selectStartTimer > 300) {
            const word = window.getSelection().toString()
            this.translate(word)
          }
        }
      },
      keyDown(e) {
        if (this.userSetting.pressKey && e.key === this.userSetting.pressKeyString) {
          this.isPressKey = true
        } else {
          this.isPressKey = false
        }
      },
      keyUp() {
        if (this.userSetting.pressKey && this.isPressKey) {
          let word = window.getSelection().toString()

          if (!word) {
            word = getWordFromPoint(this.mouseX, this.mouseY, this.$refs.app)
          }
          this.translate(word)
        } else {
          this.hide()
        }
      },
      scrollEvent() {
        if (this.isHoverTranslate) {
          this.hide()
        }
      },
      translate(word, isHover = false) {
        this.isHoverTranslate = isHover
        if (word === '' || !word.match(/\S/)) {
          return
        }
        this.$store.dispatch('WEB_TRANSLATE_KEYWORD', word).then(() => {
          this.$nextTick(() => {
            this.resizePosition()
          })
        }).catch(() => {})
      },
    },
  }
</script>

<style lang="postcss" module>
  .app {
    position: absolute;
    left: 0;
    top: 0;
    z-index: 9999;
    font-family: 'Helvetica Neue', Tahoma, Arial, PingFangSC-Regular, 'Hiragino Sans GB', 'Microsoft Yahei', sans-serif;
    text-shadow: rgb(255, 255, 255) 0px 1px 1px;
    min-width: 0;
    max-width: 280px;
    max-height: 400px;
    overflow-x: hidden;
    overflow-y: auto;
    line-height: normal;
    padding: 5px 10px;
    border-radius: 4px;
    box-shadow: rgba(143, 143, 143, 0.72) 0px 0px 2px, rgba(174, 174, 174, 0.298) 0px 1px 1px;
    background-color: rgba(255, 255, 255, 0.99);
  }
</style>



