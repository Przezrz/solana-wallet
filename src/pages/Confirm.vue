<script setup lang="ts">
import { clusterApiUrl, Connection, TransactionMessage, VersionedMessage, VersionedTransaction } from "@solana/web3.js";
import { BROADCAST_CHANNELS, BroadcastChannelHandler, broadcastChannelOptions, POPUP_RESULT } from "@toruslabs/base-controllers";
import { BroadcastChannel } from "@toruslabs/broadcast-channel";
import log from "loglevel";
import { onErrorCaptured, onMounted, ref } from "vue";

import FullDivLoader from "@/components/FullDivLoader.vue";
import { PaymentConfirm } from "@/components/payments";
import { useEstimateChanges } from "@/components/payments/EstimateChangesComposable";
import PermissionsTx from "@/components/permissionsTx/PermissionsTx.vue";
import ControllerModule from "@/modules/controllers";
import { TransactionChannelDataType } from "@/utils/enums";
import { hideCrispButton, openCrispChat } from "@/utils/helpers";
import { decodeAllInstruction, DecodedDataType, decodeInstruction } from "@/utils/instructionDecoder";
import { FinalTxData } from "@/utils/interfaces";
import { calculateTxFee, parsingTransferAmount } from "@/utils/solanaHelpers";

const channel = `${BROADCAST_CHANNELS.TRANSACTION_CHANNEL}_${new URLSearchParams(window.location.search).get("instanceId")}`;

const finalTxData = ref<FinalTxData>();

const tx = ref<VersionedTransaction>();
const decodedInst = ref<DecodedDataType[]>();
const origin = ref("");
const network = ref("");
const loading = ref(true);

const { hasEstimationError, estimatedBalanceChange, estimationInProgress, estimateChanges } = useEstimateChanges();

onErrorCaptured(() => {
  openCrispChat();
});

onMounted(async () => {
  hideCrispButton();
  try {
    const bcHandler = new BroadcastChannelHandler(BROADCAST_CHANNELS.TRANSACTION_CHANNEL);
    const txData = await bcHandler.getMessageFromChannel<TransactionChannelDataType>();
    origin.value = txData.origin as string;
    network.value = txData.network || "";
    const connection = new Connection(txData.networkDetails?.rpcTarget || clusterApiUrl("mainnet-beta"));
    // TODO: currently, controllers does not support multi transaction flow
    if (txData.type === "sign_all_transactions") {
      if (txData.message.length === 1) {
        txData.message = (txData.message as string[]).at(0) || "";
      } else {
        const decoded = decodeAllInstruction(txData.message as string[], txData.messageOnly || false, connection);
        decodedInst.value = decoded;
        estimationInProgress.value = false;
        hasEstimationError.value = "Failed to simulate transaction for balance changes";
        loading.value = false;
        return;
      }
    }

    if (txData.messageOnly) {
      const msgObj = VersionedMessage.deserialize(Buffer.from(txData.message as string, "hex"));
      tx.value = new VersionedTransaction(msgObj);
    } else {
      tx.value = VersionedTransaction.deserialize(Buffer.from(txData.message as string, "hex"));
    }

    estimateChanges(tx.value, connection, txData.selectedAddress);
    const txFee = await calculateTxFee(tx.value.message, ControllerModule.connection);

    const { instructions } = TransactionMessage.decompile(tx.value.message, txFee.lookupArgs);

    try {
      decodedInst.value = instructions.map((inst) => {
        return decodeInstruction(inst);
      });

      finalTxData.value = await parsingTransferAmount(tx.value, txFee.fee, false, txFee.lookupArgs);
      loading.value = false;
    } catch (e) {
      log.error(e);
    }
  } catch (error) {
    log.error(error, "error in tx");
    openCrispChat();
  }
});

const approveTxn = async (): Promise<void> => {
  loading.value = true;
  const bc = new BroadcastChannel(channel, broadcastChannelOptions);
  await bc.postMessage({
    data: { type: POPUP_RESULT, approve: true },
  });
  bc.close();
};

const closeModal = async () => {
  loading.value = true;
  const bc = new BroadcastChannel(channel, broadcastChannelOptions);
  await bc.postMessage({ data: { type: POPUP_RESULT, approve: false } });
  bc.close();
};

const rejectTxn = async () => {
  closeModal();
};
</script>

<template>
  <FullDivLoader v-if="loading" />
  <PaymentConfirm
    v-else-if="finalTxData"
    :is-open="true"
    :sender-pub-key="finalTxData.slicedSenderAddress"
    :receiver-pub-key="finalTxData.slicedReceiverAddress"
    :crypto-amount="finalTxData.totalSolAmount"
    :crypto-tx-fee="finalTxData.totalSolFee"
    :is-gasless="finalTxData.isGasless"
    :decoded-inst="decodedInst || []"
    :network="network"
    :estimation-in-progress="estimationInProgress"
    :estimated-balance-change="estimatedBalanceChange"
    :has-estimation-error="hasEstimationError"
    :price-per-sol="ControllerModule.conversionRate"
    :currency="ControllerModule.currentCurrency"
    @transfer-confirm="approveTxn()"
    @transfer-cancel="rejectTxn()"
  />
  <PermissionsTx
    v-else-if="decodedInst"
    :decoded-inst="decodedInst || []"
    :origin="origin"
    :network="network"
    :estimation-in-progress="estimationInProgress"
    :estimated-balance-change="estimatedBalanceChange"
    :has-estimation-error="hasEstimationError"
    @on-approved="approveTxn()"
    @on-cancel="rejectTxn()"
  />
</template>
