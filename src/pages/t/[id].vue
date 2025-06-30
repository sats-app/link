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
          <template v-if="!error">
            <h1 class="text-h4 font-weight-bold mb-6">
              Claim {{ tokenValue }} sat.
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
                {{ token?.mint }}
              </v-list-item-subtitle>
            </v-card>
            <v-btn
              color="primary"
              class="mb-3 mx-auto w-100"
              @click="claimInApp"
              prepend-icon="mdi-open-in-app"
              size="large"
            >
              Claim in App
            </v-btn>
            <v-btn
              color="primary"
              class="mb-6 mx-auto w-100"
              variant="outlined"
              @click="transferNow"
              size="large"
            >
              Transfer Now
            </v-btn>
            <v-dialog v-model="showTransfer" max-width="400">
              <v-card>
                <v-card-title>Transfer Token</v-card-title>
                <v-card-text>
                  <v-text-field v-model="invoice" label="Invoice" />
                  <div v-if="transferError" class="text-error">
                    {{ transferError }}
                  </div>
                  <div v-if="transferSuccess" class="text-success">
                    {{ transferSuccess }}
                  </div>
                </v-card-text>
                <v-card-actions>
                  <v-btn color="primary" @click="submitTransfer"
                    >Submit</v-btn
                  >
                  <v-btn text @click="backToButtons">Cancel</v-btn>
                </v-card-actions>
              </v-card>
            </v-dialog>
          </template>
          <v-card v-if="error" class="mx-auto" max-width="500">
            <v-card-text>
              <div class="text-error mb-4">
                <span v-if="error === 'expired'"
                  >Token expired or not found.</span
                >
                <span v-else-if="error === 'server'"
                  >Server error. Please try again later.</span
                >
                <span v-else-if="error === 'spent'"
                  >Token has already been claimed.</span
                >
                <span v-else>Unknown error.</span>
              </div>
            </v-card-text>
          </v-card>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts" setup>
import { onMounted, ref, watch } from "vue";
import { useRoute } from "vue-router";
import {
  type Token,
  CashuWallet,
  type GetInfoResponse,
  CashuMint,
  getDecodedTokenBinary,
  CheckStateEnum,
  MeltQuoteState,
} from "@cashu/cashu-ts";

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || "";
const route = useRoute();
const loading = ref(true);
const error = ref<string | null>(null);
const token = ref<Token | null>(null);
const tokenValue = ref<number | null>(null);
const showTransfer = ref(false);
const invoice = ref("");
const mintInfo = ref<GetInfoResponse | null>(null);
const wallet = ref<CashuWallet | null>(null);
const transferError = ref<string | null>(null);
const transferSuccess = ref<string | null>(null);

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

function base64UrlToBytes(base64: string): Uint8Array {
  const bin = atob(base64.replace(/-/g, "+").replace(/_/g, "/"));
  return new Uint8Array([...bin].map((c) => c.charCodeAt(0)));
}

async function decryptToken(
  encryptedTokenBytes: Uint8Array,
  passwordBytes: Uint8Array,
  idBytes: Uint8Array,
): Promise<Uint8Array> {
  // 1. Derive key using PBKDF2-HMAC-SHA256, 10,000 iterations, 128 bits
  const keyMaterial = await window.crypto.subtle.importKey(
    "raw",
    passwordBytes,
    { name: "PBKDF2" },
    false,
    ["deriveKey"],
  );
  const key = await window.crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt: idBytes,
      iterations: 10000,
      hash: "SHA-256",
    },
    keyMaterial,
    { name: "AES-GCM", length: 128 },
    false,
    ["decrypt"],
  );

  // 2. Parse nonce, ciphertext, mac
  const nonce = encryptedTokenBytes.slice(0, 12); // AES-GCM nonce is 12 bytes
  const mac = encryptedTokenBytes.slice(-16); // 16 bytes MAC (tag)
  const ciphertext = encryptedTokenBytes.slice(12, -16); // rest is ciphertext
  // 3. Decrypt
  const decrypted = await window.crypto.subtle.decrypt(
    {
      name: "AES-GCM",
      iv: nonce,
      tagLength: 128,
    },
    key,
    // AES-GCM expects ciphertext + tag
    new Uint8Array([...ciphertext, ...mac]),
  );
  return new Uint8Array(decrypted);
}

async function fetchMintInfo(mintUrl: string) {
  const wallet = new CashuWallet(new CashuMint(mintUrl));
  mintInfo.value = await wallet.getMintInfo();
}

async function fetchToken() {
  loading.value = true;
  error.value = null;
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
    const idBytes = base64UrlToBytes(idB64);
    const fragment = window.location.hash.slice(1);
    let passwordBytes: Uint8Array | undefined;
    if (fragment) {
      passwordBytes = base64UrlToBytes(fragment);
    } else {
      error.value = "unknown";
      loading.value = false;
      return;
    }
    const res = await fetch(`${API_BASE_URL}/tokens/${id}`);
    if (res.status === 404) {
      error.value = "expired";
    } else if (res.status === 500) {
      error.value = "server";
    } else if (!res.ok) {
      error.value = "unknown";
    } else {
      const data = await res.json();
      const encryptedToken = data["encrypted_token"];
      const encryptedTokenBytes = Uint8Array.from(atob(encryptedToken), (c) =>
        c.charCodeAt(0),
      );
      const decryptedBytes = await decryptToken(
        encryptedTokenBytes,
        passwordBytes!,
        idBytes,
      );
      const decodedToken = getDecodedTokenBinary(decryptedBytes);
      token.value = decodedToken;
      tokenValue.value = decodedToken.proofs
        .map((p: any) => p.amount)
        .reduce((a: number, b: number) => a + b, 0);

      wallet.value = new CashuWallet(new CashuMint(decodedToken.mint));
      try {
        const states = await wallet.value.checkProofsStates(decodedToken.proofs);
        if (states.some((p) => p.state === CheckStateEnum.SPENT)) {
          error.value = "spent";
          return;
        }
      } catch (e) {
        console.error("Error checking proofs states:", e);
      }
      await fetchMintInfo(decodedToken.mint);
    }
  } catch (e) {
    console.error("Error decrypting token:", e);
    error.value = "server";
  } finally {
    loading.value = false;
  }
}

async function claimInApp() {
  if (!token.value) return;
  const { getEncodedTokenV4 } = await import("@cashu/cashu-ts");
  const encodedToken = getEncodedTokenV4(token.value);
  window.location.href = `cashu:${encodeURIComponent(encodedToken)}`;
}

function transferNow() {
  showTransfer.value = true;
}

async function submitTransfer() {
  transferError.value = null;
  transferSuccess.value = null;
  if (!wallet.value || !token.value) {
    transferError.value = "Wallet or token not loaded.";
    return;
  }
  if (!invoice.value) {
    transferError.value = "Invoice is required.";
    return;
  }
  try {
    const quote = await wallet.value.createMeltQuote(invoice.value);
    const totalRequired = quote.amount + quote.fee_reserve;
    if (tokenValue.value === null || tokenValue.value < totalRequired) {
      transferError.value = `Insufficient value. Required: ${totalRequired}, available: ${tokenValue.value}`;
      return;
    }
    const meltResult = await wallet.value.meltProofs(quote, token.value.proofs);
    if (meltResult && meltResult.quote.state === MeltQuoteState.PAID) {
      transferSuccess.value = "Transfer successful!";
      showTransfer.value = false;
    } else {
      transferError.value = "Transfer failed. Invoice not paid.";
    }
  } catch (e: any) {
    transferError.value = e?.message || "Error during transfer.";
  }
}

function backToButtons() {
  showTransfer.value = false;
  invoice.value = "";
}

onMounted(fetchToken);
</script>
