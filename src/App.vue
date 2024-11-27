<script setup>
import {onMounted, onUnmounted, ref} from "vue";

let blockUpdate = ref(false)
let imageModel = ref("")
let buttonDisabled = ref(false)
let updatingList = ref(false)
let imageList = ref([])
let imageCount = ref({cur: 0, total: 0})

const imageTimer = setInterval(async () => {
  if (blockUpdate.value) {
    return
  }

  blockUpdate.value = true
  const response = await fetch('/cgi-bin/live.cgi')
  if (response.ok) {
    const imageData = await response.blob()
    imageModel.value = URL.createObjectURL(imageData)
  }

  blockUpdate.value = false

}, 33)

async function takeSnapshot(event) {
  buttonDisabled.value = true;
  blockUpdate.value = true

  const response = await fetch('/cgi-bin/capture.cgi')
  if (response.ok) {
    const data = await response.json()
    const image_resp = await fetch(data.path)
    if (image_resp.ok) {
      const img_data = await image_resp.blob()
      imageList.value.unshift({
        uri: URL.createObjectURL(img_data),
        path: data.path
      })
    }
  }

  blockUpdate.value = false
  buttonDisabled.value = false;
}

async function updateImageList() {
  blockUpdate.value = true
  updatingList.value = true

  const response = await fetch('/cgi-bin/list.cgi')
  if (response.ok) {
    const data = await response.json()
    imageCount.value.total = data.items.length

    for (let i = 0; i < data.items.length; i++) {
      const path = data.items[i];
      const image_resp = await fetch(path)
      if (image_resp.ok) {
        const img_data = await image_resp.blob()
        imageList.value.unshift({
          uri: URL.createObjectURL(img_data),
          path: path
        })

        imageCount.value.cur = i
      }
    }
  }

  updatingList.value = false
  blockUpdate.value = false
}

onMounted(() => {
  updateImageList()
})

onUnmounted(() => {
  clearInterval(imageTimer)
})
</script>

<template>
  <div class="container">
    <div class="row">
      <div class="col-4">
        <div v-if="updatingList === true">
          <br/>
          <div class="alert alert-primary" role="alert">
            Updating image list, please wait...
            <br/>
            <br/>
            <div class="progress" role="progressbar" aria-label="load" style="height: 40px" aria-valuenow="{{ imageCount.cur }}" aria-valuemin="0" aria-valuemax="{{ imageCount.total }}">
              <div class="progress-bar" :style="'width:' + (imageCount.cur * 100 / imageCount.total) + '%'">
                {{ imageCount.cur }} / {{ imageCount.total }}
              </div>
            </div>
          </div>
        </div>
        <div class="card" v-else>
          <img :src="imageModel" alt="Live image" class="img-fluid"/>
          <div class="card-body">
            <h5 class="card-title">Live Preview</h5>
            <p class="card-text">Take a picture using the button below. The image will be stored on the external SPI
              flash.</p>
            <button type="button" @click="takeSnapshot" :class="'btn ' + (buttonDisabled ? 'btn-disabled' : 'btn-primary')" :disabled="buttonDisabled">
              <span v-if="buttonDisabled">
                <span class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
              <span class="visually-hidden">Loading...</span>
              </span>
              <span v-else>Shoot!</span>
            </button>
          </div>
        </div>
      </div>
      <div class="col-8">
        <div class="row overflow-auto" style="height: 90vh">
          <div class="col-4" v-for="image in imageList">
            <div class="card">
              <img :src="image.uri" :alt="image.path" class="card-img-top"/>
              <div class="card-body">
                <a class="btn btn-primary" target="_blank" :href="image.path">Open</a>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
