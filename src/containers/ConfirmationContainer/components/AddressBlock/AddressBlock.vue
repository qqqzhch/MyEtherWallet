<template lang="html">
  <div class="address-container">
    <div class="currency-container">
      <img :src="require(`@/assets/images/currency/${currency}.svg`)" />
      <p>
        <span class="currency-amt">
          {{ direction === 'from' ? '-' : '+' }}
          {{ tokenTransferVal !== '' ? tokenTransferVal : converter(value) }}
        </span>
        <span class="currency-type"
          >{{ tokenSymbol !== '' ? tokenSymbol : currency.toUpperCase() }}
        </span>
      </p>
    </div>
    <div class="identicon-container">
      <p>{{ direction | capitalize }} Address</p>
    </div>
    <div class="address">{{ checksumAddress }}</div>
  </div>
</template>

<script>
import web3 from 'web3';
// import BigNumber from 'bignumber.js';
export default {
  props: {
    address: {
      type: String,
      default: ''
    },
    direction: {
      type: String,
      default: ''
    },
    value: {
      type: Number,
      default: 0
    },
    currency: {
      type: String,
      default: 'eth'
    },
    tokenTransferTo: {
      type: String,
      default: ''
    },
    tokenTransferVal: {
      type: String,
      default: ''
    },
    tokenSymbol: {
      type: String,
      default: ''
    }
  },
  computed: {
    checksumAddress() {
      if (web3.utils.isAddress(this.tokenTransferTo))
        return web3.utils.toChecksumAddress(this.tokenTransferTo);
      if (web3.utils.isAddress(this.address))
        return web3.utils.toChecksumAddress(this.address);
      return '';
    }
  },
  methods: {
    converter(num) {
      return web3.utils.fromWei(num.toString(), 'ether');
    }
  }
};
</script>

<style lang="scss">
@import 'AddressBlock.scss';
</style>
