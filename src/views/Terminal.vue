<template>
	<div class="terminal-page">
		<terminal-panel @close="goHome" />
	</div>
</template>

<script>
import TerminalPanel from '@/components/logsAndTerminal/TerminalPanel.vue'

export default {
	name: 'Terminal',
	components: {
		TerminalPanel
	},
	methods: {
		goHome() {
			this.$router.push('/')
		}
	}
}
</script>

<style lang="scss" scoped>
.terminal-page {
	width: 100%;
	height: 100vh;
	background: hsla(208, 20%, 12%, 1);
}
</style>
