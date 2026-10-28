<template>
	<div class="terminal-page">
		<terminal-card class="terminal-card"></terminal-card>
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
	width: 100%;
	height: 100vh;
	background: hsla(208, 20%, 12%, 1);
	display: flex;
	align-items: center;
	justify-content: center;
}

.terminal-card {
	width: 100%;
	height: 100%;
}
</style>
