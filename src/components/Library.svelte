<script lang="ts">
import { getRelativeLocaleUrl } from "astro:i18n";
import ProgressRing from "$components/ProgressRing.svelte";
import { flip } from "svelte/animate";
import { fade } from "svelte/transition";
import { monolocale } from "$config";
import Time from "$utils/time";
import i18nit from "$i18n";

type KnowledgeItem = {
	id: string;
	data: any;
	body: string;
	progress: number;
	excerpt: string;
	latestDate: Date | null;
};

import type { Snippet } from "svelte";

type Icons = {
	"icon-book"?: Snippet;
	"icon-video"?: Snippet;
	"icon-course"?: Snippet;
	"icon-book-large"?: Snippet;
	"icon-video-large"?: Snippet;
	"icon-course-large"?: Snippet;
	"icon-empty"?: Snippet;
};

let {
	locale,
	books,
	allTags,
	theme = "plain",
	...icons
}: {
	locale: string;
	books: KnowledgeItem[];
	allTags: string[];
	theme?: string;
} & Icons = $props();

const t = i18nit(locale);

// State for tag filtering
let selectedTags = $state<string[]>([]);

// State for hover interaction
let hoveredBook = $state<KnowledgeItem | null>(null);

// Filter books by selected tags
const filteredBooks = $derived.by(() => {
	if (selectedTags.length === 0) {
		return books;
	}

	return books.filter(book => {
		if (!book.data.tags || book.data.tags.length === 0) {
			return false;
		}
		// Check if book contains ALL selected tags (AND logic, same as Note/Jotting)
		return selectedTags.every(tag => book.data.tags.includes(tag));
	});
});

// Group filtered books by status
const inProgressBooks = $derived(filteredBooks.filter(book => book.data.status === "inProgress"));
const todoBooks = $derived(filteredBooks.filter(book => book.data.status === "todo"));
const doneBooks = $derived(filteredBooks.filter(book => book.data.status === "done"));

const statusGroups = $derived(
	[
		{ key: "inProgress", label: t("library.status.inProgress"), items: inProgressBooks },
		{ key: "todo", label: t("library.status.todo"), items: todoBooks },
		{ key: "done", label: t("library.status.done"), items: doneBooks }
	].filter(group => group.items.length > 0)
);

// Determine which book to display in the sidebar
// Priority: hovered book > most recently edited book from filtered results
const displayBook = $derived(
	hoveredBook ||
		filteredBooks.filter(book => book.latestDate).sort((a, b) => b.latestDate!.getTime() - a.latestDate!.getTime())[0] ||
		filteredBooks[0]
);

// Toggle tag selection
function toggleTag(tag: string) {
	if (selectedTags.includes(tag)) {
		selectedTags = selectedTags.filter(t => t !== tag);
	} else {
		selectedTags = [...selectedTags, tag];
	}
}

// Set hovered book
function setHoveredBook(book: KnowledgeItem) {
	hoveredBook = book;
}

// Clear hovered book
function clearHoveredBook() {
	hoveredBook = null;
}
</script>

<main class="flex flex-col sm:flex-row gap-8 grow">
	<!-- Main content area with status rows -->
	<article class="flex flex-col gap-10 grow">
		{#each statusGroups as group (group.key)}
			<div class="status-row" transition:fade={{ duration: 150 }} animate:flip={{ duration: 150 }}>
				<h2>{group.label}</h2>
				<div class="books-grid">
					{#each group.items as book (book.id)}
						<a
							href={getRelativeLocaleUrl(locale, `/library/${monolocale ? book.id : book.id.split("/").slice(1).join("/")}`)}
							class="book-card"
							onmouseenter={() => setHoveredBook(book)}
							onmouseleave={() => clearHoveredBook()}
							animate:flip={{ duration: 150 }}
						>
							<ProgressRing status={book.data.status} progress={book.progress} size={100} {theme}>
								<div class="w-full h-full flex items-center justify-center c-weak">
									{#if book.data.type === "book" && icons["icon-book"]}
										{@render icons["icon-book"]()}
									{:else if book.data.type === "video" && icons["icon-video"]}
										{@render icons["icon-video"]()}
									{:else if book.data.type === "course" && icons["icon-course"]}
										{@render icons["icon-course"]()}
									{/if}
								</div>
							</ProgressRing>
							<div class="book-info">
								<div class="book-title">{book.data.title}</div>
								{#if book.data.author}
									<div class="book-author">{book.data.author}</div>
								{/if}
								<div class="book-progress">{book.progress}%</div>
							</div>
						</a>
					{/each}
				</div>
			</div>
		{/each}

		{#if statusGroups.length === 0}
			<div class="flex flex-col items-center justify-center py-20 c-weak">
				{#if icons["icon-empty"]}
					{@render icons["icon-empty"]()}
				{/if}
				<p class="mt-4 text-lg">
					{selectedTags.length > 0 ? t("library.noMatches") : t("library.empty")}
				</p>
			</div>
		{/if}
	</article>

	<!-- Sidebar -->
	<aside class="sm:flex-basis-280px flex flex-col gap-8 sm:border-l sm:border-l-solid sm:pl-8">
		<!-- Tag filter section -->
		{#if allTags.length > 0}
			<section>
				<h3>{t("library.filterByTag")}</h3>
				<div class="tag-list">
					{#each allTags as tag (tag)}
						<button class:selected={selectedTags.includes(tag)} onclick={() => toggleTag(tag)}>
							{tag}
						</button>
					{/each}
				</div>
			</section>
		{/if}

		<!-- Latest activity section -->
		{#if displayBook}
			<section>
				<h3>{hoveredBook ? t("library.thisReadingSession") : t("library.latestActivity")}</h3>
				<div class="activity-card">
					<div class="activity-info">
						<div class="activity-title">{displayBook.data.title}</div>
						{#if displayBook.latestDate}
							<time datetime={displayBook.latestDate.toISOString()} class="activity-date">{Time.date.locale(displayBook.latestDate, locale)}</time>
						{/if}
						<div class="activity-excerpt">{displayBook.excerpt || t("library.noExcerpt")}</div>
					</div>
					<div class="activity-cover">
						{#if displayBook.data.cover}
							{@const isRemote = typeof displayBook.data.cover === "string"}
							{@const coverSrc = isRemote ? displayBook.data.cover : displayBook.data.cover.src}
							{@const coverWidth = isRemote ? undefined : displayBook.data.cover.width}
							{@const coverHeight = isRemote ? undefined : displayBook.data.cover.height}

							<img src={coverSrc} alt={displayBook.data.title} class="cover-image object-cover" width={coverWidth} height={coverHeight} loading="lazy" decoding="async" />
						{:else}
							<div class="cover-image placeholder-container">
								<div class="placeholder">
									{#if displayBook.data.type === "book" && icons["icon-book-large"]}
										{@render icons["icon-book-large"]()}
									{:else if displayBook.data.type === "video" && icons["icon-video-large"]}
										{@render icons["icon-video-large"]()}
									{:else if displayBook.data.type === "course" && icons["icon-course-large"]}
										{@render icons["icon-course-large"]()}
									{/if}
								</div>
							</div>
						{/if}
					</div>
				</div>
			</section>
		{/if}
	</aside>
</main>

<style lang="less">
	main {
		border-color: var(--border-color);
	}

	// Main content area with status rows
	article {
		.status-row {
			display: flex;
			flex-direction: column;
			gap: 1rem;

			h2 {
				font-size: 1.25rem;
				font-weight: 600;
				color: var(--primary-color);
				margin-bottom: 0.5rem;
			}

			.books-grid {
				display: flex;
				flex-wrap: wrap;
				gap: 1.5rem;

				.book-card {
					display: flex;
					flex-direction: column;
					align-items: center;
					gap: 0.75rem;
					padding: 1rem;
					border-radius: 0.5rem;
					transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
					cursor: pointer;
					text-decoration: none;

					&:hover {
						background-color: var(--block-color);
						transform: translateY(-2px);
					}

					.book-info {
						display: flex;
						flex-direction: column;
						align-items: center;
						text-align: center;
						gap: 0.25rem;
						width: 100%;

						.book-title {
							font-weight: 600;
							color: var(--primary-color);
							font-size: 1rem;
							line-height: 1.4;
						}

						.book-author {
							color: var(--remark-color);
							font-size: 0.875rem;
						}

						.book-progress {
							color: var(--weak-color);
							font-size: 0.75rem;
						}
					}
				}
			}
		}
	}

	// Sidebar styling
	aside {
		border-color: var(--border-color);

		section {
			display: flex;
			flex-direction: column;
			gap: 5px;

			// Tag filter buttons
			.tag-list {
				display: flex;
				flex-direction: row;
				flex-wrap: wrap;
				gap: 0.5rem;

				button {
					border-bottom: 1px solid var(--primary-color);
					padding: 0.125rem 0.5rem;
					font-size: 0.9rem;
					transition:
						color 0.15s ease,
						background-color 0.15s ease;

					&.selected {
						color: var(--background-color);
						background-color: var(--primary-color);
					}

					@media (min-width: 640px) {
						&:hover {
							color: var(--background-color);
							background-color: var(--primary-color);
						}
					}
				}
			}

			// Latest activity card
			.activity-card {
				display: flex;
				flex-direction: row;
				gap: 0.75rem;
				padding: 1rem;
				border: 1px solid var(--border-color);
				border-radius: 0.5rem;
				background-color: var(--block-color);
				transition: opacity 0.3s ease;
				align-items: flex-start;

				@media (min-width: 640px) {
					flex-direction: column;
					align-items: stretch;
				}

				.activity-cover {
					width: 40%;
					flex-shrink: 0;
					aspect-ratio: 3 / 4;
					border-radius: 0.375rem;
					overflow: hidden;
					order: 1;

					@media (min-width: 640px) {
						width: 100%;
					}

					.cover-image {
						width: 100%;
						height: 100%;
						display: block;

						&.object-cover {
							object-fit: cover;
						}

						&.placeholder-container {
							background-color: var(--background-color);
						}

						.placeholder {
							width: 100%;
							height: 100%;
							display: flex;
							align-items: center;
							justify-content: center;
							color: var(--weak-color);
						}
					}
				}

				.activity-info {
					display: flex;
					flex-direction: column;
					flex-grow: 1;
					gap: 0.5rem;
					order: 0;

					.activity-title {
						font-weight: 600;
						color: var(--primary-color);
						font-size: 1rem;
						line-height: 1.4;
					}

					.activity-date {
						color: var(--remark-color);
						font-size: 0.75rem;
					}

					.activity-excerpt {
						color: var(--remark-color);
						font-size: 0.875rem;
						line-height: 1.6;
						overflow: hidden;
						max-width: 100%;
						word-wrap: break-word;
					}
				}
			}
		}
	}
</style>
