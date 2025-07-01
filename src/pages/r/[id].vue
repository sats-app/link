<template>
  <v-container class="fill-height" max-width="600">
    <v-row align="center" justify="center" class="h-100">
      <v-col cols="12" class="text-center">
        <v-progress-circular
          v-if="loading"
          indeterminate
          color="primary"
          size="64"
        />
        <div v-else>
          <template v-if="paid">
            <v-card class="mx-auto mb-6 py-6 px-4 d-flex flex-column align-center" max-width="500">
              <v-icon color="success" size="64" class="mb-4">mdi-check-circle</v-icon>
              <div class="text-h5 font-weight-bold mb-2">Payment Sent</div>
              <div class="mb-2">This payment request has been paid.</div>
            </v-card>
          </template>
          <template v-else-if="!error">
            <h1 class="text-h4 font-weight-bold mb-6">
              Pay {{ paymentReqValue }} sat.
            </h1>
            <v-card
              v-if="mintInfo"
              class="mx-auto mb-6 py-6 px-4 d-flex flex-column align-center"
              max-width="500"
            >
              <v-avatar size="64" class="mb-4" v-if="mintInfo.icon_url">
                <v-img :src="mintInfo.icon_url" />
              </v-avatar>
              <v-list-item-title class="text-h6 font-weight-bold mb-1">
                {{ mintInfo.name }}
              </v-list-item-title>
              <v-list-item-subtitle class="mb-2">
                {{ paymentRequestMintUrl }}
              </v-list-item-subtitle>
            </v-card>
            <v-btn
              color="primary"
              class="mb-3 mx-auto w-100"
              @click="payInApp"
              prepend-icon="mdi-open-in-app"
              size="large"
            >
              Pay in App
            </v-btn>
            <v-btn
              color="primary"
              class="mb-6 mx-auto w-100"
              variant="outlined"
              @click="payNow"
              size="large"
            >
              Pay Now
            </v-btn>
          </template>
          <v-card v-if="error" class="mx-auto" max-width="500">
            <v-card-text>
              <div class="text-error mb-4">
                <span v-if="error === 'expired'">Request expired or not found.</span>
                <span v-else-if="error === 'server'">Server error. Please try again later.</span>
                <span v-else>Unknown error.</span>
              </div>
            </v-card-text>
          </v-card>
          <!-- Pay Now Dialog -->
          <v-dialog v-model="payNowDialog" max-width="400">
            <v-card>
              <v-card-title class="text-h6 font-weight-bold">Pay Now</v-card-title>
              <v-card-text>
                <div v-if="payNowLoading" class="d-flex flex-column align-center justify-center py-6">
                  <v-progress-circular indeterminate color="primary" size="48" class="mb-2" />
                  <div>Waiting for quote...</div>
                </div>
                <div v-else>
                  <div v-if="payNowError" class="text-error mb-2">{{ payNowError }}</div>
                  <div v-else>
                    <div v-if="payNowRequest" class="d-flex flex-column align-center justify-center">
                      <qrcode-vue :value="payNowRequest" :size="200" class="mb-2" />
                      <div class="text-caption mb-2">Scan to pay</div>
                      <v-sheet class="pa-2 d-flex align-center justify-center" color="#f5f5f5" rounded>
                        <span class="text-truncate" style="max-width: 220px;">{{ payNowRequest }}</span>
                        <v-btn icon size="small" @click="copyPayNowRequest" class="ml-2">
                          <v-icon v-if="!payNowCopySuccess">mdi-content-copy</v-icon>
                          <v-icon v-else color="success">mdi-check</v-icon>
                        </v-btn>
                      </v-sheet>
                    </div>
                  </div>
                </div>
              </v-card-text>
              <v-card-actions>
                <v-spacer />
                <v-btn text @click="payNowDialog = false">Close</v-btn>
              </v-card-actions>
            </v-card>
          </v-dialog>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts" setup>
import { onMounted, ref } from "vue";
import { useRoute } from "vue-router";
import {
  PaymentRequest,
  CashuWallet,
  type GetInfoResponse,
  CashuMint,
  decodePaymentRequest,
  type MintQuoteResponse,
} from "@cashu/cashu-ts";

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || "";
const route = useRoute();
const loading = ref(true);
const error = ref<string | null>(null);
const paymentRequest = ref<PaymentRequest | null>(null);
const paymentReqValue = ref<number | null>(null);
const mintInfo = ref<GetInfoResponse | null>(null);
const encodedPaymentRequest = ref<string>("");
const paymentRequestMintUrl = ref<string>("");
const paid = ref(false);

async function fetchMintInfo(mintUrl: string) {
  const wallet = new CashuWallet(new CashuMint(mintUrl));
  mintInfo.value = await wallet.getMintInfo();
}

function base64UrlToUuid(base64: string): string {
  const bin = atob(base64.replace(/-/g, "+").replace(/_/g, "/"));
  if (bin.length !== 16) throw new Error("Invalid base64 UUID");
  const hex = Array.from(bin, (c) =>
    c.charCodeAt(0).toString(16).padStart(2, "0"),
  ).join("");
  return [
    hex.slice(0, 8),
    hex.slice(8, 12),
    hex.slice(12, 16),
    hex.slice(16, 20),
    hex.slice(20),
  ].join("-");
}

async function fetchPaymentRequest() {
  loading.value = true;
  error.value = null;
  paid.value = false;
  try {
    const params = route.params as { id?: string | string[] };
    const idParam = params.id;
    const idB64 = typeof idParam === "string" ? idParam : Array.isArray(idParam) ? idParam[0] : undefined;
    if (!idB64) {
      error.value = "unknown";
      loading.value = false;
      return;
    }
    const id = base64UrlToUuid(idB64);
    const res = await fetch(`${API_BASE_URL}/payment-requests/${id}`);
    if (res.status === 404) {
      error.value = "expired";
    } else if (res.status === 500) {
      error.value = "server";
    } else if (!res.ok) {
      error.value = "unknown";
    } else {
      const data = await res.json();
      if (data["token"] != null) {
        paid.value = true;
        return;
      }
      encodedPaymentRequest.value = data["encoded"];
      let decoded = decodePaymentRequest(encodedPaymentRequest.value);
      paymentRequest.value = decoded;
      paymentReqValue.value = decoded.amount ?? null;
      if (decoded.mints != null && decoded.mints.length > 0) {
        paymentRequestMintUrl.value = decoded.mints[0];
        await fetchMintInfo(decoded.mints[0]);
      } 
    }
  } catch (e) {
    console.error("Error fetching payment request:", e);
    error.value = "server";
  } finally {
    loading.value = false;
  }
}

async function payInApp() {
  if (!encodedPaymentRequest.value) return;
  window.location.href = `cashu:${encodeURIComponent(encodedPaymentRequest.value)}`;
}


// Dialog state
const payNowDialog = ref(false);
const payNowLoading = ref(false);
const payNowQuote = ref<any>(null);
const payNowRequest = ref("");
const payNowCopySuccess = ref(false);
const payNowError = ref<string | null>(null);
let payNowWallet: CashuWallet | null = null;
let payNowQuoteId: string | null = null;
let payNowMintUrl: string | null = null;

async function payNow() {
  if (!paymentRequest.value || !paymentRequestMintUrl.value) return;
  payNowDialog.value = true;
  payNowLoading.value = true;
  payNowError.value = null;
  payNowQuote.value = null;
  payNowRequest.value = "";
  payNowCopySuccess.value = false;
  try {
    payNowMintUrl = paymentRequestMintUrl.value;
    payNowWallet = new CashuWallet(new CashuMint(payNowMintUrl));
    const amount = (paymentRequest.value.amount ?? 0) * 1000;
    const quote = await payNowWallet.createMintQuote(amount);
    payNowQuote.value = quote;
    payNowQuoteId = quote.quote;
    payNowRequest.value = quote.request;
    payNowWallet.onMintQuotePaid(
      quote.quote,
      (payload: MintQuoteResponse) => {
        (async () => {
          try {
            payNowLoading.value = true;
            const proofs = await payNowWallet!.mintProofs(amount, payNowQuoteId!);
            if (
              paymentRequest.value &&
              paymentRequest.value.transport &&
              paymentRequest.value.transport.length > 0
            ) {
              const url = paymentRequest.value.transport[0].target;
              const payload = { proofs };
              await fetch(url, {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify(payload),
              });
            }
            payNowDialog.value = false;
            paid.value = true;
          } catch (err) {
            payNowError.value = "Failed to mint proofs or send payment.";
          } finally {
            payNowLoading.value = false;
          }
        })();
      },
      (err: any) => {
        payNowError.value = err?.message || "Error waiting for payment.";
        payNowLoading.value = false;
      }
    );
  } catch (err) {
    payNowError.value = "Failed to create mint quote.";
  } finally {
    payNowLoading.value = false;
  }
}

function copyPayNowRequest() {
  if (!payNowRequest.value) return;
  navigator.clipboard.writeText(payNowRequest.value).then(() => {
    payNowCopySuccess.value = true;
    setTimeout(() => (payNowCopySuccess.value = false), 1500);
  });
}

onMounted(fetchPaymentRequest);
</script>
