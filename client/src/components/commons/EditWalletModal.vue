<template>
	<b-modal 
		id="editWallet" 
		:okDisabled="isOkDisabled" 
		@close="link=false" 
		:hide-footer="!!link" 
		:title="getTitle"
		@ok="handleOk">
		<b-container v-if="!link" fluid >

			<b-row v-if="!propExchange">
				<label for="input-with-list">{{$t("editModalSelectExchange")}}</label>
				<b-form-input 
					v-on:input="reset" 
					:state="validExchange"
					list="input-list"
					v-model="selectedExchange"
					id="input-with-list"
					></b-form-input>
				<b-form-datalist 
					id="input-list"
					:options="assocExchanges"></b-form-datalist>
			</b-row >

			<b-row v-if="!propWalletId && !isRemoving">
				<b-col cols="9" >
				<b-form-input
					v-on:input="reset"
					type='text'
					no-wheel
					v-model="selectedWalletId"
					:placeholder="$t('editModalHolderEnterIdOrAddress')"></b-form-input>
				</b-col>
				<b-col cols="2" >
				<b-button
					variant="primary"
					v-if="!isSpinnerActive && isCheckButtonActive"
					v-on:click="check" 
					size="s">check</b-button>
				<div v-if="isSpinnerActive" class="text-center w-100">
					<b-spinner label="Spinning"></b-spinner>
				</div>
				</b-col>
			</b-row >

			<b-row v-if="!propWalletId && isRemoving">
				<div>
					<b-form-group label="Select wallet to be removed" >
						<b-form-radio  v-for="wallet_id in wallet_choices" name="some-radios" :value="wallet_id" :key="wallet_id" @change="onRadioSelected">{{wallet_id}} </b-form-radio>
					</b-form-group>
				</div>
			</b-row >

			<b-row >
				<span v-if="text_error" class="pt-3">{{text_error}}</span>
				<div v-if="rewardAmount>0" class="pt-3">
					<i18n v-if="isRemoving" path="editModalGainIfRemoved" id="potential-gain">
						<template #stake_amount>
							<byte-amount :amount="stakeAmount" />
						</template>
						<template #gain_amount>
							<byte-amount :amount="rewardAmount" /> 
						</template>
						<template #wallet>
							{{selectedWalletId}}
						</template>
						<template #exchange>
							<exchange :id="selectedExchange" noUrl />
						</template>
					</i18n>
					<i18n v-else path="editModalGainIfAdded" id="potential-gain">
						<template #stake_amount>
							<byte-amount :amount="stakeAmount" />
						</template>
						<template #gain_amount>
							<byte-amount :amount="rewardAmount" /> 
						</template>
						<template #wallet>
							{{selectedWalletId}}
						</template>
						<template #exchange>
							<exchange :id="selectedExchange" noUrl />
						</template>
  			</i18n>
				<div class="mt-4">
					{{$t('editModalProofExplanation')}}
				</div>
				<UrlInputs :requireOneUrl="true" v-on:url_1_update="update_url_1" v-on:url_2_update="update_url_2"/>
				</div>
			</b-row >

		</b-container>
		<b-container v-else fluid >
			<b-row class="pt-3">
				{{$t("editModalLinkHeader")}}
			</b-row >
			<b-row class="pt-3">
				<span class="text-break">
					<a :href="link">{{link}}</a>
				</span>
			</b-row >
			<b-row class="py-3">
				{{$t("editModalLinkFooter")}}
			</b-row >
		</b-container>
	</b-modal>
</template>

<script>
const conf = require("../../conf.js");
import ByteAmount from './ByteAmount.vue';
import Exchange from './Exchange.vue';
import UrlInputs from './UrlInputs.vue';
import validate from 'bitcoin-address-validation';
const isUrl = require('is-url');

export default {	
	components: {
		ByteAmount,
		UrlInputs,
		Exchange
	},
	props: {
		propWalletId: {
			type: Number,
			required: false,
			default: null
		},
		propExchange: {
			type: String,
			required: false,
			default: null
		},
		isRemoving: {
			type: Boolean,
			required: false,
			default: false
		}
	},
	data(){
		return {
			text_error: null,
			selectedWalletId: null,
			wallet_choices: [],
			selectedExchange: null,
			isOperationAllowed: false,
			isSpinnerActive: false,
			isOkDisabled: true,
			isPoolAvailable: false,
			rewardAmount: false,
			isCheckButtonActive: false,
			stakeAmount: conf.challenge_min_stake_gb*conf.gb_to_bytes,
			link: false,
			url_1: null,
			url_2: null
			}
	},

	computed:{

		getTitle:function(){
			if (this.propExchange){
				if (this.selectedWalletId && this.selectedExchange){
					return (this.isRemoving ? this.$t('editModalRemoveXFromX', {exchange: this.assocExchanges[this.selectedExchange], wallet: this.selectedWalletId}): 
					this.$t('editModalAddXToX', {exchange:this.assocExchanges[this.selectedExchange], wallet: this.isWalletId(this.selectedWalletId)? this.selectedWalletId : ""}));
				}
				else if (this.selectedExchange){
					return (this.isRemoving ? this.$t('editModalRemoveFromX',{exchange:this.assocExchanges[this.selectedExchange]}):
					this.$t('editModalAddToX',{exchange:this.assocExchanges[this.selectedExchange]}));
				}
				else if (this.selectedWalletId){
					return (this.isRemoving ? this.$t('editModalRemoveXFrom', {wallet: this.selectedWalletId}): 
					this.$t('editModalAddXTo', {wallet: this.isWalletId(this.selectedWalletId) ? this.selectedWalletId : ""}));
				}
			} else if (this.propWalletId){
					return this.$t('editModalAddXToX', {exchange:this.assocExchanges[this.selectedExchange], wallet: this.isWalletId(this.selectedWalletId) ? this.selectedWalletId : ""});
			 } else
				 return "";
		},
		validExchange() {
			return !!this.assocExchanges[this.selectedExchange]
		},
		assocExchanges() {
			return this.$store.state.exchangesById;
		}
	},
	watch:{
		propExchange:function(){
			this.selectedExchange = this.propExchange;
			this.selectedWalletId = this.propWalletId;
			this.reset();
		},
		propWalletId:function(){
			this.selectedExchange = this.propExchange;
			this.selectedWalletId = this.propWalletId;

			if (this.propExchange) // we have an exchange and wallet id as prop, let's go to check directly
				this.check()
			this.reset();
		},
		selectedWalletId: function(){
			if (this.selectedWalletId)
				this.isCheckButtonActive = validate(this.selectedWalletId.toString()) || this.isWalletId(this.selectedWalletId);
			else
				this.isCheckButtonActive = false;
		},
		isRemoving: function(){

		},
		selectedExchange: function(){
			if (this.propWalletId && this.assocExchanges[this.selectedExchange])// we have a wallet id as prop and exchange input is valid, let's go to check
				this.check();
		},
		url_1: function(){
			this.isOkDisabled = !isUrl(this.url_1) && this.isPoolAvailable;
		}
	},
	created(){
		if (this.propWalletId)
			this.selectedWalletId = this.propWalletId;
	},
	methods:{
		isWalletId(wallet){
				return (Number(wallet) && parseInt(wallet))
		},
		update_url_1(value){
			this.url_1 = value;
		},
		update_url_2(value){
			this.url_2 = value;
		},
		reset(){
			this.isSpinnerActive = false;
			this.isCheckButtonActive = false;
			this.text_error = null;
			this.isOkDisabled = true;
			this.isPoolAvailable = false;
			this.bestPoolId = false;
			this.rewardAmount = 0;
			if (!this.selectedWalletId && this.selectedExchange && this.isRemoving){
				this.axios.get('/api/exchange-wallets/'+this.selectedExchange).then((response) => {
					this.wallet_choices = response.data;
				});
			}
		},
		onRadioSelected(arg){
			this.selectedWalletId = arg;
			this.check();
		},
		check(){
			this.text_error = null;
			this.isOkDisabled = true;
			this.bestPoolId = false;
			this.rewardAmount = 0;
			this.isSpinnerActive = true;
			this.isCheckButtonActive = false;

			this.axios.get('/api/redirection/'+this.selectedWalletId).then((response) => {
				this.selectedWalletId = response.data.redirected_id;
				if (!this.selectedWalletId)
					this.text_error = this.$t("editModalWalletNotFound");

				this.axios.get('/api/operations/'+this.selectedExchange).then((response) => {
					const operations = response.data;
					var bFound = false;
					for (var i=0; i< operations.length; i++){
						if (operations[i].wallet_id != this.selectedWalletId || operations[i].exchange != this.selectedExchange)
							continue;
						bFound = true;
						if (operations[i].committed_outcome == "in" && !this.isRemoving){
							this.text_error = this.$t("editModalAlreadyBelongs", {wallet: this.selectedWalletId, exchange: this.selectedExchange});
							break;
						}
						if ((!operations[i].committed_outcome || operations[i].committed_outcome == "out") && this.isRemoving){
							this.text_error = this.$t("editModalDoesntBelong", {wallet: this.selectedWalletId, exchange: this.selectedExchange});
							break;
						}
						if (operations[i].status == "onreview")
							this.text_error = this.$t("editModalOperationOnGoing", {wallet: this.selectedWalletId, exchange: this.selectedExchange});
					}

						if(!bFound && this.isRemoving)
							this.text_error = this.$t("editModalDoesntBelong", {wallet: this.selectedWalletId, exchange: this.selectedExchange});

						if (!this.text_error) {
							this.getBestPool()
						} else {
							this.isSpinnerActive = false;
						}

				});
			});
		},
		getBestPool(){
			this.axios.get('/api/pool/'+this.selectedExchange).then((response) => {
				if (response.data.pool_id){
					this.bestPoolId = response.data.pool_id;
					this.rewardAmount = Number(response.data.reward_amount);
					this.isPoolAvailable = true;
				} else {
					this.text_error = this.$t("editModalNoRewardAvailable");
				}
					this.isSpinnerActive = false;
			});
		},
		handleOk(bvModalEvt){
				bvModalEvt.preventDefault();
				const base64url = require('base64url');
				const data = {
						exchange: this.selectedExchange,
						pool_id: this.bestPoolId
				};

				if (this.url_1)
					data.url_1 = this.url_1;
				if (this.url_2)
					data.url_2 = this.url_2;

				if (this.isRemoving)
					data.remove_wallet_id = this.selectedWalletId;
				else
					data.add_wallet_id = this.selectedWalletId;

				const json_string = JSON.stringify(data);
				const base64data = base64url(json_string);
				this.link = (conf.testnet ? "byteball-tn" :"byteball")+":"+conf.aa_address+"?amount="
					+this.stakeAmount+"&base64data="+base64data;
		}
	}
}
</script>

<style lang='scss' scoped>
</style>