<template>
  <div>
    <div v-if="photos.length === 0" class="pa-2">
      <v-card class="no-results secondary-light lighten-1 ma-1" flat>
        <v-card-title primary-title>
          <div>
            <h3 v-if="filter.order === 'edited'" class="title ma-0 pa-0">
              <translate>Couldn't find recently edited</translate>
            </h3>
            <h3 v-else class="title ma-0 pa-0">
              <translate>Couldn't find anything</translate>
            </h3>
            <p class="mt-4 mb-0 pa-0">
              <translate>Try again using other filters or keywords.</translate>
            </p>
          </div>
        </v-card-title>
      </v-card>
    </div>
    <v-data-table v-else
                  v-model="selected"
                  :headers="listColumns"
                  :items="photos"
                  hide-actions
                  class="search-results photo-results list-view"
                  disable-initial-sort
                  item-key="ID"
                  :no-data-text="notFoundMessage"
    >
      <template slot:items="props">
        <td style="user-select: none;" :data-uid="props.item.UID" class="result" :class="props.item.classes()">
          <v-img :key="props.item.Hash"
                 :src="props.item.thumbnailUrl('tile_50')"
                 :alt="props.item.Title"
                 :transition="false"
                 aspect-ratio="1"
                 class="accent lighten-2 clickable"
                 @mousedown="onMouseDown($event, props.index)"
                 @contextmenu="onContextMenu($event, props.index)"
                 @click.stop.prevent="onClick($event, props.index)"
          >
            <v-btn v-if="selectMode" :ripple="false"
                   flat icon large absolute
                   class="input-select">
              <v-icon color="white" class="select-on">check_circle</v-icon>
              <v-icon color="accent lighten-3" class="select-off">radio_button_off</v-icon>
            </v-btn>
            <v-btn v-else-if="props.item.Type === 'video' || props.item.Type === 'live'"
                   :ripple="false"
                   flat icon large absolute class="input-play opacity-75"
                   @click.stop.prevent="openPhoto(props.index, true)">
              <v-icon color="white" class="action-play">play_arrow</v-icon>
            </v-btn>
          </v-img>
        </td>

        <td class="p-photo-desc clickable" :data-uid="props.item.UID" style="user-select: none;"
            @click.stop.prevent="openPhoto(props.index, false)">
          {{ props.item.Title }}
        </td>
        <td class="p-photo-desc hidden-xs-only" :title="props.item.getDateString()">
          <button style="user-select: none;" @click.stop.prevent="editPhoto(props.index)">
            {{ props.item.shortDateString() }}
          </button>
        </td>
        <td class="p-photo-desc hidden-sm-and-down" style="user-select: none;">
          <button @click.stop.prevent="editPhoto(props.index)">
            {{ props.item.CameraMake }} {{ props.item.CameraModel }}
          </button>
        </td>
        <td class="p-photo-desc hidden-xs-only">
          <button v-if="filter.order === 'name'"
                  :title="$gettext('Name')" @click.exact="downloadFile(props.index)">
            {{ props.item.FileName }}
          </button>
          <button v-else-if="props.item.Country !== 'zz' && showLocation"
                  style="user-select: none;"
                  @click.stop.prevent="openLocation(props.index)">
            {{ props.item.locationInfo() }}
          </button>
          <span v-else>
                    {{ props.item.locationInfo() }}
                </span>
        </td>
      </template>
    </v-data-table>
  </div>
</template>
<script>
export default {
  name: 'PPhotoList',
  props: {
    photos: Array,
    openPhoto: Function,
    editPhoto: Function,
    openLocation: Function,
    album: Object,
    filter: Object,
    context: String,
    selectMode: Boolean,
  },
  data() {
    let m = this.$gettext("Couldn't find anything.");

    m += " " + this.$gettext("Try again using other filters or keywords.");

    let showName = this.filter.order === 'name';

    return {
      config: this.$config.values,
      notFoundMessage: m,
      'selected': [],
      'listColumns': [
        {text: '', value: '', align: 'center', class: 'p-col-select', sortable: false},
        {text: this.$gettext('Title'), value: 'Title', sortable: false},
        {text: this.$gettext('Taken'), class: 'hidden-xs-only', value: 'TakenAt', sortable: false},
        {text: this.$gettext('Camera'), class: 'hidden-sm-and-down', value: 'CameraModel', sortable: false},
        {
          text: showName ? this.$gettext('Name') : this.$gettext('Location'),
          class: 'hidden-xs-only',
          value: showName ? 'FileName' : 'PlaceLabel',
          sortable: false
        },
      ],
      showName: showName,
      showLocation: this.$config.values.settings.features.places,
      hidePrivate: this.$config.values.settings.features.private,
      mouseDown: {
        index: -1,
        timeStamp: -1,
      },
    };
  },
  methods: {
    downloadFile(index) {
      const photo = this.photos[index];
      const link = document.createElement('a');
      link.href = `/api/v1/dl/${photo.Hash}?t=${this.$config.downloadToken()}`;
      link.download = photo.FileName;
      link.click();
    },
    onSelect(ev, index) {
      if (ev.shiftKey) {
        this.selectRange(index);
      } else {
        this.toggle(this.photos[index]);
      }
    },
    onMouseDown(ev, index) {
      this.mouseDown.index = index;
      this.mouseDown.timeStamp = ev.timeStamp;
    },
    onClick(ev, index) {
      let longClick = (this.mouseDown.index === index && ev.timeStamp - this.mouseDown.timeStamp > 400);

      if (longClick || this.selectMode) {
        if (longClick || ev.shiftKey) {
          this.selectRange(index);
        } else {
          this.toggle(this.photos[index]);
        }
      } else if (this.photos[index]) {
        let photo = this.photos[index];

        if (photo.Type === 'video' && photo.isPlayable()) {
          this.openPhoto(index, true);
        } else {
          this.openPhoto(index, false);
        }
      }
    },
    onContextMenu(ev, index) {
      if (this.$isMobile) {
        ev.preventDefault();
        ev.stopPropagation();
        this.selectRange(index);
      }
    },
    toggle(photo) {
      this.$clipboard.toggle(photo);
    },
    selectRange(index) {
      this.$clipboard.addRange(index, this.photos);
    },
  }
};
</script>
