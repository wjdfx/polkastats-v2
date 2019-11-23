<template>
  <div>
    <section>
      <b-container class="main pt-4">
        <!-- <b-alert show dismissible variant="info" class="text-center">
          Connected to chain <strong>{{system.chain}}</strong> using <strong>{{ system.client_name}}</strong> client version <strong>{{system.client_version}}</strong>
        </b-alert> -->
        <b-alert show dismissible variant="success" class="text-center">
          Total issuance is <strong>{{ formatDot(totalIssuance) }}</strong>, total stake bonded is <strong>{{ formatDot(totalStakeBonded) }} ({{ totalStakeBondedPercen.toString(10) }}% of total issuance)</strong>
        </b-alert>
        <Network :bestblocknumber="bestblocknumber" :bestBlockFinalized="bestBlockFinalized" :session="session" />
        <!-- Table with sorting and pagination-->
        <div class="table-responsive">
          <b-table
            stacked="md"
            id="intentions-table"
            head-variant="dark"
            :fields="Fields"
            :items="intentions"
            :per-page="perPage"
            :current-page="CurrentPage"
            :sort-by.sync="SortBy"
            :sort-desc.sync="SortDesc"
          >
            <template slot="rank" slot-scope="data">
              <p class="text-right mb-0">
                {{ data.item.rank }}
              </p>
            </template>
            <template slot="accountId" slot-scope="data">
              <div class="d-block d-sm-block d-md-none d-lg-none d-xl-none text-center">

                <div v-if="hasIdentity(data.item.accountId)">
                  <div v-if="getIdentity(data.item.accountId).logo !== ''">
                    <img v-bind:src="getIdentity(data.item.accountId).logo" class="identity mt-2" />
                    <h4 class="mt-2 mb-2" v-if="getIdentity(data.item.accountId).full_name !== ''">{{ getIdentity(data.item.accountId).full_name }}</h4>
                  </div>
                </div>
                <div v-else>
                  <Identicon :value="data.item.accountId" :size="80" :theme="'polkadot'" />
                </div>
                <nuxt-link :to="{name: 'intention', query: { accountId: data.item.accountId } }" title="Intention details">
                  <h4 v-if="hasIdentity(data.item.accountId)" class="mt-2 mb-2">
                    {{ getIdentity(data.item.accountId).full_name }}
                  </h4>
                  <h4 v-else-if="hasNickname(data.item.accountId)" class="mt-2 mb-2">
                    {{ getNickname(data.item.accountId) }}
                  </h4>
                  <h4 v-else class="mt-2 mb-2">
                    <span class="d-inline d-sm-inline d-md-inline d-lg-inline d-xl-none">{{ shortAddress(data.item.accountId) }}</span>
                    <span class="d-none d-sm-none d-md-none d-lg-none d-xl-inline"></span>
                  </h4>
                </nuxt-link>

                <p class="mt-3 mb-0 rank">
                  rank #{{ data.item.rank }}
                </p>
                <p class="bonded mb-0" v-b-tooltip.hover title="Active bonded">
                  {{ formatDot(data.item.activeStake) }}
                </p>
                <p class="mb-0">
                  <small>
                    <span v-b-tooltip.hover title="Total bonded">
                      {{ formatDot(data.item.totalStake) }}
                    </span>
                  </small>
                </p>
              
              </div>
              <div class="d-none d-sm-none d-md-block d-lg-block d-xl-block">
                <Identicon :value="data.item.accountId" :size="20" :theme="'polkadot'" />
                <nuxt-link :to="{name: 'intention', query: { accountId: data.item.accountId } }" title="Intention details">
                  <span v-if="hasIdentity(data.item.accountId)">
                    {{ getIdentity(data.item.accountId).full_name }}
                  </span>
                  <span v-else-if="hasNickname(data.item.accountId)">
                    {{ getNickname(data.item.accountId) }}
                  </span>
                  <span v-else>
                    <span class="d-inline d-sm-inline d-md-inline d-lg-inline d-xl-none">{{ shortAddress(data.item.accountId) }}</span>
                    <span class="d-none d-sm-none d-md-none d-lg-none d-xl-inline">{{ data.item.accountId }}</span>
                  </span>
                </nuxt-link>
              </div>
            </template>
            <template slot="totalStake" slot-scope="data">
              <p class="text-right mb-0">
                {{ formatDot(data.item.totalStake) }}
              </p>
            </template> 
            <template slot="comission" slot-scope="data">
              <p class="text-right mb-0">
                {{ formatDot(data.item.comission) }}
              </p>
            </template>
            <template slot="favorite" slot-scope="data">
              <p class="text-center mb-0">
                <a class="favorite" v-on:click="toggleFavorite(data.item.accountId)">
                  <i v-if="data.item.favorite" class="fas fa-star" style="color: #f1bd23" v-b-tooltip.hover title="Remove from Favorites"></i>
                  <i v-else class="fas fa-star" style="color: #e6dfdf;" v-b-tooltip.hover title="Add to Favorites"></i>
                </a>
              </p>
            </template>
          </b-table>
          <b-pagination
            v-model="CurrentPage"
            :total-rows="Rows"
            :per-page="perPage"
            aria-controls="validators-table"
          ></b-pagination>
        </div>
      </b-container>
    </section>
  </div>
</template>
<script>
import { mapMutations } from 'vuex';
import axios from 'axios';
import bootstrap from 'bootstrap';
import Identicon from '../components/identicon.vue';
import Network from '../components/network.vue';
import { formatBalance, isHex } from '@polkadot/util';
import BN from 'bn.js';
import { decimals, unit, backendBaseURL, blockExplorer} from '../polkastats.config.js';

formatBalance.setDefaults({ decimals, unit });

export default {
  head () {
    return {
      title: 'PolkaStats - Polkadot network statistics',
      meta: [
        { hid: 'description', name: 'description', content: 'Polkadot network statistics' }
      ]
    }
  },
  data: function() {
    return {
      perPage: 20,
      CurrentPage: 1,
      SortBy: 'rank',
      SortDesc: false,
      Fields: [
        { key: 'rank', label: '#', sortable: true, class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell` },
        { key: 'accountId', label: 'Intention', sortable: true },
        { key: 'totalStake', label: 'Total Stake', sortable: true, class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell` },
        { key: 'comission', label: 'Comission', sortable: true, class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell` },
        { key: 'favorite', label: '⭐', sortable: true, class: `d-none d-sm-none d-md-table-cell d-lg-table-cell d-xl-table-cell` }
      ],
      system: {
        chain: "",
        client_name: "",
        client_version: "",
        timestamp: 0
      },
      blockExplorer,
      backendBaseURL,
      favorites: [],
      polling: null,
      bestblocknumber: 0,
      bestBlockFinalized: 0,
      session: {  
        currentEra: 0,
        currentIndex: 0,
        eraLength: 0,
        eraProgress: 0,
        lastEraLengthChange: 0,
        lastLengthChange: 0,
        sessionLength: 0,
        sessionsPerEra: 0,
        sessionProgress: 0
      },
      totalIssuance: ""
    }
  },
  computed: {
    intentions () {
      let intentionsObject = [];
      for(let i = 0; i < this.$store.state.intentions.list.length; i++) {
        let intention = this.$store.state.intentions.list[i];
        intentionsObject.push({
          rank: i+1,
          accountId: intention.accountId,
          totalStake: intention.stakingLedger.total,
          activeStake: intention.stakingLedger.active,
          comission: intention.validatorPrefs.validatorPayment,
          favorite: this.isFavorite(intention.accountId)
        });
      }
      return intentionsObject;
    },
    identities() {
      return this.$store.state.identities.list;
    },
    nicknames() {
      return this.$store.state.nicknames.list;
    },
    Rows() {
      return this.$store.state.intentions.list.length;
    },
    totalStakeBondedPercen() {
      if (this.totalStakeBonded !== 0 && this.totalIssuance !== "") {
        let totalIssuance = new BN(this.totalIssuance, 10);
        let totalStakeBonded = this.totalStakeBonded.mul(new BN('100', 10));
        return totalStakeBonded.div(totalIssuance);
      } else {
        return 0;
      }
    },
    totalStakeBonded () {
      return this.$store.state.validators.totalStakeBonded
    },
  },
  created: function () {
    var vm = this;

    // Get favorites from cookie
    if (this.$cookies.get('favorites')) {
      this.favorites = this.$cookies.get('favorites');
    }

    // First time
    this.getSystemData();
    this.getChainData();
    
    // Force update of intentions list if empty
    if (this.$store.state.intentions.list.length == 0) {
      vm.$store.dispatch('intentions/update');
    }

    // Force update of indentity list if empty
    if (this.$store.state.identities.list.length == 0) {
      vm.$store.dispatch('identities/update');
    }

    // Force update of nicknames list if empty
    if (this.$store.state.nicknames.list.length == 0) {
      vm.$store.dispatch('nicknames/update');
    }

    /* Update intention validators, best block and session info every 10 seconds */
    this.polling = setInterval(() => {
      vm.$store.dispatch('intentions/update');
      this.getChainData();
    }, 10000);

  },
  beforeDestroy: function () {
    clearInterval(this.polling);
    clearInterval(this.sessionPolling);
  },
  methods: {
    getSystemData: function () {
      var vm = this;
      axios.get(`${backendBaseURL}/system`)
        .then(function (response) {
          vm.system = response.data;
        })
    },
    getChainData: function () {
      var vm = this;
      axios.get(`${backendBaseURL}/chain`)
        .then(function (response) {
          vm.bestblocknumber = response.data.block_height;
          vm.bestBlockFinalized = response.data.block_height_finalized;
          vm.session = response.data.session;
          vm.totalIssuance = response.data.total_issuance;
        });
    },
    formatNumber(n) {
      if (isHex(n)) {
        return (parseInt(n, 16).toString()).replace(/(\d)(?=(\d{3})+(?!\d))/g, '$1,');
      } else {
        return (n.toString()).replace(/(\d)(?=(\d{3})+(?!\d))/g, '$1,');
      }
    },
    formatDot(amount) {
      let bn;
      if (isHex(amount)) {
        bn = new BN(amount.substring(2, amount.length), 16);
      } else {
        bn = new BN(amount.toString());
      }
      return formatBalance(bn.toString(10));
    },  
    shortAddress(address) {
      return (address).substring(0,5) + ' .... ' + (address).substring(address.length - 5);
    },
    thousandsSeparator(n) {
        return n.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
    },
    toggleFavorite(validator) {
      // Receives validator accountId
      if (this.isFavorite(validator)) {
        this.favorites.splice(this.getIndex(validator), 1);
      } else {
        this.favorites.push({ accountId: validator, name: 'Edit validator name...'});
      }
      return true;
    },
    isFavorite(validator) {
      // Receives validator accountId
      for (var i=0; i < this.favorites.length; i++) {
        if (this.favorites[i].accountId == validator) {
          return true;
        }
      }
      return false;
    },
    getIndex(validator) {
      // Receives validator accountId
      for (var i=0; i < this.favorites.length; i++) {
        if (this.favorites[i].accountId == validator) {
          return i;
        }
      }
      return false;
    },
    getRank(validator) {
      // Receives validator accountId
      for (var i=0; i < this.validators.length; i++) {
        if (this.validators[i].accountId == validator) {
          return i + 1;
        }
      }
      return false;
    },
    makeToast(content = '', title = '', variant = null, solid = false) {
      this.$bvToast.toast(content, {
        title: title,
        variant: variant,
        solid: solid
      })
    },
    formatRewardDest(rewardDestination) {
      if (rewardDestination === 0) {
        return `Stash account (increase stake)`;
      }
      if (rewardDestination === 1) {
        return `Stash account (do not increase stake)`;
      }
      if (rewardDestination === 2) {
        return `Controller account`;
      }
      return rewardDestination;
    },
    hasIdentity(stashId) {
      return this.$store.state.identities.list.some(obj => {
        return obj.stashId === stashId;
      });
    },
    getIdentity(stashId) {
      let filteredArray =  this.$store.state.identities.list.filter(obj => {
        return obj.stashId === stashId
      });
      return filteredArray[0];
    },
    hasNickname(accountId) {
      return this.$store.state.nicknames.list.some(obj => {
        return obj.accountId === accountId;
      });
    },
    getNickname(accountId) {
      let filteredArray =  this.$store.state.nicknames.list.filter(obj => {
        return obj.accountId === accountId
      });
      return filteredArray[0].nickname;
    }
  },
  watch: {
    favorites: function (val) {
      //console.log(val);
      this.$cookies.set('favorites', val, {
        path: '/',
        maxAge: 60 * 60 * 24 * 7
      });
    }
  },
  components: {
    Identicon,
    Network
  }
}
</script>
<style>
body {
  font-size: 0.9rem;
}
.page-intentions .favorite {
  cursor: pointer;
  position: absolute;
  top: 0.4rem;
  right: 0.4rem;
  z-index: 10;
  font-size: 1.1rem;
}
.validator .bg-offline {
  background-color: rgba(239, 57, 74, 0.12) !important;
}
.validator .bg-candidate {
  background-color: rgba(21, 240, 86, 0.12) !important;
}
.rank {
  font-size: 1.6rem;
  color: #7d7378;
}
.account {
  color: #670d35;
}
.bonded {
  color: #d75ea1;
  font-weight: 700;
  font-size: 1.3rem;
}
[data-toggle="collapse"] .fas:before {   
  content: "\f078";
}
[data-toggle="collapse"][aria-expanded="false"] .fas:before {
  content: "\f054";
}
.nominators {
  color: #670d35;
}
.nominator {
  font-size: 0.9rem;
}
.nominator .value {
  color: #d75ea1;
  font-weight: 700;
}
.fee, .unstake {
  color: #d75ea1;
  font-weight: 700;
}
.block {
  color: #d75ea1;
  font-weight: bold;
}
.block:hover {
  color: #d75ea1;
}
.tab-content .validator:nth-child(1) {
  border-top: 0;
}
.fas.fa-copy {
  cursor: pointer;
  color: #d75ea1;
  font-size: 0.9rem;
  margin-left: 0.1rem;
}
.identity {
  max-width: 80px;
}
#intentions-table th {
  text-align: center;
}
#intentions-table th:first-child {
  width: 8%;
}
#intentions-table th:nth-child(2) {
  width: 42%;
}
#intentions-table th:nth-child(3) {
  width: 20%;
}
#intentions-table th:nth-child(4) {
  width: 20%;
}
#intentions-table th:nth-child(5) {
  width: 10%;
}
#intentions-table .identicon {
  display: inline;
  margin-right: 0.2rem;
  cursor: copy;
}
#intentions-table .identicon div {
  display: inline;
}
.page-item.active .page-link {
    z-index: 1;
    color: #fff;
    background-color: #343a40;
    border-color: #343a40;
}
.page-link {
    position: relative;
    display: block;
    padding: 0.5rem 0.75rem;
    margin-left: -1px;
    line-height: 1.25;
    color: #343a40;
    background-color: #fff;
    border: 1px solid #dee2e6;
}
</style>