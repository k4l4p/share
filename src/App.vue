<script setup>
import { ref, watch, watchEffect } from 'vue'

let p 
let pass = "i_am_an_id"


let start = () => {

}

let recv = () => {

}


</script>

<template>

  <div class="container mx-auto">
    <button 
    @click="start()"
    class="bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
    id="start">
    Start
  </button>
  <button 
    @click="recv"
    class="bg-sky-500 hover:bg-sky-700 px-5 py-2 text-sm leading-5 rounded-full font-semibold text-white"
    id="start">
    Receive
  </button>
  </div>
  
</template>