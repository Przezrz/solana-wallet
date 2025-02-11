<script setup lang="ts">
import { createTransfer, parseURL, TransactionRequestURL, TransferRequestURL } from "@solana/pay";
import { LAMPORTS_PER_SOL, PublicKey, TransactionMessage, VersionedTransaction } from "@solana/web3.js";
import { findAllLookUpTable } from "@toruslabs/solana-controllers";
import log from "loglevel";
import { onMounted, ref, watch } from "vue";
import { useRouter } from "vue-router";

import MessageModal from "@/components/common/MessageModal.vue";
import ControllerModule from "@/modules/controllers";
import { STATUS } from "@/utils/enums";
import { DecodedDataType, decodeInstruction } from "@/utils/instructionDecoder";
import { parseSolanaPayRequestLink } from "@/utils/solanaHelpers";

import FullDivLoader from "../FullDivLoader.vue";
import PermissionsTx from "../permissionsTx/PermissionsTx.vue";
import { useEstimateChanges } from "./EstimateChangesComposable";
import PaymentConfirm from "./PaymentConfirm.vue";

const { hasEstimationError, estimatedBalanceChange, estimationInProgress, estimateChanges } = useEstimateChanges();
const props = withDefaults(
  defineProps<{
    requestLink: string;
  }>(),
  {}
);
const invalidLink = ref("");
const transaction = ref<VersionedTransaction>();
const decodedInstructions = ref<DecodedDataType[]>([]);
const requestParams = ref<TransferRequestURL>();
const linkParams = ref<{ icon: string; label: string; decodedInst: DecodedDataType[]; origin: string; message: string }>();
const symbol = ref<string>("");
const pricePerToken = ref(0);
const emits = defineEmits(["onApproved", "onCloseModal"]);
const closeModal = () => {
  requestParams.value = undefined;
  linkParams.value = undefined;
  emits("onCloseModal");
};
const onCancel = () => {
  closeModal();
};
const onConfirm = () => {
  emits("onApproved", transaction.value);
};

const estimateTxFee = ref(0);
const router = useRouter();
watch(transaction, async () => {
  if (transaction.value) {
    const args = await findAllLookUpTable(ControllerModule.connection, transaction.value.message);
    const transactionMessage = TransactionMessage.decompile(transaction.value.message, args);

    const legacyMessage = transactionMessage.compileToLegacyMessage();

    const response = await ControllerModule.connection.getFeeForMessage(legacyMessage);
    estimateTxFee.value = response.value / LAMPORTS_PER_SOL;

    decodedInstructions.value = transactionMessage.instructions.map((inst) => decodeInstruction(inst));
  }
});
onMounted(async () => {
  // set loading
  invalidLink.value = "";
  const { requestLink } = props;
  try {
    const pubkey = new PublicKey(requestLink);
    router.push({
      name: "walletTransfer",
      query: {
        receiverPubKey: pubkey.toBase58(),
      },
    });
  } catch (e) {
    log.info(e);
  }
  try {
    if (!requestLink.length) {
      // set loaded
      invalidLink.value = "Invalid Link";
      return;
    }

    // check for publickey format, redirect to transfer page
    try {
      const address = new PublicKey(requestLink);
      router.push({
        name: "walletTransfer",
        query: {
          receiverPubKey: address.toBase58(),
        },
      });
      return;
    } catch (e) {}

    const parsed = parseURL(requestLink);

    if ((parsed as TransactionRequestURL).link) {
      // request link is an url. fetch transaction from url
      const result = await parseSolanaPayRequestLink(parsed as TransactionRequestURL, ControllerModule.selectedAddress, ControllerModule.connection);
      if (result.transaction.feePayer && result.transaction.recentBlockhash) {
        const messageV0 = new TransactionMessage({
          payerKey: result.transaction.feePayer,
          instructions: result.transaction.instructions,
          recentBlockhash: result.transaction.recentBlockhash,
        }).compileToV0Message();
        log.info(result);
        transaction.value = new VersionedTransaction(messageV0);
        linkParams.value = {
          icon: result.icon,
          label: result.label,
          origin: result.link.origin,
          decodedInst: result.decodedInst,
          message: result.message || "",
        };
      }
    } else {
      // parse solanapay format
      const result = parsed as TransferRequestURL;
      const { recipient, splToken, reference, memo, amount, message } = result;
      // redirect to transfer page if amount is not available
      if (amount === undefined) {
        // redirect to transfer page
        if (splToken) {
          const tokenOwned = ControllerModule.torusState.TokensTrackerState.tokens;
          if (!tokenOwned || !tokenOwned[ControllerModule.selectedAddress].find((v) => v.mintAddress === splToken.toBase58()))
            throw new Error("sender not found");
        }
        router.push({
          name: "walletTransfer",
          query: {
            receiverPubKey: recipient.toBase58(),
            mint: splToken?.toBase58(),
            reference: reference?.map((item) => item.toBase58()),
            memo,
            message,
          },
        });
        return;
      }
      // create transaction based on the solanapay format
      const tx = await createTransfer(ControllerModule.connection, new PublicKey(ControllerModule.selectedAddress), {
        recipient,
        amount,
        splToken,
        reference,
        memo,
      });
      // get symbol if spl token
      if (splToken) {
        // const tokenInfo = await getTokenInfo(splToken.toBase58());
        const tokenInfo = ControllerModule.torusState.TokenInfoState.tokenInfoMap[splToken.toBase58()];
        if (tokenInfo.symbol) symbol.value = tokenInfo.symbol;
        else {
          symbol.value = `${splToken.toBase58().substring(0, 5)}...`;
        }
        pricePerToken.value =
          ControllerModule.torusState.CurrencyControllerState.tokenPriceMap[splToken.toBase58()][ControllerModule.currentCurrency] || 0;
      }
      // set blockhash and feepayer
      const block = await ControllerModule.connection.getLatestBlockhash();
      tx.recentBlockhash = block.blockhash;
      tx.lastValidBlockHeight = block.lastValidBlockHeight;
      tx.feePayer = new PublicKey(ControllerModule.selectedAddress);
      // update ref state (ui)
      const messageV0 = new TransactionMessage({
        payerKey: new PublicKey(ControllerModule.selectedAddress),
        instructions: tx?.instructions,
        recentBlockhash: block.blockhash,
      }).compileToV0Message();
      transaction.value = new VersionedTransaction(messageV0);
      requestParams.value = result;
      log.info(result);
    }
    // estimate changes if transaction available
    if (transaction.value) estimateChanges(transaction.value, ControllerModule.connection, ControllerModule.selectedAddress);
  } catch (e) {
    // invalidLink.value = true;
    if (e instanceof Error) {
      if (e.name === "TokenInvalidAccountOwnerError") e.message = "mint not initialize";
      invalidLink.value = e.message || e.name;
    } else {
      invalidLink.value = "Invalid Link";
    }
    log.error(e);
  }
});
</script>

<template>
  <div v-if="invalidLink" class="">
    <MessageModal :is-open="!!invalidLink || false" :title="invalidLink" :status="STATUS.ERROR" class="capitalize" @on-close="onCancel" />
  </div>

  <PaymentConfirm
    v-else-if="requestParams"
    class="payContainer"
    :is-gasless="false"
    :label="requestParams.label"
    :message="requestParams.message"
    :memo="requestParams.memo"
    :receiver-pub-key="requestParams.recipient.toBase58()"
    :crypto-amount="requestParams.amount?.toNumber()"
    :token="symbol || 'SOL'"
    :crypto-tx-fee="estimateTxFee"
    :network="ControllerModule.selectedNetworkDisplayName"
    :decoded-inst="decodedInstructions"
    :estimation-in-progress="estimationInProgress"
    :estimated-balance-change="estimatedBalanceChange"
    :has-estimation-error="hasEstimationError"
    :price-per-sol="ControllerModule.conversionRate"
    :currency="ControllerModule.currentCurrency"
    :price-per-token="pricePerToken"
    @transfer-confirm="onConfirm"
    @transfer-cancel="onCancel"
    @on-close-modal="onCancel"
  />
  <PermissionsTx
    v-else-if="linkParams"
    :decoded-inst="linkParams.decodedInst"
    :network="ControllerModule.selectedNetworkDisplayName"
    :logo-url="linkParams.icon"
    :label="linkParams.label"
    :origin="linkParams.origin"
    :message="linkParams.message"
    :estimation-in-progress="estimationInProgress"
    :estimated-balance-change="estimatedBalanceChange"
    :has-estimation-error="hasEstimationError"
    @on-approved="onConfirm"
    @on-cancel="onCancel"
    @on-close-modal="onCancel"
  />
  <FullDivLoader v-else />
</template>
<style>
.wrapper {
  @apply overflow-hidden align-middle transform shadow-xl flex-col justify-center items-center  text-center py-6;
}
</style>
