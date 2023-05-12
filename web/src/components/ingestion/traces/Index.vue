<!-- Copyright 2022 Zinc Labs Inc. and Contributors

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http:www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License. 
-->

<!-- eslint-disable vue/x-invalid-end-tag -->
<template>
  <q-splitter
    v-model="splitterModel"
    unit="px"
    style="min-height: calc(100vh - 130px)"
  >
    <template v-slot:before>
      <q-tabs
        v-model="ingestiontabs"
        indicator-color="transparent"
        class="text-secondary"
        inline-label
        vertical
      >
        <q-route-tab
          name="openTelemetry"
          :to="{
            name: 'tracesOTLP',
            query: {
              org_identifier: store.state.selectedOrganization.identifier,
            },
          }"
          :icon="'img:' + getImageURL('images/ingestion/otlp.svg')"
          label="Open Telemetry"
          content-class="tab_content"
        />
      </q-tabs>
    </template>

    <template v-slot:after>
      <router-view
        title="Metrics"
        :currOrgIdentifier="currentOrgIdentifier"
        :currUserEmail="currentUserEmail"
        @copy-to-clipboard-fn="copyToClipboardFn"
      >
      </router-view>
    </template>
  </q-splitter>
</template>

<script lang="ts">
// @ts-ignore
import { defineComponent, ref, onMounted, onActivated } from "vue";
import { useI18n } from "vue-i18n";
import { useStore } from "vuex";
import { useRouter } from "vue-router";
import { copyToClipboard, useQuasar } from "quasar";
import organizationsService from "@/services/organizations";
// import { config } from "../constants/config";
import config from "../../../aws-exports";
import segment from "@/services/segment_analytics";
import { getImageURL } from "@/utils/zincutils";

export default defineComponent({
  name: "IngestTraces",
  components: {},
  data() {
    return {
      ingestiontabs: "openTelemetry",
    };
  },
  setup() {
    const { t } = useI18n();
    const store = useStore();
    const q = useQuasar();
    const router: any = useRouter();
    const rowData: any = ref({});
    const confirmUpdate = ref<boolean>(false);
    const currentOrgIdentifier: any = ref(
      store.state.selectedOrganization.identifier
    );

    onMounted(() => {
      const ingestRoutes = ["tracesOTLP"];
      if (ingestRoutes.includes(router.currentRoute.value.name)) {
        router.push({
          name: router.currentRoute.value.name,
          query: {
            org_identifier: store.state.selectedOrganization.identifier,
          },
        });
        return;
      }
      if (router.currentRoute.value.name === "ingestTraces") {
        router.push({
          name: "tracesOTLP",
          query: {
            org_identifier: store.state.selectedOrganization.identifier,
          },
        });
        return;
      }
      getOrganizationPasscode();
    });

    const getOrganizationPasscode = () => {
      organizationsService
        .get_organization_passcode(store.state.selectedOrganization.identifier)
        .then((res) => {
          if (res.data.data.token == "") {
            q.notify({
              type: "negative",
              message: "API Key not found.",
              timeout: 5000,
            });
          } else {
            store.dispatch("setOrganizationPasscode", res.data.data.passcode);
            currentOrgIdentifier.value =
              store.state.selectedOrganization.identifier;
          }
        });
    };

    const copyToClipboardFn = (content: any) => {
      copyToClipboard(content.innerText)
        .then(() => {
          q.notify({
            type: "positive",
            message: "Content Copied Successfully!",
            timeout: 5000,
          });
        })
        .catch(() => {
          q.notify({
            type: "negative",
            message: "Error while copy content.",
            timeout: 5000,
          });
        });

      segment.track("Button Click", {
        button: "Copy to Clipboard",
        ingestion: router.currentRoute.value.name,
        user_org: store.state.selectedOrganization.identifier,
        user_id: store.state.userInfo.email,
        page: "Ingestion",
      });
    };

    const updatePasscode = () => {
      organizationsService
        .update_organization_passcode(
          store.state.selectedOrganization.identifier
        )
        .then((res) => {
          if (res.data.data.token == "") {
            q.notify({
              type: "negative",
              message: "API Key not found.",
              timeout: 5000,
            });
          } else {
            q.notify({
              type: "positive",
              message: "Token reset successfully.",
              timeout: 5000,
            });
            store.dispatch("setOrganizationPasscode", res.data.data.passcode);
            currentOrgIdentifier.value =
              store.state.selectedOrganization.identifier;
          }
        })
        .catch((e) => {
          q.notify({
            type: "negative",
            message: "Error while updating Token." + e.error,
            timeout: 5000,
          });
        });

      segment.track("Button Click", {
        button: "Update Passcode",
        user_org: store.state.selectedOrganization.identifier,
        user_id: store.state.userInfo.email,
        page: "Ingestion",
      });
    };

    const showUpdateDialogFn = () => {
      confirmUpdate.value = true;
    };

    return {
      t,
      store,
      router,
      config,
      rowData,
      splitterModel: ref(200),
      getOrganizationPasscode,
      currentUserEmail: store.state.userInfo.email,
      currentOrgIdentifier,
      copyToClipboardFn,
      updatePasscode,
      showUpdateDialogFn,
      confirmUpdate,
      getImageURL,
    };
  },
  computed: {
    selectedOrg() {
      return this.store.state.selectedOrganization.identifier;
    },
  },
  watch: {
    selectedOrg(newVal: any, oldVal: any) {
      console.log(this.router.currentRoute.value.name);
      if (
        newVal != oldVal &&
        (this.router.currentRoute.value.name === "ingestTraces" ||
          this.router.currentRoute.value.name === "tracesOTLP")
      ) {
        this.getOrganizationPasscode();
      }
    },
  },
});
</script>

<style scoped lang="scss">
.ingestionPage {
  padding: 1.5rem 0 0;
  .head {
    padding-bottom: 1rem;
  }
  .q-tabs {
    &--vertical {
      margin: 1.5rem 1rem 0 1rem;
      .q-tab {
        justify-content: flex-start;
        padding: 0 0.6rem 0 0.6rem;
        border-radius: 0.5rem;
        margin-bottom: 0.5rem;
        color: $dark;
        text-transform: capitalize;

        &__content.tab_content {
          .q-tab {
            &__icon + &__label {
              padding-left: 0.875rem;
              font-weight: 600;
            }
          }
        }
        &--active {
          background-color: $accent;
        }
      }
    }
  }
}
</style>