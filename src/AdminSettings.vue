<script>
/**
 * @copyright Copyright (c) 2022, 2023 Claus-Justus Heine <himself@claus-justus-heine.de>
 *
 * @author Claus-Justus Heine <himself@claus-justus-heine.de>
 *
 * @license AGPL-3.0-or-later
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU Affero General Public License as
 * published by the Free Software Foundation, either version 3 of the
 * License, or (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU Affero General Public License for more details.
 *
 * You should have received a copy of the GNU Affero General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
 *
 */
</script>
<template>
  <SettingsSection :class="cloudVersionClasses"
                   :title="t(appName, 'Recursive PDF Downloader')"
  >
    <AppSettingsSection v-if="dependencies.missing.required + dependencies.missing.suggested > 0"
                        :title="t(appName, 'Missing Dependencies')"
    >
      <div v-if="dependencies.missing.required > 0" class="required-dependencies">
        <div><label>{{ t(appName, 'Required Missing') }}</label></div>
        <ul>
          <ListItem v-for="(path, program) in dependencies.required"
                    :key="program"
                    :title="program"
                    :details="path"
                    :bold="false"
          >
            <template #subtitle>
              <div class="hint">
                {{ t(appName, 'The app will not work unless you install {program} such that it can be found by the web-server.', { program }) }}
              </div>
            </template>
          </ListItem>
        </ul>
      </div>
      <div v-if="dependencies.missing.suggested > 0" class="suggested-dependencies">
        <div><label>{{ t(appName, 'Suggested Missing') }}</label></div>
        <ul>
          <ListItem v-for="(path, program) in dependencies.suggested"
                    :key="program"
                    :title="program"
                    :details="path"
                    :bold="false"
          >
            <template #subtitle>
              <div v-if="path === 'missing'" class="hint">
                {{ t(appName, 'The app will work without installing {program}, but the conversion results may be degraded.', { program }) }}
              </div>
            </template>
          </ListItem>
        </ul>
      </div>
    </AppSettingsSection>
    <AppSettingsSection :title="t(appName, 'Archive Extraction')">
      <div :class="['flex-container', 'flex-center', { extractArchiveFiles }]">
        <input id="extract-archive files"
               v-model="extractArchiveFiles"
               type="checkbox"
               :disabled="loading"
               @change="saveSetting('extractArchiveFiles')"
        >
        <label for="extract-archive files">
          {{ t(appName, 'On-the-fly extraction of archive files. If enabled users can control this setting on a per-user basis.') }}
        </label>
      </div>
      <SettingsInputText
        v-model="humanArchiveSizeLimit"
        :label="t(appName, 'Archive Size Limit')"
        :hint="t(appName, 'Disallow archive extraction for archives with decompressed size larger than this limit.')"
        :disabled="loading || !extractArchiveFiles"
        @update="saveTextInput(...arguments, 'archiveSizeLimit')"
      />
    </AppSettingsSection>
    <AppSettingsSection :title="t(appName, 'Custom Converter Scripts')">
      <div :class="['flex-container', 'flex-center']">
        <input id="disable-builtin-converters"
               v-model="disableBuiltinConverters"
               type="checkbox"
               :disabled="loading"
               @change="saveSetting('disableBuiltinConverters')"
        >
        <label for="disable-builtin-converters">
          {{ t(appName, 'Disable the builtin converters.') }}
        </label>
      </div>
      <SettingsInputText
        v-model="universalConverter"
        :label="t(appName, 'Universal Converter')"
        :hint="t(appName, 'Full path to a filter program to be executed first for all files. If it fails, the other converters will be tried in turn.')"
        :disabled="loading"
        @update="saveTextInput(...arguments, 'universalConverter')"
      />
      <SettingsInputText
        v-model="fallbackConverter"
        :label="t(appName, 'Fallback Converter')"
        :hint="t(appName, 'Full path to a filter program to be run when all other filters have failed. If it fails an error page will be substituted for the failing document.')"
        :disabled="loading || builtinConvertersDisabled"
        @update="saveTextInput(...arguments, 'fallbackConverter')"
      />
    </AppSettingsSection>
    <AppSettingsSection :title="t(appName, 'Converters')">
      <div class="converter-status">
        <div><label>{{ t(appName, 'Status of the configured Converters') }}</label></div>
        <ul>
          <ListItem v-for="(value, mimeType) in converters"
                    :key="mimeType"
                    :title="mimeType"
                    :details="value.length > 1 ? t(appName, 'converter chain') : t(appName, 'single converter')"
                    :bold="true"
          >
            <template #subtitle>
              <ul>
                <ListItem v-for="(items, index) in value"
                          :key="index"
                          :title="Object.values(items).length > 1 ? t(appName, 'alternatives') : t(appName, 'converter')"
                          :show-counter="value.length > 1"
                          :counter-number="value.length > 1 ? index + 1 : 0"
                >
                  <template #subtitle>
                    <ListItem v-for="(executable, converter) in items"
                              :key="converter"
                              title=""
                              :details="items.length > 1 ? t(appName, 'converter') : ''"
                    >
                      <template #subtitle>
                        <span>{{ converter }}: {{ executable }}</span>
                      </template>
                    </ListItem>
                  </template>
                </ListItem>
              </ul>
            </template>
          </ListItem>
        </ul>
      </div>
    </AppSettingsSection>
  </SettingsSection>
</template>

<script>
import { appName } from './config.js'
import SettingsSection from '@nextcloud/vue/dist/Components/SettingsSection'
import AppSettingsSection from '@nextcloud/vue/dist/Components/AppSettingsSection'
import SettingsInputText from '@rotdrop/nextcloud-vue-components/lib/components/SettingsInputText'
import ListItem from '@rotdrop/nextcloud-vue-components/lib/components/ListItem'
import { generateUrl } from '@nextcloud/router'
import { showError, showSuccess, showInfo, TOAST_PERMANENT_TIMEOUT } from '@nextcloud/dialogs'
import axios from '@nextcloud/axios'
import settingsSync from './toolkit/mixins/settings-sync'
import cloudVersionClasses from './toolkit/util/cloud-version-classes.js'

export default {
  name: 'AdminSettings',
  components: {
    AppSettingsSection,
    ListItem,
    SettingsSection,
    SettingsInputText,
  },
  data() {
    return {
      cloudVersionClasses,
      extractArchiveFiles: false,
      archiveSizeLimit: null,
      humanArchiveSizeLimit: '',
      disableBuiltinConverters: false,
      universalConverter: '',
      fallbackConverter: '',
      converters: {},
      dependencies: {
        missing: {
          required: 0,
          suggested: 0,
        },
        required: {},
        suggested: {},
      },
      loading: true,
    }
  },
  mixins: [
    settingsSync,
  ],
  created() {
    this.getData()
  },
  computed: {
    builtinConvertersDisabled() {
      return !!this.disableBuiltinConverters
    },
  },
  watch: {
  },
  methods: {
    async getData() {
      // slurp in all settings
      this.fetchSettings('admin');
      this.loading = false
    },
    async saveTextInput(value, settingsKey, force) {
      if (await this.saveConfirmedSetting(value, 'admin', settingsKey, force)) {
        if (settingsKey.endsWith('Converter')) {
          this.fetchSetting('converters', 'admin')
        }
      }
    },
    async saveSetting(setting) {
      if (await this.saveSimpleSetting(setting, 'admin')) {
        if (setting === 'disableBuiltinConverters') {
          this.fetchSetting('converters', 'admin')
        }
      }
    },
  },
}
</script>
<style lang="scss" scoped>
.cloud-version {
  --cloud-theme-filter: none;
  &.cloud-version-major-25 {
    --cloud-theme-filter: var(--background-invert-if-dark);
  }
}
.settings-section {
  .flex-container {
    display:flex;
    &.flex-center {
      align-items:center;
    }
  }
  :deep() &__title {
    position: relative;
    padding-left:60px;
    height:32px;
    &::before {
      content: "";
      position: absolute;
      top: 0;
      left: 0;
      width: 32px;
      height: 32px;
      background-size:32px;
      background-image:url('../img/app-dark.svg');
      background-repeat:no-repeat;
      background-origin:border-box;
      background-position:left center;
      filter: var(--cloud-theme-filter);
    }
  }
  :deep(.app-settings-section) {
    margin-bottom: 40px;
  }
}
</style>
