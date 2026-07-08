<template>
  <FadedScrollableDiv
    class="quick-filters flex flex-1 items-center overflow-x-auto py-1 gap-2 pe-4 ps-1 -ms-1"
    orientation="horizontal"
    v-if="!quickFilters.loading"
  >
    <div
      v-for="filter in quickFilters.data"
      :key="filter.name"
      class="min-w-36"
    >
      <QuickFilterField
        :filter="filter"
        :value="getValue(filter, list.params?.filters)"
        :linkFilters="getLinkFilters(filter)"
        @applyQuickFilter="(f, v) => applyQuickFilter(f, v)"
      />
    </div>
  </FadedScrollableDiv>
</template>

<script setup>
import { FadedScrollableDiv } from "@/components";
import { useAuthStore } from "@/stores/auth";
import { storeToRefs } from "pinia";
import { call } from "frappe-ui";
import { computed, inject, ref, watch } from "vue";
import QuickFilterField from "./QuickFilterField.vue";

const listViewData = inject("listViewData");
const listViewActions = inject("listViewActions");
const { list, quickFilters } = listViewData;

const { isAdmin, ignoreTeamRestrictions, userTeams } = storeToRefs(useAuthStore());

const directValueFilterTypes = ["Check", "Select", "Link", "Date", "Datetime"];
const teamMembersFilter = ref(null);

const canSeeAllTeams = computed(() => {
  return isAdmin.value || ignoreTeamRestrictions.value;
});

const visibleTeams = computed(() => {
  return userTeams.value || [];
});

const selectedTeam = computed(() => {
  const filters = list.params?.filters || {};
  return filters.agent_group || "";
});

const getTeamMembersMethod =
  "helpdesk.helpdesk.doctype.hd_team.hd_team.get_team_members";
let teamMembersRequest = 0;

async function refreshTeamMembersFilter(team) {
  const request = ++teamMembersRequest;
  if (canSeeAllTeams.value) {
    teamMembersFilter.value = null;
    return;
  }
  teamMembersFilter.value = [];

  if (team) {
    const members = await call(getTeamMembersMethod, { team });
    if (request === teamMembersRequest) {
      teamMembersFilter.value = members || [];
    }
    return;
  }

  const teams = visibleTeams.value;
  if (!teams.length) {
    teamMembersFilter.value = [];
    return;
  }

  const members = await Promise.all(
    teams.map((teamName) => call(getTeamMembersMethod, { team: teamName }))
  );
  if (request === teamMembersRequest) {
    teamMembersFilter.value = [...new Set(members.flat().filter(Boolean))];
  }
}

watch(
  [selectedTeam, visibleTeams, canSeeAllTeams],
  ([team]) => {
    refreshTeamMembersFilter(team);
  },
  { immediate: true }
);

function applyQuickFilter(filter, value) {
  let filters = { ...list.params?.filters };

  let field = filter.name;
  if (value) {
    if (directValueFilterTypes.includes(filter.type)) {
      filters[field] = value;
    } else {
      filters[field] = ["LIKE", `%${value}%`];
    }
  } else {
    delete filters[field];
  }

  if (field === "agent_group") {
    delete filters._assign;
  }

  listViewActions.applyFilters(filters);
}

function getLinkFilters(filter) {
  if (canSeeAllTeams.value || filter.type !== "Link") return null;

  if (filter.name === "agent_group" && filter.options === "HD Team") {
    return { name: ["in", visibleTeams.value] };
  }

  if (filter.name === "_assign" && filter.options === "HD Agent") {
    if (teamMembersFilter.value === null) return null;
    return { name: ["in", teamMembersFilter.value] };
  }

  return null;
}

function getDefaultValue(quickFilter) {
  return quickFilter.type === "Check" ? false : "";
}

function getValue(quickFilter, filters) {
  if (!filters || !(quickFilter && quickFilter.name)) {
    return getDefaultValue(quickFilter);
  }
  const filter = filters[quickFilter.name];
  if (filter === undefined) {
    // not a part of the customizable filters
    return getDefaultValue(quickFilter);
  }
  if (directValueFilterTypes.includes(quickFilter.type)) {
    return filter;
  }
  if (
    !Array.isArray(filter) ||
    filter.length !== 2 ||
    filter[0].toLowerCase() !== "like"
  ) {
    return getDefaultValue(quickFilter);
  }
  let value = filter[1];
  if (value.startsWith("%")) {
    value = value.substring(1);
  }
  if (value.endsWith("%")) {
    value = value.substring(0, value.length - 1);
  }
  return value;
}
</script>

<style scoped>
.quick-filters {
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE/Edge */
}
.quick-filters::-webkit-scrollbar {
  display: none; /* Chrome/Safari */
}
</style>
