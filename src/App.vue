<template>
  <router-view />
  
</template>

<style>
* {
  padding: 0;
  margin: 0;
  font-family: Merriweather;
}
</style>

<script>
import swal from 'sweetalert'
export default {
  data() {
    return {
    }
  },
  mounted() {
    this.checkDevice()
    // alert(`tinggi: ${window.innerHeight} || lebar: ${window.innerWidth}`)
  },
  methods: {
    checkDevice() {
      const width = window.innerWidth
      if (width < 1000) {
        swal({
          icon: 'error',
          title: `Sorry, your device size does not support this website`
        })
      }
    }
  }
}
</script>