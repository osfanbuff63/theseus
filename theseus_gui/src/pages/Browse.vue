<script setup>
import { ref } from 'vue'
import { ofetch } from 'ofetch'
import {
  Pagination,
  ProjectCard,
  Checkbox,
  Button,
  ClearIcon,
  SearchIcon,
  DropdownSelect,
  SearchFilter,
  Card,
  ClientIcon,
  ServerIcon,
  AnimatedLogo,
} from 'omorphia'
import Multiselect from 'vue-multiselect'
import { useSearch } from '@/store/state'
import { useBreadcrumbs } from '@/store/breadcrumbs'
import { get_categories, get_loaders, get_game_versions } from '@/helpers/tags'
import { useRoute } from 'vue-router'

const searchStore = useSearch()
const breadcrumbs = useBreadcrumbs()
const route = useRoute()
breadcrumbs.setContext({ name: 'Browse', link: route.path })

const showSnapshots = ref(false)
const loading = ref(true)

const [categories, loaders, availableGameVersions] = await Promise.all([
  get_categories(),
  get_loaders(),
  get_game_versions(),
])

const getSearchResults = async (shouldLoad = false) => {
  const queryString = searchStore.getQueryString()
  if (shouldLoad === true) {
    loading.value = true
  }
  const response = await ofetch(`https://api.modrinth.com/v2/search${queryString}`)
  loading.value = false
  searchStore.setSearchResults(response)
}

getSearchResults(true)

const toggleFacet = async (facet) => {
  const index = searchStore.facets.indexOf(facet)

  if (index !== -1) searchStore.facets.splice(index, 1)
  else searchStore.facets.push(facet)

  await getSearchResults()
}

const toggleOrFacet = async (orFacet) => {
  const index = searchStore.orFacets.indexOf(orFacet)

  if (index !== -1) searchStore.orFacets.splice(index, 1)
  else searchStore.orFacets.push(orFacet)

  await getSearchResults()
}

const switchPage = async (page) => {
  searchStore.currentPage = parseInt(page)
  if (page === 1) searchStore.offset = 0
  else searchStore.offset = (searchStore.currentPage - 1) * searchStore.limit
  await getSearchResults()
}

const handleReset = async () => {
  searchStore.resetFilters()
  await getSearchResults()
}
</script>

<template>
  <div class="search-container">
    <aside class="filter-panel">
      <Button
        role="button"
        :disabled="
          !(
            searchStore.facets.length > 0 ||
            searchStore.orFacets.length > 0 ||
            searchStore.environments.server === true ||
            searchStore.environments.client === true ||
            searchStore.openSource === true ||
            searchStore.activeVersions.length > 0
          )
        "
        @click="handleReset"
        ><ClearIcon />Clear Filters</Button
      >
      <div class="categories">
        <h2>Categories</h2>
        <div
          v-for="category in categories.filter((cat) => cat.project_type === 'modpack')"
          :key="category.name"
        >
          <SearchFilter
            :active-filters="searchStore.facets"
            :icon="category.icon"
            :display-name="category.name"
            :facet-name="`categories:${encodeURIComponent(category.name)}`"
            class="filter-checkbox"
            @toggle="toggleFacet"
          />
        </div>
      </div>
      <div class="loaders">
        <h2>Loaders</h2>
        <div
          v-for="loader in loaders.filter((l) => l.supported_project_types?.includes('modpack'))"
          :key="loader"
        >
          <SearchFilter
            :active-filters="searchStore.orFacets"
            :icon="loader.icon"
            :display-name="loader.name"
            :facet-name="`categories:${encodeURIComponent(loader.name)}`"
            class="filter-checkbox"
            @toggle="toggleOrFacet"
          />
        </div>
      </div>
      <div class="environment">
        <h2>Environments</h2>
        <SearchFilter
          v-model="searchStore.environments.client"
          display-name="Client"
          facet-name="client"
          class="filter-checkbox"
          @click="getSearchResults"
        >
          <ClientIcon aria-hidden="true" />
        </SearchFilter>
        <SearchFilter
          v-model="searchStore.environments.server"
          display-name="Server"
          facet-name="server"
          class="filter-checkbox"
          @click="getSearchResults"
        >
          <ServerIcon aria-hidden="true" />
        </SearchFilter>
      </div>
      <div class="versions">
        <h2>Minecraft versions</h2>
        <Checkbox v-model="showSnapshots" class="filter-checkbox">Show snapshots</Checkbox>
        <multiselect
          v-model="searchStore.activeVersions"
          :options="
            showSnapshots
              ? availableGameVersions.map((x) => x.version)
              : availableGameVersions
                  .filter((it) => it.version_type === 'release')
                  .map((x) => x.version)
          "
          :multiple="true"
          :searchable="true"
          :show-no-results="false"
          :close-on-select="false"
          :clear-search-on-select="false"
          :show-labels="false"
          placeholder="Choose versions..."
          @update:model-value="getSearchResults"
        />
      </div>
      <div class="open-source">
        <h2>Open source</h2>
        <Checkbox
          v-model="searchStore.openSource"
          class="filter-checkbox"
          @click="getSearchResults"
        >
          Open source
        </Checkbox>
      </div>
    </aside>
    <div class="search">
      <Card class="search-panel-container">
        <div class="search-panel">
          <div class="iconified-input">
            <SearchIcon aria-hidden="true" />
            <input
              v-model="searchStore.searchInput"
              type="text"
              placeholder="Search modpacks..."
              @input="getSearchResults"
            />
          </div>
          <span>Sort by</span>
          <DropdownSelect
            v-model="searchStore.filter"
            name="Sort dropdown"
            :options="[
              'Relevance',
              'Download count',
              'Follow count',
              'Recently published',
              'Recently updated',
            ]"
            class="sort-dropdown"
            @change="getSearchResults"
          />
          <span>Show per page</span>
          <DropdownSelect
            v-model="searchStore.limit"
            name="Limit dropdown"
            :options="[5, 10, 15, 20, 50, 100]"
            :default-value="searchStore.limit.toString()"
            :model-value="searchStore.limit.toString()"
            class="limit-dropdown"
            @change="getSearchResults"
          />
        </div>
      </Card>
      <Pagination
        :page="searchStore.currentPage"
        :count="searchStore.pageCount"
        @switch-page="switchPage"
      />
      <AnimatedLogo v-if="loading" class="loading" />
      <section v-else class="project-list display-mode--list instance-results" role="list">
        <ProjectCard
          v-for="result in searchStore.searchResults"
          :id="`${result?.project_id}/`"
          :key="result?.project_id"
          class="result-project-item"
          :type="result?.project_type"
          :name="result?.title"
          :description="result?.description"
          :icon-url="result?.icon_url"
          :downloads="result?.downloads?.toString()"
          :follows="result?.follows?.toString()"
          :created-at="result?.date_created"
          :updated-at="result?.date_modified"
          :categories="[
            ...categories.filter(
              (cat) =>
                result?.display_categories.includes(cat.name) && cat.project_type === 'modpack'
            ),
            ...loaders.filter(
              (loader) =>
                result?.display_categories.includes(loader.name) &&
                loader.supported_project_types?.includes('modpack')
            ),
          ]"
          :project-type-display="result?.project_type"
          project-type-url="project"
          :server-side="result?.server_side"
          :client-side="result?.client_side"
          :show-updated-date="false"
          :color="result?.color"
        />
      </section>
    </div>
  </div>
</template>

<style src="vue-multiselect/dist/vue-multiselect.css"></style>
<style lang="scss">
.search-panel-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 100%;
  margin-top: 1rem;
  padding: 0.8rem !important;

  .search-panel {
    display: flex;
    align-items: center;
    justify-content: space-evenly;
    width: 100%;
    gap: 1rem;
    margin: 1rem auto;
    white-space: nowrap;

    .sort-dropdown {
      min-width: 12.18rem;
    }

    .limit-dropdown {
      width: 5rem;
    }

    .iconified-input {
      width: 75%;

      input {
        flex-basis: initial;
      }
    }
  }

  .filter-panel {
    button {
      display: flex;
      align-items: center;
      justify-content: space-evenly;

      svg {
        margin-right: 0.4rem;
      }
    }
  }
}

.search-container {
  display: flex;

  .filter-panel {
    position: fixed;
    width: 16rem;
    background: var(--color-raised-bg);
    padding: 1rem 1rem 3rem 1rem;
    display: flex;
    flex-direction: column;
    min-height: 100vh;
    height: fit-content;
    max-height: 100%;
    overflow-y: auto;

    h2 {
      color: var(--color-contrast);
      margin-top: 1rem;
      margin-bottom: 0.5rem;
      font-size: 1.16rem;
    }

    .filter-checkbox {
      margin-bottom: 0.3rem;
      font-size: 1rem;
      text-transform: capitalize;

      svg {
        display: flex;
        align-self: center;
        justify-self: center;
      }

      button.checkbox {
        border: none;
      }
    }
  }

  .search {
    margin: 0 1rem 0 17rem;
    width: 100%;

    .loading {
      margin: 2rem;
      text-align: center;
    }
  }
}
</style>
