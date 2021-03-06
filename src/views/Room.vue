<template>
  <section>
    <div class="container is-fluid dashboard">
      <div class="columns holder">
        <div class="column is-half code-editor">
          <codemirror ref="cmEditor" :value="room.code" :options="cmOptions" />
          <transition name="fade">
            <Console v-if="showConsole" :output="consoleOutput" :exit-code="consoleExitCode" />
          </transition>
          <Toolbar
            :language="room.language"
            :name="room.name"
            @change-language="changeLanguage"
            @edit-name="updateName"
            @save="updateRoom"
            @run="runCode"
            @toggle-console="toggleConsole"
            @add-user="addGuest"
          />
        </div>
        <div class="column resizer"></div>
        <div class="column is-half drawing-board">
          <DrawingBoard ref="drawingBoard" @drawing="handleDrawing" />
        </div>
      </div>
    </div>
    <div class="video-chat">
      <VideoChat v-if="sockets" ref="videoChat" @host-connected="handleHostConnected" />
    </div>
  </section>
</template>

<script>
import Console from '../components/Console'
import Toolbar from '../components/Toolbar'
import DrawingBoard from '../components/DrawingBoard'
import VideoChat from '../components/VideoChat'
import { codemirror } from 'vue-codemirror'

// import base style
import 'codemirror/lib/codemirror.css'
// import language js
import 'codemirror/mode/javascript/javascript.js'
import 'codemirror/mode/python/python.js'

// import theme style
import 'codemirror/theme/monokai.css'

// socketIO
import socket from 'socket.io-client'

export default {
  components: {
    codemirror,
    Console,
    Toolbar,
    DrawingBoard,
    VideoChat
  },
  props: {
    roomId: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      room: {
        code: '',
        language: ''
      },
      cmOptions: {
        tabSize: 4,
        mode: 'text/javascript',
        theme: 'monokai',
        lineNumbers: true,
        line: true
      },
      languageModes: {
        python: 'x-python',
        javascript: 'javascript'
      },
      showConsole: false,
      consoleOutput: '',
      consoleExitCode: 0,

      // sockets
      sockets: null
    }
  },
  computed: {
    codemirror() {
      return this.$refs.cmEditor.codemirror
    }
  },
  beforeDestroy() {
    this.sockets.close()
  },
  async created() {
    const response = await this.$http.get(`rooms/${this.roomId}`)
    this.room = { ...this.room, ...response.data.room }
    this.setLanguageMode()

    // connect to sockets
    this.sockets = socket.connect(
      `${process.env.VUE_APP_API_SOCKETS_URL}sockets/room`
    )

    this.sockets.emit('join-room', { roomId: this.roomId })
    this.sockets.on('code-edited', this.syncCode)
    this.sockets.on('drawing', this.syncDrawingBoard)
    this.sockets.on('user-video-connected', this.handlePeerConnected)
    this.sockets.on('user-disconnected', this.handleUserDisconnected)
  },
  async mounted() {
    /* Using codemirrors native change event...
       ...to detect change Origin

       Vue Codemirror does not provide access to changeObj
    */
    this.codemirror.on('change', (cm, changeObj) => {
      this.onCmCodeChange(cm.getValue(), changeObj.origin)
    })

    this.addResizeListeners()
  },
  methods: {
    onCmCodeChange(newCode, origin) {
      this.room.code = newCode
      /* only emit if user types
         if the code is set by some methods
         origin value will be setValue
         if the code is set by typing event
         origin value will be something like +input, +delete, +undo...etc

         So we only emit if user types
      */
      if (origin === 'setValue') return
      this.sockets.emit('code-edited', {
        roomId: this.room.id,
        code: this.room.code
      })
    },
    async updateRoom() {
      await this.$http.put(`rooms/${this.roomId}`, this.room)
    },
    setLanguageMode() {
      this.cmOptions.mode = `text/${this.languageModes[this.room.language]}`
    },

    // toolbar event handlers
    changeLanguage(language) {
      this.room.language = language
      this.setLanguageMode()
    },
    updateName(name) {
      this.room.name = name
    },
    toggleConsole() {
      this.showConsole = !this.showConsole
    },
    async runCode() {
      let response
      try {
        response = await this.$http.post('code/run', {
          code: this.room.code,
          language: this.room.language
        })
      } catch {
        return console.error('Failed to run code!')
      }

      this.showConsole = true
      const runResult = response.data
      if (runResult.run_status.status === 'CE') {
        // compilation error
        this.consoleOutput = runResult.compile_status
        this.consoleExitCode = 1
      } else if (runResult.run_status.status === 'AC') {
        // accepted
        this.consoleOutput = runResult.run_status.output
        this.consoleExitCode = 0
      } else if (runResult.run_status.status === 'RE') {
        this.consoleOutput = runResult.run_status.stderr
        this.consoleExitCode = 1
      }
    },
    async addGuest(guestEmail) {
      try {
        const response = await this.$http.post(`rooms/${this.roomId}/guests`, {
          email: guestEmail
        })
        this.room = { ...this.room, ...response.data.room }
      } catch (err) {
        console.log(err)
      }
    },
    // end of toolbar event handers

    // sockets event handlers
    syncCode(data) {
      this.room.code = data.code
    },
    syncDrawingBoard(data) {
      this.$refs.drawingBoard.drawingHelper(data)
    },

    // child component event handlers
    handleDrawing(data) {
      this.sockets.emit('drawing', { ...data, ...{ roomId: this.roomId } })
    },

    // video chat handlers
    handleHostConnected(hostId) {
      this.sockets.emit('user-video-connected', {
        roomId: this.roomId,
        peerId: hostId
      })
    },
    handlePeerConnected(data) {
      this.$refs.videoChat.connectToPeer(data.peerId)
    },
    handleUserDisconnected() {
      this.$refs.videoChat.disconnectPeer()
    },

    // Resize listenrs
    addResizeListeners() {
      const resizer = document.querySelector('.resizer')
      const holder = document.querySelector('.holder')
      const editor = document.querySelector('.code-editor')
      const drawingBoard = document.querySelector('.drawing-board')
      const parentWidth = holder.offsetWidth
      let isMouseDown = false

      resizer.addEventListener('mousedown', (e) => {
        isMouseDown = true
      })

      document.addEventListener('mousemove', (e) => {
        if (!isMouseDown) return
        editor.style.width = `${e.clientX}px`
        drawingBoard.style.width = `${parentWidth - e.clientX}px`
      })

      document.addEventListener('mouseup', () => {
        isMouseDown = false
      })
    }
  }
}
</script>
<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
<style scoped>
/* TODO: refactor it into scss (need to learn :D) */
.dashboard {
  padding: 0;
}
.dashboard .columns {
  width: 100%;
  margin: 0;
}
.dashboard .columns .code-editor,
.dashboard .columns .drawing-board {
  padding: 0;
  border: 3px solid #1f364d;
  background: #001528;
}
.dashboard .columns .code-editor,
.dashboard .columns .drawing-board {
  height: calc(100vh - 50px);
  display: flex;
  flex-direction: column;
  justify-content: stretch;
}
.vue-codemirror {
  height: 100%;
  border-radius: 4px;
}
.vue-codemirror >>> .CodeMirror {
  height: 100% !important;
}
.vue-codemirror >>> .cm-s-monokai.CodeMirror,
.vue-codemirror >>> .cm-s-monokai .CodeMirror-gutters {
  background: #001528 !important;
}

/* vue transistion classes for console component*/
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
.video-chat {
  position: fixed;
  bottom: 0px;
  right: 0px;
  display: flex;
}
.resizer{
  padding: 0 2px;
  background-color: #1f364d;
  cursor: col-resize;
}
</style>
