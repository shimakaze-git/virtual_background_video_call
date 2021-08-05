<template>
  <div>
    <h1>仮想の背景をMediaStreamTrackProcessorで実装する</h1>
    <h2>（requestAnimationFrame）</h2>
    <div>
      <video
        id="videoTag"
        ref="videoTag"
        width="240px"
        height="120px"
        autoplay
        muted
        playsinline
      />
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
    />
  </div>
</template>
<script>
import Peer from 'skyway-js'

export default {
  data() {
    return {
      prevTime: null,
      videoTag: null,

      localMediaStream: null,
      selfieSegmentationInstance: null,
      segmentedLocalMediaStream: null,
      peer: null,
      peerId: '',
      theirId: '',
    }
  },
  created() {
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
    executeDraw() {
      //   await new Promise((resolve, reject) => {
      //     setTimeout(resolve, 500)
      //   })

      this.videoTag = this.$refs.videoTag
      this.videoTag.srcObject = this.localMediaStream
      this.videoTag.play()

      this.prevTime = Date.now()
      window.requestAnimationFrame(this.drawImage)
    },

    // 実際の描画処理を行う.
    async drawImage() {
      if (Date.now() - this.prevTime > 30) {
        // if (Date.now() - this.prevTime > 60) {
        this.prevTime = Date.now()
        if (this.videoTag.currentTime !== 0) {
          const imageBitMap = await createImageBitmap(this.videoTag)
          //   console.log('imageBitMap', imageBitMap)
          //   console.log('selfieSegmentation', this.selfieSegmentationInstance)
          await this.selfieSegmentationInstance.send({ image: imageBitMap })
        }
      }
      window.requestAnimationFrame(this.drawImage)
    },
  },
}
</script>
<style scoped>
#videoTag {
  display: none;
}

#output_canvas_video {
  display: none;
}
</style>
