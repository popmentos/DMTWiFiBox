<template>
  <v-content style="height:calc(100vh - 80px)">
    <v-card flat>
      <v-card-text v-if="!logined">
        <v-btn @click="login">{{ $t('home.login') }}</v-btn>
      </v-card-text>
      <v-card-text fluid v-if="logined">
        <v-layout>
          <v-flex xs30 sm6 md3>
            <v-layout column>
              <v-flex xs4>
                <v-btn @click="registerDevice">Register</v-btn>
              </v-flex>
              <v-flex xs12>
                <v-text-field
                  label="Device Name"
                  placeholder=""
                  outline
                  v-model="devname"
                ></v-text-field>
              </v-flex>
              <v-flex xs12>
                <v-text-field
                  label="Device Address"
                  placeholder=""
                  outline
                  v-model="devaddr"
                ></v-text-field>
              </v-flex>
            </v-layout>
          </v-flex>
        </v-layout>
      </v-card-text>
      <hr>
      <v-card-text v-if="logined">
         <v-layout>
          <v-flex  xs12 sm6 md3>
            <v-layout>
              <v-flex xs5>
                <v-btn @click="getUserDevices">List Devices</v-btn>
              </v-flex>
              <v-flex xs6>
              </v-flex>
            </v-layout>
          </v-flex>
        </v-layout>
        address - device name
        <ul>
          <li v-for="item in userDevices" v-bind:value="item.address" :key="item.address">
            {{ item.address }} - {{ item.bytes32 }}
          </li>
        </ul>
      </v-card-text>
      <hr>
      <!-- <v-card-text v-if="logined">
         <v-layout>
          <v-flex  xs12 sm6 md3>
            <v-layout>
              <v-flex xs6>
                <v-btn @click="offline">Set User Offline</v-btn>
              </v-flex>
              <v-flex xs6>
                <v-text-field
                  label="User Address"
                  placeholder=""
                  outline
                  v-model="offlineUserAddr"
                ></v-text-field>
              </v-flex>
            </v-layout>
          </v-flex>
        </v-layout>
      </v-card-text> -->
      <hr>
      <v-card-text fluid>
         <v-layout>
          <v-flex xs16 sm6 md3>
            <v-layout column>
              <v-flex xs8>
                <v-btn @click="setPrivateKey">Set Private Key of RPi</v-btn>
              </v-flex>
              <v-flex xs8>
                <v-text-field
                  label="Owner Address"
                  placeholder=""
                  outline
                  v-model="piOwnerAddr"
                ></v-text-field>
              </v-flex>
              <v-flex xs8>
              <v-text-field
                  label="Private Key"
                  placeholder=""
                  outline
                  v-model="piPrivateKey"
                ></v-text-field>
              </v-flex>
            </v-layout>
          </v-flex>
        </v-layout>
      </v-card-text>
    </v-card>
    <hr>
    <v-dialog
      v-model="dialog"
      persistent
      >
      <v-card
        class="white--text"
        style="background-image: radial-gradient(circle at 50% 45%, rgb(149, 74, 151), rgb(12, 0, 12) 65%);"
        >
          <a target="_blank" href="https://chrome.google.com/webstore/detail/dekusan/anlicggbddjeebblaidciapponbpegoj">
          <img src="/img/install dekusan wallet.svg"
            style="margin:0 auto"
            alt="Install DekuSan Wallet"/>
          </a>
      </v-card>
    </v-dialog>
    <v-dialog
      persistent
      v-model="alertdialog">
      <v-card>
        <v-card-title>
          {{ alerttext }}
        </v-card-title>
        <v-card-actions>
          <v-btn @click="alertdialog=false">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-content>
</template>

<script>
// @ is an alias to /src
import dmtContract from '@/../build/contracts/DMTWiFiBox.json'
import Web3 from 'web3'
import rp from 'request-promise'
const DEXON_TESTNET_ID = 238

export default {
  name: 'home',
  data () {
    return {
      WS_PROVIDER: () => {
        if (window.ethereum) {
          return 'ws://' + window.location.hostname + ':8545'
        }
        return (window.location.hostname === 'localhost')
          ? 'ws://127.0.0.1:8545'
          : 'wss://ws-proxy.dexon.org'
      },
      INJECTED: window.dexon || window.ethereum,
      netid: window.location.hostname === 'localhost' ? 5777 : 238,
      hsABI: dmtContract.abi,
      hsAddress: dmtContract.networks[ window.location.hostname === 'localhost' ? 5777 : 238 ].address,
      dmtContractWrite: null,
      dmtContractRead: null,
      logined: false,
      userdata: {
        addr: null,
        balance: null
      },
      currency: 'ether',
      wsHandler: null,
      dialog: false,
      alertdialog: false,
      alerttext: '',
      piaddr: null,
      ipaddr: null,
      devname: "",
      devaddr: null,
      offlineUserAddr: null,
      userDevices: [],
      piPrivateKey: null,
      piOwnerAddr: null
    }
  },
  mounted () {
    this.wsHandler = new Web3(this.WS_PROVIDER())
    this.dmtContractRead = new this.wsHandler.eth.Contract(this.hsABI, this.hsAddress)
  },
  methods: {
    setPrivateKey: function() {
      rp('http://192.168.254.1:3000/setup/' + this.piOwnerAddr + '/' + this.piPrivateKey)
        .then(function (res) {
            // Process...
        })
        .catch(function (err) {
            // Failed...
            console.log(err);
        });  
    },
    registerDevice: function () {
      console.log('-----------------------------------');
      console.log('registerDevice');
      console.log(this.devname);
      console.log('-----------------------------------');
      this.dmtContractWrite.methods.registerDevice(this.devaddr, this.wsHandler.utils.fromAscii(this.devname))
        .send({ from: this.userdata.addr, gas: this.gasFee })
    },
    getUserDevices: function () {
      console.log('-----------------------------------');
      console.log('getUserDevices');
      console.log('-----------------------------------');
      let that = this      
      this.dmtContractRead.methods.getUserDevices()
        .call().then(res=>{
          var i=0;
          for (i = 0; i < res[0].length; i++) { 
            that.userDevices.push({address:res[0][i], bytes32:this.wsHandler.utils.toAscii(res[1][i])});
          }
        });

    },
    offline: function () {
      console.log('-----------------------------------');
      console.log('offline');
      console.log(this.offlineUserAddr);
      console.log('-----------------------------------');
      this.dmtContractWrite.methods.userOffline(this.offlineUserAddr)
        .send({ from: this.userdata.addr, gas: this.gasFee })
    },
    getWSHandler: async function () {
      let WS_PROVIDER = (
        window.ethereum
          ? 'ws://' + window.location.hostname + ':8545'
          : (window.location.hostname === 'localhost'
            ? 'ws://127.0.0.1:8545'
            : 'wss://ws-proxy.dexon.org'
          )
      )
      let handler = await new Web3(WS_PROVIDER)()
      return handler
    },
    enableDexon: async function () {
      let e
      await this.INJECTED.enable().catch(err => { e = err })
      if (e) {
        return false
      }
      return true
    },
    login: async function () {
      if (!this.INJECTED) {
        this.needDekuSan()
        return
      }
      if (!this.enableDexon()) {
        this.alerttext = 'Please accept to interact with DekuSan'
        this.alertdialog = true
        return
      }
      this.walletHandler = await new Web3(this.INJECTED)
      this.netid = await this.walletHandler.eth.net.getId()
      if ((this.netid !== DEXON_TESTNET_ID) && (window.location.hostname !== 'localhost')) {
        this.alerttext = 'Please Select "DEXON Test Network" in DekuSan wallet'
        this.alertdialog = true
        return
      }
      this.dmtContractWrite = await new this.walletHandler.eth.Contract(
        this.hsABI, this.hsAddress)
      this.getUserData()
    },
    getUserData: async function () {
      let that = this
      await this.walletHandler.eth.getAccounts().then(accounts => {
        if (accounts.length === 0) {
          that.alerttext = 'Account not found'
          that.alert = true
          return false
        }
        that.walletHandler.eth.defaultAccount = accounts[0]
        that.userdata.addr = accounts[0]       
        that.getBalance(that.userdata.addr).then((result) => {
          let r = that.walletHandler.utils.fromWei(result, 'ether')
          that.userdata.balance = r
          that.logined = true
        })
      }).catch(e => {
        console.log(e)
      })
      return this.userdata
    },
    getBalance: async function () {
      let coinbase = await this.walletHandler.eth.getCoinbase()
      let balance = await this.walletHandler.eth.getBalance(coinbase)
      return balance
    },
    needDekuSan: function () {
      this.dialog = true
    }
  }
}
</script>
