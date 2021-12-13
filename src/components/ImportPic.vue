<template>
<div>
  <h1>Hi, Nano!</h1>

  <input type="file" id="file" accept="image/*" @change="loadFile">
  <div v-for="(color, idx) in matchedColors" :key="idx">
    <template v-if="idx < maxNum">
      {{color}}
    </template>

  </div>
  <button @click="matchColors">Match colors</button>

  <canvas id="canvas"></canvas>

</div>
</template>

<script>
import {
  deltaE
} from "chroma-js";
import chroma from 'chroma-js';
import DMC from "@/assets/dmc_colors.json";
import chunk from "lodash/chunk";
import countBy from "lodash/countBy";

export default {
  name: 'ImportPic',
  data() {
    return {
      imageWidth: 80,
      imageHeight: 80,
      matchedColors: [],
      maxNum: 10,
      roundedRGBA: [],
      chunks: []
      // imageWidth: 820,
      // imageHeight: 1005
    }
  },
  mounted() {
    DMC.forEach(d => {
      d["color"] = chroma(`rgb(${d.r}, ${d.g}, ${d.b})`);
      d["hex"] = d.color.hex();
    })
  },
  methods: {
    roundRGBA(arr) {
      return (`${this.round(arr[0])},${this.round(arr[1])},${this.round(arr[2])},${this.round(arr[3])/255}`)
    },
    round(value, precision = 5) {
      return Math.ceil(value / precision) * precision;
    },
    calcDist(color) {
      DMC.forEach(d => {
        d.dist = deltaE(color, d.color);
      })

      DMC.sort((a, b) => a.dist - b.dist);

      const closestColor = DMC[0];
      return (closestColor)
    },
    matchColors() {
      const imgDMC = this.chunks.map((d, i) => {
        if (i % 1000 === 0) {
          console.log(i / this.chunks.length)
        }
        let obj = {};
        const rgba = d.id.split(",")
        obj["color"] = chroma(`rgba(${rgba[0]}, ${rgba[1]}, ${rgba[2]}, ${rgba[3]/255})`);
        const closest = this.calcDist(obj.color);
        obj["hex"] = obj.color.hex();
        obj["color_count"] = d.count;
        obj["closest_hex"] = closest.hex;
        obj["closest_name"] = closest.name;
        obj["closest_dmc_id"] = closest.floss;
        return obj;
      })

      const dmcObj = countBy(imgDMC, "closest_name");
      this.matchedColors = Object.keys(dmcObj).map(key => ({
        id: key,
        count: dmcObj[key]
      }));
      this.matchedColors.sort((a, b) => b.count - a.count);
      // console.log(imgDMC)
      console.log(this.matchedColors)
    },
    loadFile(event) {
      var canvas = document.getElementById('canvas'); // load context of canvas
      var ctx = canvas.getContext('2d'); // load context of canvas
      var img = new Image();
      img.src = URL.createObjectURL(event.target.files[0]); // use first selected image from input element

      img.onload = (e) => {
        this.imageWidth = img.width;
        this.imageHeight = img.height;
        canvas.width = this.imageWidth;
        canvas.height = this.imageHeight;

        ctx.drawImage(img, 0, 0); // draw the image to the canvas
        let imgData = ctx.getImageData(0, 0, this.imageWidth, this.imageHeight);

        // split into RGBA values
        let chunks = chunk(imgData.data, 4);

        // round the RGB values to the nearest 5 units, to reduce the number of duplicate calculations to make.
        this.roundedRGBA = chunks.map(d => this.roundRGBA(d));

        // count the number of occurrences of RGBA values, to reduce to single values to compare.
        let chunkCount = countBy(this.roundedRGBA);

        // convert from an object to an array
        this.chunks = Object.keys(chunkCount).map(key => ({
          id: key,
          count: chunkCount[key]
        }));

        // sort high to low
        this.chunks.sort((a, b) => b.count - a.count);
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
</style>
