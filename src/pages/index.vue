<script setup lang="ts">
import { io } from 'socket.io-client'

const socket = io({
  path: '/socket.io',
})
function postMessage(data: any) {
  socket.emit('webrtc', data)
}
let pc: RTCPeerConnection | null
let sendChannel: RTCDataChannel | null
let receiveChannel: RTCDataChannel | null
const startDisabled = ref(true)
// send
const sendDisabled = ref(true)
// stop
const closeDisabled = ref(true)
const sendTextData = ref('')
const receiveTextData = ref('')
//
function onSendChannelStateChange() {
  const readyState = sendChannel!.readyState
  console.log(`发送通道状态 is: ${readyState}`)
  if (readyState === 'open') {
    sendDisabled.value = false
    closeDisabled.value = false
  }
  else {
    sendDisabled.value = true
    closeDisabled.value = true
  }
}

function onReceiveChannelStateChange() {
  const readyState = receiveChannel!.readyState
  console.log(`接收信息通道 状态 is: ${readyState}`)
  if (readyState === 'open') {
    sendDisabled.value = false
    closeDisabled.value = false
  }
  else {
    sendDisabled.value = true
    closeDisabled.value = true
  }
}

function onReceiveMessageCallback(event: MessageEvent<any>) {
  console.log('从机接收到信息', event.data)
  receiveTextData.value = event.data
}

function receiveChannelCallback(event: RTCDataChannelEvent) {
  console.log('从机设置接收到信并设置回调函数')
  receiveChannel = event.channel
  receiveChannel.onmessage = onReceiveMessageCallback
  receiveChannel.onopen = onReceiveChannelStateChange
  receiveChannel.onclose = onReceiveChannelStateChange
}

interface CandidateMessage {
  type: string
  data: RTCIceCandidateInit
}

function createPeerConnection() {
  pc = new RTCPeerConnection()
  pc.onicecandidate = (e) => {
    const message: CandidateMessage = {
      type: 'candidate',
      data: {
        candidate: '',
        sdpMid: null as string | null,
        sdpMLineIndex: null as number | null,
      },
    }
    if (e.candidate) {
      message.data.candidate = e.candidate.candidate
      message.data.sdpMid = e.candidate.sdpMid
      message.data.sdpMLineIndex = e.candidate.sdpMLineIndex
    }
    postMessage(message)
  }
}

function onSendChannelMessageCallback(event: any) {
  console.log('主机接收到信息', event.data)
  receiveTextData.value = event.data
}
/**
 * 创建主机
 */
async function createConnection() {
  const configuration = {
    iceServers: [
      {
        urls: 'stun:stun.miwifi.com',
      },
    ],
  }
  pc = new RTCPeerConnection(configuration)
  console.log('createConnection: 创建了本地对等连接对象pc')

  createPeerConnection()

  sendChannel = pc.createDataChannel('sendDataChannel')
  console.log('sendChannel: 创建了发送数据通道 [sendDataChannel]')
  sendChannel.onopen = onSendChannelStateChange
  sendChannel.onmessage = onSendChannelMessageCallback
  sendChannel.onclose = onSendChannelStateChange

  const offer = await pc.createOffer()
  // 主机发送信令
  postMessage({
    type: 'offer',
    sdp: offer.sdp,
  })
  // 将本地对等连接对象 pc 的本地描述设置为刚创建的 offer。本地描述包含了本地对等连接的配置信息和媒体流信息。
  await pc.setLocalDescription(offer)
  startDisabled.value = true
  closeDisabled.value = false
}
function sendData() {
  const data = sendTextData.value
  // 主机
  if (sendChannel) {
    sendChannel.send(data)
  }
  else {
    receiveChannel!.send(data)
  }
  console.log(`Sent Data: ${data}`)
}

async function hangup() {
  if (pc) {
    pc.close()
    pc = null
  }
  sendChannel = null
  receiveChannel = null
  console.log('Closed peer connections')
  startDisabled.value = false
  sendDisabled.value = true
  closeDisabled.value = true
  sendTextData.value = ''
  receiveTextData.value = ''
};
function closeDataChannels() {
  hangup()
  postMessage({ type: 'bye' })
}

async function handleCandidate(candidate: CandidateMessage) {
  console.log('handleCandidate')
  if (!pc) {
    console.error('无 对等连接')
    return
  }
  if (!candidate.data.candidate) {
    await pc.addIceCandidate(undefined)
  }
  else {
    await pc.addIceCandidate(candidate.data)
  }
}

// 收到主机信令
async function handleOffer(offer: RTCSessionDescriptionInit) {
  if (pc) {
    console.error('对等链接已存在')
  }
  else {
    await createPeerConnection()
    pc!.ondatachannel = receiveChannelCallback
    await pc!.setRemoteDescription(offer)
    const answer = await pc!.createAnswer()
    // 将自己的信令发送给主机
    postMessage({ type: 'answer', sdp: answer.sdp })
    await pc!.setLocalDescription(answer)
  }
}

async function handleAnswer(answer: RTCSessionDescriptionInit) {
  if (!pc) {
    console.error('对等链接不存在')
    return
  }
  await pc.setRemoteDescription(answer)
}

onMounted(() => {
  socket.on('webrtc', (data) => {
    console.log(data)
    switch (data.type) {
      // 第一个接收到信令
      case 'offer':
        handleOffer(data)
        break
      case 'answer':
        handleAnswer(data)
        break
        // 第二个接收到信令
      case 'candidate':
        handleCandidate(data)
        break
      case 'ready':
      // A second tab joined. This tab will enable the start button unless in a call already.
        if (pc) {
          console.log('already in call, ignoring')
          return
        }
        startDisabled.value = false
        break
      case 'bye':
        if (pc) {
          hangup()
        }
        break
      default:
        console.log('unhandled', data)
        break
    }
  })
  postMessage({ type: 'ready' })
  // signaling.onmessage = (e) => {
  //   console.log(e)
  //   switch (e.data.type) {
  //     // 第一个接收到信令
  //     case 'offer':
  //       handleOffer(e.data)
  //       break
  //     case 'answer':
  //       handleAnswer(e.data)
  //       break
  //       // 第二个接收到信令
  //     case 'candidate':
  //       handleCandidate(e.data)
  //       break
  //     case 'ready':
  //     // A second tab joined. This tab will enable the start button unless in a call already.
  //       if (pc) {
  //         console.log('already in call, ignoring')
  //         return
  //       }
  //       startDisabled.value = false
  //       break
  //     case 'bye':
  //       if (pc) {
  //         hangup()
  //       }
  //       break
  //     default:
  //       console.log('unhandled', e)
  //       break
  //   }
  // }
  // signaling.postMessage({ type: 'ready' })
})
</script>

<template>
  <div>
    <div>WebRTC</div>
    <div class="flex flex-col items-center">
      <ASpace>
        <AButton :disabled="startDisabled" @click="createConnection">
          Start
        </AButton>
        <AButton :disabled="sendDisabled" @click="sendData">
          Send
        </AButton>
        <AButton :disabled="closeDisabled" @click="closeDataChannels">
          Stop
        </AButton>
      </ASpace>
      <div class="w-900px flex justify-between gap-2">
        <div class="flex-1">
          <div>Send</div>
          <ATextarea v-model:model-value="sendTextData" :disabled="sendDisabled" />
        </div>
        <div class="flex-1">
          <div>Receive</div>
          <ATextarea v-model:model-value="receiveTextData" disabled />
        </div>
      </div>
    </div>
  </div>
</template>
