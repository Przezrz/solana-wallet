<script setup lang="ts">
import { CheckCircleIcon } from "@heroicons/vue/solid";
import { ref } from "vue";

import GoToLinkLogo from "@/assets/go-to-link.svg";
import SolanaLogoURL from "@/assets/solana-mascot.svg";
import { Button } from "@/components/common";
import ControllerModule from "@/modules/controllers";
import { i18n } from "@/plugins/i18nPlugin";
import { getDomainFromUrl } from "@/utils/helpers";
import { DecodedDataType } from "@/utils/instructionDecoder";
import { AccountEstimation } from "@/utils/interfaces";
import { getWhiteLabelLogoDark, getWhiteLabelLogoLight } from "@/utils/whitelabel";

import NetworkDisplay from "../common/NetworkDisplay.vue";
import EstimateChanges from "../payments/EstimateChanges.vue";
import InstructionDisplay from "../payments/InstructionDisplay.vue";

const { t } = i18n.global;
const props = withDefaults(
  defineProps<{
    logoUrl?: string;
    decodedInst: DecodedDataType[];
    origin: string;
    network: string;
    estimationInProgress: boolean;
    estimatedBalanceChange: AccountEstimation[];
    hasEstimationError: string;
    message?: string;
    label?: string;
  }>(),
  {
    logoUrl: SolanaLogoURL,
    message: "",
    label: "",
  }
);

const expand_inst = ref(false);
const emits = defineEmits(["onApproved", "onCancel"]);

const onCancel = () => {
  emits("onCancel");
};

const onConfirm = () => {
  emits("onApproved");
};
function openLink() {
  window?.open(props?.origin, "_blank")?.focus();
}
</script>
<template>
  <div class="w-full h-full overflow-hidden text-left align-middle bg-white dark:bg-app-gray-800 shadow-xl flex flex-col justify-center items-center">
    <div class="content-box w-full h-full transition-all bg-white dark:bg-app-gray-800 shadow-xl flex flex-col relative">
      <div class="shadow dark:shadow-dark bg-white dark:bg-app-gray-700 text-center py-6 flex flex-row justify-start items-center px-4">
        <img
          class="h-7 left-5 absolute"
          :src="(ControllerModule.isDarkMode ? getWhiteLabelLogoLight() : getWhiteLabelLogoDark()) || props.logoUrl || SolanaLogoURL"
          alt="Dapp Logo"
        />
        <p class="text-center font-header text-lg font-bold text-app-text-600 dark:text-app-text-dark-500 w-full">
          {{ `${t("dappProvider.confirm")} ${t("dappProvider.permission")}` }}
        </p>
      </div>
      <div class="mt-4 items-center px-4 flex flex-col justify-start h-full no-scrollbar overflow-y-auto">
        <div class="flex flex-col justify-start items-start w-full mt-4 mb-6">
          <NetworkDisplay :network="network" />
          <p class="text-sm text-app-text-600 dark:text-app-text-dark-500">{{ t("dappProvider.requestFrom") }} {{ props.label }}</p>

          <div class="w-full flex flex-row justify-between items-center bg-white dark:bg-app-gray-700 h-12 px-5 mt-3 rounded-md">
            <a :href="props.origin" target="_blank" rel="noopener noreferrer" class="text-sm text-app-primary-500">{{
              getDomainFromUrl(props.origin)
            }}</a>
            <div class="h-6 w-6 flex items-center justify-center rounded-md cursor-pointer" @click="openLink" @keydown="openLink">
              <img :src="GoToLinkLogo" alt="GoToLink" />
            </div>
          </div>
        </div>
        <div class="mb-5 w-full">
          <EstimateChanges
            :estimated-balance-change="props.estimatedBalanceChange"
            :has-estimation-error="props.hasEstimationError"
            :is-expand="true"
            :estimation-in-progress="props.estimationInProgress"
          />
        </div>

        <!-- Specific for Solana Pay -->
        <div v-if="message" class="w-full mb-5 text-sm text-app-text-600 dark:text-app-text-dark-500">
          <div>Message :</div>
          {{ message }}
        </div>

        <div class="flex flex-col justify-start items-start w-full">
          <div class="w-full flex flex-row justify-start items-center">
            <CheckCircleIcon class="w-4 h-4 mr-2 text-app-primary-500" />
            <p class="text-sm text-app-text-600 dark:text-app-text-dark-500">
              {{ decodedInst.length }} {{ t("walletSettings.transactionInstructions") }}
            </p>
          </div>
          <p
            class="text-right mt-4 text-sm cursor-pointer text-app-primary-500 w-full"
            @click="() => (expand_inst = !expand_inst)"
            @keydown="() => (expand_inst = !expand_inst)"
          >
            {{ expand_inst ? t("dappPermission.hideDetails") : t("dappPermission.viewMoreDetails") }}
          </p>
          <InstructionDisplay :is-expand="expand_inst" :decoded-inst="decodedInst" />
        </div>
      </div>
      <hr class="mx-6 mt-auto" />
      <div class="flex flex-row items-center my-4 mx-4">
        <Button class="flex-auto mx-1" :block="true" variant="tertiary" @click="onCancel">{{ t("dappTransfer.cancel") }}</Button>
        <Button class="flex-auto mx-1" :block="true" variant="primary" @click="onConfirm">{{ t("dappTransfer.approve") }}</Button>
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
