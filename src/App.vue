<script setup>
import { onMounted, ref, watch, watchEffect } from 'vue'

const MAXIMUM_MESSAGE_SIZE = 65535;
const END_OF_FILE_MESSAGE = 'EOF';
const recv_buff = []

const phase = ref('')
const ice = ref(null)
const showice = ref(null)
const showDialog = ref(false)

const ws = new WebSocket('ws://localhost:8000')

ws.onopen = () => {
  console.log('opened, request key')
  ws.send(JSON.stringify({
    phase: 'req_key'
  }))
}

ws.onmessage = (msg) => {
  console.log(JSON.parse(msg.data))
}

ws.onclose = () => {
  ws.send(JSON.stringify({
    phase: 'close'
  }))
  console.log('close')
}

let pass = "i_am_an_id"
let set_connection
let recv_connection
let channel
let isNameSet = false

const ice_list = {
  "iceServers": [
    { "url": "stun:stun.l.google.com:19302" },
  ]
}

const start = () => {
  showDialog.value = true
  phase.value = 'start'
  set_connection = new RTCPeerConnection(ice_list)
  set_connection.onicecandidate = e => {
    showice.value = JSON.stringify(set_connection.localDescription)
    let msg = {
      phase: 'start',
      ice: set_connection.localDescription
    }
    ws.send(JSON.stringify(msg))
  }
  channel = set_connection.createDataChannel(pass)
  // channel.binaryType = "arraybuffer"
  channel.onopen = e => {
    let fileElem = document.getElementById("fileElem")
    fileElem.click()
    console.log('open!')
  }
  channel.onclose = e => {
    console.log('close!!')
  }
  set_connection.createOffer()
    .then(offer => set_connection.setLocalDescription(offer))

}

const recv = () => {
  showDialog.value = true
  phase.value = 'recv'
  recv_connection = new RTCPeerConnection(ice_list)
  recv_connection.onicecandidate = e => {
    showice.value = JSON.stringify(recv_connection.localDescription)
  }
  recv_connection.ondatachannel = (e) => {
    channel = e.channel
    // channel.binaryType = 'blob'
    channel.onmessage = (msg) => {
      try {
        if (isNameSet) {
          if (msg.data !== END_OF_FILE_MESSAGE) {
            recv_buff.push(msg.data)
          } else {
            // convert array of arrayBuffer to one arraybuffer
            const arrayBuffer = recv_buff.reduce((acc, arrayBuffer) => {
              const tmp = new Uint8Array(acc.byteLength + arrayBuffer.byteLength);
              tmp.set(new Uint8Array(acc), 0);
              tmp.set(new Uint8Array(arrayBuffer), acc.byteLength);
              return tmp;
            }, new Uint8Array());
            const blob = new Blob([arrayBuffer]);
            const a = document.createElement('a');
            const url = window.URL.createObjectURL(blob);
            a.href = url;
            a.download = channel.filename;
            a.click();
            window.URL.revokeObjectURL(url);
            a.remove()
            channel.close();
          }
        } else {
          //dirty way to save filename
          channel.filename = msg.data
          isNameSet = true
        }

      } catch (e) {
        console.log('Cannot transmit!')
      }


    }
    channel.onopen = () => console.log(channel.readyState)
    channel.onclose = () => console.log(channel.readyState)
  }
}

const submitIce = async () => {
  if (phase.value === 'start') {
    console.log(channel.negotiated)

    let temp = JSON.parse(ice.value)
    await set_connection.setRemoteDescription(temp)
  } else if (phase.value === 'recv') {
    let temp = JSON.parse(ice.value)
    await recv_connection.setRemoteDescription(temp)
    await recv_connection.createAnswer().then((e) => {
      recv_connection.setLocalDescription(e)
    })
  }
}

const select = async (e) => {
  let file = e.currentTarget.files[0]
  const arrayBuffer = await file.arrayBuffer()
  channel.send(file.name)
  for (let i = 0; i < arrayBuffer.byteLength; i += MAXIMUM_MESSAGE_SIZE) {
    channel.send(arrayBuffer.slice(i, i + MAXIMUM_MESSAGE_SIZE));
    // channel.send(file.slice(i, i + MAXIMUM_MESSAGE_SIZE))
  }
  channel.send(END_OF_FILE_MESSAGE);
}

</script>

<template>
  <div class="container mx-auto h-screen">
    <div class="relative w-full h-full">
      <div
        :class="[showDialog ? 'max-h-screen' : 'max-h-24']"
        class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 transition-all duration-500 ease-out overflow-hidden w-full"
      >
        <div class="flex w-full justify-center">
          <div
            @click="start"
            :class="[phase === 'recv' ? 'pointer-events-none opacity-70' : '']"
            class="rounded-full bg-indigo-400 shadow-lg h-20 w-20 flex justify-center items-center cursor-pointer hover:bg-sky-600 transition-all duration-100 ease-out"
          >
            <h1 class="text-white font-bold text-xl">Send</h1>
          </div>
          <div
            @click="recv"
            :class="[phase === 'start' ? 'pointer-events-none opacity-70' : '']"
            class="ml-2 rounded-full bg-indigo-400 shadow-lg h-20 w-20 flex justify-center items-center cursor-pointer hover:bg-sky-600 transition-all duration-100 ease-out"
          >
            <h1 class="text-white font-bold text-xl">Receive</h1>
          </div>
        </div>
        <div class="mt-5 w-full">
          <div class="p-4 my-2 bg-purple-100">
            <h1 class="text-lg font-medium text-indigo-600">Show ICE</h1>
            <textarea
              v-model="showice"
              rows="4"
              name="ice"
              id="ice"
              class="focus:ring-indigo-500 focus:border-indigo-500 block w-full sm:text-sm border-gray-300 rounded-md"
            />
            <h1 class="text-lg font-medium text-indigo-600">Input ICE</h1>
            <textarea
              v-model="ice"
              rows="4"
              name="ice"
              id="ice"
              class="focus:ring-indigo-500 focus:border-indigo-500 block w-full sm:text-sm border-gray-300 rounded-md"
            />
            <button
              @click="submitIce()"
              class="bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
              id="start"
            >Submit</button>
            <input type="file" id="fileElem" style="display:none" @change="select" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>