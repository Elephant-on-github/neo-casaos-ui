<template>
	<div class="modal-card">
		<!-- Modal-Card Header Start -->
		<header class="modal-card-head">
			<div class="is-flex-grow-1">
				<h3 class="title is-header">CasaOS</h3>
			</div>
			<b-tooltip label="Download logs" position="is-bottom" size="is-small" type="is-black">
				<b-icon v-show="showTabName === 'logs'" class="close-button" icon="downloads-outline" pack="casa"
					@click.native="downloadSystemLog" />
			</b-tooltip>
			<b-icon class="close-button ml-2" icon="close-outline" pack="casa" @click.native="$emit('close');" />

		</header>
		<!-- Modal-Card Header End -->
		<!-- Modal-Card Body Start -->
		<section class="modal-card-body " style="overflow:hidden">
			<div class="is-flex-grow-1">
				<b-tabs :animated="false" @input="onInput">
					<b-tab-item :label="$t('Terminal')" value="terminal">
						<terminal-card ref="terminal" :initWsUrl="wsUrl"></terminal-card>
					</b-tab-item>
					<b-tab-item :label="$t('Logs')" value="logs">
						<logs-card ref="logs" :data="logData"></logs-card>
					</b-tab-item>
				</b-tabs>

			</div>

		</section>
		<!-- Modal-Card Body End -->
		<b-loading v-model="isLoading" :is-full-page="false"></b-loading>
	</div>
</template>

<script>
import TerminalCard from './TerminalCard.vue';
import LogsCard from './LogsCard.vue';

export default {
	name: 'terminal-panel',
	components: {
		TerminalCard,
		LogsCard
	},
	props: {
		initialTab: {
			type: String,
			default: 'terminal'
		}
	},
	data() {
		return {
			isLoading: false,
			wsUrl: ``,
			logData: "",
			timer: '',
			showTabName: "terminal"
		}
	},
	mounted() {
		this.showTabName = this.initialTab || 'terminal';
		this.getLogs();
		this.timer = setInterval(() => {
			this.getLogs();
		}, 1000 * 5);
	},
	methods: {
		getLogs() {
			this.$api.sys.getLogs().then(res => {
				this.logData = (res && res.data && res.data.data) ? res.data.data : '';
			}).catch(e => {
				console.error('getLogs', e);
			});
		},
		onInput(e) {
			if (e == "terminal") {
				this.showTabName = "terminal"
				if (this.$refs.terminal) this.$refs.terminal.active(true)
				if (this.$refs.logs) this.$refs.logs.active(false)
				this.$messageBus('terminallogs_terminal')
			} else {
				this.showTabName = "logs"
				if (this.$refs.terminal) this.$refs.terminal.active(false)
				if (this.$refs.logs) this.$refs.logs.active(true)
				this.$messageBus('terminallogs_logs')
			}
		},
		downloadSystemLog() {
			this.$api.sys.getLogs().then(res => {
				const data = (res && res.data && res.data.data) ? res.data.data : '';
				const blob = new Blob([data], { type: 'text/plain' });
				const url = URL.createObjectURL(blob);
				const a = document.createElement('a');
				a.href = url;
				a.download = 'casaos.log';
				document.body.appendChild(a);
				a.click();
				document.body.removeChild(a);
				URL.revokeObjectURL(url);
			}).catch(e => {
				console.error('downloadSystemLog', e);
			});
		},
	},
	destroyed() {
		clearInterval(this.timer);
	}
}
</script>

<style lang="scss" scoped>
.cursor-pointer {
	cursor: pointer;
}
</style>
