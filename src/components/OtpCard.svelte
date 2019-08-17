<section class="mui-panel otp-card" bind:this={element}>
  <h3>{ name }</h3>
  <div class="content">
    <div>
      <div class="primary-card-content">
        <h1>
          OTP:
          <input
            class="form-control copy-this"
            value={otp}
            name="otp"
            on:click|preventDefault|capture={clipboard}
          />
        </h1>
        <h4>
          next: { seconds }
          <span class="text-muted">second{ seconds === 1 ? '' : 's' }</span>
        </h4>
      </div>
    </div>
  </div>
</section>

<script>
import { onMount, onDestroy } from 'svelte';
import * as otplib from 'otplib'

export let name;   // String, required
export let secret; // String, required, secretb32

let element;
let seconds = 0;
let otp = null;
let generator = null;
let qrContent = null;
let interval = null;

$: qrContent = `otpauth://totp/${name}?secret=${secret}`;

onMount(() => {
  updateToken();
  interval = setInterval(updateToken, 120);
});
onDestroy(() => {
  clearInterval(interval);
});

function updateToken () {
  const now = new Date()
  seconds = (60 - now.getSeconds()) % 30
  // this.time = now.toLocaleString()
  otp = otplib.authenticator.generate(secret)
}
function clipboard (index) {
  const input = element.querySelector(
    '.copy-this',
  )
  if (input) {
    input.select()
    document.execCommand('copy')
  }
}
</script>

<style>
/*.clock-container {
  width: 200px;
  height: 200px;
  margin: 0 auto;
}*/

/*.qr-container {
  text-align: center;
}
*/
/*.qrcode {
  padding: 12px;
}*/
</style>
