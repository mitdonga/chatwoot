<template>
  <div
    role="button"
    class="flex flex-col pl-5 pr-3 gap-2.5 py-3 w-full border-b border-slate-50 dark:border-slate-800/50 hover:bg-slate-25 dark:hover:bg-slate-800 cursor-pointer"
    :class="
      isInboxCardActive
        ? 'bg-slate-25 dark:bg-slate-800 click-animation'
        : 'bg-white dark:bg-slate-900'
    "
    @contextmenu="openContextMenu($event)"
    @click="openConversation(notificationItem)"
  >
    <div class="flex relative items-center justify-between w-full">
      <div
        v-if="isUnread"
        class="absolute -left-3.5 flex w-2 h-2 rounded bg-woot-500 dark:bg-woot-500"
      />
      <InboxNameAndId :inbox="inbox" :conversation-id="primaryActor.id" />

      <div class="flex gap-2">
        <PriorityIcon :priority="primaryActor.priority" />
        <StatusIcon :status="primaryActor.status" />
      </div>
    </div>

    <div class="flex flex-row justify-between items-center w-full">
      <div class="flex gap-1.5 items-center max-w-[calc(100%-70px)]">
        <Thumbnail
          v-if="assigneeMeta"
          :src="assigneeMeta.thumbnail"
          :username="assigneeMeta.name"
          size="20px"
        />
        <div class="flex min-w-0">
          <span
            class="font-medium text-slate-800 dark:text-slate-50 text-sm overflow-hidden text-ellipsis whitespace-nowrap"
          >
            <span class="font-normal text-sm">
              {{ pushTitle }}
            </span>
          </span>
        </div>
      </div>
      <span
        class="font-medium max-w-[60px] text-slate-600 dark:text-slate-300 text-xs whitespace-nowrap"
      >
        {{ lastActivityAt }}
      </span>
    </div>
    <inbox-context-menu
      v-if="isContextMenuOpen"
      :context-menu-position="contextMenuPosition"
      :menu-items="menuItems"
      @close="closeContextMenu"
      @click="handleAction"
    />
  </div>
</template>
<script>
import PriorityIcon from './PriorityIcon.vue';
import StatusIcon from './StatusIcon.vue';
import InboxNameAndId from './InboxNameAndId.vue';
import InboxContextMenu from './InboxContextMenu.vue';
import Thumbnail from 'dashboard/components/widgets/Thumbnail.vue';
import timeMixin from 'dashboard/mixins/time';
import { INBOX_EVENTS } from 'dashboard/helper/AnalyticsHelper/events';
export default {
  components: {
    PriorityIcon,
    InboxContextMenu,
    StatusIcon,
    InboxNameAndId,
    Thumbnail,
  },
  mixins: [timeMixin],
  props: {
    notificationItem: {
      type: Object,
      default: () => {},
    },
  },
  data() {
    return {
      isContextMenuOpen: false,
      contextMenuPosition: { x: null, y: null },
    };
  },
  computed: {
    primaryActor() {
      return this.notificationItem?.primary_actor;
    },
    isInboxCardActive() {
      return this.$route.params.conversation_id === this.primaryActor?.id;
    },
    inbox() {
      return this.$store.getters['inboxes/getInbox'](
        this.primaryActor.inbox_id
      );
    },
    isUnread() {
      return !this.notificationItem?.read_at;
    },
    meta() {
      return this.primaryActor?.meta;
    },
    assigneeMeta() {
      return this.meta?.assignee;
    },
    pushTitle() {
      return this.$t(
        `INBOX.TYPES.${this.notificationItem.notification_type.toUpperCase()}`
      );
    },
    lastActivityAt() {
      const dynamicTime = this.dynamicTime(
        this.notificationItem?.last_activity_at
      );
      return this.shortTimestamp(dynamicTime, true);
    },
    menuItems() {
      const items = [
        {
          key: 'delete',
          label: this.$t('INBOX.MENU_ITEM.DELETE'),
        },
      ];

      if (!this.isUnread) {
        items.push({
          key: 'mark_as_unread',
          label: this.$t('INBOX.MENU_ITEM.MARK_AS_UNREAD'),
        });
      } else {
        items.push({
          key: 'mark_as_read',
          label: this.$t('INBOX.MENU_ITEM.MARK_AS_READ'),
        });
      }
      return items;
    },
  },
  unmounted() {
    this.closeContextMenu();
  },
  methods: {
    openConversation(notification) {
      const {
        id,
        primary_actor_id: primaryActorId,
        primary_actor_type: primaryActorType,
        primary_actor: { id: conversationId, inbox_id: inboxId },
        notification_type: notificationType,
      } = notification;

      if (this.$route.params.conversation_id !== conversationId) {
        this.$track(INBOX_EVENTS.OPEN_CONVERSATION_VIA_INBOX, {
          notificationType,
        });

        this.$store.dispatch('notifications/read', {
          id,
          primaryActorId,
          primaryActorType,
          unreadCount: this.meta.unreadCount,
        });

        this.$router.push({
          name: 'inbox_view_conversation',
          params: { inboxId, conversation_id: conversationId },
        });
      }
    },
    closeContextMenu() {
      this.isContextMenuOpen = false;
      this.contextMenuPosition = { x: null, y: null };
    },
    openContextMenu(e) {
      this.closeContextMenu();
      e.preventDefault();
      this.contextMenuPosition = {
        x: e.pageX || e.clientX,
        y: e.pageY || e.clientY,
      };
      this.isContextMenuOpen = true;
    },
    handleAction(key) {
      switch (key) {
        case 'mark_as_read':
          this.$emit('mark-notification-as-read', this.notificationItem);
          break;
        case 'mark_as_unread':
          this.$emit('mark-notification-as-unread', this.notificationItem);
          break;
        case 'delete':
          this.$emit('delete-notification', this.notificationItem);
          break;
        default:
      }
    },
  },
};
</script>
<style scoped>
.click-animation {
  animation: click-animation 0.3s ease-in-out;
}
@keyframes click-animation {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(0.99);
  }
  100% {
    transform: scale(1);
  }
}
</style>
