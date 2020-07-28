<template>
  <v-container>
    <v-row class="text-center">
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

    <v-row>
      <p>Your polynomial is:</p>
    </v-row>
    <v-row>
      <pre>{{ polynomial }}</pre>
    </v-row>
  </v-container>
</template>

<script>
import chart from "./chart.vue";
import * as tf from "@tensorflow/tfjs";

Number.prototype.map = function(in_min, in_max, out_min, out_max) {
  return ((this - in_min) * (out_max - out_min)) / (in_max - in_min) + out_min;
};

export default {
  name: "Calibration",
  components: { chart },
  data: () => ({
    tf: {
      optimiser: tf.train.sgd(0.5),
      polynomial: {
        a: tf.variable(tf.scalar(Math.random())),
        b: tf.variable(tf.scalar(Math.random())),
        c: tf.variable(tf.scalar(Math.random())),
      },
    },
    lineData: [],
    newSG: null,
    newAngle: null,
    headers: [
      { text: "Specific Gravity", value: "sg" },
      { text: "Angle", value: "angle" },
      { text: "Remove", value: "controls" },
    ],
    measurements: [
      { sg: 1, angle: 27.4 },
      { sg: 1.008, angle: 32 },
      { sg: 1.02, angle: 39 },
      { sg: 1.027, angle: 42.4 },
      { sg: 1.036, angle: 46.7 },
      { sg: 1.039, angle: 48.4 },
      { sg: 1.048, angle: 52 },
      { sg: 1.067, angle: 58.85 },
      { sg: 1.069, angle: 59.95 },
      { sg: 1.071, angle: 60.6 },
      { sg: 1.074, angle: 61.25 },
      { sg: 1.077, angle: 62.63 },
    ],
    chartOptions: {
      responsive: true,
      maintainAspectRatio: true,
      scales: {
        xAxes: [
          {
            ticks: {
              reverse: false,
            },
          },
        ],
      },
    },
    minX: null,
    maxX: null,
    minY: null,
    maxY: null,
  }),
  computed: {
    polynomial() {
      let ret = `${this.tf.polynomial.a.dataSync()}*(tilt)*(tilt)`;
      const b = this.tf.polynomial.b.dataSync();
      ret += b >= 0 ? "+" : "";
      ret += `${b}*(tilt)`;
      const c = this.tf.polynomial.c.dataSync();
      ret += c >= 0 ? "+" : "";
      ret += `${c}`;
      return ret;
    },
    chartData: function() {
      let ret = [];
      for (let i = 0; i < this.measurements.length; i++) {
        ret.push({ x: this.measurements[i].angle, y: this.measurements[i].sg });
      }
      return {
        labels: ["SG", "Angle"],
        datasets: [
          { label: "Measurements", backgroundColor: "black", data: ret },
          {
            label: "Regression",
            backgroundColor: "red",
            data: this.lineData,
            type: "line",
            fill: false,
            pointRadius: 0,
            showLine: true,
          },
        ],
      };
    },
  },
  mounted() {
    if (this.measurements.length > 0) {
      this.train();
    }
  },
  methods: {
    addMeasurement: function() {
      if (this.newSG == "" || this.newAngle == "") {
        return;
      }
      const newSG = parseFloat(this.newSG);
      const newAngle = parseFloat(this.newAngle);
      if (isNaN(newSG) || isNaN(newAngle)) {
        alert("One of those values is not a number.");
        return;
      }
      this.measurements.push({
        sg: newSG,
        angle: newAngle,
      });
      this.measurements.sort(function(a, b) {
        if (a.sg < b.sg) {
          return -1;
        }
        if (a.sg > b.sg) {
          return 1;
        }
        return 0;
      });
      this.newSG = null;
      this.newAngle = null;
      if (this.measurements.length > 0) {
        this.train();
      }
    },
    deleteMeasurement: function(item) {
      const i = this.measurements.indexOf(item);
      this.measurements.splice(i, 1);
      if (this.measurements.length > 0) {
        this.train();
      }
    },
    getXs: function() {
      const xs = [];
      const angles = [];
      for (let i = 0; i < this.measurements.length; i++) {
        angles.push(this.measurements[i].angle);
      }
      this.minX = Math.min(...angles);
      this.maxX = Math.max(...angles);
      for (let i = 0; i < angles.length; i++) {
        xs.push(angles[i].map(this.minX, this.maxX, -1, 1));
      }
      return xs;
    },
    getYs: function() {
      const ys = [];
      const sgs = [];
      for (let i = 0; i < this.measurements.length; i++) {
        sgs.push(this.measurements[i].sg);
      }
      this.minY = Math.min(...sgs);
      this.maxY = Math.max(...sgs);
      for (let i = 0; i < sgs.length; i++) {
        ys.push(sgs[i].map(this.minY, this.maxY, -1, 1));
      }
      return ys;
    },
    train: function() {
      const xs = tf.tensor1d(this.getXs());
      const ys = tf.tensor1d(this.getYs());
      for (let i = 0; i < 50; i++) {
        this.tf.optimiser.minimize(() => this.loss(this.predict(xs), ys));
      }
      this.setLineData();
    },
    predict: function(x) {
      return tf.tidy(() =>
        // x
        //   .pow(tf.scalar(3))
        //   .mul(this.tf.polynomial.a)
        //   .add(x.square().mul(this.tf.polynomial.b))
        //   .add(x.mul(this.tf.polynomial.c))
        //   .add(this.tf.polynomial.d)
        x
          .square()
          .mul(this.tf.polynomial.a)
          .add(x.mul(this.tf.polynomial.b))
          .add(this.tf.polynomial.c)
      );
    },
    loss: function(pred, expectation) {
      const ret = pred
        .sub(expectation)
        .square()
        .mean();
      return ret;
    },
    setLineData: function() {
      const xs = [];
      for (let x = this.minX; x <= this.maxX; x += 0.1) {
        xs.push(x.map(this.minX, this.maxX, -1, 1));
      }
      const ys = tf.tidy(() => this.predict(tf.tensor1d(xs)));
      const curveY = ys.dataSync();
      ys.dispose();
      this.lineData = [];
      let j = 0;
      for (let x = this.minX; x <= this.maxX; x += 0.1) {
        this.lineData.push({
          x: x,
          y: curveY[j].map(-1, 1, this.minY, this.maxY),
        });
        j++;
      }
    },
  },
};
</script>
