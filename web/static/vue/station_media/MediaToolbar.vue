<template>
    <div class="row pt-4" id="app-toolbar">
        <div class="col-md-8">
            <div class="btn-group dropdown allow-focus">
                <b-dropdown size="sm" variant="primary" ref="setPlaylistsDropdown" v-b-tooltip.hover
                            :title="langPlaylistDropdown">
                    <template v-slot:button-content>
                        <i class="material-icons" aria-hidden="true">clear_all</i>
                        <translate>Playlists</translate>
                        <span class="caret"></span>
                    </template>
                    <b-dropdown-form class="pt-2" @submit.prevent="setPlaylists">
                        <div v-for="playlist in playlists" class="form-group">
                            <div class="custom-control custom-checkbox">
                                <input type="checkbox" class="custom-control-input"
                                       v-bind:id="'chk_playlist_' + playlist.id" name="playlists[]"
                                       v-model="checkedPlaylists" v-bind:value="playlist.id">
                                <label class="custom-control-label" v-bind:for="'chk_playlist_'+playlist.id">
                                    {{ playlist.name }}
                                </label>
                            </div>
                        </div>
                        <div class="form-group">
                            <div class="custom-control custom-checkbox">
                                <input type="checkbox" class="custom-control-input" id="chk_playlist_new"
                                       v-model="checkedPlaylists" value="new">
                                <label class="custom-control-label" for="chk_playlist_new">
                                    <input type="text" class="form-control p-2" id="new_playlist_name"
                                           name="new_playlist_name" v-model="newPlaylist" style="min-width: 150px;"
                                           :placeholder="langNewPlaylist">
                                </label>
                            </div>
                        </div>

                        <b-button type="submit" size="sm" variant="primary" v-translate>
                            Save
                        </b-button>
                        <b-button type="button" size="sm" variant="warning" @click="clearPlaylists()" v-translate>
                            Clear
                        </b-button>
                    </b-dropdown-form>
                </b-dropdown>
            </div>
            <b-button size="sm" variant="primary" @click="doQueue" v-b-tooltip.hover :title="langQueue">
                <i class="material-icons" aria-hidden="true">queue_play_next</i>
                <translate>Queue</translate>
            </b-button>
            <b-button size="sm" variant="primary" v-b-modal.move_file>
                <i class="material-icons" aria-hidden="true">open_with</i>
                <translate>Move</translate>
            </b-button>
            <b-button size="sm" variant="danger" @click="doDelete">
                <i class="material-icons" aria-hidden="true">delete</i>
                <translate>Delete</translate>
            </b-button>
        </div>
        <div class="col-md-4 text-right">
            <b-button size="sm" variant="primary" v-b-modal.create_directory>
                <i class="material-icons" aria-hidden="true">folder</i>
                <translate>New Folder</translate>
            </b-button>
        </div>
    </div>
</template>
<script>
  import axios from 'axios'

  export default {
    name: 'station-media-toolbar',
    props: {
      currentDirectory: String,
      selectedFiles: Array,
      initialPlaylists: Array,
      batchUrl: String
    },
    data () {
      return {
        playlists: this.initialPlaylists,
        checkedPlaylists: [],
        newPlaylist: ''
      }
    },
    watch: {
      newPlaylist (text) {
        if (text !== '') {
          if (!this.checkedPlaylists.includes('new')) {
            this.checkedPlaylists.push('new')
          }
        }
      }
    },
    computed: {
      langPlaylistDropdown () {
        return this.$gettext('Set or clear playlists from the selected media')
      },
      langNewPlaylist () {
        return this.$gettext('New Playlist')
      },
      langQueue () {
        return this.$gettext('Queue the selected media to play next')
      },
      langErrors () {
        return this.$gettext('The request could not be processed.')
      },
      newPlaylistIsChecked () {
        return this.newPlaylist !== ''
      }
    },
    methods: {
      doQueue (e) {
        this.doBatch('queue', this.$gettext('Files queued for playback:'))
      },
      doDelete (e) {
        let buttonText = this.$gettext('Delete')
        let buttonConfirmText = this.$gettext('Delete %{ num } media file(s)?')

        swal({
          title: this.$gettextInterpolate(buttonConfirmText, { num: this.selectedFiles.length }),
          buttons: [true, buttonText],
          dangerMode: true
        }).then((value) => {
          if (value) {
            this.doBatch('delete', this.$gettext('Files removed:'))
          }
        })
      },
      doBatch (action, notifyMessage) {
        if (this.selectedFiles.length) {
          this.notifyPending()

          axios.put(this.batchUrl, {
            'do': action,
            'files': this.selectedFiles,
            'file': this.currentDirectory
          }).then((resp) => {
            if (resp.data.success) {
              notify('<b>' + notifyMessage + '</b><br>' + this.selectedFiles.join('<br>'), 'success', false)
            } else {
              notify('<b>' + this.langErrors + '</b><br>' + resp.data.errors.join('<br>'), 'danger')
            }

            this.$emit('relist')
          }).catch((err) => {
            this.handleError(err)
          })
        } else {
          this.notifyNoFiles()
        }
      },
      clearPlaylists () {
        this.checkedPlaylists = []
        this.newPlaylist = ''

        this.setPlaylists()
      },
      setPlaylists () {
        this.$refs.setPlaylistsDropdown.hide()

        if (this.selectedFiles.length) {
          this.notifyPending()

          axios.put(this.batchUrl, {
            'do': 'playlist',
            'playlists': this.checkedPlaylists,
            'new_playlist_name': this.newPlaylist,
            'files': this.selectedFiles,
            'file': this.currentDirectory
          }).then((resp) => {
            if (resp.data.success) {
              if (resp.data.record) {
                this.playlists.push(resp.data.record)
              }

              let notifyMessage = (this.checkedPlaylists.length > 0)
                ? this.$gettext('Playlists updated for selected files:')
                : this.$gettext('Playlists cleared for selected files:')
              notify('<b>' + notifyMessage + '</b><br>' + this.selectedFiles.join('<br>'), 'success')

              this.checkedPlaylists = []
              this.newPlaylist = ''
            } else {
              notify(resp.data.errors.join('<br>'), 'danger')
            }

            this.$emit('relist')
          }).catch((err) => {
            this.handleError(err)
          })
        } else {
          this.notifyNoFiles()
        }
      },
      notifyPending () {
        notify('<b>' + this.$gettext('Applying changes...') + '</b>', 'warning', {
          delay: 3000
        })
      },
      notifyNoFiles () {
        notify('<b>' + this.$gettext('No files selected.') + '</b>', 'danger')
      },
      handleError (err) {
        console.error(err)
        if (err.response.message) {
          notify('<b>' + err.response.message + '</b>', 'danger')
        }
      }
    }
  }
</script>