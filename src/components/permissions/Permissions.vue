<script setup lang="ts">
import { CheckCircleIcon } from "@heroicons/vue/solid";

import GoToLinkLogo from "@/assets/go-to-link.svg";
import SolanaLogoURL from "@/assets/solana-mascot.svg";
import { Button } from "@/components/common";
import ControllerModule from "@/modules/controllers";
import { i18n } from "@/plugins/i18nPlugin";
import { getDomainFromUrl } from "@/utils/helpers";
import { getWhiteLabelLogoDark, getWhiteLabelLogoLight } from "@/utils/whitelabel";

const { t } = i18n.global;
const props = withDefaults(
  defineProps<{
    requestedFrom: string;
    logoUrl?: string;
    approvalMessage?: string;
  }>(),
  {
    logoUrl: SolanaLogoURL,
    requestedFrom: "",
    approvalMessage: "",
  }
);

const emits = defineEmits(["onApproved", "onRejected"]);

const onCancel = () => {
  emits("onRejected");
};

const onConfirm = () => {
  emits("onApproved");
};
function openLink() {
  window?.open(props?.requestedFrom, "_blank")?.focus();
}
</script>
<template>
  <div
    class="w-full h-full overflow-hidden text-left align-middle transition-all bg-white dark:bg-app-gray-800 shadow-xl flex flex-col justify-center items-center"
  >
    <div class="content-box w-full h-full transition-all bg-white dark:bg-app-gray-800 shadow-xl flex flex-col relative">
      <div class="shadow dark:shadow-dark bg-white dark:bg-app-gray-700 text-center py-6 flex flex-row justify-start items-center px-4">
        <img
          class="h-7 left-5 absolute"
          :src="(ControllerModule.isDarkMode ? getWhiteLabelLogoLight() : getWhiteLabelLogoDark()) || props.logoUrl"
          alt="Dapp Logo"
        />
        <p class="text-center font-header text-lg font-bold text-app-text-600 dark:text-app-text-dark-500 w-full">
          {{ t("dappTransfer.permission") }}
        </p>
      </div>
      <div class="mt-4 items-center px-4 flex flex-col justify-start w-full">
        <div class="flex flex-col justify-start items-start mt-4 mb-8 w-full">
          <p class="text-sm text-app-text-600 dark:text-app-text-dark-500">{{ `${t("dappInfo.requestFrom")}:` }}</p>
          <div class="w-full flex flex-row justify-between items-center bg-white dark:bg-app-gray-700 h-12 px-5 mt-3 rounded-md">
            <a :href="props.requestedFrom" target="_blank" rel="noopener noreferrer" class="text-sm text-app-primary-500">{{
              getDomainFromUrl(props.requestedFrom)
            }}</a>
            <div class="h-6 w-6 flex items-center justify-center rounded-md cursor-pointer" @click="openLink" @keydown="openLink">
              <img :src="GoToLinkLogo" alt="GoToLink" />
            </div>
          </div>
        </div>

        <div class="flex flex-col justify-start items-start w-full">
          <div class="w-full flex flex-row justify-start items-center">
            <CheckCircleIcon class="w-4 h-4 mr-2 text-app-primary-500" />
            <p class="text-sm text-app-text-600 dark:text-app-text-dark-500">
              {{ `${t("dappTransfer.data")} ${t("dappTransfer.to")} ${t("dappTransfer.approve")}` }}
            </p>
          </div>
          <div class="w-full bg-white dark:bg-app-gray-700 h-12 mt-3 rounded-md approval-msg">
            <p class="whitespace-pre-line break-all text-sm text-app-text-600 dark:text-app-text-dark-500 m-4">
              {{ props.approvalMessage }}
            </p>
          </div>
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
.approval-msg {
  overflow: auto;
  height: 100%;
  max-height: 180px !important;
  min-height: 180px;
}

@screen gt-xs {
  .content-box {
    max-width: 400px;
    max-height: 600px;
  }
}
</style>
