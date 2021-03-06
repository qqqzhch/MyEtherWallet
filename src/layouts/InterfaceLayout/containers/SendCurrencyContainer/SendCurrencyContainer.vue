<template>
  <div class="send-currency-container">
    <interface-container-title :title="$t('common.sendTx')" />

    <div class="send-form">
      <div class="form-block amount-to-address">
        <div class="amount">
          <div class="single-input-block">
            <div class="title">
              <h4>{{ $t('interface.sendTxType') }}</h4>
            </div>
            <currency-picker
              :currency="tokensWithBalance"
              :page="'sendEgasAmountthAndTokens'"
              :token="true"
              @selectedCurrency="setSelectedCurrency"
            />
          </div>
          <div class="single-input-block">
            <div class="title">
              <h4>{{ $t('interface.sendTxAmount') }}</h4>
              <p
                class="title-button prevent-user-select"
                @click="setBalanceToAmt"
              >
                Entire Balance
              </p>
            </div>
            <div class="the-form amount-number">
              <input
                v-validate="'min_value:10'"
                :value="amount"
                type="number"
                placeholder="Amount"
                min="0"
                name="amount"
                @input="debouncedAmount"
              />
              <i
                :class="[
                  selectedCurrency.symbol === network.type.name
                    ? parsedBalance.lt(amount)
                      ? 'not-good'
                      : ''
                    : errors.has('amount')
                    ? 'not-good'
                    : '',
                  'fa fa-check-circle good-button'
                ]"
                aria-hidden="true"
              />
            </div>
          </div>
          <div
            v-if="
              selectedCurrency.symbol === network.type.name
                ? parsedBalance.lt(amount)
                : selectedCurrency.balance < amount
            "
            class="error-message-container"
          >
            <p>{{ $t('common.dontHaveEnough') }}</p>
          </div>
        </div>
        <div class="to-address">
          <div class="title">
            <h4>
              {{ $t('interface.sendTxToAddr') }}
              <blockie
                v-show="isValidAddress && address.length !== 0"
                :address="hexAddress"
                :size="8"
                :scale="16"
                width="32px"
                height="32px"
                class="blockie-image"
              />
            </h4>

            <p
              class="copy-button prevent-user-select"
              @click="copyToClipboard('address')"
            >
              {{ $t('common.copy') }}
            </p>
          </div>
          <div class="the-form address-block">
            <input
              v-ens-resolver="'address'"
              ref="address"
              type="text"
              name="name"
              autocomplete="off"
              @input="debounceInput"
            />
            <i
              :class="[
                isValidAddress && address.length !== 0 ? '' : 'not-good',
                'fa fa-check-circle good-button'
              ]"
              aria-hidden="true"
            />
          </div>
        </div>
      </div>
    </div>

    <div class="send-form advanced">
      <div class="advanced-content">
        <div class="toggle-button-container">
          <h4>{{ $t('common.advanced') }}</h4>
          <div class="toggle-button">
            <span>{{ $t('interface.dataGas') }}</span>
            <!-- Rounded switch -->
            <div class="sliding-switch-white">
              <label class="switch">
                <input
                  type="checkbox"
                  @click="advancedExpend = !advancedExpend"
                />
                <span class="slider round" />
              </label>
            </div>
          </div>
        </div>
        <div
          :class="advancedExpend && 'input-container-open'"
          class="input-container"
        >
          <div class="margin-container">
            <div
              v-show="selectedCurrency.symbol === network.type.name"
              class="the-form user-input"
            >
              <p>Add Data</p>
              <input
                :value="data"
                type="text"
                placeholder="Add Data (e.g. 0x7834f874g298hf298h234f)"
                autocomplete="off"
                @input="debounceData"
              />
              <i
                :class="[
                  data.length !== 0 ? '' : 'not-good',
                  'fa fa-check-circle good-button'
                ]"
                aria-hidden="true"
              />
            </div>
            <div class="the-form user-input">
              <p>{{ $t('common.gasLimit') | capitalize }}</p>
              <input
                v-model="gasLimit"
                :placeholder="$t('common.gasLimit')"
                type="number"
                name
              />
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="submit-button-container">
      <div
        :class="[
          isValidAddress &&
          address.length !== 0 &&
          isValidAmount &&
          data.length !== 0
            ? ''
            : 'disabled',
          'submit-button large-round-button-green-filled'
        ]"
        @click="confirmationModalOpen"
      >
        {{ $t('interface.sendTx') }}
      </div>
      <interface-bottom-text
        :link-text="$t('interface.helpCenter')"
        :question="$t('interface.haveIssues')"
        link="https://kb.myetherwallet.com"
      />
    </div>
  </div>
</template>

<script>
/* eslint-disable */
import { mapGetters } from 'vuex';
import InterfaceContainerTitle from '../../components/InterfaceContainerTitle';
import CurrencyPicker from '../../components/CurrencyPicker';
import InterfaceBottomText from '@/components/InterfaceBottomText';
import Blockie from '@/components/Blockie';
import normalise from '@/helpers/normalise';
import { Misc } from '@/helpers';
import BigNumber from 'bignumber.js';
import * as unit from 'ethjs-unit';
import utils from 'web3-utils';

export default {
  components: {
    'interface-container-title': InterfaceContainerTitle,
    'interface-bottom-text': InterfaceBottomText,
    blockie: Blockie,
    'currency-picker': CurrencyPicker
  },
  props: {
    tokensWithBalance: {
      type: Array,
      default: function() {
        return [];
      }
    },
    getBalance: {
      type: Function,
      default: function() {}
    },
    highestGas: {
      type: Number,
      default: 0
    }
  },
  data() {
    return {
      advancedExpend: false,
      amount: 0,
      isValidAmount: true,
      nonce: 0,
      gasLimit: 21000,
      data: '0x',
      transactionFee: 0,
      selectedCurrency: {},
      raw: {},
      signedTx: '',
      address: '',
      hexAddress: '',
      isValidAddress: false
    };
  },
  computed: {
    ...mapGetters({
      account: 'account',
      gasPrice: 'gasPrice',
      web3: 'web3',
      network: 'network',
      ens: 'ens'
    }),
    parsedBalance() {
      return new BigNumber(unit.fromWei(this.account.balance, 'ether'));
    }
  },
  watch: {
    gasPrice(newVal) {
      if (this.verifyAddr()) this.estimateGas();
    },
    selectedCurrency(newVal) {
      if (this.verifyAddr()) this.estimateGas();
    }
  },
  methods: {
    validateHexString: Misc.validateHexString,
    debouncedAmount: utils._.debounce(function(e) {
      const decimals =
        this.selectedCurrency.symbol === this.network.type.name
          ? 18
          : this.selectedCurrency.decimals;
      this.amount =
        new BigNumber(e.target.value).decimalPlaces() > decimals
          ? new BigNumber(e.target.value).decimalPlaces(decimals).toFixed()
          : new BigNumber(e.target.value).isGreaterThanOrEqualTo(0) ? e.target.value : 0;
      if (this.amount < 0) {
        this.isValidAmount = false;
      } else {
        this.isValidAmount = true;
      }
      e.target.value = this.amount;
      if (this.verifyAddr()) {
        this.estimateGas();
      }
    }, 300),
    debounceInput: utils._.debounce(function(e) {
      this.address = e.target.value;
      if (this.verifyAddr()) this.estimateGas();
    }, 500),
    debounceData: utils._.debounce(function(e) {
      if (this.validateHexString(e.target.value)) {
        this.data = e.target.value;
        if (this.verifyAddr()) this.estimateGas();
      } else {
        this.data = '0x';
      }
    }, 500),
    copyToClipboard(ref) {
      this.$refs[ref].select();
      document.execCommand('copy');
    },
    async createTx() {
      const isEth = this.selectedCurrency.symbol === this.network.type.name;
      const coinbase = await this.web3.eth.getCoinbase();
      this.nonce = await this.web3.eth.getTransactionCount(coinbase);

      this.raw = {
        from: coinbase,
        gas: this.gasLimit,
        nonce: this.nonce,
        gasPrice: unit.toWei(this.gasPrice, 'gwei'),
        value: isEth
          ? this.amount === ''
            ? 0
            : unit.toWei(this.amount, 'ether')
          : 0,
        to: isEth ? this.hexAddress : this.selectedCurrency.addr,
        data: Misc.sanitizeHex(this.data),
        chainId: this.network.type.chainID || 1
      };
      if (this.hexAddress === '') {
        delete this.raw['to'];
      }
      this.web3.eth.sendTransaction(this.raw);
    },
    confirmationModalOpen() {
      this.createTx();
      window.scrollTo(0, 0);
    },
    setBalanceToAmt() {
      if (this.selectedCurrency.symbol === this.network.type.name) {
        const txFee = new BigNumber(this.gasLimit)
          .times(unit.toWei(this.gasPrice, 'gwei'))
          .toString();
        this.amount = this.amount > 0 ? this.parsedBalance
          .minus(unit.fromWei(txFee, 'ether'))
          .toString() : 0;
      } else {
        this.amount = this.selectedCurrency.balance;
      }
    },
    async createDataHex() {
      let amount;
      if (this.selectedCurrency.name !== 'Ether') {
        if (this.amount !== 0 && this.amount !== '') {
          amount = this.amount;
        } else {
          amount = 0;
        }
        const jsonInterface = [
          {
            constant: false,
            inputs: [
              { name: '_to', type: 'address' },
              { name: '_amount', type: 'uint256' }
            ],
            name: 'transfer',
            outputs: [{ name: '', type: 'bool' }],
            payable: false,
            stateMutability: 'nonpayable',
            type: 'function'
          }
        ];
        const contract = new this.web3.eth.Contract(
          jsonInterface,
          this.selectedCurrency.addr
        );
        this.data = await contract.methods
          .transfer(
            this.hexAddress,
            new BigNumber(amount)
              .times(new BigNumber(10).pow(this.selectedCurrency.decimals))
              .toFixed()
          )
          .encodeABI();
      } else {
        this.data = this.data !== '0x' ? this.data : '0x';
      }
    },
    setSelectedCurrency(e) {
      this.selectedCurrency = e;
    },
    async estimateGas() {
      if (this.hexAddress !== '') {
        const isEth = this.selectedCurrency.symbol === this.network.type.name;
        const bnAmount = new BigNumber(this.amount);
        const coinbase = await this.web3.eth.getCoinbase();
        if (!isEth) {
          await this.createDataHex();
        }
        const params = {
          from: coinbase,
          value: isEth
            ? this.amount === ''
              ? 0
              : unit.toWei(bnAmount, 'ether')
            : 0,
          to: isEth ? this.hexAddress : this.selectedCurrency.addr,
          data: Misc.sanitizeHex(this.data)
        };

        this.web3.eth
          .estimateGas(params)
          .then(res => {
            const resBN = new BigNumber(res);
            const txFee = resBN.times(unit.toWei(this.gasPrice, 'gwei'));
            this.transactionFee = txFee.toString();
            this.gasLimit = res ? res : this.gasLimit;
          })
          .catch(err => {
            // eslint-disable-next-line no-console
            console.error(err);
          });
      }
    },
    verifyAddr() {
      return this.web3.utils.isAddress(this.hexAddress);
    }
  }
};
</script>

<style lang="scss" scoped>
@import 'SendCurrencyContainer.scss';
</style>
