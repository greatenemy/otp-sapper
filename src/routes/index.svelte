<style>

</style>

<svelte:head>
  <title>OTP - HTML5</title>
</svelte:head>

<h2 class="text-center">HTML5 OTP</h2>
<h4 class="text-center title">HTML5 OTP</h4>
<p>This is a proof of concept drop-in-replacement for Google Authenticator, do not use in production.</p>
<br />

{#if process.browser}
  <Vault />
{/if}

<script>
  import Vault from '~/components/Vault.svelte';
</script>
