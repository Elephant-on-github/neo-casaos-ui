<template>
	<div class="modal-card">
		<header class="modal-card-head">
			<div class="is-flex-grow-1">
				<h3 class="title is-header">{{ $t('About') }}</h3>
			</div>
			<b-icon class="close-button" icon="close-outline" pack="casa" @click.native="$emit('close');" />
		</header>
		<section class="modal-card-body">
			<div class="info-list">
				<div class="info-row">
					<span class="info-label">{{ $t('CasaOS (neo)') }}</span>
					<span class="info-value">{{ backendVersion || '—' }}</span>
				</div>
				<div class="info-row">
					<span class="info-label">{{ $t('UI') }}</span>
					<span class="info-value">{{ uiVersion }}</span>
				</div>
				<div class="info-row">
					<span class="info-label">{{ $t('Gateway') }}</span>
					<span class="info-value">{{ gatewayVersion }}</span>
				</div>
				<div v-if="components" class="info-row info-components">
					<span class="info-label">{{ $t('Components') }}</span>
					<div class="info-value">
						<div v-for="(c, name) in components" :key="name" class="info-component-row">
							<span class="info-component-name">{{ name }}</span>
							<span class="info-component-versions">{{ c.installed || '-' }} -&gt; {{ c.latest }}</span>
							<span :class="c.need_update ? 'info-status-update' : 'info-status-ok'">
								{{ c.need_update ? $t('Update available') : $t('Up to date') }}
							</span>
						</div>
					</div>
				</div>
				<div class="info-row">
					<span class="info-label">{{ $t('Architecture') }}</span>
					<span class="info-value">{{ arch || '—' }}</span>
				</div>
			</div>
		</section>
	</div>
</template>

<script>
export default {
	name: 'info-panel',
	data() {
		return {
			backendVersion: '',
			uiVersion: process.env.UI_VERSION || 'dev',
			gatewayVersion: 'v0.4.10',
			arch: '',
			components: null,
		}
	},
	created() {
		this.fetchVersion();
		this.fetchArch();
	},
	methods: {
		async fetchVersion() {
			try {
				const r = await fetch('/v1/sys/version/current');
				if (r.ok) {
					this.backendVersion = (await r.text()).trim();
					return;
				}
			} catch (e) {
				console.error('fetchVersion', e);
			}
			try {
				const res = await this.$api.sys.getVersion();
				if (res && res.data) {
					if (typeof res.data === 'string') {
						this.backendVersion = res.data;
					} else if (res.data.data && res.data.data.current_version) {
						this.backendVersion = res.data.data.current_version;
						this.components = res.data.data.components || null;
					} else if (res.data.current_version) {
						this.backendVersion = res.data.current_version;
						this.components = res.data.components || null;
					}
				}
			} catch (e) {
				console.error('fetchVersion fallback', e);
			}
		},
		fetchArch() {
			this.arch = localStorage.getItem('arch') || '';
			if (!this.arch) {
				this.$api.sys.hardwareInfo().then((res) => {
					if (res && res.data && res.data.success === 200) {
						this.arch = res.data.data.arch || '';
						localStorage.setItem('arch', this.arch);
					}
				}).catch(() => {});
			}
		},
	}
}
</script>

<style lang="scss" scoped>
.info-list {
	padding: 0.5rem 0;
}

.info-row {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 0.625rem 0;
	border-bottom: 1px solid rgba(0, 0, 0, 0.06);

	&:last-child {
		border-bottom: none;
	}
}

.info-label {
	font-weight: 500;
	color: #555;
}

.info-value {
	font-family: 'Monaco', 'Consolas', monospace;
	font-size: 0.875rem;
	color: #222;
}
</style>


