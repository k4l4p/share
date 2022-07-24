<script setup>
import { ref, watch, watchEffect } from 'vue'

const phase = ref('')
const ice = ref(null)
const showice = ref(null)
const content = ref('')
const msg = ref(null)

let p
let pass = "i_am_an_id"
let set_connection
let recv_connection
let channel
const start = () => {
  phase.value = 'start'
  set_connection = new RTCPeerConnection()
  set_connection.onicecandidate = e => {
    console.log('new ice: ' + JSON.stringify(set_connection.localDescription))
    showice.value = JSON.stringify(set_connection.localDescription)
  }
  channel = set_connection.createDataChannel(pass)
  channel.onmessage = (msg) => {
      content.value = content.value + '\n' + msg.data
    }
  channel.onopen = e => {
    console.log('open!')
  }
  channel.onclose = e => {
    console.log('close!!')
  }
  set_connection.createOffer()
    .then(offer => set_connection.setLocalDescription(offer))

}

const recv = () => {
  phase.value = 'recv'
  recv_connection = new RTCPeerConnection()
  recv_connection.onicecandidate = e => {
    console.log('new ice: ' + JSON.stringify(recv_connection.localDescription))
    showice.value = JSON.stringify(recv_connection.localDescription)
  }
  recv_connection.ondatachannel = (e) => {
    channel = e.channel
    channel.onmessage = (msg) => {
      content.value = content.value + '\n' + msg.data
    }
    channel.onopen = () => console.log(channel.readyState)
    channel.onclose = () => console.log(channel.readyState)
  }
}

const submitIce = async () => {
  if (phase.value === 'start') {
    let temp = JSON.parse(ice.value)
    await set_connection.setRemoteDescription(temp)
  } else if (phase.value === 'recv') {
    let temp = JSON.parse(ice.value)
    console.log(temp)
    await recv_connection.setRemoteDescription(temp)
    await recv_connection.createAnswer().then((e) => {
      recv_connection.setLocalDescription(e)
    })
  }
}

const send = () => {
  channel.send(msg.value)
}

</script>

<template>
  <div class="container mx-auto h-screen">
    <div class="flex align-middle items-center justify-center h-full flex-col gap-2">
      <div class="flex flex-col">
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
        </div>
        <div class="flex bg-gray-50 shadow-md rounded-md p-4 my-2">
          <div class="flex flex-col gap-2 item-center">
            <button
              @click="start()"
              class="bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
              id="start"
            >Start</button>
            <button
              @click="recv"
              class="bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
              id="start"
            >Receive</button>
          </div>
          <textarea
            v-model="msg"
            rows="4"
            name="comment"
            id="comment"
            class="ml-2 shadow-sm focus:ring-indigo-500 focus:border-indigo-500 block w-96 sm:text-sm border-gray-300 rounded-md"
          />
        </div>
        <button
          @click="send"
          type="button"
          class="inline-flex items-center px-6 py-3 border border-transparent text-base font-medium rounded-md shadow-sm text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
        >Send</button>
        <div class="border border-sky-400">{{content}}</div>
      </div>
    </div>
  </div>
</template>