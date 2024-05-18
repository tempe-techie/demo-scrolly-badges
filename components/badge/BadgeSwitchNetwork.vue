<template>
  <div class="row">
    <p class="text-break text-center mt-3">
      You're on the wrong network. Please switch the chain using the button below.
    </p>
  </div>

  <div class="d-flex justify-content-center mt-4 mb-5">
    <SwitchChainButton />
  </div>
</template>

<script>
import SwitchChainButton from '~/components/SwitchChainButton.vue';

export default {
 name: 'BadgeSwitchNetwork',

 components: {
    SwitchChainButton
  },
}
</script>
