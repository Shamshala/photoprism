<template>
  <div>
    <v-container v-if="selection.length > 0" fluid class="pa-0">
      <v-speed-dial
          id="t-clipboard" v-model="expanded"
          fixed
          bottom
          direction="top"
          transition="slide-y-reverse-transition"
          :right="!rtl"
          :left="rtl"
          :class="`p-clipboard ${!rtl ? '--ltr' : '--rtl'} p-label-clipboard`"
      >
        <template v-slot:activator>
          <v-btn
              fab
              dark
              color="accent darken-2"
              class="action-menu"
          >
            <v-icon v-if="selection.length === 0">menu</v-icon>
            <span v-else class="count-clipboard">{{ selection.length }}</span>
          </v-btn>
        </template>

        <!-- v-btn
                fab
                dark
                small
                :title="$gettext('Download')"
                color="download"
                @click.stop="download()"
                class="p-label-clipboard-download"
                :disabled="selection.length !== 1"
        >
            <v-icon>cloud_download</v-icon>
        </v-btn -->
        <v-btn
            v-if="$config.feature('albums')"
            fab dark small
            :title="$gettext('Add to album')"
            color="album"
            :disabled="selection.length === 0"
            class="action-album"
            @click.stop="dialog.album = true"
        >
          <v-icon>bookmark</v-icon>
        </v-btn>
        <v-btn
            fab dark small
            color="remove"
            :title="$gettext('Delete')"
            :disabled="selection.length === 0"
            class="action-delete"
            @click.stop="dialog.delete = true"
        >
          <v-icon>delete</v-icon>
        </v-btn>

        <v-btn
            fab dark small
            color="accent"
            class="action-clear"
            @click.stop="clearClipboard()"
        >
          <v-icon>clear</v-icon>
        </v-btn>
      </v-speed-dial>
    </v-container>
    <p-photo-album-dialog :show="dialog.album" @cancel="dialog.album = false"
                          @confirm="addToAlbum"></p-photo-album-dialog>
    <p-label-delete-dialog :show="dialog.delete" @cancel="dialog.delete = false"
                           @confirm="batchDelete"></p-label-delete-dialog>
  </div>
</template>
<script>
import Api from "common/api";
import Notify from "common/notify";

export default {
  name: 'PLabelClipboard',
  props: {
    selection: Array,
    refresh: Function,
    clearSelection: Function,
  },
  data() {
    return {
      expanded: false,
      dialog: {
        delete: false,
        album: false,
        edit: false,
      },
      rtl: this.$rtl,
    };
  },
  methods: {
    clearClipboard() {
      this.clearSelection();
      this.expanded = false;
    },
    addToAlbum(ppid) {
      this.dialog.album = false;

      Api.post(`albums/${ppid}/photos`, {"labels": this.selection}).then(() => this.onAdded());
    },
    onAdded() {
      this.clearClipboard();
    },
    batchDelete() {
      this.dialog.delete = false;

      Api.post("batch/labels/delete", {"labels": this.selection}).then(this.onDeleted.bind(this));
    },
    onDeleted() {
      Notify.success(this.$gettext("Labels deleted"));
      this.clearClipboard();
    },
    download() {
      if (this.selection.length !== 1) {
        Notify.error(this.$gettext("You can only download one label"));
        return;
      }

      this.onDownload(`/api/v1/labels/${this.selection[0]}/dl?t=${this.$config.downloadToken()}`);

      this.expanded = false;
    },
    onDownload(path) {
      Notify.success(this.$gettext("Downloading…"));
      const link = document.createElement('a');
      link.href = path;
      link.download = "label.zip";
      link.click();
    },
  }
};
</script>
