<template>
	<radio4000-player
		v-if="canLoad"
		:channel="channel"
		:tracks="tracks"
		:originTrack="track"
		:image="image"
		:autoplay="autoplay"
		:r4Url="r4Url"
		:volume="localVolume"
		:shuffle="shuffle"
		@trackChanged="onTrackChanged"
		@trackEnded="onTrackEnded"
		@mediaNotAvailable="onMediaNotAvailable">
	</radio4000-player>
	<div v-else class="Console">
		<p>Radio4000-player is ready to start playing:
			<a href="https://github.com/internet4000/radio4000-player-vue">documentation</a>
		</p>
	</div>
</template>

<script>
	import debounce from 'lodash.debounce'
	import Radio4000Player from './Radio4000Player.vue'
	import {
		findChannelById,
		findChannelBySlug,
		findChannelTracks,
		findChannelImage,
		findTrack
	} from './utils/store'

	export default {
		name: 'player-data',
		components: {Radio4000Player},
		props: {
			autoplay: Boolean,
			channelSlug: String,
			channelId: String,
			trackId: String,
			r4Url: {
				type: Boolean,
				default: false
			},
			volume: {
				type: Number,
				default: 100
			},
			shuffle: Boolean
		},
		data () {
			return {
				channel: {},
				image: '',
				tracks: [],
				track: {}
			}
		},
		created() {
			this.$root.$on('setVolume', debounce(vol => {
				this.localVolume = vol
			}, 100))

			// Decide which method to use to load data.
			const { channelSlug, channelId, trackId } = this
			if (trackId) {
				return this.loadChannelByTrack(trackId)
			}
			if (channelSlug) {
				return this.loadChannelBySlug(channelSlug)
			}
			if (channelId) {
				return this.loadChannelById(channelId)
			}
		},
		watch: {
			channelSlug(slug) {
				if (slug) this.loadChannelBySlug(slug)
			},
			channelId(id) {
				if (id) this.loadChannelById(id)
			},
			trackId(id) {
				if (id) this.loadChannelByTrack(id)
			}
		},
		computed: {
			localVolume: {
				get() {
					return this.volume
				},
				set(volume) {
					const el = this.$root.$el.parentNode
					el.setAttribute('volume', volume)
				}
			},
			// When either of these is set, it means we can load and show the player.
			canLoad() {
				return this.channel || this.channelSlug || this.channelId || this.trackId
			}
		},
		methods: {
			// start player session by:
			// all start method must return a `channel@r4` model
			loadChannelBySlug(slug) {
				return findChannelBySlug(slug)
					.then(this.updateChannel)
					.catch(err => {console.log(err)})
			},
			loadChannelById(id) {
				return findChannelById(id)
					.then(this.updateChannel)
					.catch(err => {console.log(err)})
			},
			loadChannelByTrack(id) {
				// avoid loading track twice
				let track = this.tracks && this.tracks.find(t => t.id === id)
				if (track) {
					return this.track = track
				}

				return findTrack(id).then(track => {
					this.track = track

					// avoid loading the same channel twice
					const hasQuery = this.channel.query
					const sameChannel = this.channel.id === track.channel
					if (sameChannel && hasQuery || sameChannel && !hasQuery) {
						return
					}

					// refresh
					return this.loadChannelById(track.channel)
				})
			},
			loadChannelTracks(channel) {
				findChannelTracks(channel.id)
					.then(this.updateTracks)
					.catch(err => {console.log(err)})
			},
			loadChannelImage(channel) {
				findChannelImage(channel)
					.then(this.updateImage)
					.catch(this.updateImage)
			},

			/*
				 As an alternative to letting the player load the right data for you,
				 you can pass a "channel" object.
			 */
			updateChannel(channel) {
				/* Reset tracks and image to show loading UX immediately.*/
				this.tracks = []
				this.image = ''

				if (!channel.image) {
					this.loadChannelImage(channel)
				} else {
					this.updateImage(channel.image)
				}

				if (!channel.tracks.length) {
					this.loadChannelTracks(channel)
				} else {
					this.updateTracks(channel.tracks)
				}

				this.channel = channel
			},
			updateTracks(tracks) {
				return this.tracks = tracks
			},
			updateImage(image) {
				return this.image = image ? image : ''
			},

			/*
				 Public API: events sent to the outside world
			 */

			/* methods*/
			updatePlaylist(playlist) {
				this.updateChannel(playlist)
			},

			/* events */
			onTrackChanged(...args) {
				this.$emit('trackChanged', ...args)
			},
			onTrackEnded(...args) {
				this.$emit('trackEnded', ...args)
			},
			onMediaNotAvailable(...args) {
				this.$emit('mediaNotAvailable', ...args)
			}
		}
	}
</script>

<style scoped>
	.Console {
		font-family: monospace;
		padding: 0 1em;
		line-height: 1.4;
	}
</style>
