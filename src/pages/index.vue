<script setup lang="ts">
import type { DataConnection, Peer } from 'peerjs'
export interface ReceiveData {
  file: Blob
  type: string
  size: string
  filename: string
}

// const user = useUserStore()
// const name = $ref(user.savedName)
let peerId = $ref<string>('')
let isConnect = $ref<boolean>(false)
let destPeerId = $ref<string>('')
const video = ref<HTMLVideoElement | null>(null)
const file = ref<HTMLInputElement | null>(null)
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

const onData = (data: any) => {
  if (data === 'close stream') { stop() }
  else if (typeof data === 'object' && data.type) {
    const newData: ReceiveData = data
    const blob = new Blob([newData.file], { type: newData.type })
    const url = URL.createObjectURL(blob)
    if (newData.type.includes('image')) {
      messages.push(`<img src='${url}' w-250px inline-block />`)
    }
    else if (newData.type.includes('video')) {
      messages.push(`<video src='${url}' w-350px inline-block controls />`)
    }
    else {
      const a = document.createElement('a')
      a.href = url
      a.download = data.filename
      document.body.appendChild(a)
      a.click()
      a.remove()
      URL.revokeObjectURL(url)
      // console.log(url)
      messages.push(`[note]:accept new file ${data.filename} size ${data.size}`)
    }
  }
  else { messages.push(`other:${data}`) }
}

onMounted(async () => {
  if (typeof window !== 'undefined') {
    const { Peer } = await import('peerjs')
    peerId = `test_peer_${Date.now()}`
    messages.push(`[note]init local peerId ${peerId}`)
    peer = new Peer(peerId)
    peer?.on('connection', (remoteCon) => {
      conn = remoteCon
      destPeerId = remoteCon.peer
      isConnect = true
      remoteCon.on('data', (data: any) => {
        onData(data)
      })
      remoteCon.on('open', () => {
        messages.push('[note]connected!')
      })
    })
    peer?.on('disconnected', (currentId) => {
      isConnect = false
      messages.push('[note]disconnected')
    })
    peer?.on('close', () => {
      messages.push('close')
    })
    peer?.on('error', (err) => {
      console.log(err)

      messages.push(`[error]${err.message}`)
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
        console.log(`[error]remote stream err:${err.message}`)
      })
      call.on('close', () => {
        messages.push('[note]close stream!')
        if (video.value)
          video.value.srcObject = null
      })
    })
  }
  file.value?.addEventListener('change', (e) => {
    const target = e.target as HTMLInputElement
    const files = target.files

    if (!files)
      return
    const firstFile = files[0]
    const blob = new Blob([firstFile], { type: firstFile.type || 'application/octet-stream' })
    console.log(blob)

    let sOutput = `${firstFile.size} bytes`
    const nBytes = firstFile.size

    // optional code for multiples approximation
    const aMultiples = ['KiB', 'MiB', 'GiB', 'TiB', 'PiB', 'EiB', 'ZiB', 'YiB']
    for (let nMultiple = 0, nApprox = nBytes / 1024; nApprox > 1; nApprox /= 1024, nMultiple++)
      sOutput = `${nApprox.toFixed(3)} ${aMultiples[nMultiple]} (${nBytes} bytes)`

    conn?.send({
      file: blob,
      filename: firstFile.name,
      type: firstFile.type || 'application/octet-stream',
      size: sOutput,
    })
    if (file.value)
      file.value.value = ''
    messages.push(`[note]send file:name:${firstFile.name},size:${sOutput}`)
  })
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
  destPeerId = message
  message = ''
  conn?.on('open', () => {
    isConnect = true
    messages.push('[note]connected!')
  })
  conn?.on('data', (data: any) => {
    // Will print 'hi!'
    onData(data)
  })
}

const sendMessage = () => {
  console.log(message)
  if (message === 'close stream')
    stop()
  if (!message)
    return
  conn?.send(message)
  messages.push(`me:${message}`)
  message = ''
}

const startVideo = () => {
  console.log('startVideo', peerId, destPeerId)
  if (!destPeerId)
    return
  navigator.mediaDevices.getDisplayMedia(displayMediaOptions).then(
    (stream) => {
      console.log('local stream')
      if (!peer)
        return
      const call = peer.call(destPeerId, stream)
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
        console.log(`local stream error:${err}`)
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
const sendFile = () => {
  file.value?.click()
}

// const { t } = useI18n()
</script>

<template>
  <div>
    <div text-4xl>
      <div i-carbon-campsite inline-block />
    </div>
    <div>
      <div un-v-text-bottom inline-block :class="{ 'i-carbon-unlink': !isConnect, 'i-carbon-link': isConnect }" />
      {{ destPeerId }}
    </div>
    <!-- <div>
      <span>{{ peerid }}</span>
    </div> -->
    <input
      id="input"
      v-model="message"
      :placeholder="isConnect ? '发点啥' : '输入房间号'"
      :aria-label="isConnect ? '发点啥' : '输入房间号'"
      type="text"
      autocomplete="false"
      p="x4 y2"
      w="250px"
      text="center"
      bg="transparent"
      border="~ rounded gray-200 dark:gray-700"
      outline="none active:none"
    >
    <ul>
      <li v-for="(item, index) in messages" :key="index">
        <span v-html="item" />
      </li>
    </ul>
    <button
      v-if="isConnect" btn m-3
      text-sm
      @click="sendMessage"
    >
      send
    </button>
    <button
      v-else btn m-3
      text-sm
      @click="connect"
    >
      connect
    </button>
    <button
      v-if="isConnect"
      btn m-3
      text-sm
      @click="startVideo"
    >
      start video
    </button>
    <button
      v-if="isConnect"
      btn m-3
      text-sm
      @click="sendFile"
    >
      send file
    </button>
    <button
      btn m-3 text-sm
      @click="getVideoInfo"
    >
      getinfo
    </button>
    <div>
      <video ref="video" inline-block controls />
    </div>
    <input ref="file" type="file" display-none>
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
