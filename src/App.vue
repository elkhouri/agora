<template>
  <Agora />
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";
import Agora from "./components/Agora.vue";

@Options({
  components: {
    Agora
  }
})
export default class App extends Vue {}
</script>
