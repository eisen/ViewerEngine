<template>
  <div id="unity-container" class="unity-desktop">
    <canvas id="unity-canvas"></canvas>
    <div id="rtx-loading-bar">
      <div id="rtx-logo"></div>
      <div id="rtx-progress-bar-empty">
        <div id="rtx-progress-bar-full"></div>
      </div>
    </div>
    <div id="unity-mobile-warning">
      WebGL builds are not supported on mobile devices.
    </div>
  </div>
</template>

<script>
export default {
  /* global createUnityInstance */

  mounted: () => {
    const buildUrl = 'Build'
    const loaderUrl = buildUrl + '/static.loader.js'
    const config = {
      dataUrl: buildUrl + '/static.data',
      frameworkUrl: buildUrl + '/static.framework.js',
      codeUrl: buildUrl + '/static.wasm',
      streamingAssetsUrl: 'StreamingAssets',
      companyName: 'Raytheon Technologies',
      productName: 'Othello',
      productVersion: '0.1',
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
    } else {
      canvas.style.width = '1280px'
      canvas.style.height = '720px'
    }
    loadingBar.style.display = 'block'

    const script = document.createElement('script')
    script.src = loaderUrl
    script.onload = () => {
      createUnityInstance(canvas, config, (progress) => {
        progressBarFull.style.width = 100 * progress + '%'
      })
        .then((unityInstance) => {
          loadingBar.style.display = 'none'
        })
        .catch((message) => {
          alert(message)
        })
    }
    document.body.appendChild(script)
  },
}
</script>

<style>
body {
  padding: 0;
  margin: 0;
}

#unity-container {
  position: absolute;
}

#unity-container.unity-desktop {
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
}

#unity-container.unity-mobile {
  width: 100%;
  height: 100%;
}

#unity-canvas {
  background: white;
}

.unity-mobile #unity-canvas {
  width: 100%;
  height: 100%;
}

#rtx-loading-bar {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  display: none;
}

#rtx-logo {
  width: 400px;
  height: 100px;
  background: url('static/TemplateData/rtx-logo.svg') no-repeat center;
}

#rtx-progress-bar-empty {
  width: 400px;
  height: 18px;
  margin-top: 10px;
  background: url('static/TemplateData/progress-bar-empty-rtx.png') no-repeat
    center;
}

#rtx-progress-bar-full {
  width: 0%;
  height: 18px;
  margin-top: 10px;
  background: url('static/TemplateData/progress-bar-full-rtx.png') no-repeat
    center;
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
</style>
