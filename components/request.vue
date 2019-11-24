<template>
  <pw-section class="green" icon="send" ref="request">
    <h1> this is the new request section</h1>
  </pw-section>
</template>

<script>
export default {
  components: {
    "pw-section": () => import("./section")
  },
  data() {
    return { hello: "world" }
  }
}
</script>
