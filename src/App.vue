<script setup>
import { onMounted, ref, watch, watchEffect } from 'vue'

const MAXIMUM_MESSAGE_SIZE = 65535;
const END_OF_FILE_MESSAGE = 'EOF';
const recv_buff = []

const side = ref('')
const ID = ref(null)
const showID = ref(null)
const showDialog = ref(false)

const ws = new WebSocket('ws://192.168.128.87:8000')

ws.onopen = () => {
  console.log('opened')
}

ws.onmessage = (msg) => {
  let data = JSON.parse(msg.data)
  console.log(data)
  switch (side.value) {
    case 'start':
      if (data.action === 'req_key') {
        showID.value = data.key
      }
      if (data.action === 'send_decr') {
        set_connection.setRemoteDescription(data.decr)
        ws.send(JSON.stringify({
          side: 'start',
          action: 'get_ice'
        }))
      }
      if (data.action === 'send_ice') {
        set_connection.addIceCandidate(data.ice)
      }

      break
    case 'recv':
      if (data.action === 'send_decr') {
        let temp = data.decr
        recv_connection.setRemoteDescription(temp).then(() => {
          recv_connection.createAnswer().then((e) => {
            recv_connection.setLocalDescription(e).then(() => {
              ws.send(JSON.stringify({
                side: 'recv',
                action: 'send_decr',
                key: ID.value,
                decr: recv_connection.localDescription
              }))
            })
          })
        })
      }
      if (data.action === 'send_ice') {
        recv_connection.addIceCandidate(data.ice)
      }
      break
    default:
      break
  }
}

ws.onclose = () => {
  if (side.value === 'start') {
    ws.send(JSON.stringify({
      action: 'close'
    }))
  }
  console.log('close')
}

window.onbeforeunload = function () {
  ws.close();
};


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
  side.value = 'start'
  ws.send(JSON.stringify({
    side: 'start',
    action: 'req_key'
  }))
  set_connection = new RTCPeerConnection(ice_list)
  set_connection.onicecandidate = e => {
    console.log('start send ice')
    let msg = {
      side: 'start',
      action: 'send_ice',
      ice: e.candidate
    }
    ws.send(JSON.stringify(msg))
  }


  channel = set_connection.createDataChannel(pass)
  channel.onopen = e => {
    console.log('channel open!')
    document.getElementById("fileElem").click()
  }
  channel.onclose = e => {
    console.log('close!!')
  }
  set_connection.createOffer()
    .then(offer => set_connection.setLocalDescription(offer).then(() => {
      let msg = {
        side: 'start',
        action: 'send_decr',
        decr: set_connection.localDescription
      }
      ws.send(JSON.stringify(msg))
    }))

}

const recv = () => {
  showDialog.value = true
  side.value = 'recv'
  recv_connection = new RTCPeerConnection(ice_list)
  recv_connection.onicecandidate = e => {
    console.log('recv send ice')
    ws.send(JSON.stringify({
      side: 'recv',
      action: 'send_ice',
      key: ID.value,
      ice: e.candidate
    }))
  }
  recv_connection.ondatachannel = (e) => {
    channel = e.channel
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
  if (ID.value) {
    ws.send(JSON.stringify({
      side: side.value,
      action: 'send_key',
      key: ID.value
    }))
  }
}

const select = async (e) => {
  let file = e.currentTarget.files[0]
  const arrayBuffer = await file.arrayBuffer()
  channel.send(file.name)
  for (let i = 0; i < arrayBuffer.byteLength; i += MAXIMUM_MESSAGE_SIZE) {
    channel.send(arrayBuffer.slice(i, i + MAXIMUM_MESSAGE_SIZE));
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
            :class="[side === 'recv' ? 'pointer-events-none opacity-70' : '']"
            class="rounded-full bg-indigo-400 shadow-lg h-20 w-20 flex justify-center items-center cursor-pointer hover:bg-sky-600 transition-all duration-100 ease-out"
          >
            <h1 class="text-white font-bold text-xl">Send</h1>
          </div>
          <div
            @click="recv"
            :class="[side === 'start' ? 'pointer-events-none opacity-70' : '']"
            class="ml-2 rounded-full bg-indigo-400 shadow-lg h-20 w-20 flex justify-center items-center cursor-pointer hover:bg-sky-600 transition-all duration-100 ease-out"
          >
            <h1 class="text-white font-bold text-xl">Receive</h1>
          </div>
        </div>
        <div class="mt-5 flex justify-center">
          <div class="p-4 my-2 bg-indigo-100 rounded-lg ">
            <div v-if="side === 'start'">
              <label for="email" class="block text-sm font-medium text-gray-700">ID</label>
              <div class="mt-1">
                <input
                  v-model="showID"
                  type="text"
                  name="showID"
                  id="showID"
                  readonly
                  class="shadow-sm focus:ring-indigo-500 focus:border-indigo-500 block w-full sm:text-sm border-gray-300 rounded-md"
                />
                 <input type="file" id="fileElem" hidden @change="select" />
              </div>
            </div>
            <div v-else-if="side === 'recv'">
              <label for="email" class="block text-sm font-medium text-gray-700">ID</label>
              <div class="mt-1">
                <input
                  v-model="ID"
                  type="text"
                  name="ID"
                  id="ID"
                  class="shadow-sm focus:ring-indigo-500 focus:border-indigo-500 block w-full sm:text-sm border-gray-300 rounded-md"
                />
              </div>
              <button
              @click="submitIce()"
              class="mt-2 bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
              id="start"
            >Submit</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>