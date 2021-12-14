<template>
<div class="m-5">
  <h1>Hi, Nano!</h1>

  <!-- input options -->
  <div id="input-opts" class="d-flex flex-wrap mb-3 align-items-center">
    <!-- select file -->
    <input type="file" id="file" accept="image/*" @change="loadFile">

    <template v-if="fileLoaded">
      <!-- number of colors -->
      <div class="input-num-colors mr-5 d-flex align-items-center">
        <label for="num-colors" class="flex-shrink-0 mr-2">Number of colors</label>
        <b-form-input id="num-colors" v-model="numMatches" type="number" min="1" placeholder="Number of colors"></b-form-input>
      </div>

      <!-- execute -->
      <b-button @click="matchColors">Match colors</b-button>
    </template>
  </div>

  <!-- Progress bar -->
  <div class="w-100 m-auto">

    <div class="d-flex flex-column arrange-items-center w-50" v-if="isMatching">
      <small class="text-muted">Percent complete</small>
      <b-progress :value="matchProgress" max="1" show-progress animated></b-progress>
    </div>
  </div>

  <!-- image preview -->
  <div id="image-preview">
    <h6>original image</h6>
    <canvas id="canvas"></canvas>
  </div>


  <!-- results -->
  <div id="results" v-if="filteredMatches.length">
    <table>
      <thead>
        <tr class="font-weight-bold text-left">
          <td>

          </td>
          <td>
            DMC code
          </td>
          <td>
            name
          </td>
          <td>
            Percent of image
          </td>
          <td>
            Avg. score (lower is a closer match)
          </td>
        </tr>
      </thead>

      <tbody>
        <tr v-for="(color, idx) in filteredMatches" :key="idx" class="text-left">
          <td :style="{width: '50px', height: '25px', background: color.dmc_hex, border: '4px solid white'}">
          </td>
          <td>
            {{color.dmc_id}}
          </td>
          <td>
            {{color.dmc_name}}
          </td>
          <td>
            {{color.pct}}
          </td>
          <td>
            {{color.score}}
          </td>
        </tr>
      </tbody>

    </table>
  </div>

</div>
</template>

<script>
import {
  deltaE
} from "chroma-js";
import chroma from 'chroma-js';
import DMC from "@/assets/dmc_colors.json";
import chunk from "lodash/chunk";
import sumBy from "lodash/sumBy";
import countBy from "lodash/countBy";
import groupBy from "lodash/groupBy";
import * as _ from "lodash";
import * as d3 from "d3-format";

export default {
  name: 'ImportPic',
  data() {
    return {
      imageWidth: 80,
      imageHeight: 80,
      matchedColors: [],
      matchProgress: 0,
      isMatching: false,
      fileLoaded: false,
      numMatches: 10,
      roundedRGBA: [],
      imagePixels: []
    }
  },
  computed: {
    filteredMatches() {
      return (this.matchedColors.slice(0, this.numMatches))
    }
  },
  mounted() {
    DMC.forEach(d => {
      d["color"] = chroma(`rgb(${d.r}, ${d.g}, ${d.b})`);
      d["hex"] = d.color.hex();
    })
  },
  methods: {
    loadFile(event) {
      // resetting the values on new file load.
      this.isMatching = false;
      this.matchProgress = 0;
      this.matchedColors = [];
      this.fileLoaded = false;

      var canvas = document.getElementById('canvas'); // load context of canvas
      var ctx = canvas.getContext('2d'); // load context of canvas
      var img = new Image();
      img.src = URL.createObjectURL(event.target.files[0]); // use first selected image from input element

      img.onload = (e) => {
        this.imageWidth = img.width * 0.5;
        this.imageHeight = img.height * 0.5;
        canvas.width = this.imageWidth;
        canvas.height = this.imageHeight;

        ctx.drawImage(img, 0, 0, this.imageWidth, this.imageHeight); // draw the image to the canvas
        let imgData = ctx.getImageData(0, 0, this.imageWidth, this.imageHeight);

        // split into RGBA values
        let pixels = chunk(imgData.data, 4);

        // round the RGB values to the nearest 5 units, to reduce the number of duplicate calculations to make.
        this.roundedRGBA = pixels.map(d => this.roundRGBA(d));

        // count the number of occurrences of RGBA values, to reduce to single values to compare.
        let chunkCount = countBy(this.roundedRGBA);

        // convert from an object to an array
        this.imagePixels = Object.keys(chunkCount).map(key => ({
          id: key,
          count: chunkCount[key]
        }));

        // sort high to low
        this.imagePixels.sort((a, b) => b.count - a.count);

        // allow color matching to happen
        this.fileLoaded = true;
      }
    },
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
      this.isMatching = true;

      const imgDMC = this.imagePixels.map((d, i) => {
        this.matchProgress = i / this.imagePixels.length;

        if (i % 1000 === 0) {
          console.log(this.matchProgress)
        }

        let obj = {};
        const rgba = d.id.split(",")
        obj["color"] = chroma(`rgba(${rgba[0]}, ${rgba[1]}, ${rgba[2]}, ${rgba[3]/255})`);
        const closest = this.calcDist(obj.color);
        obj["hex"] = obj.color.hex();
        obj["pixel_count"] = d.count;
        obj["closest_hex"] = closest.hex;
        obj["closest_name"] = closest.name;
        obj["closest_dmc_id"] = closest.floss;
        obj["closest_score"] = closest.dist;
        return obj;
      })

      // sum number of occurrences of the DMC color
      this.matchedColors = _(imgDMC).groupBy("closest_dmc_id")
        .map((values, i) => ({
          values: values,
          dmc_id: values[0]["closest_dmc_id"],
          dmc_name: values[0]["closest_name"],
          dmc_hex: values[0]["closest_hex"],
          count: sumBy(values, "pixel_count"),
          score: d3.format("0.2f")(_.meanBy(values, "closest_score")),
          pct: d3.format("0.1%")(sumBy(values, "pixel_count") / this.roundedRGBA.length)
        }))
        .value()

      // sort descendingly by count
      this.matchedColors.sort((a, b) => b.count - a.count);
      console.log(this.matchedColors)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
td {
    padding: 0.25rem 0.5rem;
}

.input-num-colors {
    width: 225px;
}
</style>
