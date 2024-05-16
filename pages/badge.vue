<template>
  <Head>
    <Title>Badge | {{ $config.projectMetadataTitle }}</Title>
    <Meta property="og:title" :content="'Badge | ' + $config.projectMetadataTitle" />
  </Head>

  <div class="card border scroll-500">
    <div class="card-body">
      <p class="fs-3">
        <i class="bi bi-arrow-left-circle cursor-pointer" @click="$router.back()"></i>
      </p>

      <h3 class="mb-3 mt-3">Get a Scroll Badge here</h3>

      <p v-if="!isActivated" class="text-break mt-3">
        Connect your wallet to see if you're eligible for a badge.
      </p>

      <p v-if="isActivated && !isSupportedChain" class="text-break mt-3">
        You're on the wrong network. Please switch the chain using the button below.
      </p>

      <div v-if="!isActivated || !isSupportedChain" class="d-flex justify-content-center mt-4">
        <ConnectWalletButton v-if="!isActivated" class="btn btn-outline-primary" btnText="Connect wallet" />
        <SwitchChainButton v-if="isActivated && !isSupportedChain" />
      </div>

      <div v-if="isActivated && isSupportedChain">
        <p class="text-break mt-3">
          Check if you're eligible for a badge by clicking the button below.
        </p>

        <button class="btn btn-primary mt-4" @click="checkEligibility" :disabled="waiting">
          <span v-if="waiting" class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
          Check eligibility
        </button>

        <p v-if="isEligible" class="text-break mt-3">
          Success! You're eligible for a badge.
        </p>
      </div>

    </div>
  </div>
</template>

<script>
import axios from 'axios';
import { ethers } from 'ethers';
import { useEthers } from 'vue-dapp';
import { useToast } from "vue-toastification/dist/index.mjs";
import WaitingToast from "~/components/WaitingToast";
import ConnectWalletButton from "~/components/ConnectWalletButton.vue";
import SwitchChainButton from '~/components/SwitchChainButton.vue';

export default {
  name: 'Badge',

  components: {
    ConnectWalletButton,
    SwitchChainButton
  },

  data() {
    return {
      apiBaseUrl: "https://api.scrolly.xyz/api/badge/",
      badgeContractAddress: "0x9bc5af171bCE66c647E17D010664a3366d2CeA28",
      isEligible: false,
      waiting: false
    }
  },

  computed: {
    isSupportedChain() {
      if (this.chainId === this.$config.supportedChainId) {
        return true;
      } else {
        return false;
      }
    },
  },

  methods: {
    async claim() {
      // https://api.scrolly.xyz/api/badge/claim?badge=0x9bc5af171bCE66c647E17D010664a3366d2CeA28&recipient=0xb29050965A5AC70ab487aa47546cdCBc97dAE45D
    },

    async checkEligibility() {
      this.waiting = true;

      if (!this.address) {
        this.toast.error("Please connect your wallet first.");
        return this.waiting = false;
      }

      const checkUrl = `${this.apiBaseUrl}check?badge=${this.badgeContractAddress}&recipient=${this.address}`;

      try {
        const response = await axios.get(checkUrl);

        console.log(response.data);

        if (response.data.eligibility) {
          this.toast.success("You're eligible for a badge!");
          this.isEligible = true;
        } else {
          this.toast.error("You're not eligible for a badge.");
          this.isEligible = false;
        }
        
      } catch (error) {
        console.error(error);
        this.toast.error("An error occurred. Please try again later.");
      } finally {
        this.waiting = false;
      }
      
    }


  },

  setup() {
    const { address, chainId, isActivated, signer } = useEthers();
    const toast = useToast();

    return {
      address,
      chainId,
      isActivated,
      signer,
      toast
    }
  },
}
</script>