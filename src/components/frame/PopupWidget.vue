<script setup lang="ts">
import { SolanaTransactionActivity } from "@toruslabs/solana-controllers";

import LoginUrl from "@/assets/login.png";
import SolanaLogoLight from "@/assets/solana-light.svg";
import { RoundLoader } from "@/components/common";
import { i18n } from "@/plugins/i18nPlugin";
import { getWhiteLabelLogoLight } from "@/utils/whitelabel";

import PopupWidgetPanel from "./PopupWidgetPanel.vue";

defineProps<{
  isLoggedIn: boolean;
  isLoginInProgress: boolean;
  isIframeFullScreen: boolean;
  buttonPosition: string;
  lastTransaction: SolanaTransactionActivity;
}>();
const { t } = i18n.global;
const emits = defineEmits(["closePanel", "togglePanel", "showLoginModal", "showWallet"]);
const closePanel = () => {
  emits("closePanel");
};

const togglePanel = () => {
  emits("togglePanel");
};

const onLogin = () => {
  emits("showLoginModal");
};

const showWallet = (path: string) => {
  emits("showWallet", path);
};
</script>

<template>
  <div class="torus-widget" :class="[buttonPosition]">
    <button v-if="isLoginInProgress" type="button" class="torus-widget__button">
      <RoundLoader class="w-5 h-5" color="border-white" />
    </button>
    <button v-else-if="!isLoggedIn && !isIframeFullScreen" type="button" class="torus-widget__button torus-widget__button--toggle" @click="onLogin">
      <img class="torus-widget__button-img" :src="LoginUrl" alt="Login icon" />
      <span class="torus-widget__button-text">{{ t("emailLogin.loginNoSpace") }}</span>
    </button>
    <div v-else>
      <PopupWidgetPanel
        :last-transaction="lastTransaction"
        :is-open="isLoggedIn && isIframeFullScreen"
        @on-close="closePanel"
        @show-wallet="showWallet"
      />
      <button type="button" class="torus-widget__button" @click="togglePanel">
        <img class="torus-widget__button-img" :src="getWhiteLabelLogoLight() || SolanaLogoLight" alt="Login icon" />
      </button>
    </div>
  </div>
</template>

<style scoped>
.torus-widget {
  position: absolute;
  --positionWidth: 14px;
}
.torus-widget.top-left {
  top: var(--positionWidth);
  left: var(--positionWidth);
}
.torus-widget.top-right {
  top: var(--positionWidth);
  right: var(--positionWidth);
}
.torus-widget.bottom-right {
  bottom: var(--positionWidth);
  right: var(--positionWidth);
}
.torus-widget.bottom-left {
  bottom: var(--positionWidth);
  left: var(--positionWidth);
}
.torus-widget__button {
  @apply bg-app-primary-500 flex items-center justify-center shadow;
  height: 56px;
  width: 56px;
  border-radius: 28px;
}
.torus-widget__button-img {
  @apply block;
  max-width: 30px;
  max-height: 30px;
}
.torus-widget__button-text {
  @apply hidden text-sm text-white;
}
.torus-widget__button--toggle:hover .torus-widget__button-img {
  @apply hidden;
}
.torus-widget__button--toggle:hover .torus-widget__button-text {
  @apply block;
}
</style>
