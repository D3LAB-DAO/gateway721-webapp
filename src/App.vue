<script>
import ChatBox from "./components/ChatBox.vue";

export default {
  data() {
    return {};
  },
  components: {
    ChatBox: ChatBox,
  },
  methods: {},
};
</script>

<template>
  <div class="black-bg"></div>
  <div class="container">
    <ChatBox id="chatbox" />
  </div>
</template>

<style scoped>
#chatbox {
  height: calc(100vh);
}

.black-bg {
  width: 100vw;
  height: 100vh;
  position: absolute;
  overflow: hidden;
  background: rgb(0, 0, 0);
  top: 0;
  left: 0;
  z-index: -2;
}
</style>
