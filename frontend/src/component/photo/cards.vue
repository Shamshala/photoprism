<template>
  <v-container grid-list-xs fluid class="pa-2 p-photos p-photo-cards">
    <v-card v-if="photos.length === 0" class="no-results secondary-light lighten-1 ma-1" flat>
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
            <translate>If a file you expect is missing, please re-index your library and wait until indexing has been completed.</translate>
            <template v-if="$config.feature('review')" class="mt-2 mb-0 pa-0">
              <translate>Non-photographic and low-quality images require a review before they appear in search results.</translate>
            </template>
          </p>
        </div>
      </v-card-title>
    </v-card>
    <v-layout row wrap class="search-results photo-results cards-view">
      <v-flex
          v-for="(photo, index) in photos"
          :key="index"
          xs12 sm6 md4 lg3 xlg2 xxxl1 d-flex
      >
        <v-card tile
                :data-id="photo.ID"
                :data-uid="photo.UID"
                class="result accent lighten-3"
                :class="photo.classes()"
                @contextmenu="onContextMenu($event, index)">
          <div class="card-background accent lighten-3"></div>
          <v-img :key="photo.Hash"
                 :src="photo.thumbnailUrl('tile_500')"
                 :alt="photo.Title"
                 :title="photo.Title"
                 :transition="false"
                 aspect-ratio="1"
                 class="accent lighten-2 clickable"
                 @mousedown="onMouseDown($event, index)"
                 @click.stop.prevent="onClick($event, index)"
                 @mouseover="playLive(photo)"
                 @mouseleave="pauseLive(photo)"
          >
            <v-layout v-if="photo.Type === 'live'" class="live-player">
              <video :id="'live-player-' + photo.ID" :key="photo.ID" width="500" height="500" preload="none"
                     loop muted playsinline>
                <source :src="photo.videoUrl()" type="video/mp4">
              </video>
            </v-layout>

            <v-btn :ripple="false" :depressed="false" class="input-open"
                   icon flat absolute
                   @click.stop.prevent="openPhoto(index, true)">
              <v-icon color="white" class="default-hidden action-raw" :title="$gettext('RAW')">photo_camera</v-icon>
              <v-icon color="white" class="default-hidden action-live" :title="$gettext('Live')">adjust</v-icon>
              <v-icon color="white" class="default-hidden action-stack" :title="$gettext('Stack')">burst_mode</v-icon>
            </v-btn>

            <v-btn :ripple="false" :depressed="false" class="input-view"
                   icon flat absolute :title="$gettext('View')"
                   @click.stop.prevent="openPhoto(index, false)">
              <v-icon color="white" class="action-fullscreen">zoom_in</v-icon>
            </v-btn>

            <v-btn :ripple="false" :depressed="false" color="white" class="input-play"
                   outline fab absolute :title="$gettext('Play')"
                   @click.stop.prevent="openPhoto(index, true)">
              <v-icon color="white" class="action-play">play_arrow</v-icon>
            </v-btn>

            <v-btn v-if="hidePrivate" :ripple="false"
                   icon flat absolute
                   class="input-private">
              <v-icon color="white" class="select-on">lock</v-icon>
            </v-btn>

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
                   @click.stop.prevent="photo.toggleLike()">
              <v-icon color="white" class="select-on">favorite</v-icon>
              <v-icon color="accent lighten-3" class="select-off">favorite_border</v-icon>
            </v-btn>
          </v-img>

          <v-card-title primary-title class="pa-3 card-details" style="user-select: none;">
            <div>
              <h3 class="body-2 mb-2" :title="photo.Title">
                <button class="action-title-edit" :data-uid="photo.UID"
                        @click.exact="editPhoto(index)">
                  {{ photo.Title | truncate(80) }}
                </button>
              </h3>
              <div v-if="photo.Description" class="caption mb-2" :title="labels.description">
                <button @click.exact="editPhoto(index)">
                  {{ photo.Description }}
                </button>
              </div>
              <div class="caption">
                <button class="action-date-edit" :data-uid="photo.UID"
                        @click.exact="editPhoto(index)">
                  <v-icon size="14" :title="labels.taken">date_range</v-icon>
                  {{ photo.getDateString() }}
                </button>
                <template v-if="!photo.Description">
                  <br/>
                  <button v-if="photo.Type === 'video'" :title="labels.video"
                          @click.exact="openPhoto(index, true)">
                    <v-icon size="14">movie</v-icon>
                    {{ photo.getVideoInfo() }}
                  </button>
                  <button v-else :title="labels.camera" class="action-camera-edit"
                          :data-uid="photo.UID" @click.exact="editPhoto(index)">
                    <v-icon size="14">photo_camera</v-icon>
                    {{ photo.getPhotoInfo() }}
                  </button>
                </template>
                <template v-if="filter.order === 'name' && $config.feature('download')">
                  <br/>
                  <button :title="labels.name"
                          @click.exact="downloadFile(index)">
                    <v-icon size="14">insert_drive_file</v-icon>
                    {{ photo.baseName() }}
                  </button>
                </template>
                <template v-if="showLocation && photo.Country !== 'zz'">
                  <br/>
                  <button :title="labels.location" class="action-location"
                          :data-uid="photo.UID" @click.exact="openLocation(index)">
                    <v-icon size="14">location_on</v-icon>
                    {{ photo.locationInfo() }}
                  </button>
                </template>
              </div>
            </div>
          </v-card-title>

          <v-card-actions v-if="photo.Quality < 3 && context === 'review'" class="card-details">
            <v-layout row wrap align-center>
              <v-flex xs12>
                <div class="text-xs-center">
                  <v-btn color="secondary-dark" small depressed dark class="action-archive text-xs-center"
                         :title="labels.archive" @click.stop="photo.archive()">
                    <v-icon dark>archive</v-icon>
                  </v-btn>
                  <v-btn color="secondary-dark" small depressed dark class="action-approve text-xs-center"
                         :title="labels.approve" @click.stop="photo.approve()">
                    <v-icon dark>check</v-icon>
                  </v-btn>
                </div>
              </v-flex>
            </v-layout>
          </v-card-actions>
        </v-card>
      </v-flex>
    </v-layout>
  </v-container>
</template>
<script>
export default {
  name: 'PPhotoCards',
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
    return {
      showLocation: this.$config.settings().features.places,
      hidePrivate: this.$config.settings().features.private,
      debug: this.$config.get('debug'),
      labels: {
        location: this.$gettext("Location"),
        description: this.$gettext("Description"),
        taken: this.$gettext("Taken"),
        approve: this.$gettext("Approve"),
        archive: this.$gettext("Archive"),
        camera: this.$gettext("Camera"),
        video: this.$gettext("Video"),
        name: this.$gettext("Name"),
      },
      mouseDown: {
        index: -1,
        timeStamp: -1,
      },
    };
  },
  methods: {
    livePlayer(photo) {
      return document.querySelector("#live-player-" + photo.ID);
    },
    playLive(photo) {
      const player = this.livePlayer(photo);
      if (player) player.play();
    },
    pauseLive(photo) {
      const player = this.livePlayer(photo);
      if (player) player.pause();
    },
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
        this.$clipboard.toggle(this.photos[index]);
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
          this.$clipboard.toggle(this.photos[index]);
        }
      } else {
        this.openPhoto(index, false);
      }
    },
    onContextMenu(ev, index) {
      if (this.$isMobile) {
        ev.preventDefault();
        ev.stopPropagation();
        this.selectRange(index);
      }
    },
    selectRange(index) {
      this.$clipboard.addRange(index, this.photos);
    },
  }
};
</script>
