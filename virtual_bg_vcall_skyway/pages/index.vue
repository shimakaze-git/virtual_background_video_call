<template>
  <div class="container">
    <div>
      <h1 class="subtitle">バーチャル背景の実装</h1>
      <ul>
        <!-- 仮想の背景をMediaStreamTrackProcessorで実装する -->
        <li>
          <nuxt-link to="mstreamtpbg">
            MediaPipe Selfie Segmentationでの実装(MediaStreamTrackProcessor)
          </nuxt-link>
        </li>
        <li>
          <nuxt-link to="mssegmentationfaframe">
            MediaPipe Selfie Segmentationでの実装(requestAnimationFrame)
          </nuxt-link>
        </li>
        <li>
          <nuxt-link to="bodypix"> BodyPixでの実装 </nuxt-link>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {}
  },
  // created() {},
  // updated() {},
  // methods: {},
}
</script>

<style>
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.title {
  font-family: 'Quicksand', 'Source Sans Pro', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}
</style>
