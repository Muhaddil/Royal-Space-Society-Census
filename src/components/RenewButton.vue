<script setup lang="ts">
import { computed, ref } from 'vue';
import ConfirmDialog from './ConfirmDialog.vue';
import type { CensusEntry } from '@/types/censusQueryResponse';
import { updateLocalStorage } from '@/helpers/localStorage';
import { renewCensus } from '@/helpers/renewMessage';
import { renewWebhook } from '@/variables/env';
import { useRenewDataStore } from '@/stores/renewDataStore';
import { storeToRefs } from 'pinia';
import { delay } from '@/variables/delay';
import { blurActiveElement } from '@/helpers/domHelpers';

const props = defineProps<{
  userObject: CensusEntry;
  buttonText: string;
  buttonTextFailed: string;
  buttonTextSuccess?: string;
}>();

const isSending = ref(false);
const isFailed = ref(false);
const isSuccess = ref(false);

const dialogElement = ref<HTMLDialogElement | null>(null);

function openModal() {
  // yes, this double-blur is necessary. Yes, this is stupid. I don't like it. But it is necessary. Please don't question this, or else you'll break it.
  // remove the focus on the button itself
  blurActiveElement();
  dialogElement.value?.showModal();
  // remove the focus on the close button
  blurActiveElement();
};

const closeModal = () => dialogElement.value?.close();

const renewText = computed(() => {
  if (isSending.value) return '';
  if (isFailed.value) return props.buttonTextFailed;
  if (isSuccess.value && props.buttonTextSuccess) return props.buttonTextSuccess;
  return props.buttonText;
});

const renewDataStore = useRenewDataStore();
const { tries, requested, triesExceeded } = storeToRefs(renewDataStore);

const isDisabled = computed(() => triesExceeded.value || props.userObject.renewRequested || props.userObject.renewed);

async function requestRenewal() {
  if (!renewWebhook || isDisabled.value) return;
  const playerName = props.userObject.CensusPlayer;
  isSending.value = true;
  try {
    await renewCensus(props.userObject);
    isSuccess.value = true;
    props.userObject.renewRequested = true;
    updateLocalStorage(requested, tries, playerName);
  } catch (error) {
    isFailed.value = true;
    console.warn(error);
  } finally {
    isSending.value = false;
    setTimeout(() => {
      isSuccess.value = false;
      isFailed.value = false;
    }, delay);
  }
}
</script>

<template>
  <div>
    <button
      :aria-busy="isSending"
      :class="{ 'is-danger': isFailed }"
      :disabled="isDisabled"
      class="renew-button"
      type="button"
      @click="openModal"
    >
      {{ renewText }}
    </button>

    <!--teleporting out of the div in order to remove any unwanted hover effects-->
    <Teleport to="body">
      <dialog
        ref="dialogElement"
        @click.self="closeModal"
      >
        <ConfirmDialog
          :user-name="userObject.CensusPlayer"
          @confirm="requestRenewal"
        />
      </dialog>
    </Teleport>
  </div>
</template>

<style scoped>
.renew-button {
  margin: 0;
  width: 100%;
}
</style>
