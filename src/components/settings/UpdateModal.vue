<template>
	<div class="modal-card">
		<!-- Modal-Card Header Start -->
		<header class="modal-card-head">
			<div class="is-flex-grow-1">
				<h3 class="title is-header">{{ $t('Update') }}</h3>
			</div>
			<b-icon class="close-button" icon="close-outline" pack="casa" @click.native="$emit('close');" />
		</header>
		<!-- Modal-Card Header End -->
		<!-- Modal-Card Body Start -->
		<section class="modal-card-body ">
			<div class="node-card fixed-height">
				<div v-if="!isUpdating" class="update-info-container  is-size-14px" v-dompurify-html="markdownToHtml"></div>
				<div v-else class="update-info-container  is-size-14px" v-dompurify-html="updateMarkdownHtml"></div>
			</div>
		</section>
		<!-- Modal-Card Body End -->
		<!-- Modal-Card Footer Start-->
		<footer class="modal-card-foot is-flex is-align-items-center">
			<div class="is-flex-grow-1"></div>
			<div>
				<div v-if="reloadNeeded" class="mr-3 is-size-7 has-text-success">
					{{ $t('UI update applied. Reload to use the new version.') }}
				</div>
				<b-button v-if="reloadNeeded" rounded size="is-small" type="is-primary" @click="location.reload()">
					{{ $t('Reload now') }}
				</b-button>
				<b-button v-else :label="$t('Upgrade Now')" :loading="isUpdating" expaned rounded type="is-primary"
						  @click="updateSystem"/>
			</div>
		</footer>
		<!-- Modal-Card Footer End -->
	</div>
</template>

<script>
import {marked} from 'marked'

export default {
	props: {
		changeLog: {
			type: String,
			default: ""
		},
	},
	data() {
		return {
			timer: 0,
			updateTimer: 0,
			updateWatchdog: 0,
			isUpdating: false,
			markdown: ``,
			updateLogs: ``,
			updatedComponents: [],
			reloadNeeded: false,
		};
	},
	computed: {
		markdownToHtml() {
			return marked.parse(this.changeLog);
		},
		updateMarkdownHtml() {

			return marked.parse(this.updateLogs);
		}
	},
	beforeDestroy() {
		// Closing the modal mid-update must not leave the poll and the
		// watchdog running against an unmounted component.
		clearInterval(this.updateTimer);
		clearTimeout(this.updateWatchdog);
	},
	methods: {
		/**
		 * @description: Update System Version and check update state
		 * @return {*} void
		 */
		async updateSystem() {
			this.isUpdating = true;
			try {
				const res = await this.$api.sys.updateCasaOS();
				if (res && res.data && res.data.data && res.data.data.updated_components) {
					this.updatedComponents = res.data.data.updated_components;
				}
			} catch (e) {
				// Backend POST /sys/update itself failed (network, 500). Without
				// this the button stays in loading forever and getUpdateLogs never
				// starts, so the user has no way to retry.
				this.handleUpdateFailure();
				return;
			}
			// this.checkUpdateState();
			this.getUpdateLogs()
		},

		/**
		 * @description: Get update logs
		 * @return {*} void
		 */
		getUpdateLogs() {
			// Watchdog: the update script can die before writing anything
			// (UpdateUrl unreachable, curl 404, disk full). Without this the
			// poll below would spin forever with the button stuck in loading.
			this.updateWatchdog = setTimeout(() => {
				this.handleUpdateFailure();
			}, 15 * 60 * 1000);

			this.updateTimer = setInterval(() => {
				this.$api.file.getContent(`/var/log/casaos/upgrade.log`).then(res => {

					this.updateLogs = res.data.data;
					if (this.updateLogs.includes(`CasaOS upgrade successfully`)) {
						localStorage.setItem('is_update', 'true')
						clearInterval(this.updateTimer);
						clearTimeout(this.updateWatchdog);
						const uiOnly = this.updatedComponents.length > 0
							&& this.updatedComponents.includes('ui')
							&& !this.updatedComponents.includes('backend')
							&& !this.updatedComponents.includes('gateway');
						if (uiOnly) {
							this.reloadNeeded = true;
							this.isUpdating = false;
						} else {
							setTimeout(() => {
								this.$router.replace({
									path: '/logout'
								})
							}, 1000);
						}
					} else if (this.updateLogs.includes(`CasaOS upgrade failed`) || this.updateLogs.includes(`FAILED`)) {
						// The neo update script (build/update.sh) never writes the
						// canonical failure line - it prints "- FAILED <reason>" and
						// exits 1. Match that too, or the modal never recovers.
						this.handleUpdateFailure();
					}
				}).catch(() => {
					// Transient errors while the service restarts mid-update: keep polling.
				})
			}, 200);
		},

		/**
		 * @description: Stop polling, surface the error, re-enable retry
		 * @return {*} void
		 */
		handleUpdateFailure() {
			// Guard against a double call: the 15-min watchdog and the poll could
			// both resolve in the same tick, or a retry could race the watchdog.
			if (!this.isUpdating) return;
			clearInterval(this.updateTimer);
			clearTimeout(this.updateWatchdog);
			this.$buefy.toast.open({
				message: this.$t(`There seems to be a problem with the upgrade process, please try again!`),
				type: 'is-danger'
			})
			setTimeout(() => {
				this.isUpdating = false;
			}, 1000);
		},
		/**
		 * @description: check update state if is_need is false then reload page
		 * @return {*} void
		 */
		checkUpdateState() {
			this.timer = setInterval(() => {
				this.$api.sys.getVersion().then(res => {
					if (res.data.success == 200) {
						if (!res.data.data.is_need) {
							clearInterval(this.timer);
							location.reload();
						}
					}
				})
			}, 3000)
		},
	},
}
</script>

<style lang="scss">
.fixed-height {
	max-height: 20rem;
	overflow-y: auto;
}

.update-info-container {
	line-height: 1.5rem;
	border-radius: 4px;
	overflow: hidden;
	min-height: 20rem;

	h1,
	h2,
	h3,
	h4,
	h5,
	h6 {
		font-weight: bold;
		margin-bottom: 0.4rem;
	}

	h1 {
		font-size: 2em;
	}

	h2 {
		font-size: 1.5em;
	}

	h3 {
		font-size: 1.17em;
	}

	h4 {
		font-size: 1em;
	}

	h5 {
		font-size: 0.83em;
	}

	h6 {
		font-size: 0.67em;
	}

	ul {
		margin-bottom: 0.5em;

		li {
			list-style: disc;
			margin-left: 1rem;
		}
	}
}
</style>
