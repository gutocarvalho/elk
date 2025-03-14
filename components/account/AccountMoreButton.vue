<script setup lang="ts">
import type { mastodon } from 'masto'
import { toggleBlockAccount, toggleBlockDomain, toggleMuteAccount } from '~~/composables/masto/relationship'

const { account } = defineProps<{
  account: mastodon.v1.Account
  command?: boolean
}>()
const emit = defineEmits<{
  (evt: 'addNote'): void
  (evt: 'removeNote'): void
}>()

let relationship = $(useRelationship(account))

const isSelf = $(useSelfAccount(() => account))

const { t } = useI18n()
const { client } = $(useMasto())
const useStarFavoriteIcon = usePreferences('useStarFavoriteIcon')

async function toggleReblogs() {
  if (!relationship!.showingReblogs && await openConfirmDialog({
    title: t('confirm.show_reblogs.title', [account.acct]),
    confirm: t('confirm.show_reblogs.confirm'),
    cancel: t('confirm.show_reblogs.cancel'),
  }) !== 'confirm')
    return

  const showingReblogs = !relationship?.showingReblogs
  relationship = await client.v1.accounts.follow(account.id, { reblogs: showingReblogs })
}

async function addUserNote() {
  emit('addNote')
}

async function removeUserNote() {
  if (!relationship!.note || relationship!.note.length === 0)
    return

  const newNote = await client.v1.accounts.createNote(account.id, { comment: '' })
  relationship!.note = newNote.note
  emit('removeNote')
}
</script>

<template>
  <CommonDropdown :eager-mount="command">
    <button flex gap-1 items-center w-full rounded op75 hover="op100 text-purple" group aria-label="More actions">
      <div rounded-5 p2 elk-group-hover="bg-purple/10">
        <div i-ri:more-2-fill />
      </div>
    </button>

    <template #popper>
      <NuxtLink :to="account.url" external target="_blank">
        <CommonDropdownItem
          :text="$t('menu.open_in_original_site')"
          icon="i-ri:arrow-right-up-line"
          :command="command"
        />
      </NuxtLink>

      <template v-if="currentUser">
        <template v-if="!isSelf">
          <CommonDropdownItem
            :text="$t('menu.mention_account', [`@${account.acct}`])"
            icon="i-ri:at-line"
            :command="command"
            @click="mentionUser(account)"
          />
          <CommonDropdownItem
            :text="$t('menu.direct_message_account', [`@${account.acct}`])"
            icon="i-ri:message-3-line"
            :command="command"
            @click="directMessageUser(account)"
          />

          <CommonDropdownItem
            v-if="!relationship?.showingReblogs"
            icon="i-ri:repeat-line"
            :text="$t('menu.show_reblogs', [`@${account.acct}`])"
            :command="command"
            @click="toggleReblogs()"
          />
          <CommonDropdownItem
            v-else
            :text="$t('menu.hide_reblogs', [`@${account.acct}`])"
            icon="i-ri:repeat-line"
            :command="command"
            @click="toggleReblogs()"
          />

          <CommonDropdownItem
            v-if="!relationship?.note || relationship?.note?.length === 0"
            :text="$t('menu.add_personal_note', [`@${account.acct}`])"
            icon="i-ri-edit-2-line"
            :command="command"
            @click="addUserNote()"
          />
          <CommonDropdownItem
            v-else
            :text="$t('menu.remove_personal_note', [`@${account.acct}`])"
            icon="i-ri-edit-2-line"
            :command="command"
            @click="removeUserNote()"
          />

          <CommonDropdownItem
            v-if="!relationship?.muting"
            :text="$t('menu.mute_account', [`@${account.acct}`])"
            icon="i-ri:volume-mute-line"
            :command="command"
            @click="toggleMuteAccount (relationship!, account)"
          />
          <CommonDropdownItem
            v-else
            :text="$t('menu.unmute_account', [`@${account.acct}`])"
            icon="i-ri:volume-up-fill"
            :command="command"
            @click="toggleMuteAccount (relationship!, account)"
          />

          <CommonDropdownItem
            v-if="!relationship?.blocking"
            :text="$t('menu.block_account', [`@${account.acct}`])"
            icon="i-ri:forbid-2-line"
            :command="command"
            @click="toggleBlockAccount (relationship!, account)"
          />
          <CommonDropdownItem
            v-else
            :text="$t('menu.unblock_account', [`@${account.acct}`])"
            icon="i-ri:checkbox-circle-line"
            :command="command"
            @click="toggleBlockAccount (relationship!, account)"
          />

          <template v-if="getServerName(account) !== currentServer">
            <CommonDropdownItem
              v-if="!relationship?.domainBlocking"
              :text="$t('menu.block_domain', [getServerName(account)])"
              icon="i-ri:shut-down-line"
              :command="command"
              @click="toggleBlockDomain(relationship!, account)"
            />
            <CommonDropdownItem
              v-else
              :text="$t('menu.unblock_domain', [getServerName(account)])"
              icon="i-ri:restart-line"
              :command="command"
              @click="toggleBlockDomain(relationship!, account)"
            />
          </template>

          <CommonDropdownItem
            :text="$t('menu.report_account', [`@${account.acct}`])"
            icon="i-ri:flag-2-line"
            :command="command"
            @click="openReportDialog(account)"
          />
        </template>

        <template v-else>
          <NuxtLink to="/pinned">
            <CommonDropdownItem :text="$t('account.pinned')" icon="i-ri:pushpin-line" :command="command" />
          </NuxtLink>
          <NuxtLink to="/favourites">
            <CommonDropdownItem :text="$t('account.favourites')" :icon="useStarFavoriteIcon ? 'i-ri:star-line' : 'i-ri:heart-3-line'" :command="command" />
          </NuxtLink>
          <NuxtLink to="/mutes">
            <CommonDropdownItem :text="$t('account.muted_users')" icon="i-ri:volume-mute-line" :command="command" />
          </NuxtLink>
          <NuxtLink to="/blocks">
            <CommonDropdownItem :text="$t('account.blocked_users')" icon="i-ri:forbid-2-line" :command="command" />
          </NuxtLink>
          <NuxtLink to="/domain_blocks">
            <CommonDropdownItem :text="$t('account.blocked_domains')" icon="i-ri:shut-down-line" :command="command" />
          </NuxtLink>
        </template>
      </template>
    </template>
  </CommonDropdown>
</template>
