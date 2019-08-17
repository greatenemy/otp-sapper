<div bind:this={element}>
  {#if err}
  <div class="vault-error">
    <div class="alert alert-danger">{ err }
      <button class="btn btn-primary" on:click|preventDefault|capture={() => err = ''}>OK</button>
    </div>
  </div>
  {:else if opened}
  <div class="open-vault">
    <div class="row">
      <div class="col-xs-6 col-sm-4">
        <div class="form-group">
          {#if mode === 'add'}
          <div class="add-entry">
            <div class="form-group">
              <label class="form-label">Label</label>
              <input class="form-control" bind:value={newLabel} name="newLabel" />
            </div>
            <div class="form-group">
              <label class="form-label">Value ({ newValue ? newValue.length : '0' })</label>
              <input class="form-control" bind:value={newValue} name="newValue" />
            </div>
            <button class="btn btn-success"
              class:disabled={!newValue}
              disabled={!newValue}
              on:click|preventDefault|capture={saveEntry}>Save new value</button>
            <br/>
            <div class="btn-group">
              <button class="btn btn-info" on:click|preventDefault|capture={() => newValue = randomaz09()}>
                Generate Random&#32;<tt><code>az09</code></tt>
              </button>
              <button class="btn btn-info" on:click|preventDefault|capture={() => newValue = randomAZaz09ShiftNumbers()}>
                <tt><code>AZaz09!)</code></tt>
              </button>
              <button class="btn btn-info" on:click|preventDefault|capture={() => newValue = randomAZaz09ShiftNumbers()}>
                <tt><code>AZaz09!)</code></tt>
              </button>
            </div>
          </div>
          {:else if mode === 'otp'}
          <div class="otp-entry">
            <OtpCard name={otp.label} secret={otp.value}></OtpCard>
            <button class="btn btn-primary" on:click|preventDefault|capture={() => mode = null}>Close OTP</button>
          </div>
          {:else}
          <div class="otp-entry"></div>
          {/if}
        </div>
      </div>
      <div class="col-xs-6 col-sm-8">
        <div class="form-group">
          <div class="row">
            <div class="col">
              <button class="btn btn-primary" on:click|preventDefault|capture={close}>Close Vault</button>
            </div>
            <div class="col">
              <button class="btn btn-primary" on:click|preventDefault|capture={onUserRequestDownload}>Download</button>
            </div>
            <div class="col">
              <button class="btn btn-primary" on:click|preventDefault|capture={() => mode = 'add'}>New Entry</button>
            </div>
            <div class="col">
              <div class="btn btn-primary" on:click|capture on:change={importFromFileInput}>
                <input type="file" name="import" />
              </div>
            </div>
          </div>
          {#each vaultItems as entry, index (index) }
          <div class="row entry">
            <div class="col">
              <div class="card">
                <div class="card-body">
                  <div>
                    <div class="btn-group float-right">
                      <button class="btn btn-secondary" on:click|preventDefault|capture={() => removeFromData(entry.originalDataEntry)}>X</button>
                      <button class="btn btn-secondary" on:click|preventDefault|capture={() => clipboard(index)}>COPY</button>
                      <button class="btn btn-secondary" on:click|preventDefault|capture={() => setOtp(entry)}>OTP</button>
                    </div>
                  </div>
                  <label class="form-label">{ entry.label || 'unlabeled' }</label>
                  <input class="form-control" value={entry.value} name="value" data-index={index} />
                </div>
              </div>
            </div>
          </div>
          {/each}
        </div>
      </div>
    </div>
  </div>
  {:else}
  <div class="closed-vault">
    <div class="form" on:submit|preventDefault|capture={open}>
      <div class="row">
        <div class="col-xs-6 col-sm-4">
          <div class="form-group">
            <label class="form-label">Passphrase</label>
            <input class="form-control" type="password" on:keyup={onInputKeyup} bind:value={passphrase} />
          </div>
        </div>
        <div class="col-xs-6 col-sm-4">
          <div class="form-group">
            <button class="btn btn-primary" on:click|preventDefault|capture={open}>Create/Open Vault</button>
          </div>
        </div>
      </div>
    </div>
  </div>
  {/if}
</div>

<script>
import { onMount, onDestroy } from 'svelte';
import isArray from 'lodash/isArray'
import reduce from 'lodash/reduce'
import uniq from 'lodash/uniq'
import cryptico from 'cryptico'
import OtpCard from '~/components/OtpCard.svelte'

let opened = false;
let mode = null;
let otp = null;
let passphrase = '';
let err = '';
let data = null;
let bits = 1024;
let newValue = '';
let newLabel = '';
let element = null; // bind this element to variable

let rsakey;
let publickeystring;
let vaultItems;
$: vaultItems = (rsakey ? reduce(data, vaultItemReducer, []) : []);

function vaultItemReducer (carry, entry) {
  try {
    const result = cryptico.decrypt(entry, rsakey)
    if (result.status === 'success') {
      const item = JSON.parse(result.plaintext)
      item.originalDataEntry = entry
      carry.push(item)
    }
  } catch (err) {
    // ignore
    // eslint-disable-next-line
    // console.error(err)
  }
  return carry
}

onMount(() => {
  window.addEventListener('keyup', onGlobalKeyup)
})
onDestroy(() => {
  window.removeEventListener('keyup', onGlobalKeyup)
})

function open() {
  otp = null
  mode = null
  opened = true
  rsakey = (passphrase&&bits) ? cryptico.generateRSAKey(passphrase, bits) : null;
  publickeystring = rsakey ? cryptico.publicKeyString(rsakey) : null;
  loadFromLs()
}
function close() {
  opened = false
  otp = null
  mode = null
  passphrase = ''
}
function clipboard (index) {
  const input = element.querySelector(
    `.entry input[name=value][data-index="${index}"]`,
  )
  if (input) {
    input.select()
    document.execCommand('copy')
  }
}
function onInputKeyup (event) {
  // @todo if enter, do something, else allow default
}
function onUserRequestDownload () {
  return download('backup.dat', JSON.stringify(data))
}
function importFromFileInput ($event) {
  console.log($event)
  const input = $event.srcElement || $event.target
  const file = input.files[0]

  const reader = new FileReader()

  reader.onload = $fileReadEvent => {
    const binary = $fileReadEvent.target.result
    const fileData = JSON.parse(binary)
    data = uniq([...data, ...fileData].filter(Boolean))
    saveToLs()
    input.value = null
  }

  reader.readAsText(file)
}
function setOtp (entry) {
  otp = entry
  mode = 'otp'
}
function loadFromLs () {
  const dataFromLs = window.localStorage.getItem('v')
  if (dataFromLs && dataFromLs[0] === '[') {
    data = JSON.parse(dataFromLs) || []
  }
  if (!isArray(data)) {
    data = []
  }
}
function removeFromData (originalDataEntry) {
  data = data.filter(entry => entry !== originalDataEntry)
  saveToLs()
}
function saveEntry () {
  const plaintext = JSON.stringify({
    label: newLabel,
    value: newValue,
  })
  const result = cryptico.encrypt(plaintext, publickeystring)
  data.push(result.cipher)
  data = data;
  saveToLs()
  newValue = ''
  newLabel = ''
}
function saveToLs () {
  if (isArray(data)) {
    window.localStorage.setItem('v', JSON.stringify(data))
  }
}
function onGlobalKeyup (event) {
  event = event || window.event
  if (parseInt(event.keyCode) === 27) {
    close()
  }
}
function randomaz09 (len = 32) {
  let text = ''
  const possible = 'abcdefghijklmnopqrstuvwxyz0123456789'

  for (let i = 0; i < len; i++)
    text += possible.charAt(Math.floor(Math.random() * possible.length))

  return text
}
function randomAZaz09 (len = 32) {
  let text = ''
  const possible =
    'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'

  for (let i = 0; i < len; i++)
    text += possible.charAt(Math.floor(Math.random() * possible.length))

  return text
}
function randomAZaz09ShiftNumbers (len = 32) {
  let text = ''
  const possible =
    'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()'

  for (let i = 0; i < len; i++)
    text += possible.charAt(Math.floor(Math.random() * possible.length))

  return text
}
function download(filename, text) {
  const element = document.createElement('a')
  element.setAttribute(
    'href',
    'data:text/plain;charset=utf-8,' + encodeURIComponent(text),
  )
  element.setAttribute('download', filename)

  element.style.display = 'none'
  document.body.appendChild(element)

  element.click()

  document.body.removeChild(element)
}
</script>
