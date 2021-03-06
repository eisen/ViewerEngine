<template>
  <section>
    <transition name="fade">
      <div v-if="show" id="rtx-loading-bar">
        <div id="rtx-logo"></div>
        <div id="rtx-progress-bar-empty">
          <div id="rtx-progress-bar-full"></div>
        </div>
      </div>
    </transition>
    <div class="title-bar">
      <div class="header">Othello Challenge</div>
    </div>
    <div class="clients-wrapper">
      <div class="header">Participants</div>
      <div class="participants-wrapper">
        <div v-if="NoParticipants" class="no-participants-yet">
          Waiting for participants to connect...
        </div>
        <div v-else class="participants">
          <div
            v-for="participant in participants"
            :key="participant.id"
            class="participant"
            :class="{ selected: selectedParticipant === participant.id }"
            @click="FilterGames(participant)"
          >
            <div class="participant-name">
              {{ participant.name }}
            </div>
            <div class="participant-scores">
              <div class="participant-score">
                W: {{ participant.win_count }}
              </div>
              <div class="participant-score">
                L: {{ participant.lost_count }}
              </div>
              <div class="participant-score">
                T: {{ participant.tie_count }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="unity-wrapper">
      <div id="unity-container" class="unity-desktop">
        <canvas id="unity-canvas"></canvas>
        <div id="unity-mobile-warning">
          WebGL builds are not supported on mobile devices.
        </div>
      </div>
    </div>
    <div class="tournament-wrapper">
      <div class="header">
        Active Games
        <div class="disconnected-color">Disconnected</div>
        <div class="completed-color">Finished</div>
        <div class="active-color">Running</div>
        <div class="pending-color">Waiting</div>
        <div class="status-legend">Game Status:</div>
      </div>
      <div class="games-wrapper">
        <div v-if="NoGames" class="no-games-yet">
          The tournament hasn't started yet
        </div>
        <div v-else class="games">
          <div
            v-for="game in FilteredGames"
            :key="game.id"
            class="game"
            :class="{ selected: selectedGame === game.id }"
            @click="DisplayGame(game.id)"
          >
            <div v-if="game.disconnected" class="game-state-disconnected">
              <span>VS</span>
            </div>
            <div v-else-if="game.pending" class="game-state-pending">
              <span>VS</span>
            </div>
            <div v-else-if="game.ended" class="game-state-ended">
              <span>VS</span>
            </div>
            <div v-else class="game-state-active"><span>VS</span></div>
            <div class="black-contender">
              <span>{{ game.black }}</span>
            </div>
            <div class="white-contender">
              <span>{{ game.white }}</span>
            </div>
            <div class="scores">
              <div class="black-score">
                {{ game.board.split('1').length - 1 }}
              </div>
              <div class="white-score">
                {{ game.board.split('2').length - 1 }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="footer-bar">
      <img class="rtx-logo" src="TemplateData/rtx-logo.svg" />
      <div class="header right-align">2021 RMD Hackathon</div>
    </div>
  </section>
</template>

<script>
import socket from '~/plugins/socket.io.js'

export default {
  /* global createUnityInstance */
  /* eslint object-shorthand: "off" */

  data: function () {
    return {
      app: null,
      show: true,
      participants: [],
      games: [],
      filter: '',
      selectedParticipant: '',
      selectedGame: '',
    }
  },
  computed: {
    NoParticipants: function () {
      return this.participants.length === 0
    },
    NoGames: function () {
      return this.games.length === 0
    },
    FilteredGames: function () {
      if (this.filter === '') {
        return this.games
      } else {
        return this.games.filter((game) => {
          return game.black === this.filter || game.white === this.filter
        })
      }
    },
  },
  beforeMount: function () {
    socket.on('participants-list', (data) => {
      this.participants = data
    })

    socket.on('games-list', (data) => {
      this.games = data
    })

    socket.on('disconnect', () => {
      this.participants = []
      this.games = []
      this.filter = ''
      this.app.SendMessage('Base', 'ClearBoard')
    })
  },
  mounted: function () {
    const buildUrl = 'Build'
    const loaderUrl = buildUrl + '/static.loader.js'
    const config = {
      dataUrl: buildUrl + '/static.data',
      frameworkUrl: buildUrl + '/static.framework.js',
      codeUrl: buildUrl + '/static.wasm',
      streamingAssetsUrl: 'StreamingAssets',
      companyName: 'Raytheon Technologies',
      productName: 'Othello',
      productVersion: '1.0',
    }

    const container = document.querySelector('#unity-container')
    const canvas = document.querySelector('#unity-canvas')
    const loadingBar = document.querySelector('#rtx-loading-bar')
    const progressBarFull = document.querySelector('#rtx-progress-bar-full')
    const mobileWarning = document.querySelector('#unity-mobile-warning')

    // By default Unity keeps WebGL canvas render target size matched with
    // the DOM size of the canvas element (scaled by window.devicePixelRatio)
    // Set this to false if you want to decouple this synchronization from
    // happening inside the engine, and you would instead like to size up
    // the canvas DOM size and WebGL render target sizes yourself.
    // config.matchWebGLToCanvasSize = false

    if (/iPhone|iPad|iPod|Android/i.test(navigator.userAgent)) {
      container.className = 'unity-mobile'
      // Avoid draining fillrate performance on mobile devices,
      // and default/override low DPI mode on mobile browsers.
      config.devicePixelRatio = 1
      mobileWarning.style.display = 'block'
      setTimeout(() => {
        mobileWarning.style.display = 'none'
      }, 5000)
    }
    loadingBar.style.display = 'block'

    const script = document.createElement('script')
    // const that = this
    script.src = loaderUrl
    script.onload = () => {
      createUnityInstance(canvas, config, (progress) => {
        progressBarFull.style.width = 100 * progress + '%'
      })
        .then((unityInstance) => {
          this.app = unityInstance
          this.show = false // Hide loading bar
          this.app.SendMessage(
            'Base',
            'ConnectToServer',
            'engine.rmd-hackathon.xyz:8080/viewers'
          )
        })
        .catch((message) => {
          alert(message)
        })
    }
    document.body.appendChild(script)
  },
  methods: {
    DisplayGame: function (gameID) {
      this.selectedGame = gameID
      this.app.SendMessage('Base', 'GetGameStatus', gameID)
    },
    FilterGames: function (participant) {
      if (this.filter === '' || this.filter !== participant.name) {
        this.filter = participant.name
        this.selectedParticipant = participant.id
      } else {
        this.filter = ''
        this.selectedParticipant = ''
      }
    },
  },
}
</script>

<style>
html {
  height: 100%;
}

body {
  padding: 0;
  margin: 0;
  height: 100%;
  background-image: url('~@/static/TemplateData/Hackathon_Globe_2021.png');
  background-size: 50%;
  background-position: center;
  background-repeat: no-repeat;
}

#unity-container {
  position: absolute;
}

#unity-container.unity-desktop {
  width: 100%;
  height: 100%;
}

#unity-container.unity-mobile {
  width: 100%;
  height: 100%;
}

#unity-canvas {
  background: transparent;
  width: 100%;
  height: 100%;
  border: 2px solid #ce1126;
}

.unity-mobile #unity-canvas {
  width: 100%;
  height: 100%;
}

#rtx-loading-bar {
  position: absolute;
  left: 0px;
  top: 0px;
  right: 0px;
  bottom: 0px;
  background-color: white;
  z-index: 10;
}

#rtx-logo {
  width: 100%;
  height: 100%;
  background: url('static/TemplateData/hackathon-logo-red.svg') no-repeat center;
  background-size: 50% 50%;
  position: relative;
  top: -160px;
  right: -20px;
}

#rtx-progress-bar-empty {
  width: 400px;
  height: 18px;
  margin-top: 10px;
  background: url('static/TemplateData/progress-bar-empty-rtx.png') no-repeat
    center;
  position: absolute;
  left: 50%;
  top: 65%;
  z-index: 20;
  transform: translate(-50%, -50%);
}

#rtx-progress-bar-full {
  width: 0%;
  height: 18px;
  background: url('static/TemplateData/progress-bar-full-rtx.png') no-repeat
    center;
  position: absolute;
  left: 50%;
  top: 65%;
  z-index: 20;
  transform: translate(-50%, -50%);
}

#unity-footer {
  position: relative;
}

.unity-mobile #unity-footer {
  display: none;
}

#unity-build-title {
  float: right;
  margin-right: 10px;
  line-height: 38px;
  font-family: arial;
  font-size: 18px;
}

#unity-mobile-warning {
  position: absolute;
  left: 50%;
  top: 5%;
  transform: translate(-50%);
  background: white;
  padding: 10px;
  display: none;
}

.title-bar {
  position: absolute;
  top: 0px;
  left: 0px;
  right: 0px;
  height: 50px;
  text-align: center;
  padding: 10px;
  padding-top: 0px;
  font-size: 25px;
}

.footer-bar {
  position: absolute;
  bottom: 0px;
  left: 0px;
  right: 0px;
  height: 50px;
  text-align: center;
  font-size: 25px;
  overflow: hidden;
}

.title-bar > .header,
.footer-bar > .header {
  border: none;
}

.clients-wrapper {
  position: absolute;
  top: 50px;
  left: 0px;
  bottom: 50%;
  width: 50%;
  border-bottom: 2px solid #ce1126;
  border-top: 2px solid #ce1126;
}

.unity-wrapper {
  position: absolute;
  top: 50px;
  right: 0px;
  bottom: 50%;
  width: 50%;
}

.tournament-wrapper {
  position: absolute;
  top: 50%;
  left: 0px;
  right: 0px;
  bottom: 50px;
  border-bottom: 2px solid #ce1126;
}

.header {
  padding: 10px;
  font-weight: bold;
  color: #ce1126;
  border-bottom: 2px solid #ce1126;
}

.right-align {
  text-align: right;
}

.participants-wrapper,
.games-wrapper {
  position: absolute;
  overflow-y: auto;
  top: 40px;
  left: 0px;
  right: 0px;
  bottom: 0px;
}

.participants,
.games {
  display: flex;
  flex-wrap: wrap;
  align-content: flex-start;
  justify-content: space-between;
}

.participant {
  margin: 10px;
  padding: 10px;
  border: 2px solid #ce1126;
  border-radius: 5px;
  cursor: pointer;
}

.selected {
  background-color: #009245;
  color: white;
}

.participant-scores {
  display: flex;
  justify-content: space-evenly;
}

.participant-score {
  padding-left: 10px;
  padding-right: 10px;
}

.no-participants-yet {
  color: #ce1126;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.no-games-yet {
  color: #ce1126;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.rtx-logo {
  position: absolute;
  left: 10px;
  height: 50px;
}

.game {
  position: relative;
  border: 2px solid #ce1126;
  padding: 10px;
  border-radius: 5px;
  margin: 10px;
  width: 200px;
  height: 50px;
  cursor: pointer;
}

.game:hover {
  border: 3px solid #ce1126;
}

.game-state-active {
  position: absolute;
  top: 0px;
  left: 0px;
  bottom: 0px;
  width: 30px;
  background-color: #ce1126;
  color: white;
}

.game-state-ended {
  position: absolute;
  top: 0px;
  left: 0px;
  bottom: 0px;
  width: 30px;
  background-color: blue;
  color: white;
}

.game-state-disconnected {
  position: absolute;
  top: 0px;
  left: 0px;
  bottom: 0px;
  width: 30px;
  background-color: gray;
  color: white;
}

.game-state-pending {
  position: absolute;
  top: 0px;
  left: 0px;
  bottom: 0px;
  width: 30px;
  background-color: orange;
  color: white;
}

.game-state-active > span,
.game-state-ended > span,
.game-state-pending > span,
.game-state-disconnected > span {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

.black-contender > span,
.white-contender > span {
  position: absolute;
  top: 50%;
  left: 10px;
  transform: translateY(-50%);
}

.black-contender {
  position: absolute;
  top: 0px;
  left: 30px;
  bottom: 50%;
  width: 125px;
  padding-left: 10px;
  padding-right: 10px;
  background-color: black;
  color: white;
}

.white-contender {
  position: absolute;
  top: 50%;
  left: 30px;
  bottom: 0px;
  width: 125px;
  padding-left: 10px;
  padding-right: 10px;
  background-color: white;
  color: black;
}

.scores {
  display: flex;
  flex-direction: column;
  justify-content: space-around;
  align-items: center;
  position: absolute;
  top: 0px;
  right: 0px;
  bottom: 0px;
  width: 41px;
  border-left: 2px solid #ce1126;
}

.pending-color {
  background-color: orange;
  border: 2px solid #ce1126;
  padding: 5px;
  color: white;
  border-radius: 5px;
  float: right;
  font-size: 10px;
  margin-left: 5px;
  position: relative;
  top: -3px;
}

.active-color {
  background-color: #ce1126;
  border: 2px solid #ce1126;
  padding: 5px;
  color: white;
  border-radius: 5px;
  float: right;
  font-size: 10px;
  margin-left: 5px;
  position: relative;
  top: -3px;
}

.completed-color {
  background-color: blue;
  border: 2px solid #ce1126;
  padding: 5px;
  color: white;
  border-radius: 5px;
  float: right;
  font-size: 10px;
  margin-left: 5px;
  position: relative;
  top: -3px;
}

.disconnected-color {
  background-color: gray;
  border: 2px solid #ce1126;
  padding: 5px;
  color: white;
  border-radius: 5px;
  float: right;
  font-size: 10px;
  margin-left: 5px;
  position: relative;
  top: -3px;
}

.status-legend {
  padding: 5px;
  color: #ce1126;
  float: right;
  font-size: 12px;
  margin-left: 5px;
  position: relative;
  top: -2px;
}
</style>
