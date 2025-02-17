<script setup lang="ts">
import { addressSlicer, significantDigits } from "@toruslabs/base-controllers";
import { ExternalLinkIcon } from "@toruslabs/vue-icons/basic";
import { BigNumber } from "bignumber.js";
import { computed, ref } from "vue";

import QuestionMark from "@/assets/question-circle.svg";
import SolanaLogoURL from "@/assets/solana-mascot.svg";
import { Button, NetworkDisplay } from "@/components/common";
import ControllerModule from "@/modules/controllers";
import { i18n } from "@/plugins/i18nPlugin";
import { DecodedDataType } from "@/utils/instructionDecoder";
import { AccountEstimation } from "@/utils/interfaces";
import { getWhiteLabelLogoDark, getWhiteLabelLogoLight } from "@/utils/whitelabel";

import EstimateChanges from "./EstimateChanges.vue";
import InstructionDisplay from "./InstructionDisplay.vue";

const { t } = i18n.global;

const props = withDefaults(
  defineProps<{
    receiverPubKey: string;
    cryptoAmount: number;
    cryptoTxFee: number;
    token?: string;
    isGasless?: boolean;
    isOpen?: boolean;
    tokenLogoUrl?: string;
    decodedInst: DecodedDataType[];
    network: string;
    estimationInProgress: boolean;
    estimatedBalanceChange: AccountEstimation[];
    hasEstimationError: string;
    label?: string;
    message?: string;
    memo?: string;
    pricePerToken?: number;
    pricePerSol: number;
    currency: string;
  }>(),
  {
    senderPubKey: "",
    receiverPubKey: "",
    cryptoAmount: 0.0,
    cryptoTxFee: 0,
    token: "SOL",
    isOpen: false,
    isGasless: true,
    tokenLogoUrl: "",
    label: "",
    message: "",
    memo: "",
    pricePerToken: 0,
  }
);

const expand_inst = ref(false);
const emits = defineEmits(["transferConfirm", "transferCancel"]);

const onCancel = () => {
  emits("transferCancel");
};

const onConfirm = () => {
  emits("transferConfirm");
};

const totalCryptoCostString = computed(() => {
  if (props.token !== "SOL") {
    let total = `${props.cryptoAmount} ${props.token}`;
    if (!props.isGasless) total += ` + ${props.cryptoTxFee} SOL`;
    return total;
  }
  const totalCost = new BigNumber(props.cryptoAmount).plus(props.cryptoTxFee);
  return `${totalCost.toString(10)} ${props.token}`;
});

const totalFiatCostString = computed(() => {
  if (props.token === "SOL") {
    const totalCost = props.cryptoTxFee + props.cryptoAmount;
    const total = significantDigits(totalCost * props.pricePerSol, false, 2);
    return `${total.toString(10)} ${props.currency}`;
  }
  let total = props.cryptoAmount * props.pricePerToken;
  if (!props.isGasless) total += props.cryptoTxFee * props.pricePerSol;
  return `${significantDigits(total, false, 2)} ${props.currency}`;
});

const explorerUrl = computed(() => {
  return `${ControllerModule.torus.blockExplorerUrl}/account/${props.receiverPubKey}/?cluster=${props.network}`;
});
</script>
<template>
  <div class="w-full h-full overflow-hidden bg-white dark:bg-app-gray-800 flex items-center justify-center">
    <div class="content-box h-full w-full transition-all bg-white dark:bg-app-gray-800 shadow-xl flex flex-col justify-between relative">
      <div class="shadow dark:shadow-dark text-center py-6">
        <img
          class="h-7 absolute left-5"
          :src="(ControllerModule.isDarkMode ? getWhiteLabelLogoLight() : getWhiteLabelLogoDark()) || props.tokenLogoUrl || SolanaLogoURL"
          alt="Solana Logo"
        />
        <p class="font-header text-lg font-bold text-app-text-600 dark:text-app-text-dark-500">
          {{ t("walletSettings.paymentConfirmation") }}
        </p>
      </div>
      <div class="mt-4 px-6">
        <div class="flex flex-col">
          <NetworkDisplay :network="network" />
          <div
            v-if="props.label"
            class="flex flex-row justify-between items-center w-full whitespace-no-wrap overflow-hidden overflow-ellipsis text-sm text-app-text-500 dark:text-app-text-dark-500 mt-3"
          >
            <p>{{ `${t("walletTransfer.pay")} ${t("walletActivity.to")}` }} : {{ props.label }}</p>
            <a :href="explorerUrl" target="_blank" rel="noopener noreferrer" class="underline flex"
              >{{ addressSlicer(props.receiverPubKey) }}
              <ExternalLinkIcon class="w-4 h-4" />
            </a>
          </div>
          <p v-else class="whitespace-no-wrap overflow-hidden overflow-ellipsis text-xs text-app-text-500 dark:text-app-text-dark-500 mt-3">
            {{ `${t("walletTransfer.pay")} ${t("walletActivity.to")}` }} : {{ props.receiverPubKey }}
          </p>
        </div>
      </div>
      <hr class="m-5" />
      <div class="mt-4 px-6 items-center no-scrollbar overflow-y-auto">
        <div class="flex flex-col justify-start items-start">
          <span class="flex flex-row justify-between items-center w-full text-sm text-app-text-500 dark:text-app-text-dark-500">
            <p v-if="props.message">{{ props.message }}</p>
            <p v-else>{{ t("walletTopUp.youSend") }}</p>
            <!-- <p>{{ t("walletTopUp.youSend") }}</p> -->
            <p>{{ props.cryptoAmount }} {{ props.token }}</p>
          </span>
          <div v-if="false" class="mt-3 w-full">
            <EstimateChanges
              :estimated-balance-change="props.estimatedBalanceChange"
              :has-estimation-error="props.hasEstimationError"
              :is-expand="true"
              :estimation-in-progress="props.estimationInProgress"
            />
          </div>
          <span class="flex flex-row mt-3 justify-between items-center w-full text-sm text-app-text-500 dark:text-app-text-dark-500">
            <p>{{ t("walletTransfer.transferFee") }} <img :src="QuestionMark" alt="QuestionMark" class="ml-2 float-right mt-1 cursor-pointer" /></p>
            <p>{{ props.isGasless ? "Paid by DApp" : props.cryptoTxFee + " SOL" }}</p>
          </span>

          <p
            class="text-right mt-4 text-sm cursor-pointer ml-auto text-app-primary-500"
            @click="() => (expand_inst = !expand_inst)"
            @keydown.enter="() => (expand_inst = !expand_inst)"
          >
            {{ expand_inst ? "Hide details" : "View more details" }}
          </p>
          <InstructionDisplay :is-expand="expand_inst" :decoded-inst="decodedInst" />
          <div v-if="props.memo">
            <span class="text-sm text-app-text-500 dark:text-app-text-dark-500 mt-3">Memo</span>
            <br />
            <span class="text-sm text-app-text-500">{{ props.memo }}</span>
          </div>
        </div>
      </div>
      <hr class="m-5" />
      <div class="flex px-6">
        <p class="text-sm text-app-text-600 dark:text-app-text-dark-400 font-bold">{{ t("walletTransfer.totalCost") }}</p>
        <div class="ml-auto text-right">
          <p class="text-sm font-bold text-app-text-600 dark:text-app-text-dark-400">~ {{ totalCryptoCostString }}</p>
          <p class="text-xs text-app-text-400 dark:text-app-text-dark-400">~ {{ totalFiatCostString }}</p>
        </div>
      </div>
      <div class="flex flex-row items-center my-4 mx-4">
        <Button class="flex-auto mx-1" :block="true" variant="tertiary" @click="onCancel">{{ t("dappTransfer.cancel") }}</Button>
        <Button class="flex-auto mx-1" :block="true" variant="primary" @click="onConfirm">{{ t("dappTransfer.confirm") }}</Button>
      </div>
    </div>
  </div>
</template>
<style scoped>
hr {
  border-color: #555555;
}
@screen gt-xs {
  .content-box {
    max-width: 400px;
    max-height: 600px;
  }
}
</style>
