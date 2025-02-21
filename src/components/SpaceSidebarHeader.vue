<script setup>
import { ref, watchEffect, computed } from 'vue';
import { useSpaceSubscription } from '@/composables/useSpaceSubscription';
import { useFollowSpace } from '@/composables/useFollowSpace';
import verified from '@/../snapshot-spaces/spaces/verified.json';
import { useIntl } from '@/composables/useIntl';

const props = defineProps({
  space: {
    type: Object,
    default: null
  }
});

const { formatCompactNumber } = useIntl();

const isVerified = computed(() => verified[props.space.id] || 0);

const {
  loading,
  toggleSubscription,
  isSubscribed,
  loadSubscriptions,
  subscriptions
} = useSpaceSubscription(props.space.id);

const { isFollowing } = useFollowSpace(props.space.id);

const notificationIcon = ref('notifications-off');

watchEffect(() => {
  if (subscriptions.value === undefined) {
    loadSubscriptions();
  }
  if (isSubscribed.value) {
    notificationIcon.value = 'notifications-on';
  } else notificationIcon.value = 'notifications-off';
});
</script>

<template>
  <div class="text-center bg-skin-header-bg h-[253px]">
    <AvatarSpace
      :space="space"
      symbolIndex="space"
      size="80"
      class="mt-3 mb-2"
    />
    <h3 class="mb-[2px] mx-3 flex justify-center items-center">
      <div
        class="truncate mr-1"
        v-tippy="{
          content: space.name.length > 16 ? space.name : null
        }"
      >
        {{ space.name }}
      </div>
      <BaseIcon
        v-if="isVerified === 1"
        v-tippy="{
          content: $t('verifiedSpace'),
          placement: 'right'
        }"
        name="check"
        size="20"
      />
      <BaseIcon v-if="isVerified === -1" name="warning" size="20" />
    </h3>
    <div class="mb-[12px] text-skin-text">
      {{
        $tc('members', space.followersCount, {
          count: formatCompactNumber(space.followersCount)
        })
      }}
    </div>

    <div class="flex justify-center gap-x-2">
      <ButtonFollow :space="space" />
      <UiSidebarButton
        class="inline"
        v-if="isFollowing"
        @click="toggleSubscription()"
      >
        <LoadingSpinner v-if="loading" />
        <BaseIcon
          v-else
          size="20"
          class="text-skin-link"
          :name="notificationIcon"
        />
      </UiSidebarButton>
    </div>
  </div>
</template>
