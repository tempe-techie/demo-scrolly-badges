<template>
  <div class="row">
    <p class="text-break text-center mt-3">
      You don't have a profile yet. Mint it here.
    </p>
  </div>

  <div class="d-flex justify-content-center mb-3" v-if="waitingUsername">
    <span class="spinner-border spinner-border-lg" role="status" aria-hidden="true"></span>
  </div>

  <div class="d-flex justify-content-center mt-3 mb-1">
    <div>
      <label for="inputUsername" class="form-label">Choose username</label>
      <input type="text" class="form-control" id="inputUsername" v-model="username" />
    </div>
  </div>

  <div class="d-flex justify-content-center mt-2 mb-5">
    <button @click="mintUsername" :disabled="waiting || waitingUsername || !username" class="btn btn-primary">
      <span v-if="waiting" class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
      Mint Profile
    </button>
  </div>
</template>

<script>
import { useEthers } from 'vue-dapp';
import { ethers } from 'ethers';
import { useToast } from "vue-toastification/dist/index.mjs";
import WaitingToast from "~/components/WaitingToast";
import { useUserStore } from '~/store/user';
import { getDomainName } from '~/utils/domainUtils';

export default {
  name: 'BadgeMintProfile',
  props: ["profileRegistryAddress"],
  emits: ["checkIfProfileMinted"],

  data() {
    return {
      domainName: null,
      username: null,
      waiting: false,
      waitingUsername: false,
    };
  },

  mounted() {
    this.fetchScrollyName();
  },

  methods: {
    async fetchScrollyName() {
      let provider = this.$getFallbackProvider(this.$config.supportedChainId);

      if (this.signer) {
        provider = this.signer;
      }

      this.domainName = await getDomainName(this.address, provider);

      if (this.domainName) {
        this.username = this.domainName;
      }
    },

    async isUsernameUsed() {
      if (!this.username) {
        this.toast.error("Please enter a username");
        this.waiting = false;
        return;
      }

      const intrfc = new ethers.utils.Interface([
        "function isUsernameUsed(string calldata username) external view returns (bool)",
      ]);

      const contract = new ethers.Contract(this.profileRegistryAddress, intrfc, this.signer);

      try {
        const usernameUsed = await contract.isUsernameUsed(this.username);
        return usernameUsed;
      } catch (error) {
        console.error("Error checking if username is used", error);
      } finally {
        this.waiting = false;
      }
    },

    async mintUsername() {
      this.waiting = true;

      const usernameUsed = await this.isUsernameUsed();

      console.log("usernameUsed in mintUsername():", usernameUsed);

      if (usernameUsed) {
        this.toast.error("Username already used. Please pick another one.");
        this.waiting = false;
        return;
      }

      const intrfc = new ethers.utils.Interface([
        "function mint(string calldata username, bytes memory referral) external payable returns (address)"
      ]);

      const contract = new ethers.Contract(this.profileRegistryAddress, intrfc, this.signer);

      console.log("minting username:", this.username);

      try {
          const tx = await contract.mint(
            this.username, // username
            "0x", // referral
            {
              value: ethers.utils.parseEther("0.001")
            }
          );

          console.log("tx sent");

          const toastWait = this.toast(
            {
              component: WaitingToast,
              props: {
                text: "Please wait for your transaction to confirm. Click on this notification to see transaction in the block explorer."
              }
            },
            {
              type: "info",
              onClick: () => window.open(this.$config.blockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
            }
          );

          const receipt = await tx.wait();

          if (receipt.status === 1) {
            this.toast.dismiss(toastWait);
            
            this.toast("You have successfully minted a Scroll Canvas profile!", {
              type: "success",
              onClick: () => window.open(this.$config.blockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
            });
            
            this.$emit("checkIfProfileMinted");
          } else {
            this.toast.dismiss(toastWait);
            this.toast("Transaction has failed.", {
              type: "error",
              onClick: () => window.open(this.$config.blockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
            });
            console.log(receipt);
          }

        } catch (e) {
          console.error(e);
          this.toast("Error: " + e, {type: "error"});
          return;
        } finally {
          this.waiting = false;
        }
    },

  },

  setup() {
    const { address, signer } = useEthers();
    const toast = useToast();
    const userStore = useUserStore();

    return { address, signer, toast, userStore };
  }

}
</script>
