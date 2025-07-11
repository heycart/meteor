<template>
  <mt-select
    v-bind="$attrs"
    :model-value="props.modelValue"
    :options="options"
    :is-loading="isLoading"
    :label-property="props.labelProperty"
    :value-property="props.valueProperty"
    :enable-multi-selection="props.enableMultiSelection"
    :disabled="props.disabled"
    :search-function="fetchSearchData"
    @update:model-value="emit('update:modelValue', $event)"
    @paginate="fetchNextPage"
    @item-add="onItemAdd"
  >
    <!-- Pass through all slots -->
    <template v-for="(_, name) in $slots" #[name]="scope">
      <slot :name="name" v-bind="scope ?? {}"></slot>
    </template>
  </mt-select>
</template>

<script setup lang="ts">
/**
 * This component is a wrapper for the mt-select component.
 *
 * It is used to add automatic data handling for the mt-select component using
 * the repository methods from the meteor-admin-sdk.
 *
 * The developer can provide an entity name, and the component will automatically fetch the
 * data from the repository and pass it to the mt-select component. Pagination and
 * searching is handled automatically.
 *
 * The fallback uses the meteor-admin-sdk's find method to fetch the data. But the developer
 * can also provide a custom repository to use their own data fetching logic.
 */
import { ref, onMounted } from "vue";
import type { PropType } from "vue";
import MtSelect from "@/components/form/mt-select/mt-select.vue";
import type Repository from "@heycart-ag/meteor-admin-sdk/es/data/repository";
import Criteria from "@heycart-ag/meteor-admin-sdk/es/data/Criteria";
import { useRepository } from "@heycart-ag/meteor-admin-sdk/es/data/composables/useRepository";

export interface Props {
  entity: keyof EntitySchema.Entities;
  repository?: typeof Repository;
  modelValue?: any;
  labelProperty?: string | string[];
  valueProperty?: string;
  disabled?: boolean;
  enableMultiSelection?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  repository: undefined,
  modelValue: null,
  labelProperty: "name",
  valueProperty: "id",
  disabled: false,
  enableMultiSelection: false,
});

const emit = defineEmits<{
  (e: "update:modelValue", value: any): void;
}>();

const options = ref<any[]>([]);
const isLoading = ref(false);
const searchTerm = ref("");
const currentPage = ref(1);
const limit = ref(50);
const total = ref(0);

// @ts-expect-error - The repository is not typed exactly the same
const repo = useRepository(props.entity, props.repository);

const getPage = async (page: number, term?: string) => {
  const criteria = new Criteria(page, limit.value);

  const currentSearchTerm = term || searchTerm.value;

  if (currentSearchTerm) {
    criteria.setTerm(currentSearchTerm);
  }

  if (typeof props.labelProperty === "string") {
    criteria.addSorting({
      field: props.labelProperty,
      order: "ASC",
      naturalSorting: false,
    });
  }

  try {
    const result = await repo.value.search(criteria);
    if (result) {
      total.value = result.total ?? 0;
      return [...result];
    }
  } catch (e) {
    console.error(`Could not fetch data for entity ${props.entity}`, e);
  }
  return [];
};

const fetchData = async (append = false, term?: string, disableOptionsUpdate = false) => {
  isLoading.value = true;
  const newOptions = await getPage(currentPage.value, term);
  isLoading.value = false;

  if (append && !disableOptionsUpdate) {
    // Iterate through all new options, if they are already in the options array, remove the
    // old one and add the new one
    newOptions.forEach((newOption) => {
      const index = options.value.findIndex(
        (option) =>
          // @ts-expect-error - The value property is not typed exactly the same
          option[props.valueProperty as string] === newOption[props.valueProperty as string],
      );
      if (index !== -1) {
        options.value.splice(index, 1);
      }
      options.value.push(newOption);
    });
  } else if (!disableOptionsUpdate) {
    options.value = newOptions;
  }

  return newOptions;
};

const fetchSelectedEntities = async () => {
  const ids = (Array.isArray(props.modelValue) ? props.modelValue : [props.modelValue]).filter(
    (v) => !!v,
  );

  if (ids.length === 0) {
    return [];
  }

  const criteria = new Criteria(1, ids.length);
  criteria.addFilter(Criteria.equalsAny("id", ids));

  try {
    const result = await repo.value.search(criteria);
    return result ? [...result] : [];
  } catch (e) {
    console.error("Could not fetch selected entities", e);
    return [];
  }
};

onMounted(async () => {
  isLoading.value = true;

  const pagePromise = getPage(currentPage.value);
  let selectedPromise = Promise.resolve<any[]>([]);

  const hasValue =
    (Array.isArray(props.modelValue) && props.modelValue.length > 0) ||
    (!Array.isArray(props.modelValue) &&
      props.modelValue !== null &&
      props.modelValue !== undefined);

  if (hasValue) {
    selectedPromise = fetchSelectedEntities();
  }

  const [pageEntities, selectedEntities] = await Promise.all([pagePromise, selectedPromise]);

  const combined = [...selectedEntities, ...pageEntities];
  const unique = combined.reduce((acc, current) => {
    if (!acc.find((item: { id: string }) => item.id === current.id)) {
      acc.push(current);
    }
    return acc;
  }, [] as any[]);

  options.value = unique;

  isLoading.value = false;
});

const onItemAdd = (item: { [key: string]: any }) => {
  if (!props.valueProperty) {
    return;
  }

  const itemExists = options.value.some(
    (option) => option[props.valueProperty as string] === item[props.valueProperty as string],
  );

  if (!itemExists) {
    options.value.push(item);
  }
};

const fetchSearchData = async ({ searchTerm: newSearchTerm }: { searchTerm: string }) => {
  currentPage.value = 1;
  searchTerm.value = newSearchTerm;

  // We need to set the options here, so the search result is actually displayed
  return fetchData(false, newSearchTerm, true);
};

const fetchNextPage = () => {
  if (isLoading.value) {
    return;
  }

  if (options.value.length >= total.value && total.value > 0) {
    return;
  }
  currentPage.value += 1;
  fetchData(true);
};
</script>
