<script setup lang="ts">
let localConnection: RTCPeerConnection | null
let remoteConnection: RTCPeerConnection | null
let sendChannel: RTCDataChannel
let receiveChannel: RTCDataChannel
const startDisabled = ref(false)
// send
const sendDisabled = ref(true)
// stop
const closeDisabled = ref(true)
const sendTextData = ref('')
const receiveTextData = ref('')
/**
 * 获取对等连接对象的名称
 */
function getName(pc: RTCPeerConnection) {
  return (pc === localConnection) ? 'localPeerConnection' : 'remotePeerConnection'
}

function getOtherPc(pc: RTCPeerConnection) {
  return (pc === localConnection) ? remoteConnection : localConnection
}
/**
 * 获取本地对等连接对象
 */
function onAddIceCandidateSuccess() {
  console.warn('AddIceCandidate success.')
}
/**
 * 获取本地对等连接对象
 */
function onAddIceCandidateError(error: any) {
  console.warn(`Failed to add Ice Candidate: ${error.toString()}`)
}
/**
 * 获取本地对等连接对象
 * @param pc
 * @param event
 */
function onIceCandidate(pc: RTCPeerConnection, event: RTCPeerConnectionIceEvent) {
  getOtherPc(pc)!
    .addIceCandidate(event.candidate!)
    .then(
      onAddIceCandidateSuccess,
      onAddIceCandidateError,
    )
  console.warn(`${getName(pc)} ICE candidate: ${event.candidate ? event.candidate.candidate : '(null)'}`)
}

function onSendChannelStateChange() {
  const readyState = sendChannel.readyState
  console.warn(`Send channel state is: ${readyState}`)
  if (readyState === 'open') {
    sendDisabled.value = false
    // sendDocus()
  }
  else {
    sendDisabled.value = true
  }
}

function onReceiveMessageCallback(event: MessageEvent<any>) {
  console.warn('Received Message')
  receiveTextData.value = event.data
}

function onReceiveChannelStateChange() {
  const readyState = receiveChannel.readyState
  console.warn(`Receive channel state is: ${readyState}`)
}

function receiveChannelCallback(event: RTCDataChannelEvent) {
  console.warn('Receive Channel Callback')
  receiveChannel = event.channel
  receiveChannel.onmessage = onReceiveMessageCallback
  receiveChannel.onopen = onReceiveChannelStateChange
  receiveChannel.onclose = onReceiveChannelStateChange
}

function onCreateSessionDescriptionError(error: any) {
  console.warn(`Failed to create session description: ${error.toString()}`)
}

function gotDescription2(desc: RTCSessionDescriptionInit) {
  remoteConnection!.setLocalDescription(desc)
  console.warn(`Answer from remoteConnection\n${desc.sdp}`)
  localConnection!.setRemoteDescription(desc)
}

function gotDescription1(desc: RTCSessionDescriptionInit) {
  localConnection!.setLocalDescription(desc)
  console.warn(`Offer from localConnection\n${desc.sdp}`)
  remoteConnection!.setRemoteDescription(desc)
  remoteConnection!.createAnswer().then(
    gotDescription2,
    onCreateSessionDescriptionError,
  )
}

function createConnection() {
  const configuration = {
    iceServers: [{
      urls: 'stun:stun.miwifi.com',
    },
    ],
  }
  localConnection = new RTCPeerConnection(configuration)
  console.warn('localConnection: 创建了本地对等连接对象localConnection')

  sendChannel = localConnection.createDataChannel('sendDataChannel')
  console.warn('sendChannel: 创建了发送数据通道')
  localConnection.onicecandidate = (e) => {
    onIceCandidate(localConnection!, e)
  }
  sendChannel.onopen = onSendChannelStateChange
  sendChannel.onclose = onSendChannelStateChange
  // remote
  remoteConnection = new RTCPeerConnection(configuration)
  console.warn('Created remote peer connection object remoteConnection')

  remoteConnection.onicecandidate = (e) => {
    onIceCandidate(remoteConnection!, e)
  }
  remoteConnection.ondatachannel = receiveChannelCallback

  localConnection.createOffer().then(
    gotDescription1,
    onCreateSessionDescriptionError,
  )
  startDisabled.value = true
  closeDisabled.value = false
}
function sendData() {
  const data = sendTextData.value
  sendChannel.send(data)
  console.warn(`Sent Data: ${data}`)
}

function closeDataChannels() {
  console.warn('Closing data channels')
  sendChannel.close()
  console.warn(`Closed data channel with label: ${sendChannel.label}`)
  receiveChannel.close()
  console.warn(`Closed data channel with label: ${receiveChannel.label}`)
  localConnection?.close()
  remoteConnection?.close()
  localConnection = null
  remoteConnection = null
  console.warn('Closed peer connections')
  startDisabled.value = false
  sendDisabled.value = true
  closeDisabled.value = true
  sendTextData.value = ''
  receiveTextData.value = ''
}
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
