<script>
import axios from "axios";
import ChatBalloon from "./ChatBalloon.vue";
import { getChatArray, disclaimerText } from "../assets/chat.js";
import { ConstantineInfo, ContractInfo } from "../assets/chainInfo";
import { SigningCosmWasmClient } from "@cosmjs/cosmwasm-stargate";
import { calculateFee, GasPrice } from "@cosmjs/stargate";

export default {
  name: "ChatBox",
  data() {
    return {
      chatArray: getChatArray(),
      prompt: "",
      inputText: "",
      typing: false,
      keplrAddress: null, // Store Keplr wallet address
      keplrClient: null, // Store Keplr client
      gasPrice: GasPrice.fromString(
        "0" + ConstantineInfo.currencies[0].coinMinimalDenom
      ),
      // gasPrice: GasPrice.fromString("0.05const"), // TODO: premium
      selected: null,
      options: [
        { value: "Create", name: "Create" },
        { value: "0", name: "0" },
        { value: "1", name: "1" },
      ],
    };
  },
  mounted() {
    this.loadOnMounted();
  },
  computed: {
    placeholderText() {
      return this.keplrAddress
        ? "Input your JavaScript code to create project ..."
        : "Connect wallet first ...";
    },
    inputDisabling() {
      return this.keplrAddress ? false : true;
    },
    connectButtonClass() {
      return this.keplrAddress ? "connected" : "";
    },
  },
  components: {
    ChatBalloon: ChatBalloon,
  },
  methods: {
    async connectKeplr() {
      if (!window.keplr) {
        throw new Error("Please install keplr extension");
      }
      try {
        await window.keplr.experimentalSuggestChain(ConstantineInfo);
      } catch (error) {
        alert("Failed to suggest the chain");
      }

      await window.keplr.enable(ConstantineInfo.chainId);
      const offlineSigner = await window.getOfflineSigner(
        ConstantineInfo.chainId
      );
      const accounts = await offlineSigner.getAccounts();
      const client = await SigningCosmWasmClient.connectWithSigner(
        ConstantineInfo.rpc,
        offlineSigner,
        {
          gasPrice: this.gasPrice,
        }
      );

      this.keplrAddress = accounts[0].address;
      this.keplrClient = client;
    },
    formatAddress(address) {
      return `${address.slice(0, 11)}...${address.slice(-4)}`;
    },

    async getPrompt(client) {
      const queryResult = await this.keplrClient.queryContractSmart(
        ContractInfo.contractAddr,
        {
          prompt: {},
        }
      );
      return queryResult ? queryResult : null;
    },
    async getNftInfo(token_id) {
      const queryResult = await this.keplrClient.queryContractSmart(
        ContractInfo.contractAddr,
        {
          nft_info: { token_id },
        }
      );
      return queryResult ? queryResult : null;
    },
    async mintNft(input) {
      let executeFee = calculateFee(300_000, this.gasPrice);
      console.log("executeFee:", executeFee);

      const msg = {
        mint: {
          token_id: "0", // has no meaning
          owner: this.keplrAddress,
          extension: {
            description: input,
          },
        },
      };
      const executeResult = await this.keplrClient.execute(
        this.keplrAddress,
        ContractInfo.contractAddr,
        msg,
        executeFee
      );
      return executeResult ? executeResult : null;
    },

    async loadOnMounted() {
      this.scrollToBottom(true);
    },
    async send(msg) {
      if (this.keplrAddress === null) return;
      if (msg === "") return;
      this.inputText = "";
      const tokenId = await this.pushMsg(msg);
      this.scrollToBottom(true);
      await this.typingMsg();
      this.scrollToBottom(true);
      await this.postMsg(tokenId, msg);
      this.scrollToBottom(true);
    },
    async pushMsg(msg) {
      this.chatArray.push({
        type: "human",
        data: {
          content: msg,
          additional_kwargs: {},
          example: false,
        },
      });
      // TODO: error catch
      const execRes = await this.mintNft(msg);
      const tokenId = execRes.logs[0].events
        .find((e) => e.type === "wasm")
        .attributes.find((attr) => attr.key === "token_id").value;
      console.log(`mint tx for ${tokenId}: ${execRes.transactionHash}`);
      return tokenId;
    },
    async typingMsg() {
      this.typing = true;
    },
    async postMsg(tokenId, msg) {
      let nftInfo;
      let words;

      const fetchNftInfo = (tokenId) => {
        const maxRetryTime = 60000; // Maximum retry time (60 seconds)
        const retryDelay = 2000; // Delay between retries (e.g., 2 seconds)
        const startTime = Date.now();

        return new Promise(async (resolve, reject) => {
          const attemptFetch = async () => {
            try {
              nftInfo = await this.getNftInfo(tokenId);
              if (nftInfo.extension.image !== null) {
                resolve(nftInfo);
              } else {
                const elapsedTime = Date.now() - startTime;
                if (elapsedTime < maxRetryTime) {
                  setTimeout(attemptFetch, retryDelay);
                } else {
                  reject(
                    new Error(
                      "Timeout: NFT info image is still null after 60 seconds."
                    )
                  );
                }
              }
            } catch (error) {
              console.error(error);
              reject(error);
            }
          };
          attemptFetch();
        });
      };

      try {
        nftInfo = await fetchNftInfo(tokenId);
        words = nftInfo.extension.image;
      } catch (error) {
        console.error(error);
        words = "Meow! Timeout. Try again, please.";
      }

      this.typing = false;
      const index = this.chatArray.push({
        type: "ai",
        data: {
          content: words,
          additional_kwargs: {},
          example: false,
        },
      });
      await this.typeWriter(index, words);
    },
    async typeWriter(index, words) {
      this.chatArray[index - 1].data.content = "";
      for (let i = 0; i < words.length; i++) {
        await new Promise((resolve) => {
          setTimeout(() => {
            this.chatArray[index - 1].data.content += words[i];
            resolve();
          }, 20);
        });
        this.scrollToBottom(false);
      }
    },
    scrollToBottom(smooth) {
      var el = this.$el.querySelector("#chatballoons-container");
      if (el && el.lastElementChild) {
        el.style.scrollBehavior = smooth ? "smooth" : "";
        el.scrollTop = el.lastElementChild.offsetTop;
      }
    },
    async disclaimer() {
      const index = this.chatArray.push({
        type: "ai",
        data: {
          content: disclaimerText,
          additional_kwargs: {},
          example: false,
        },
      });
      await this.typeWriter(index, disclaimerText);
      this.scrollToBottom(true);
    },
  },
};
</script>
<template>
  <div>
    <div id="navbar">
      <span class="text">GATEWAY 721</span>
      <button
        class="connect-button"
        :class="connectButtonClass"
        @click="connectKeplr"
      >
        {{ keplrAddress ? formatAddress(keplrAddress) : "Connect Wallet" }}
      </button>
    </div>
    <div id="chatballoons-container">
      <ChatBalloon
        v-for="chat in chatArray"
        :key="chat"
        :type="chat.type"
        :msg="chat.data.content"
      />
      <ChatBalloon v-if="typing === true" type="bubble" msg="" />
    </div>
    <div>
      <div id="input-container">
        <div class="project-select">
          <select>
            <option
              v-for="(item, index) in options"
              :key="index"
              :value="item.value"
            >
              {{ item.name }}
            </option>
          </select>
        </div>

        <div class="padded-input">
          <input
            v-model="inputText"
            type="text"
            :placeholder="placeholderText"
            :disabled="inputDisabling"
            aria-label="Input"
            @keyup.enter="send(inputText)"
          />
        </div>
        <img class="send-icon" src="/send-icon.svg" @click="send(inputText)" />
      </div>
    </div>
    <div id="copyright-container">
      <span
        >created by
        <a
          href="https://sharp-saw-d58.notion.site/D3LAB-10c829858e4c42eda1ce140f3e7e77bf"
          target="_blank"
          >@D3LAB</a
        >
        | <a @click="disclaimer()">Disclaimer</a></span
      >
    </div>
  </div>
</template>

<style scoped>
#navbar {
  height: 50px;
  width: 100%;
  border-bottom: 1px solid #ffffff99;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  position: relative;
}
.text {
  color: #ffffff99;
  letter-spacing: 2px;
  font-weight: 400;
  margin-right: auto;
  margin-left: 30px;
}
.wastebin-icon {
  position: absolute;
  width: 15px;
  right: 20px;
  top: 17px;
  filter: invert(1);
  opacity: 0.7;
}
.wastebin-icon:hover {
  cursor: pointer;
  opacity: 0.9;
}
#chatballoons-container {
  overflow: scroll;
  height: calc(100% - 140px);

  /* Firefox */
  scrollbar-width: none;

  /* IE 10+ */
  -ms-overflow-style: none;
}
#chatballoons-container::-webkit-scrollbar {
  /* Chrome */
  display: none;
}

#input-container {
  height: 45px;
  position: relative;
  padding: 20px 30px 0px 30px;
}

.project-select {
  position: absolute;
  top: 30px;
}
.padded-input {
  padding: 0px 20px 0px 80px;
  border: 1px solid #ffffff99;
  border-radius: 15px;
  color: #ffffff99;
}
.send-icon {
  position: absolute;
  width: 18px;
  right: 45px;
  top: 33px;
  filter: brightness(0) invert(1);
  opacity: 0.7;
}
.send-icon:hover {
  cursor: pointer;
  opacity: 0.9;
}

input {
  width: calc(100% - 25px);
  height: 40px;
  background-color: transparent;
  border: none;
  caret-color: #ffffff99;
  color: #ffffff99;
}

input::placeholder {
  color: #ffffff99;
}

textarea:focus,
input:focus {
  outline: none;
}

#copyright-container {
  height: 25px;
  text-align: center;
  color: #ffffff99;
  font-size: 0.5rem;
}
#copyright-container a:link {
  color: #ffffff99;
  text-decoration: none;
}
#copyright-container a:visited {
  color: #ffffff99;
  text-decoration: none;
}
#copyright-container a:hover {
  color: #ffffff99;
  text-decoration: underline;
  cursor: pointer;
}

.connect-button {
  border: 1px solid #ffffff99;
  color: white; /* White text */
  padding: 10px 20px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 12px;
  cursor: pointer;
  border-radius: 15px; /* Rounded corners */
  background-color: #00000000;
  transition: transform 0.2s; /* Animation */
  position: absolute;
  right: 30px; /* Position the button to the right */
}
.connect-button:hover {
  transform: scale(1.05); /* Slightly larger on hover */
  cursor: pointer; /* Ensures pointer cursor on hover */
}

.connect-button.connected {
  border: 1px solid #ffffff99;
  color: rgb(0, 0, 0); /* Blank text */
  padding: 10px 20px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 12px;
  cursor: pointer;
  border-radius: 15px; /* Rounded corners */
  background-color: #ffffff;
  transition: transform 0.2s; /* Animation */
  position: absolute;
  right: 30px; /* Position the but */
}
</style>
