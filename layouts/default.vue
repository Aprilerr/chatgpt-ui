<script setup>
import {useDisplay} from "vuetify";

const { $i18n, $auth } = useNuxtApp()
const runtimeConfig = useRuntimeConfig()
const colorMode = useColorMode()
const drawer = ref(null)
const themes = ref([
  { title: $i18n.t('lightMode'), value: 'light' },
  { title: $i18n.t('darkMode'), value: 'dark' },
  { title: $i18n.t('followSystem'), value: 'system'}
])
const setTheme = (theme) => {
  colorMode.preference = theme
}
const feedback = () => {
  window.open('https://github.com/WongSaang/chatgpt-ui/issues', '_blank')
}

const { locale, locales, setLocale } = useI18n()
const setLang = (lang) => {
  setLocale(lang)
}

const conversations = useConversions()
const currentConversation = useConversion()

const editingConversation = ref(null)
const deletingConversationIndex = ref(null)

const editConversation = (index) => {
  editingConversation.value = conversations.value[index]
}

const updateConversation = async (index) => {
  editingConversation.value.updating = true
  const { data, error } = await useAuthFetch(`/api/chat/conversations/${editingConversation.value.id}/`, {
    method: 'PUT',
    body: JSON.stringify({
      topic: editingConversation.value.topic
    })
  })
  if (!error.value) {
    conversations.value[index] = editingConversation.value
  }
  editingConversation.value = null
}

const deleteConversation = async (index) => {
  deletingConversationIndex.value = index
  const { data, error } = await useAuthFetch(`/api/chat/conversations/${conversations.value[index].id}/`, {
    method: 'DELETE'
  })
  deletingConversationIndex.value = null
  if (!error.value) {
    if (conversations.value[index].id === currentConversation.value.id) {
      createNewConversion()
    }
    conversations.value.splice(index, 1)
  }
}

const clearConversations = async () => {
  deletingConversations.value = true
  const { data, error } = await useAuthFetch(`/api/chat/conversations/delete_all`, {
    method: 'DELETE'
  })
  if (!error.value) {
    loadConversations()
    clearConfirmDialog.value = false
  }
  deletingConversations.value = false
}

const clearConfirmDialog = ref(false)
const deletingConversations = ref(false)
const loadingConversations = ref(false)

const loadConversations = async () => {
  loadingConversations.value = true
  conversations.value = await getConversions()
  loadingConversations.value = false
}

const {mdAndUp} = useDisplay()

const drawerPermanent = computed(() => {
  return mdAndUp.value
})

const signOut = async () => {
  const { data, error } = await useFetch('/api/account/logout/', {
    method: 'POST'
  })
  if (!error.value) {
    await $auth.logout()
  }
}

onNuxtReady(async () => {
  loadConversations()
})

</script>

<template>
  <v-app
      :theme="$colorMode.value"
  >
    <v-navigation-drawer
        v-model="drawer"
        :permanent="drawerPermanent"
        width="300"
    >
      <div class="px-2 py-2">
        <v-list>
          <v-list-item>
            <v-btn
                block
                variant="outlined"
                prepend-icon="add"
                @click="createNewConversion()"
                class="text-none"
            >
              {{ $t('newConversation') }}
            </v-btn>
          </v-list-item>
          <v-list-item v-show="loadingConversations">
            <v-list-item-title class="d-flex justify-center">
              <v-progress-circular indeterminate></v-progress-circular>
            </v-list-item-title>
          </v-list-item>
        </v-list>

        <v-list>
          <template
              v-for="(conversation, cIdx) in conversations"
              :key="conversation.id"
          >
            <v-list-item
                active-color="primary"
                rounded="xl"
                v-if="editingConversation && editingConversation.id === conversation.id"
            >
              <v-text-field
                  v-model="editingConversation.topic"
                  :loading="editingConversation.updating"
                  variant="underlined"
                  append-icon="done"
                  hide-details
                  density="compact"
                  autofocus
                  @keyup.enter="updateConversation(cIdx)"
                  @click:append="updateConversation(cIdx)"
              ></v-text-field>
            </v-list-item>
            <v-hover
                v-if="!editingConversation || editingConversation.id !== conversation.id"
                v-slot="{ isHovering, props }"
            >
              <v-list-item
                  rounded="xl"
                  active-color="primary"
                  @click="openConversationMessages(conversation)"
                  v-bind="props"
              >
                <v-list-item-title>{{ conversation.topic }}</v-list-item-title>
                <template v-slot:append>
                  <div
                      v-show="isHovering"
                  >
                    <v-btn
                        icon="edit"
                        size="small"
                        variant="text"
                        @click.stop="editConversation(cIdx)"
                    >
                    </v-btn>
                    <v-btn
                        icon="delete"
                        size="small"
                        variant="text"
                        :loading="deletingConversationIndex === cIdx"
                        @click.stop="deleteConversation(cIdx)"
                    >
                    </v-btn>
                  </div>
                </template>
              </v-list-item>
            </v-hover>
          </template>
        </v-list>
      </div>

      <template v-slot:append>
        <div class="px-1">
          <v-divider></v-divider>
          <v-list>

            <v-dialog
                v-model="clearConfirmDialog"
                persistent
            >
              <template v-slot:activator="{ props }">
                <v-list-item
                    v-bind="props"
                    rounded="xl"
                    prepend-icon="delete_forever"
                    :title="$t('clearConversations')"
                ></v-list-item>
              </template>
              <v-card>
                <v-card-title class="text-h5">
                  Are you sure you want to delete all conversations?
                </v-card-title>
                <v-card-text>This will be a permanent deletion and cannot be retrieved once deleted. Please proceed with caution.</v-card-text>
                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn
                      color="green-darken-1"
                      variant="text"
                      @click="clearConfirmDialog = false"
                      class="text-none"
                  >
                    Cancel deletion
                  </v-btn>
                  <v-btn
                      color="green-darken-1"
                      variant="text"
                      @click="clearConversations"
                      class="text-none"
                      :loading="deletingConversations"
                  >
                    Confirm deletion
                  </v-btn>
                </v-card-actions>
              </v-card>
            </v-dialog>

            <ModelParameters/>

            <v-menu
            >
              <template v-slot:activator="{ props }">
                <v-list-item
                    v-bind="props"
                    rounded="xl"
                    :prepend-icon="$colorMode.value === 'light' ? 'light_mode' : 'dark_mode'"
                    :title="$t('themeMode')"
                ></v-list-item>
              </template>
              <v-list
                  bg-color="white"
              >
                <v-list-item
                    v-for="(theme, idx) in themes"
                    :key="idx"
                    @click="setTheme(theme.value)"
                >
                  <v-list-item-title>{{ theme.title }}</v-list-item-title>
                </v-list-item>
              </v-list>
            </v-menu>

            <SettingsLanguages/>

            <v-list-item
                rounded="xl"
                prepend-icon="help_outline"
                :title="$t('feedback')"
                @click="feedback"
            ></v-list-item>

            <v-list-item
                rounded="xl"
                prepend-icon="logout"
                :title="$t('signOut')"
                @click="signOut"
            ></v-list-item>

          </v-list>
        </div>
      </template>
    </v-navigation-drawer>

    <v-app-bar
        class="d-md-none"
    >
      <v-app-bar-nav-icon @click="drawer = !drawer"></v-app-bar-nav-icon>

      <v-toolbar-title>{{ runtimeConfig.public.appName }}</v-toolbar-title>

      <v-spacer></v-spacer>

      <v-btn
          :title="$t('newConversation')"
          icon="add"
          @click="createNewConversion()"
      ></v-btn>

    </v-app-bar>

    <v-main>
      <NuxtPage/>
    </v-main>

    <div>
      <div
          v-if="$pwa?.offlineReady || $pwa?.needRefresh"
          class="pwa-toast"
          role="alert"
      >
        <div class="message">
          <span v-if="$pwa.offlineReady">
            App ready to work offline
          </span>
          <span v-else>
            New content available, click on reload button to update.
          </span>
        </div>
        <button
            v-if="$pwa.needRefresh"
            @click="$pwa.updateServiceWorker()"
        >
          Reload
        </button>
        <button @click="$pwa.cancelPrompt()">
          Close
        </button>
      </div>
      <div
          v-if="$pwa?.showInstallPrompt && !$pwa?.offlineReady && !$pwa?.needRefresh"
          class="pwa-toast"
          role="alert"
      >
        <div class="message">
          <span>
            Install PWA
          </span>
        </div>
        <button @click="$pwa.install()">
          Install
        </button>
        <button @click="$pwa.cancelInstall()">
          Cancel
        </button>
      </div>
    </div>
  </v-app>
</template>

<style>
.v-navigation-drawer__content::-webkit-scrollbar {
  width: 0;
}
.v-navigation-drawer__content:hover::-webkit-scrollbar {
  width: 6px;
}
.v-navigation-drawer__content:hover::-webkit-scrollbar-thumb {
  background-color: #999;
  border-radius: 3px;
}

.pwa-toast {
  position: fixed;
  right: 0;
  bottom: 0;
  margin: 16px;
  padding: 12px;
  border: 1px solid #8885;
  border-radius: 4px;
  z-index: 1;
  text-align: left;
  box-shadow: 3px 4px 5px 0 #8885;
}
.pwa-toast .message {
  margin-bottom: 8px;
}
.pwa-toast button {
  border: 1px solid #8885;
  outline: none;
  margin-right: 5px;
  border-radius: 2px;
  padding: 3px 10px;
}
</style>