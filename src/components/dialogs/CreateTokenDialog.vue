<template>
<v-dialog lazy persistent v-model='visible' scrollable @keydown.esc="visible = false" max-width="600">
  <div class='main-container'>
    <dialog-titlebar :title=title v-on:close='visible = false' />

    <div class='help-contents'>
      <div class='help-text'>
        <div>Create a token</div>
        <div class='sub-header'>Give your asset a symbol and create the tokens. Symbol can be 1-12 characters long</div>
      </div>
      <div class='help-email'>
        <v-text-field label='Symbol' v-model.trim="symbol" ref='input'></v-text-field>
        <v-text-field label='Amount' v-model.number="amount" type='number' @keyup.enter="createToken()"></v-text-field>
      </div>
      <div class='status-message'>{{statusMessage}}</div>
      <div class='button-holder'>
        <v-tooltip open-delay='200' bottom>
          <v-btn round small color='primary' slot="activator" @click="createToken()" :loading="loading">Create Token</v-btn>
          <span>Create Token</span>
        </v-tooltip>
      </div>

      <toast-component :absolute=true location='create-token-dialog' :bottom=false :top=true />
    </div>
  </div>
</v-dialog>
</template>

<script>
import Helper from '../../js/helper.js'
import {
  DialogTitleBar,
  StellarWallet,
  LedgerAPI
} from 'stellar-js-utils'
import StellarUtils from '../../js/StellarUtils.js'
import ToastComponent from '../ToastComponent.vue'
const StellarSdk = require('stellar-sdk')

export default {
  props: ['ping'],
  components: {
    'dialog-titlebar': DialogTitleBar,
    'toast-component': ToastComponent
  },
  data() {
    return {
      visible: false,
      title: 'Create Token',
      statusMessage: '',
      symbol: '',
      amount: 0,
      loading: false
    }
  },
  watch: {
    ping: function () {
      this.visible = true
      this.domain = ''
      this.statusMessage = ''

      // autofocus hack
      this.$nextTick(() => {
        this.$refs.input.focus()
      })
    }
  },
  methods: {
    createToken() {
      let issuerKeypair = null
      let asset = null

      if (this.amount < 1) {
        this.statusMessage = 'Create token amount must be greater than 0'
        return
      }
      if (!Helper.strOK(this.symbol)) {
        this.statusMessage = 'Type in a symbol name'
        return
      }

      const fundingWallet = StellarWallet.ledger(new LedgerAPI(), () => {
        this.statusMessage = 'Confirm transaction on Ledger Nano'
        this.displayToast(this.statusMessage)
      })

      this.loading = true

      // create issuer
      StellarUtils.newAccount(fundingWallet, '2', 'Issuer: ' + this.symbol, this.symbol)
        .then((accountInfo) => {
          issuerKeypair = accountInfo.keypair
          const issuerWallet = StellarWallet.secret(issuerKeypair.secret())
          asset = new StellarSdk.Asset(this.symbol, issuerKeypair.publicKey())

          // create distributor from issuer
          return StellarUtils.newAccountWithTokens(fundingWallet, issuerWallet, '2', asset, String(this.amount), 'Distributor: ' + this.symbol, this.symbol)
        })
        .then((accountInfo) => {
          // return results and close
          this.$emit('token-created', issuerKeypair, accountInfo.keypair, asset)
          this.visible = false

          return null
        })
        .catch((error) => {
          this.statusMessage = 'Error: ' + error.message
          Helper.debugLog(error)
        })
        .finally(() => {
          this.loading = false
        })
    },
    displayToast(message, error = false) {
      Helper.toast(message, error, 'create-token-dialog')
    }
  }
}
</script>

<style lang='scss' scoped>
@import '../../scss/styles.scss';

.main-container {
    @include standard-dialog-contents();

    .help-contents {
        @include inner-dialog-contents();

        .help-text {
            div {
                margin-bottom: 10px;
            }
            margin-bottom: 20px;

            .sub-header {
                font-size: 0.8em;
            }
        }

        .help-email {
            margin: 0 30px;
        }

        .status-message {
            font-size: 0.8em;
        }
    }
}
</style>
