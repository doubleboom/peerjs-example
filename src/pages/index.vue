<script setup lang="ts">
import type { DataConnection, Peer } from 'peerjs'
// import pkg from 'peerjs'
// const { Peer } = pkg
// import { Peer } from 'peerjs'
// const user = useUserStore()
// const name = $ref(user.savedName)
let peerid = $ref<string>('')
let destpeerid = $ref<string>('')
const video = ref<HTMLVideoElement | null>(null)
// const src = $ref('')
let message = $ref<string>('')
const messages = $ref<string[]>([])
let conn = $ref<DataConnection | null>(null)
let peer = $ref<Peer | null>(null)
// const router = useRouter()

const displayMediaOptions = {
  audio: true,
  video: true,
}

const stop = () => {
  messages.push('[note]close stream!')
  const tracks = (video.value?.srcObject as MediaStream)?.getTracks()
  tracks.forEach((track) => {
    track.stop()
  })
  if (video.value)
    video.value.srcObject = null
}

onMounted(async () => {
  if (typeof window !== 'undefined') {
    const { Peer } = await import('peerjs')
    peerid = `test_peer_${Date.now()}`
    messages.push(`init local peerid${peerid}`)
    peer = new Peer(peerid)
    peer?.on('connection', (lconn) => {
      conn = lconn
      destpeerid = lconn.peer

      lconn.on('data', (data) => {
        // Will print 'hi!'
        if (data === 'close stream')
          stop()

        else
          messages.push(`other:${data}`)
      })
      lconn.on('open', () => {
        messages.push('connected!')
      })
    })
    peer?.on('disconnected', (currentId) => {
      messages.push('disconnected')
    })
    peer?.on('close', () => {
      messages.push('close')
    })
    peer?.on('error', (err) => {
      messages.push(err.message)
    })

    peer.on('call', (call) => {
      call.answer() // Answer the call with an A/V stream.
      call.on('stream', (remoteStream) => {
        // Show stream in some <video> element.
        messages.push('[note]start stream!')
        if (!video.value)
          return
        video.value.srcObject = remoteStream
        video.value.play()
        video.value.srcObject.getVideoTracks()[0].addEventListener('ended', (event) => {
          console.log(event)

          call.emit('close')
          call.close()
        })
      })
      call.on('error', (err) => {
        console.log(`remotestream err:${err}`)
      })
      call.on('close', () => {
        messages.push('[note]close stream!')
        if (video.value)
          video.value.srcObject = null
      })
    })
  }
})
// const go = () => {
//   if (name)
//     router.push(`/hi/${encodeURIComponent(name)}`)
// }

const connect = () => {
  if (!message)
    alert('value is not null')
  messages.push('connecting!')
  if (!peer)
    return
  conn = peer.connect(message)
  destpeerid = message
  message = ''
  conn?.on('open', () => {
    messages.push('connected!')
    // conn?.send('hi!')
  })
  conn?.on('data', (data) => {
    // Will print 'hi!'
    if (data === 'close stream')
      stop()

    else
      messages.push(`other:${data}`)
  })
}

const sendMessage = () => {
  console.log(message)
  if (message === 'close stream')
    stop()
  conn?.send(message)
  messages.push(`me:${message}`)
  message = ''
}

const startAudio = () => {
  console.log('startVideo', peerid, destpeerid)
  if (!destpeerid)
    return
  navigator.mediaDevices.getDisplayMedia(displayMediaOptions).then(
    (stream) => {
      console.log('localstream')
      if (!peer)
        return
      const call = peer.call(destpeerid, stream)
      if (!video.value)
        return
      video.value.srcObject = stream
      video.value.play()
      video.value.srcObject.getVideoTracks()[0].addEventListener('ended', (event) => {
        console.log(event)
        conn?.send('close stream')
        stop()
        call.close()
      })
      call.on('error', (err) => {
        // Show stream in some <video> element.
        console.log(`localstream error:${err}`)
        // src = URL.createObjectURL(remoteStream)
      })
      call.on('close', () => {
        messages.push('[note]close stream!')
        if (video.value)
          video.value.srcObject = null
      })
      call.on('iceStateChanged', (state) => {
        console.log(state)

        if (state === 'connected')
          messages.push('[note]start stream!')
        if (state === 'disconnected') {
          messages.push('[note]disconnected!')
          stop()
        }

        // video.value.srcObject = null
      })
    },
    (err) => {
      console.error('Failed to get local stream', err)
    },
  )
}

const getVideoInfo = () => {
  const videoTrack = (video.value?.srcObject as MediaStream)?.getVideoTracks()[0]

  console.info('Track settings:')
  console.info(JSON.stringify(videoTrack.getSettings(), null, 2))
  console.info('Track constraints:')
  console.info(JSON.stringify(videoTrack.getConstraints(), null, 2))
}

const { t } = useI18n()
</script>

<template>
  <div>
    <div text-4xl>
      <div i-carbon-campsite inline-block />
    </div>
    <div>
      <span>{{ peerid }}</span>
    </div>
    <input
      id="input"
      v-model="message"
      placeholder="输入房间号"
      aria-label="输入房间号"
      type="text"
      autocomplete="false"
      p="x4 y2"
      w="250px"
      text="center"
      bg="transparent"
      border="~ rounded gray-200 dark:gray-700"
      outline="none active:none"
      @keydown.enter="conn ? sendMessage() : connect()"
    >
    <ul>
      <li v-for="(item, index) in messages" :key="index">
        {{ item }}
      </li>
    </ul>
    <button
      btn m-3 text-sm
      @click="startAudio"
    >
      {{ t('button.go') }}
    </button>
    <button
      btn m-3 text-sm
      @click="getVideoInfo"
    >
      getinfo
    </button>
    <video ref="video" controls />
    <!-- <p>
      <a rel="noreferrer" href="https://github.com/antfu/vitesse" target="_blank">
        Vitesse
      </a>
    </p>
    <p>
      <em text-sm opacity-75>{{ t('intro.desc') }}</em>
    </p>

    <div py-4 />

    <input
      id="input"
      v-model="name"
      :placeholder="t('intro.whats-your-name')"
      :aria-label="t('intro.whats-your-name')"
      type="text"
      autocomplete="false"
      p="x4 y2"
      w="250px"
      text="center"
      bg="transparent"
      border="~ rounded gray-200 dark:gray-700"
      outline="none active:none"
      @keydown.enter="go"
    >
    <label class="hidden" for="input">{{ t('intro.whats-your-name') }}</label>

    <div>
      <button
        btn m-3 text-sm
        :disabled="!name"
        @click="go"
      >
        {{ t('button.go') }}
      </button>
    </div> -->
  </div>
</template>

<route lang="yaml">
meta:
  layout: home
</route>
