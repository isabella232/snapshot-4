<script setup>
import { ref, watch, onMounted, onUnmounted, computed } from 'vue';
import { useRouter, useRoute } from 'vue-router';
import { useWeb3 } from '@/composables/useWeb3';
import { useModal } from '@/composables/useModal';
import { useI18n } from '@/composables/useI18n';
import { useEns } from '@/composables/useEns';
import { useSpaces } from '@/composables/useSpaces';
import { useSpaceController } from '@/composables/useSpaceController';

const router = useRouter();
const route = useRoute();
const { web3, web3Account } = useWeb3();
const { modalAccountOpen } = useModal();
const { loadOwnedEnsDomains, ownedEnsDomains } = useEns();
const { setPageTitle } = useI18n();
const { spaces } = useSpaces();
const { ensAddress } = useSpaceController();

onMounted(() => {
  setPageTitle('page.title.setup');
});

const ownedEnsDomainsNoExistingSpace = computed(() => {
  //  filter ownedEnsDomains with spaces
  return ownedEnsDomains.value.filter(
    d => !Object.keys(spaces.value).includes(d.name)
  );
});

// used either on click on existing owned domain OR once a newly registered
// domain is returned by the ENS subgraph.
const goToStepTwo = key => {
  router.push({ name: 'setup', params: { step: 'controller' } });
  ensAddress.value = key;
};

// input for new domain registration
const newDomain = ref('');

// indicates whether to periodically check for new domains
// (after clicking on "Register")
const waitingForRegistration = ref(false);
// handle periodic lookup (every 10s) while registering new domain
let waitingForRegistrationInterval;
const waitForRegistration = () => {
  waitingForRegistration.value = true;
  clearInterval(waitingForRegistrationInterval);
  waitingForRegistrationInterval = setInterval(loadOwnedEnsDomains, 10000);
};

// If after loading domains, we have more than before, there was a new registration.
// If the new domain matches the one that was input, directly jump to settings page.
watch(ownedEnsDomains, (newVal, oldVal) => {
  if (newVal.length > oldVal.length) {
    waitingForRegistration.value = false;
    clearInterval(waitingForRegistrationInterval);
    if (newVal.find(d => d.name === newDomain.value)) {
      goToStepTwo(newDomain.value);
    }
  }
});

// load domains initially and update on account change
// using finally() here because await at top level would require the component to be inside a <Suspense> block
// https://v3.vuejs.org/guide/migration/suspense.html#introduction
const loadingOwnedEnsDomains = ref(true);
loadOwnedEnsDomains().finally(() => (loadingOwnedEnsDomains.value = false));
watch(web3Account, async () => {
  // Reset ensAddress to empty string
  ensAddress.value = '';

  loadingOwnedEnsDomains.value = true;
  await loadOwnedEnsDomains();
  loadingOwnedEnsDomains.value = false;
  waitingForRegistration.value = false;
});

// stop lookup when leaving page
onUnmounted(() => clearInterval(waitingForRegistrationInterval));
</script>

<template>
  <TheLayout>
    <template #content-left>
      <div class="px-4 md:px-0">
        <h1 v-text="$t('setup.createASpace')" class="mb-4" />
      </div>
      <template v-if="web3Account">
        <LoadingRow v-if="loadingOwnedEnsDomains" block />
        <!-- Step two - setup space controller -->
        <SetupController
          v-else-if="route.params.step === 'controller' && ensAddress"
          :ensAddress="ensAddress"
          :web3Account="web3Account"
        />

        <!-- Step three - setup space profile -->
        <SetupProfile
          v-else-if="route.params.step === 'profile' && ensAddress"
          :ensAddress="ensAddress"
          :web3Account="web3Account"
        />
        <BaseBlock v-else>
          <div v-if="ownedEnsDomainsNoExistingSpace.length">
            <div class="mb-3">
              {{
                $t(
                  ownedEnsDomainsNoExistingSpace.length > 1
                    ? 'setup.chooseExistingEns'
                    : 'setup.useSingleExistingEns'
                )
              }}
            </div>
            <div class="space-y-2">
              <BaseButton
                v-for="(ens, i) in ownedEnsDomainsNoExistingSpace"
                :key="i"
                @click="goToStepTwo(ens.name)"
                class="w-full flex items-center justify-between"
                :primary="ownedEnsDomainsNoExistingSpace.length === 1"
              >
                {{ ens.name }}
                <BaseIcon name="go" size="22" class="-mr-2" />
              </BaseButton>
            </div>
            <div class="my-3">
              {{ $t('setup.orReigsterNewEns') }}
            </div>
            <RegisterENS
              v-model="newDomain"
              @waitForRegistration="waitForRegistration"
            />
          </div>
          <div v-else>
            <div class="mb-3">
              {{ $t('setup.toCreateASpace') }}
            </div>
            <RegisterENS
              v-model="newDomain"
              @waitForRegistration="waitForRegistration"
            />
          </div>
        </BaseBlock>
      </template>
      <BaseBlock v-else>
        <BaseButton
          @click="modalAccountOpen = true"
          :loading="web3.authLoading"
          class="w-full"
          primary
        >
          {{ $t('connectWallet') }}
        </BaseButton>
      </BaseBlock>
    </template>
    <template #sidebar-right>
      <BaseBlock class="text-skin-text">
        <BaseIcon
          name="gitbook"
          size="24"
          class="text-skin-text pr-2 !align-middle"
        />
        <span v-html="$t('setup.helpDocsAndDiscordLinks')" />
      </BaseBlock>
    </template>
  </TheLayout>
</template>
