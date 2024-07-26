<script>
import { thinkingMsg } from "../assets/chat.js";
// import { ContractInfo } from "../assets/chainInfo_testnet";
import { ContractInfo } from "../assets/chainInfo";

export default {
  name: "ChatBalloon",
  data() {
    return {
      thinkingMsg: thinkingMsg,
    };
  },
  computed: {
    explorerUrl() {
      return `https://www.mintscan.io/archway/address/${ContractInfo.contractAddr}`;
    },
  },
  props: ["type", "msg"],
};
</script>

<template>
  <div>
    <div class="chatballoon-container">
      <div
        class="chatballoon"
        :class="{
          agent: type === 'ai' || type === 'bubble',
          user: type === 'human',
        }"
      >
        <div v-if="type === 'bubble'" class="bubble">
          {{ thinkingMsg + " " }}
          <div class="ellipsis one"></div>
          <div class="ellipsis two"></div>
          <div class="ellipsis three"></div>
        </div>
        <img
          v-if="type === 'ai' || type === 'bubble'"
          class="profile"
          src="/profile.webp"
        />
        <span>{{ msg }}</span>
      </div>
    </div>
    <a
      class="explorer"
      v-if="type === 'ai'"
      :href="explorerUrl"
      target="_blank"
      rel="noopener noreferrer"
    >
      ↗ Explorer
    </a>
  </div>
</template>

<style scoped>
.chatballoon-container {
  padding-top: 30px;
  position: relative;
}
.chatballoon {
  font-size: 0.8rem;
  padding: 9px 20px 10px 20px;
  max-width: min(50%, 400px);
  position: relative;
  display: inline-block;
}
.agent {
  color: #000000ee;
  background-color: #ffffffcc;
  border-radius: 15px 15px 15px 0px;
  left: 65px;
}

.explorer {
  padding: 0px 20px 10px 20px;
  max-width: min(50%, 400px);
  position: relative;
  display: inline-block;
  font-size: 8pt;
  color: #ffffff;
  background-color: #ffffff00;
  left: 45px;
}

.user {
  color: #ffffffee;
  background-color: #FF481E;
  border-radius: 15px 15px 0px 15px;
  right: calc(-100% + 30px);
  transform: translate(-100%, 0%);
}

.profile {
  position: absolute;
  width: 35px;
  border-radius: 40%;
  left: -50px;
  bottom: -20px;
}
.ellipsis {
  width: 5px;
  height: 5px;
  display: inline-block;
  background: #00000077;
  border-radius: 50%;
  animation: bounce 1.3s linear infinite;
  margin: 1px;
}
.one {
  animation-delay: 0.6s;
}
.two {
  animation-delay: 0.5s;
}
.three {
  animation-delay: 0.8s;
}
@keyframes bounce {
  30% {
    transform: translateY(-2px);
  }
  60% {
    transform: translateY(0px);
  }
  80% {
    transform: translateY(2px);
  }
  100% {
    transform: translateY(0px);
    opacity: 0.5;
  }
}
</style>
