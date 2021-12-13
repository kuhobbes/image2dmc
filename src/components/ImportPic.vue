<template>
<div>
  <h1>Hi, Nano!</h1>

  <input type="file" id="file" accept="image/*" @change="loadFile">

  <canvas :width="imageWidth" :height="imageHeight" id="canvas"></canvas>

</div>
</template>

<script>
import { deltaE } from "chroma-js";

export default {
  name: 'ImportPic',
  data() {
    return {
      imageWidth: 820,
      imageHeight: 1005
    }
  },
  mounted() {
    console.log(deltaE)
  },
  methods: {
    loadFile(event) {
      var ctx = document.getElementById('canvas').getContext('2d'); // load context of canvas
      var img = new Image();
      img.src = URL.createObjectURL(event.target.files[0]); // use first selected image from input element
      img.onload = (e) => {
        // this.imageWidth = img.width;
        // this.imageHeight = img.height;

        ctx.drawImage(img, 0, 0); // draw the image to the canvas
        let imgData = ctx.getImageData(0, 0, this.imageWidth, this.imageHeight);
        console.log(imgData)
        console.log(imgData.data)
        console.log(this.imageWidth)
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
</style>
