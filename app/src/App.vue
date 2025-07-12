<template>
  <div id="app">
    <QRCodeGenerator />
  </div>
</template>

<script>
import QRCodeGenerator from './components/QRCodeGenerator.vue'

export default {
  name: 'App',
  components: {
    QRCodeGenerator
  }
}
</script>