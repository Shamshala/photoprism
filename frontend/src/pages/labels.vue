<template>
  <div v-infinite-scroll="loadMore" class="p-page p-page-labels" :infinite-scroll-disabled="scrollDisabled"
       :infinite-scroll-distance="1200" :infinite-scroll-listen-for-event="'scrollRefresh'">

    <v-form ref="form" class="p-labels-search" lazy-validation dense @submit.prevent="updateQuery">
      <v-toolbar flat color="secondary" :dense="$vuetify.breakpoint.smAndDown">
        <v-text-field id="search"
                      v-model="filter.q"
                      class="pt-3 pr-3 input-search"
                      single-line
                      :label="$gettext('Search')"
                      prepend-inner-icon="search"
                      browser-autocomplete="off"
                      clearable
                      color="secondary-dark"
                      @click:clear="clearQuery"
                      @keyup.enter.native="updateQuery"
        ></v-text-field>

        <v-spacer></v-spacer>

        <v-btn icon class="action-reload" :title="$gettext('Reload')" @click.stop="refresh">
          <v-icon>refresh</v-icon>
        </v-btn>

        <v-btn v-if="!filter.all" icon class="action-show-all" :title="$gettext('Show more')" @click.stop="showAll">
          <v-icon>visibility</v-icon>
        </v-btn>
        <v-btn v-else icon class="action-show-important" :title="$gettext('Show less')" @click.stop="showImportant">
          <v-icon>visibility_off</v-icon>
        </v-btn>
      </v-toolbar>
    </v-form>

    <v-container v-if="loading" fluid class="pa-4">
      <v-progress-linear color="secondary-dark" :indeterminate="true"></v-progress-linear>
    </v-container>
    <v-container v-else fluid class="pa-0">
      <p-label-clipboard :refresh="refresh" :selection="selection"
                         :clear-selection="clearSelection"></p-label-clipboard>

      <p-scroll-top></p-scroll-top>

      <v-container grid-list-xs fluid class="pa-2">
        <v-card v-if="results.length === 0" class="no-results secondary-light lighten-1 ma-1" flat>
          <v-card-title primary-title>
            <div>
              <h3 class="title ma-0 pa-0">
                <translate>Couldn't find anything</translate>
              </h3>
              <p class="mt-4 mb-0 pa-0">
                <translate>Try again using other filters or keywords.</translate>
              </p>
            </div>
          </v-card-title>
        </v-card>
        <v-layout row wrap class="search-results label-results cards-view">
          <v-flex
              v-for="(label, index) in results"
              :key="index"
              xs6 sm4 md3 lg2 xxl1 d-flex
          >
            <v-card tile
                    :data-uid="label.UID"
                    class="result accent lighten-3"
                    :class="label.classes(selection.includes(label.UID))"
                    :to="{name: 'all', query: {q: 'label:' + (label.CustomSlug ? label.CustomSlug : label.Slug)}}"
                    @contextmenu="onContextMenu($event, index)"
            >
              <div class="card-background accent lighten-3"></div>
              <v-img
                  :src="label.thumbnailUrl('tile_500')"
                  :alt="label.Name"
                  :transition="false"
                  aspect-ratio="1"
                  class="accent lighten-2 clickable"
                  @mousedown="onMouseDown($event, index)"
                  @click="onClick($event, index)"
              >
                <v-btn :ripple="false"
                       icon flat absolute
                       class="input-select"
                       @click.stop.prevent="onSelect($event, index)">
                  <v-icon color="white" class="select-on">check_circle</v-icon>
                  <v-icon color="accent lighten-3" class="select-off">radio_button_off</v-icon>
                </v-btn>

                <v-btn :ripple="false"
                       icon flat absolute
                       class="input-favorite"
                       @click.stop.prevent="label.toggleLike()">
                  <v-icon color="#FFD600" class="select-on">star</v-icon>
                  <v-icon color="white" class="select-off">star_border</v-icon>
                </v-btn>
              </v-img>

              <v-card-title primary-title class="pa-3 card-details" style="user-select: none;" @click.stop.prevent="">
                <v-edit-dialog
                    :return-value.sync="label.Name"
                    lazy
                    class="inline-edit"
                    @save="onSave(label)"
                >
                  <span v-if="label.Name" class="body-2 ma-0">
                      {{ label.Name | capitalize }}
                  </span>
                  <span v-else>
                      <v-icon>edit</v-icon>
                  </span>
                  <template #input>
                    <v-text-field
                        v-model="label.Name"
                        :rules="[titleRule]"
                        :label="$gettext('Label Name')"
                        color="secondary-dark"
                        single-line
                        autofocus
                    ></v-text-field>
                  </template>
                </v-edit-dialog>
              </v-card-title>
            </v-card>
          </v-flex>
        </v-layout>
      </v-container>
    </v-container>
  </div>
</template>

<script>
import Label from "model/label";
import Event from "pubsub-js";
import RestModel from "model/rest";
import {MaxItems} from "../common/clipboard";
import Notify from "../common/notify";

export default {
  name: 'PPageLabels',
  props: {
    staticFilter: Object
  },
  data() {
    const query = this.$route.query;
    const routeName = this.$route.name;
    const q = query['q'] ? query['q'] : '';
    const all = query['all'] ? query['all'] : '';
    const filter = {q: q, all: all};
    const settings = {};

    return {
      config: this.$config.values,
      subscriptions: [],
      listen: false,
      dirty: false,
      results: [],
      scrollDisabled: true,
      loading: true,
      batchSize: Label.batchSize(),
      offset: 0,
      page: 0,
      selection: [],
      settings: settings,
      filter: filter,
      lastFilter: {},
      routeName: routeName,
      titleRule: v => v.length <= this.$config.get('clip') || this.$gettext("Name too long"),
      mouseDown: {
        index: -1,
        timeStamp: -1,
      },
      lastId: "",
    };
  },
  watch: {
    '$route'() {
      const query = this.$route.query;

      this.filter.q = query['q'] ? query['q'] : '';
      this.filter.all = query['all'] ? query['all'] : '';
      this.lastFilter = {};
      this.routeName = this.$route.name;
      this.search();
    }
  },
  created() {
    this.search();

    this.subscriptions.push(Event.subscribe("labels", (ev, data) => this.onUpdate(ev, data)));

    this.subscriptions.push(Event.subscribe("touchmove.top", () => this.refresh()));
    this.subscriptions.push(Event.subscribe("touchmove.bottom", () => this.loadMore()));
  },
  destroyed() {
    for (let i = 0; i < this.subscriptions.length; i++) {
      Event.unsubscribe(this.subscriptions[i]);
    }
  },
  methods: {
    selectRange(rangeEnd, models) {
      if (!models || !models[rangeEnd] || !(models[rangeEnd] instanceof RestModel)) {
        console.warn("selectRange() - invalid arguments:", rangeEnd, models);
        return;
      }

      let rangeStart = models.findIndex((m) => m.getId() === this.lastId);

      if (rangeStart === -1) {
        this.toggleSelection(models[rangeEnd].getId());
        return 1;
      }

      if (rangeStart > rangeEnd) {
        const newEnd = rangeStart;
        rangeStart = rangeEnd;
        rangeEnd = newEnd;
      }

      for (let i = rangeStart; i <= rangeEnd; i++) {
        this.addSelection(models[i].getId());
      }

      return (rangeEnd - rangeStart) + 1;
    },
    onSelect(ev, index) {
      if (ev.shiftKey) {
        this.selectRange(index, this.results);
      } else {
        this.toggleSelection(this.results[index].getId());
      }
    },
    onMouseDown(ev, index) {
      this.mouseDown.index = index;
      this.mouseDown.timeStamp = ev.timeStamp;
    },
    onClick(ev, index) {
      let longClick = (this.mouseDown.index === index && ev.timeStamp - this.mouseDown.timeStamp > 400);

      if (longClick || this.selection.length > 0) {
        ev.preventDefault();
        ev.stopPropagation();

        if (longClick || ev.shiftKey) {
          this.selectRange(index, this.results);
        } else {
          this.toggleSelection(this.results[index].getId());
        }
      }
    },
    onContextMenu(ev, index) {
      if (this.$isMobile) {
        ev.preventDefault();
        ev.stopPropagation();

        if (this.results[index]) {
          this.selectRange(index, this.results);
        }
      }
    },
    onSave(label) {
      label.update();
    },
    showAll() {
      this.filter.all = "true";
      this.updateQuery();
    },
    showImportant() {
      this.filter.all = "";
      this.updateQuery();
    },
    clearQuery() {
      this.filter.q = '';
      this.updateQuery();
    },
    addSelection(uid) {
      const pos = this.selection.indexOf(uid);

      if (pos === -1) {
        if (this.selection.length >= MaxItems) {
          Notify.warn(this.$gettext("Can't select more items"));
          return;
        }

        this.selection.push(uid);
        this.lastId = uid;
      }
    },
    toggleSelection(uid) {
      const pos = this.selection.indexOf(uid);

      if (pos !== -1) {
        this.selection.splice(pos, 1);
        this.lastId = "";
      } else {
        if (this.selection.length >= MaxItems) {
          Notify.warn(this.$gettext("Can't select more items"));
          return;
        }

        this.selection.push(uid);
        this.lastId = uid;
      }
    },
    removeSelection(uid) {
      const pos = this.selection.indexOf(uid);

      if (pos !== -1) {
        this.selection.splice(pos, 1);
        this.lastId = "";
      }
    },
    clearSelection() {
      this.selection.splice(0, this.selection.length);
      this.lastId = "";
    },
    loadMore() {
      if (this.scrollDisabled) return;

      this.scrollDisabled = true;
      this.listen = false;

      const count = this.dirty ? (this.page + 2) * this.batchSize : this.batchSize;
      const offset = this.dirty ? 0 : this.offset;

      const params = {
        count: count,
        offset: offset,
      };

      Object.assign(params, this.lastFilter);

      if (this.staticFilter) {
        Object.assign(params, this.staticFilter);
      }

      Label.search(params).then(response => {
        this.results = this.dirty ? response.models : this.results.concat(response.models);

        this.scrollDisabled = (response.models.length < count);

        if (this.scrollDisabled) {
          this.offset = offset;
          if (this.results.length > 1) {
            this.$notify.info(this.$gettextInterpolate(this.$gettext("All %{n} labels loaded"), {n: this.results.length}));
          }
        } else {
          this.offset = offset + count;
          this.page++;

          this.$nextTick(() => {
            if (this.$root.$el.clientHeight <= window.document.documentElement.clientHeight + 300) {
              this.$emit("scrollRefresh");
            }
          });
        }
      }).catch(() => {
        this.scrollDisabled = false;
      }).finally(() => {
        this.dirty = false;
        this.loading = false;
        this.listen = true;
      });
    },
    updateQuery() {
      this.filter.q = this.filter.q.trim();

      const query = {
        view: this.settings.view
      };

      Object.assign(query, this.filter);

      for (let key in query) {
        if (query[key] === undefined || !query[key]) {
          delete query[key];
        }
      }

      if (JSON.stringify(this.$route.query) === JSON.stringify(query)) {
        return;
      }

      this.$router.replace({query: query});
    },
    searchParams() {
      const params = {
        count: this.batchSize,
        offset: this.offset,
      };

      Object.assign(params, this.filter);

      if (this.staticFilter) {
        Object.assign(params, this.staticFilter);
      }

      return params;
    },
    refresh() {
      if (this.loading) return;
      this.loading = true;
      this.page = 0;
      this.dirty = true;
      this.scrollDisabled = false;
      this.loadMore();
    },
    search() {
      this.scrollDisabled = true;

      // Don't query the same data more than once
      if (JSON.stringify(this.lastFilter) === JSON.stringify(this.filter)) {
        this.$nextTick(() => this.$emit("scrollRefresh"));
        return;
      }

      Object.assign(this.lastFilter, this.filter);

      this.offset = 0;
      this.page = 0;
      this.loading = true;
      this.listen = false;

      const params = this.searchParams();

      Label.search(params).then(response => {
        this.offset = this.batchSize;

        this.results = response.models;

        this.scrollDisabled = (response.models.length < this.batchSize);

        if (this.scrollDisabled) {
          this.$notify.info(this.$gettextInterpolate(this.$gettext("%{n} labels found"), {n: this.results.length}));
        } else {
          this.$notify.info(this.$gettext('More than 20 labels found'));

          this.$nextTick(() => {
            if (this.$root.$el.clientHeight <= window.document.documentElement.clientHeight + 300) {
              this.$emit("scrollRefresh");
            }
          });
        }
      }).finally(() => {
        this.dirty = false;
        this.loading = false;
        this.listen = true;
      });
    },
    onUpdate(ev, data) {
      if (!this.listen) return;

      if (!data || !data.entities) {
        return;
      }

      const type = ev.split('.')[1];

      switch (type) {
        case 'updated':
          for (let i = 0; i < data.entities.length; i++) {
            const values = data.entities[i];
            const model = this.results.find((m) => m.UID === values.UID);

            if (model) {
              for (let key in values) {
                if (values.hasOwnProperty(key) && values[key] != null && typeof values[key] !== "object") {
                  model[key] = values[key];
                }
              }
            }
          }
          break;
        case 'deleted':
          this.dirty = true;

          for (let i = 0; i < data.entities.length; i++) {
            const uid = data.entities[i];
            const index = this.results.findIndex((m) => m.UID === uid);

            if (index >= 0) {
              this.results.splice(index, 1);
            }

            this.removeSelection(uid);
          }

          break;
        case 'created':
          this.dirty = true;
          break;
        default:
          console.warn("unexpected event type", ev);
      }
    }
  },
};
</script>
