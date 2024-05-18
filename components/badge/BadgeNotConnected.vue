<template>
  <div class="row">
    <p class="text-break text-center mt-3">
      Connect your wallet to see if you're eligible for a badge.
    </p>
  </div>

  <div class="d-flex justify-content-center mt-4 mb-5">
    <ConnectWalletButton class="btn btn-outline-primary" btnText="Connect wallet" />
  </div>
</template>

<script>
import ConnectWalletButton from "~/components/ConnectWalletButton.vue";

export default {
 name: 'BadgeNotConnected',

 components: {
    ConnectWalletButton
  },
}
</script>
