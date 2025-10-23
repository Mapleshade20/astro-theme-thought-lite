<style lang="less">
	.ring-container {
		position: relative;
		display: inline-flex;
		align-items: center;
		justify-content: center;

		svg {
			position: absolute;
			top: 0;
			left: 0;
			width: 100%;
			height: 100%;
			transform: rotate(-90deg);
		}

		.bg-ring {
			fill: none;
			stroke: var(--weak-color);
			opacity: 0.2;
		}

		.progress-ring {
			fill: none;
			stroke: var(--primary-color);
			transition: stroke-dashoffset 0.5s ease;
		}

		// Theme: Default - Status-based animations
		&[data-theme='default'] {
			.progress-ring {
				// Todo: Breathing light effect
				&.status-todo {
					opacity: var(--todo-ring-opacity);
					animation: breathe-todo 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
				}

				// In Progress: Ripple effect with expanding circles
				&.status-in-progress {
					opacity: 0.8;
					stroke: var(--primary-color);
				}

				// Done: Color cycling with bright colors
				&.status-done {
					opacity: 1;
					animation: cycle-colors 20s cubic-bezier(0.45, 0, 0.55, 1) infinite;
				}
			}

			// Ripple circles for in_progress status
			.ripple-circle {
				fill: none;
				stroke: var(--primary-color);
				opacity: 0;
				transform-origin: center;
				animation: ripple-expand 5s cubic-bezier(0.1, 0, 0.3, 1) infinite;

				&:nth-child(2) {
					animation-delay: 1.67s;
				}

				&:nth-child(3) {
					animation-delay: 3.33s;
				}
			}
		}

		.center-content {
			position: relative;
			z-index: 1;
			display: flex;
			align-items: center;
			justify-content: center;
			width: 100%;
			height: 100%;
			border-radius: 50%;
			overflow: hidden;
			transition: filter 0.3s ease;

			// Icon brightness adjustments
			&.status-todo {
				filter: var(--todo-icon-brightness);
				animation: breathe-icon-todo 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
			}

			&.status-done {
				filter: brightness(1.3);
			}
		}
	}

	// Keyframe animations

	// Breathing animation for todo items
	@keyframes breathe-todo {
		0%,
		100% {
			opacity: var(--todo-ring-opacity-breathe-min);
		}
		50% {
			opacity: var(--todo-ring-opacity-breathe-max);
		}
	}

	@keyframes breathe-icon-todo {
		0%,
		100% {
			filter: var(--todo-icon-brightness-min);
		}
		50% {
			filter: var(--todo-icon-brightness-max);
		}
	}

	// Ripple expansion animation for in_progress items
	@keyframes ripple-expand {
		0% {
			opacity: 0.25;
		}
		100% {
			transform: scale(1.3);
			opacity: 0;
		}
	}

	// Color cycling animation for done items
	@keyframes cycle-colors {
		0% {
			stroke: var(--done-color-1);
		}
		25% {
			stroke: var(--done-color-2);
		}
		50% {
			stroke: var(--done-color-3);
		}
		75% {
			stroke: var(--done-color-4);
		}
		100% {
			stroke: var(--done-color-1);
		}
	}
</style>

<div class="ring-container" data-status={status} data-theme={theme} style="width: {size}px; height: {size}px;">
	<svg width={size} height={size} viewBox="0 0 {size} {size}" style="overflow: visible;">
		<!-- Background ring -->
		<circle class="bg-ring" cx={size / 2} cy={size / 2} r={radius} stroke-width={strokeWidth} />

		<!-- Ripple circles (only visible for in_progress status) -->
		{#if status === 'in_progress'}
			<circle
				class="ripple-circle"
				cx={size / 2}
				cy={size / 2}
				r={rippleRadius}
				stroke-width={strokeWidth * 0.3}
			/>
			<circle
				class="ripple-circle"
				cx={size / 2}
				cy={size / 2}
				r={rippleRadius}
				stroke-width={strokeWidth * 0.3}
			/>
			<circle
				class="ripple-circle"
				cx={size / 2}
				cy={size / 2}
				r={rippleRadius}
				stroke-width={strokeWidth * 0.3}
			/>
		{/if}

		<!-- Progress ring -->
		<circle
			class="progress-ring status-{status}"
			cx={size / 2}
			cy={size / 2}
			r={radius}
			stroke-width={strokeWidth}
			stroke-dasharray={circumference}
			stroke-dashoffset={offset}
			stroke-linecap="round"
		/>
	</svg>

	<div class="center-content status-{status}">
		{@render children?.()}
	</div>
</div>

<script lang="ts">
	/**
	 * ProgressRing component
	 * Displays a circular progress indicator with animated states for reading progress
	 *
	 * Visual states (default theme):
	 * - todo: Breathing light effect - ring and icon pulse between very dim and slightly dim
	 * - in_progress: Ripple effect - expanding circles continuously emanate outward
	 * - done: Color cycling - smooth transitions between bright yellow, pink, blue, white
	 */

	import type { Snippet } from 'svelte';

	let {
		status = 'todo',
		progress = 0,
		size = 120,
		strokeWidth = 4,
		theme = 'default',
		children
	}: {
		status?: 'todo' | 'in_progress' | 'done';
		progress?: number;
		size?: number;
		strokeWidth?: number;
		theme?: 'default';
		children?: Snippet;
	} = $props();

	// Calculate SVG circle parameters
	const radius = $derived((size - strokeWidth) / 2);
	const circumference = $derived(2 * Math.PI * radius);

	// Ripple starts at outer edge of the main ring (radius + half stroke width)
	const rippleRadius = $derived(radius + strokeWidth / 2);

	// Calculate stroke-dashoffset based on progress (0-100)
	// Higher progress means less offset (more visible circle)
	const offset = $derived(circumference - (progress / 100) * circumference);
</script>
