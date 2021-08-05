<template>
  <div>
    <h1>仮想の背景をMediaStreamTrackProcessorで実装する</h1>
    <div>
      <div class="canvas-container">
        <!-- canvasタグを非表示にしておく. -->
        <canvas
          id="output_canvas"
          ref="output_canvas"
          width="480px"
          height="320px"
        />

        <video
          id="output_canvas_video"
          ref="output_canvas_video"
          autoplay
          muted
          playsinline
          width="480px"
          height="320px"
        />
      </div>
    </div>

    <p v-if="peerId" id="my-id">{{ peerId }}</p>
    <input id="their-id" v-model="theirId" />
    <button id="make-call" :disabled="!theirId.length > 0" @click="makeCall">
      発信
    </button>
    <video
      id="their-video"
      ref="their_video"
      width="400px"
      autoplay
      muted
      playsinline
    ></video>
  </div>
</template>

<script>
import Peer from 'skyway-js'

export default {
  data() {
    return {
      localMediaStream: null,
      selfieSegmentationInstance: null,
      segmentedLocalMediaStream: null,
      peer: null,
      peerId: '',
      theirId: '',
    }
  },
  created() {
    console.log('this.$config', this.$config)
    this.peer = new Peer({
      key: this.$config.SKYWAY_API_KEY,
      debug: 3,
    })

    this.peer.on('open', (peerId) => {
      this.peerId = peerId
    })
    this.peer.on('call', (mediaConnection) => {
      const segmentedLocalMediaStream = this.$refs.output_canvas.captureStream()
      mediaConnection.answer(segmentedLocalMediaStream)
      this.setEventListener(mediaConnection)
    })
  },
  async mounted() {
    // SelfieSegmentationをセットする.
    await this.setSelfieSegmentation()

    // カメラと接続を行う.
    await this.connectCamera()

    // 描画を行う.
    await this.executeDraw()

    setTimeout(() => {
      // canvasをvideoタグに埋め込めるように変換.
      const stream = this.$refs.output_canvas.captureStream(30)
      const videoElm = this.$refs.output_canvas_video
      videoElm.srcObject = stream
      videoElm.play()
    }, 500)
  },
  methods: {
    makeCall() {
      const mediaConnection = this.peer.call(
        this.theirId,
        this.$refs.output_canvas.captureStream()
      )
      this.setEventListener(mediaConnection)
    },
    setEventListener(mediaConnection) {
      mediaConnection.on('stream', (stream) => {
        // video要素にカメラ映像をセットして再生
        const videoElm = this.$refs.their_video
        videoElm.srcObject = stream
        videoElm.play()
      })
    },
    async setSelfieSegmentation() {
      // @mediapipe/selfie_segmentationをCDNとして読み込む
      const scriptEl = await document.createElement('script')
      scriptEl.setAttribute('charset', 'utf-8')
      scriptEl.setAttribute(
        'src',
        'https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/selfie_segmentation.js'
      )
      document.head.appendChild(scriptEl)
      await new Promise((resolve, reject) => {
        setTimeout(resolve, 1000)
      })

      const canvasElement = this.$refs.output_canvas
      const canvasCtx = canvasElement.getContext('2d')
      this.segmentedLocalMediaStream = canvasElement.captureStream()

      const chara = new Image()
      chara.src = require('@/assets/sample.jpeg')
      this.chara = chara

      const onResults = (results) => {
        canvasCtx.save()
        canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height)
        canvasCtx.drawImage(
          results.segmentationMask,
          0,
          0,
          canvasElement.width,
          canvasElement.height
        )
        canvasCtx.globalCompositeOperation = 'source-in'
        canvasCtx.drawImage(
          results.image,
          0,
          0,
          canvasElement.width,
          canvasElement.height
        )
        canvasCtx.globalCompositeOperation = 'destination-atop'
        canvasCtx.drawImage(
          chara,
          0,
          0,
          canvasElement.width,
          canvasElement.height
        )
        canvasCtx.restore()
        results.segmentationMask.close()
        results.image.close()
      }

      const selfieSegmentationInstance = new window.SelfieSegmentation({
        locateFile: (file) => {
          return `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`
        },
      })

      selfieSegmentationInstance.setOptions({
        selfieMode: true,
        modelSelection: 1,
        effect: 'background',
      })

      selfieSegmentationInstance.onResults(onResults)
      this.selfieSegmentationInstance = selfieSegmentationInstance
    },

    // カメラと接続を行う.
    async connectCamera() {
      this.localMediaStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: false,
      })
    },

    // 描画の開始処理
    async executeDraw() {
      window.selfieSegmentationInstance = this.selfieSegmentationInstance
      if ('MediaStreamTrackProcessor' in window) {
        const processor = await new window.MediaStreamTrackProcessor(
          this.localMediaStream.getVideoTracks()[0]
        )
        const writable = await new WritableStream({
          // start() {},
          async write(videoFrame) {
            const imageBitmap = await createImageBitmap(videoFrame)
            await window.selfieSegmentationInstance.send({ image: imageBitmap })
            imageBitmap.close()
            videoFrame.close()
          },
          // stop() {},
        })
        processor.readable.pipeTo(writable)
      }
    },
  },
}
</script>

<style scoped>
/* canvasタグを非表示にしておく. */
#output_canvas {
  display: none;
}
</style>
