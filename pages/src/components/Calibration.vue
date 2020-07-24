<template>
  <v-container>
    <v-row class="text-center">
      <v-col cols="12">
        <v-img
          :src="require('../assets/logo.svg')"
          class="my-3"
          contain
          height="200"
        />
      </v-col>

      <v-col class="mb-4">
        <v-data-table
          :headers="headers"
          :items="measurements"
          disable-pagination
          hide-default-footer
        >
          <template v-slot:item.controls="props">
            <v-btn small fab @click="deleteMeasurement(props.item)">
              <v-icon dark>mdi-delete</v-icon>
            </v-btn>
          </template>
        </v-data-table>
        <v-text-field
          label="SG"
          placeholder="1.000"
          v-model="newSG"
          v-on:keyup.enter="addMeasurement"
        >
        </v-text-field>
        <v-text-field
          label="Angle"
          placeholder="45"
          v-model="newAngle"
          v-on:keyup.enter="addMeasurement"
        >
        </v-text-field>
        <v-btn @click="addMeasurement">Add</v-btn>
      </v-col>

      <v-col>
        <chart :chartData="chartData" :options="chartOptions" />
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import chart from "./chart.vue";

export default {
  name: "Calibration",
  components: { chart },
  data: () => ({
    newSG: null,
    newAngle: null,
    headers: [
      { text: "Specific Gravity", value: "sg" },
      { text: "Angle", value: "angle" },
      { text: "Remove", value: "controls" },
    ],
    measurements: [
      { sg: 1.01, angle: 69 },
      { sg: 1.02, angle: 67 },
      { sg: 1.03, angle: 65 },
      { sg: 1.04, angle: 60 },
      { sg: 1.05, angle: 56 },
      { sg: 1.07, angle: 50 },
    ],
    chartOptions: {
      responsive: true,
      maintainAspectRatio: true,
      scales: {
        xAxes: [
          {
            ticks: {
              reverse: true,
            },
          },
        ],
      },
    },
  }),
  computed: {
    chartData: function() {
      let ret = [];
      for (let i = 0; i < this.measurements.length; i++) {
        ret.push({ x: this.measurements[i].angle, y: this.measurements[i].sg });
      }
      return {
        labels: ["SG", "Angle"],
        datasets: [
          { label: "Measurements", backgroundColor: "black", data: ret },
        ],
      };
    },
  },
  methods: {
    addMeasurement: function() {
      if (this.newSG == "" || this.newAngle == "") {
        return;
      }
      this.measurements.push({
        sg: parseFloat(this.newSG),
        angle: parseFloat(this.newAngle),
      });
      console.log(
        this.measurements.sort(function(a, b) {
          if (a.sg < b.sg) {
            return -1;
          }
          if (a.sg > b.sg) {
            return 1;
          }
          return 0;
        })
      );
      this.newSG = null;
      this.newAngle = null;
    },
    deleteMeasurement: function(item) {
      const i = this.measurements.indexOf(item);
      this.measurements.splice(i, 1);
    },
  },
};
</script>
