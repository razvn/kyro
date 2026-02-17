<script setup lang="ts">
import { ApiErrorResponse } from '@/api';
import serviceApi from '@/api/service';
import ServiceCardDetails from '@/components/services/ServiceCardDetails.vue';
import ServiceCardItem from '@/components/services/ServiceCardItem.vue';
import ConfirmDialog from '@/components/shared/ConfirmDialog.vue';
import { ServiceDetails, ServiceWithBinding } from '@/components/services/models';
import useApiCall from '@/composables/useApiCall';
import useApplicationContext from '@/composables/useApplicationContext';
import useFilterData from '@/composables/useFilterData';
import useLoadingFn from '@/composables/useLoadingFn';
import { mapResources } from '@/models/cf/common';
import { onSuccess, successOf } from '@/utils/result';
import { ref } from 'vue';

type DialogState =
  | {
      opened: false;
      service: undefined;
    }
  | {
      opened: true;
      service: ServiceDetails;
    };

const context = useApplicationContext();

const {
  data: services,
  error: servicesError,
  execute: loadData,
  reset: resetData,
} = useApiCall(async () => {
  const result = await serviceApi.getBindingsForApplication(context.guid.value).then((result) =>
    onSuccess(result, ({ resources }) => {
      const guids = resources.map((binding) => binding.relationships.service_instance.data.guid);

      return serviceApi.getInstances(guids);
    }),
  );

  if (result.success) {
    const [bindings, services] = result.data;

    return successOf(
      mapResources(services, (service) => {
        const binding = bindings.resources.find(
          (binding) => binding.relationships.service_instance.data.guid === service.guid,
        );

        return {
          ...service,
          binding: binding!!,
        };
      }),
    );
  }

  return result;
}, context.loading);

loadData();

context.on('reload', () => {
  loadData();
});
context.on('reset', () => {
  resetData();
});

const { data: filteredServices, filters } = useFilterData((filters, { includesText }) => {
  return services.value?.resources.filter((service) => {
    return !filters.text || includesText(service.name);
  });
});

const dialog = ref<DialogState>({ opened: false, service: undefined });
const deleting = ref(false);
const deletingBindingGuid = ref<ServiceWithBinding['binding']['guid']>();
const deleteDialog = ref<{ opened: boolean; service?: ServiceWithBinding }>({ opened: false, service: undefined });

const wait = (delayMs: number) => new Promise((resolve) => setTimeout(resolve, delayMs));
const waitForBindingDeletion = async (bindingGuid: ServiceWithBinding['binding']['guid']) => {
  for (let i = 0; i < 20; i++) {
    const result = await serviceApi.getBindingsForApplication(context.guid.value);
    if (!result.success) {
      break;
    }

    const exists = result.data.resources.some((binding) => binding.guid === bindingGuid);
    if (!exists) {
      return;
    }

    await wait(1000);
  }
};

const openService = async (service: ServiceWithBinding) => {
  context.loading.value = true;

  const details = await serviceApi.getBindingsDetails(service.binding.guid);

  dialog.value = {
    opened: true,
    service: {
      ...service,
      details: details.success ? details.data : undefined,
    },
  };

  context.loading.value = false;
};

const requestDeleteService = (service: ServiceWithBinding) => {
  deleteDialog.value = { opened: true, service };
};

const { fn: confirmDeleteService } = useLoadingFn(async () => {
  const service = deleteDialog.value.service;
  if (!service) return;

  deletingBindingGuid.value = service.binding.guid;
  const result = await serviceApi.deleteBinding(service.binding.guid);
  if (result.success) {
    await waitForBindingDeletion(service.binding.guid);
    deleteDialog.value = { opened: false, service: undefined };
    loadData();
  } else {
    context.errors.value.push(result.error as ApiErrorResponse);
  }

  deletingBindingGuid.value = undefined;
}, deleting);
</script>

<template>
  <v-row class="flex-column" v-if="services">
    <v-col>
      <v-dialog v-model="dialog.opened" width="80%" scrollable>
        <service-card-details v-if="dialog.service" :service="dialog.service"></service-card-details>
      </v-dialog>

      <confirm-dialog v-model="deleteDialog.opened" @confirm="confirmDeleteService" :loading="deleting">
        <template #text>
          Delete service `{{ deleteDialog.service?.name }}` from application `{{ context.application.value.name }}`?
          This removes the binding from this app.
        </template>
      </confirm-dialog>

      <v-row justify="end">
        <v-col cols="3">
          <v-text-field label="Filtrer" density="compact" v-model="filters.text" clearable hide-details></v-text-field>
        </v-col>

        <v-col cols="auto">
          <v-btn variant="text" @click="loadData" size="large">
            <v-icon>mdi-cached</v-icon>
          </v-btn>
        </v-col>

        <v-col cols="auto">{{ filteredServices.length }}/{{ services.resources.length }}</v-col>
      </v-row>

      <v-row>
        <v-col cols="3" v-for="service in filteredServices" :key="`application-${service.guid}`">
          <service-card-item
            :service="service"
            :deleting="deletingBindingGuid === service.binding.guid"
            @open="openService(service)"
            @delete="requestDeleteService(service)">
          </service-card-item>
        </v-col>
      </v-row>
    </v-col>
  </v-row>
</template>
