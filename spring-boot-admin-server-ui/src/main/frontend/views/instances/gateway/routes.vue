<!--
  - Copyright 2014-2018 the original author or authors.
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <div :class="{ 'is-loading' : isLoading }">
    <sba-panel :header-sticks-below="['#navigation']" title="Routes" v-if="routes">
      <refresh-route-cache :instance="instance" @routes-refreshed="fetchRoutes" />

      <div v-if="error" class="message is-danger">
        <div class="message-body">
          <strong>
            <font-awesome-icon class="has-text-danger" icon="exclamation-triangle" />
            Fetching gateway routes failed.
          </strong>
          <p v-text="error.message" />
        </div>
      </div>

      <div class="field">
        <div class="control is-expanded">
          <input class="input" type="search" placeholder="Search routes by name" v-model="routesFilterCriteria">
        </div>
      </div>

      <routes-list :instance="instance" :is-loading="isLoading" :routes="routes" @route-deleted="fetchRoutes" />
    </sba-panel>
    <sba-panel :header-sticks-below="['#navigation']" title="Add Route">
      <add-route :instance="instance" @route-added="fetchRoutes" />
    </sba-panel>
  </div>
</template>

<script>
  import Instance from '@/services/instance';
  import {compareBy} from '@/utils/collections';
  import addRoute from './add-route';
  import refreshRouteCache from './refresh-route-cache';
  import routesList from './routes-list';

  const routeDefinitionMatches = (routeDef, keyword) => {
    if (!routeDef) {
      return false;
    }

    const uriMatches = routeDef.uri && routeDef.uri.toString().toLowerCase().includes(keyword);
    if (uriMatches) {
      return true;
    }

    const predicatesMatches = routeDef.predicates && (
      routeDef.predicates.some(p => p.name.toLowerCase().includes(keyword))
      || routeDef.predicates.some(p => Object.values(p.args).some(pv => pv.toLowerCase().includes(keyword)))
    );
    if (predicatesMatches) {
      return true;
    }

    const filtersMatches = routeDef.filters && (
      routeDef.filters.some(f => f.name.toLowerCase().includes(keyword))
      || routeDef.filters.some(f => Object.values(f.args).some(av => av.toLowerCase().includes(keyword)))
    );
    return Boolean(filtersMatches);
  };

  const routeMatches = (route, keyword) => {
    return route.route_id.toString().toLowerCase().includes(keyword) || routeDefinitionMatches(route.route_definition, keyword);
  };

  const sortRoutes = routes => {
    return [...routes].sort(compareBy(r => r.order))
  };

  export default {
    components: {
      refreshRouteCache,
      routesList,
      addRoute
    },
    props: {
      instance: {
        type: Instance,
        required: true
      }
    },
    data: () => ({
      isLoading: false,
      error: null,
      _routes: [],
      routesFilterCriteria: null
    }),
    computed: {
      routes() {
        if (!this.routesFilterCriteria) {
          return sortRoutes(this.$data._routes);
        }
        const filtered = this.$data._routes.filter(route => routeMatches(route, this.routesFilterCriteria.toLowerCase()));
        return sortRoutes(filtered);
      }
    },
    created() {
      this.fetchRoutes();
    },
    methods: {
      async fetchRoutes() {
        this.error = null;
        this.isLoading = true;
        try {
          const response = await this.instance.fetchGatewayRoutes();
          this.$data._routes = response.data
        } catch (error) {
          console.warn('Fetching routes failed:', error);
          this.error = error;
        }
        this.isLoading = false;
      }
    }
  }
</script>

