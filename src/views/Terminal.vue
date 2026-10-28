<template>
	<div class="terminal-page">
		<terminal-card class="terminal-card" standalone />
	</div>
</template>

<script>
import TerminalCard from '@/components/logsAndTerminal/TerminalCard.vue'

export default {
	name: 'Terminal',
	components: {
		TerminalCard
	}
}
</script>

<style lang="scss" scoped>
.terminal-page {
	position: relative;
	width: 100%;
	height: 100vh;
	background: hsla(208, 20%, 12%, 1);
	overflow: hidden;
}

.terminal-card {
	display: block;
	width: 100%;
	height: 100%;
}
</style>
