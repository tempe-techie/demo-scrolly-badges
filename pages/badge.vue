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

        <!-- Check Eligibility -->
        <button v-if="!isEligible" class="btn btn-primary mt-4" @click="checkEligibility" :disabled="waiting">
          <span v-if="waiting" class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
          Check eligibility
        </button>

        <!-- If user is eligible... -->
        <div v-if="isEligible">
          <div v-if="badgeMetadata">
            <div class="row">
              <img :src="badgeMetadata.image" class="img-thumbnail col-md-4" alt="..." />
            </div>

            <div class="row">
              <p>{{ badgeMetadata.description }}</p>
            </div>
          </div>

          <!-- Mint badge -->
          <button v-if="!hasBadge" class="btn btn-primary mt-2" @click="mintBadge" :disabled="waiting">
            <span v-if="waiting" class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
            Mint badge
          </button>

          <!-- Data -->
          <hr />

          <p class="text-break mt-3">
            Data:
            <ul>
              <li>Is eligible for minting: {{ isEligible }}</li>
              <li>Is profile already minted: {{ isProfileMinted }}</li>
              <li>User already has badge: {{ hasBadge }}</li>
            </ul>
          </p>

        </div>
        
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
      badgeMetadata: null,
      hasBadge: false,
      isEligible: false,
      isProfileMinted: false,
      profileRegistryAddress: "0x26aa585d5Da74A373E58c4fA723E1E1f6FD6474f",
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

      this.checkIfBadgeMinted();
      this.checkIfProfileMinted();
      
    },

    async checkIfBadgeMinted() {
      this.waiting = true;

      const intrfc = [
        "function hasBadge(address _user) external view returns (bool)"
      ];

      const contract = new ethers.Contract(this.badgeContractAddress, intrfc, this.signer);

      try {
        this.hasBadge = await contract.hasBadge(this.address);

        this.getBadgeMetadata();
      } catch (error) {
        console.error(error);
        this.toast.error("An error occurred with the hasBadge() call. Please try again later.");
      } finally {
        this.waiting = false;
      }
    },

    async checkIfProfileMinted() {
      this.waiting = true;

      const intrfc = [
        "function isProfileMinted(address _user) external view returns (bool)"
      ];

      const contract = new ethers.Contract(this.profileRegistryAddress, intrfc, this.signer);

      try {
        this.isProfileMinted = await contract.isProfileMinted(this.address);
      } catch (error) {
        console.error(error);
        this.toast.error("An error occurred with the isProfileMinted() call. Please try again later.");
      } finally {
        this.waiting = false;
      }
      
    },

    async getBadgeMetadata() {
      this.waiting = true;

      const intrfc = [
        "function badgeTokenURI(bytes32) external view returns (string memory)",
        "function getLevel(address) public view returns (uint8)"
      ];

      const contract = new ethers.Contract(this.badgeContractAddress, intrfc, this.signer);

      try {
        const level = await contract.getLevel(this.address);
        console.log(level);

        const mdUrl = await contract.badgeTokenURI("0x0000000000000000000000000000000000000000000000000000000000000000");

        if (mdUrl.startsWith("https://")) {
          // Fetch the metadata
          const response = await axios.get(mdUrl+"?level="+level);
          console.log(response.data);

          this.badgeMetadata = response.data;
          console.log(response.data.image);
        } else {
          console.error("Invalid IPFS URL");
        }
      } catch (error) {
        console.error(error);
        this.toast.error("An error occurred with the hasBadge() call. Please try again later.");
      } finally {
        this.waiting = false;
      }
    },

    async mintBadge() {
      this.waiting = true;

      const url = `${this.apiBaseUrl}claim?badge=${this.badgeContractAddress}&recipient=${this.address}`;

      try {
        const response = await axios.get(url);

        /* Send unsigned transaction */
        const txResponse = await this.signer.sendTransaction(response.data.tx);
        console.log("Transaction hash:", txResponse.hash);

        // Wait for the transaction to be mined
        const receipt = await txResponse.wait();
        console.log("Transaction was mined in block:", receipt.blockNumber);
      
        
      } catch (error) {
        console.error(error);
        this.toast.error("An error occurred. Please try again later.");
      } finally {
        this.waiting = false;
      }
    
    },
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