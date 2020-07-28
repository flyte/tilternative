<template>
  <v-app>
    <v-app-bar
      app
      color="primary"
      dark
    >
      <div class="d-flex align-center">
        <h1>Floating Hydrometer Calibration</h1>
      </div>
      <v-spacer></v-spacer>
    </v-app-bar>

    <v-main>
      <Calibration/>
    </v-main>
  </v-app>
</template>

<script>
import Calibration from './components/Calibration';

export default {
  name: 'App',

  components: {
    Calibration,
  },

  data: () => ({
    //
  }),
};
</script>
