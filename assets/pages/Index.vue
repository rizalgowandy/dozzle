<template>
  <div>
    <section class="hero is-small mt-4">
      <div class="hero-body">
        <div class="container">
          <div class="columns">
            <div class="column is-narrow" v-if="secured">
              <a class="button is-primary is-small" :href="`${base}/logout`">{{ $t("button.logout") }}</a>
            </div>
          </div>
        </div>
      </div>
    </section>
    <section class="level section">
      <div class="level-item has-text-centered">
        <div>
          <p class="title">{{ containers.length }}</p>
          <p class="heading">{{ $t("label.total-containers") }}</p>
        </div>
      </div>
      <div class="level-item has-text-centered">
        <div>
          <p class="title">{{ runningContainers.length }}</p>
          <p class="heading">{{ $t("label.running") }}</p>
        </div>
      </div>
      <div class="level-item has-text-centered">
        <div>
          <p class="title" data-ci-skip>{{ totalCpu }}%</p>
          <p class="heading">{{ $t("label.total-cpu-usage") }}</p>
        </div>
      </div>
      <div class="level-item has-text-centered">
        <div>
          <p class="title" data-ci-skip>{{ formatBytes(totalMem) }}</p>
          <p class="heading">{{ $t("label.total-mem-usage") }}</p>
        </div>
      </div>
      <div class="level-item has-text-centered">
        <div>
          <p class="title">{{ version }}</p>
          <p class="heading">{{ $t("label.dozzle-version") }}</p>
        </div>
      </div>
    </section>

    <section class="columns is-centered section is-marginless">
      <div class="column is-4">
        <div class="panel">
          <p class="panel-heading">{{ $t("label.containers") }}</p>
          <div class="panel-block">
            <p class="control has-icons-left">
              <input
                class="input"
                type="text"
                :placeholder="$t('placeholder.search-containers')"
                v-model="query"
                @keyup.esc="query = ''"
                @keyup.enter="onEnter()"
              />
              <span class="icon is-left">
                <search-icon />
              </span>
            </p>
          </div>
          <p class="panel-tabs" v-if="query === ''">
            <a :class="{ 'is-active': sort === 'running' }" @click="sort = 'running'">{{ $t("label.running") }}</a>
            <a :class="{ 'is-active': sort === 'all' }" @click="sort = 'all'">{{ $t("label.all") }}</a>
          </p>
          <router-link
            :to="{ name: 'container-id', params: { id: item.id } }"
            v-for="item in data.slice(0, 10)"
            :key="item.id"
            class="panel-block"
          >
            <span class="name">{{ item.name }}</span>

            <div class="subtitle is-7 status">
              <past-time :date="new Date(item.created * 1000)"></past-time>
            </div>
          </router-link>
        </div>
      </div>
    </section>
  </div>
</template>

<script lang="ts" setup>
import SearchIcon from "~icons/mdi-light/magnify";
import { useFuse } from "@vueuse/integrations/useFuse";

const { base, version, secured } = config;
const containerStore = useContainerStore();
const { containers } = storeToRefs(containerStore);
const router = useRouter();

const sort = $ref("running");
const query = ref("");

const mostRecentContainers = $computed(() => [...containers.value].sort((a, b) => b.created - a.created));
const runningContainers = $computed(() => mostRecentContainers.filter((c) => c.state === "running"));

const { results } = useFuse(query, containers, {
  fuseOptions: { keys: ["name"] },
  matchAllWhenSearchEmpty: false,
});
const data = computed(() => {
  if (results.value.length) {
    return results.value.map(({ item }) => item);
  }
  switch (sort) {
    case "all":
      return mostRecentContainers;
    case "running":
      return runningContainers;
    default:
      throw `Invalid sort order: ${sort}`;
  }
});

let totalCpu = $ref(0);
useIntervalFn(
  () => {
    totalCpu = runningContainers.reduce((acc, c) => acc + (c.stat?.cpu ?? 0), 0);
  },
  1000,
  { immediate: true }
);

let totalMem = $ref(0);
useIntervalFn(
  () => {
    totalMem = runningContainers.reduce((acc, c) => acc + (c.stat?.memoryUsage ?? 0), 0);
  },
  1000,
  { immediate: true }
);

function onEnter() {
  if (data.value.length > 0) {
    const item = data.value[0];
    router.push({ name: "container-id", params: { id: item.id } });
  }
}
</script>
<style lang="scss" scoped>
.panel {
  border: 1px solid var(--border-color);
  .panel-block,
  .panel-tabs {
    border-color: var(--border-color);
    .is-active {
      border-color: var(--border-hover-color);
    }
    .name {
      text-overflow: ellipsis;
      white-space: nowrap;
      overflow: hidden;
    }
    .status {
      margin-left: auto;
      white-space: nowrap;
    }
  }
}

.icon {
  padding: 10px 3px;
}
</style>
