<template>
  <div>
    <h1>仮想の背景をTensorflow.jsのBodyPixで実装する</h1>
    <div>
      <video
        id="videoTag"
        ref="videoTag"
        width="480px"
        height="320px"
        autoplay
        muted
        playsinline
      />
      <img
        id="bg"
        ref="bg"
        src="@/assets/sample.jpeg"
        width="480px"
        height="320px"
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

    <!-- <canvas id="tmp_canvas" ref="tmp_canvas" width="480px" height="320px" /> -->
    <canvas ref="canvas" width="480px" height="320px" />

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

import * as tf from '@tensorflow/tfjs'
import * as bodyPix from '@tensorflow-models/body-pix'

export default {
  data() {
    return {
      videoTag: null,
      output: null,

      localMediaStream: null,
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
      const segmentedLocalMediaStream =
        this.$refs.output_canvas.captureStream(30)
      mediaConnection.answer(segmentedLocalMediaStream)
      this.setEventListener(mediaConnection)
    })
  },

  async mounted() {
    // BodyPixのセットアップをする
    const bodypixnet = await this.setBodyPix()

    // カメラと接続を行う.
    await this.connectCamera()

    // input & onput
    const input = this.videoTag
    const output = this.output

    // const canvas = document.getElementById('canvas')
    // const ctx = canvas.getContext('2d')

    // 描画を行う.
    await this.segmentBody(input, output, bodypixnet)

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

    // BodyPixのセットアップする.
    async setBodyPix() {
      this.output = this.$refs.output_canvas

      tf.getBackend()
      const bodypixnet = await bodyPix.load()
      return bodypixnet
    },

    // カメラと接続を行う.
    async connectCamera() {
      this.videoTag = this.$refs.videoTag
      this.localMediaStream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: false,
      })

      this.videoTag.srcObject = this.localMediaStream
      return new Promise((resolve) => {
        this.videoTag.onloadedmetadata = () => {
          this.videoTag.play()
          resolve()
        }
      })
    },

    segmentBody(input, output, bodypixnet) {
      const renderFrame = async () => {
        const segmentation = await bodypixnet.segmentPerson(input)

        const backgroundBlurAmount = 3
        const edgeBlurAmount = 3
        // const flipHorizontal = true
        const flipHorizontal = false

        bodyPix.drawBokehEffect(
          output,
          input,
          segmentation,
          backgroundBlurAmount,
          edgeBlurAmount,
          flipHorizontal
        )

        // const canvas = this.$refs.canvas
        // const frontElement = this.videoTag
        // const backElement = this.$refs.bg

        // this._drawFrontBackToCanvas(
        //   canvas,
        //   segmentation,
        //   frontElement,
        //   backElement
        // )

        requestAnimationFrame(renderFrame)
      }

      //
      renderFrame()
    },

    _drawFrontBackToCanvas(canvas, segmentation, frontElement, backElement) {
      const ctx = canvas.getContext('2d')
      const width = canvas.width
      const height = canvas.height

      console.log('ctx', ctx)
      console.log('frontElement', frontElement)

      // 先に人物が映った映像（前景）を描画し、イメージとして取っておく
      ctx.drawImage(frontElement, 0, 0)
      const frontImg = ctx.getImageData(0, 0, width, height)

      // 背景になる画像を描画し、イメージを取り出す
      const srcWidth = backElement.naturalWidth
      const srcHeight = backElement.naturalHeight
      ctx.drawImage(backElement, 0, 0, srcWidth, srcHeight, 0, 0, width, height)
      const imageData = ctx.getImageData(0, 0, width, height)
      const pixels = imageData.data

      // セグメントを走査し、人体の部分だったら前景の画像の値を背景の画像に合成する
      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          const base = (y * width + x) * 4
          const segbase = y * width + x
          if (segmentation.data[segbase] === 1) {
            // console.log('pixels', pixels)
            // console.log('frontImg', frontImg)

            // is fg
            // --- 前景 ---

            pixels[base + 0] = frontImg.data[base + 0] // R
            pixels[base + 1] = frontImg.data[base + 1] // G
            pixels[base + 2] = frontImg.data[base + 2] // B
            pixels[base + 3] = frontImg.data[base + 3] // α
          }
        }
      }

      // 合成した画像を、改めてキャンバスに描画する
      ctx.putImageData(imageData, 0, 0)
    },
  },
}
</script>
<style scoped>
#videoTag {
  display: none;
}

#output_canvas {
  display: none;
}

#output_canvas_video {
  /* display: none; */
}

#bg {
  display: none;
}
</style>
