<template>
	<transition name="zoom">
		<div
			class="card-popup"
			:class="{ left: position === 'left', right: position === 'right' }"
			v-if="card"
			v-show="display"
		>
			<div class="carousel">
				<div
					class="carousel-item"
					:class="{
						'carousel-selected': currentPart === 0,
						before: currentPart === 1,
						'before-hidden': currentPart > 1,
					}"
					:key="card.id"
				>
					<CardImage
						:language="language"
						:card="card"
						:fixedLayout="true"
						:displayCardText="displayCardText"
					/>
				</div>
				<div
					class="carousel-item related-card"
					v-for="(relatedCard, idx) in relatedCards"
					:key="relatedCard.id"
					:class="{
						'carousel-selected': currentPart === idx + 1,
						before: currentPart === idx + 2,
						after: currentPart === idx,
						'before-hidden': currentPart > idx + 2,
						'after-hidden': currentPart < idx,
					}"
				>
					<img
						:src="
							relatedCard.image_uris
								? relatedCard.image_uris.border_crop
								: relatedCard.card_faces &&
								  relatedCard.card_faces[0] &&
								  relatedCard.card_faces[0].image_uris
								? relatedCard.card_faces[0].image_uris.border_crop
								: null
						"
						class="related-card-image"
					/>
					<card-text
						v-if="relatedCard.status === 'ready' && displayCardText"
						:card="relatedCard"
						class="alt-card-text"
					/>
				</div>
				<div class="all-parts">
					<div class="mouse-hint" v-if="relatedCards.length > 0">
						<i class="fas fa-arrows-alt-v"></i> <i class="fas fa-mouse"></i>
					</div>
					<i class="fas fa-angle-up" v-if="relatedCards.length > 0"></i>
					<i class="fas fa-square fa-sm" :class="{ 'carousel-selected': currentPart === 0 }"></i>
					<template v-for="(part, idx) in relatedCards">
						<template v-if="part.status === 'ready'"
							><i
								class="fas fa-square fa-sm"
								:key="part.id"
								:class="{ 'carousel-selected': currentPart === idx + 1 }"
							></i
						></template>
						<template v-else
							><i
								class="fas fa-spinner fa-spin"
								:class="{ 'carousel-selected': currentPart === idx + 1 }"
								:key="part.id"
							></i
						></template>
					</template>
					<i class="fas fa-angle-down" v-if="relatedCards.length > 0"></i>
					<div class="alt-hint" :style="displayCardText ? 'color: white' : ''">⎇ Alt</div>
				</div>
				<div v-if="hasPendingData(card.id)" class="all-parts"><i class="fas fa-spinner fa-spin" /></div>
			</div>
		</div>
	</transition>
</template>

<script>
import axios from "axios";

import CardText from "./CardText.vue";
import CardImage from "./CardImage.vue";

const scrollCooldown = 100; // ms

export default {
	components: { CardImage, CardText },
	props: {
		language: { type: String, required: true },
	},
	data() {
		return {
			display: false,
			card: null,
			position: "left",
			currentPart: 0,
			lastScroll: 0,
			displayCardText: false,
			spellbooks: {}, // Associates card names to their spellbooks (Sets of card ids)
		};
	},
	created() {
		this.$root.$on("togglecardpopup", (event, card) => {
			if (!this.display) {
				this.position = event.clientX < window.innerWidth / 2 ? "right" : "left";
				this.card = card;
				this.currentPart = 0;
				let promise = this.$cardCache.request(card.id);
				// Also request associated spellbook if necessary
				promise?.then(() => {
					const cardData = this.$cardCache.get(card.id);
					if (!(card.name in this.spellbooks)) {
						const url = `https://api.scryfall.com/cards/search?q=spellbook%3A%22${encodeURI(
							cardData.name
						)}%22&unique=cards`;
						axios
							.get(url)
							.then((response) => {
								if (response.status === 200 && response.data?.data?.length > 0) {
									this.$set(this.spellbooks, cardData.name, new Set());
									for (const card of response.data.data) {
										card.status = "ready";
										this.$set(this.$cardCache.cardCache, card.id, card);
										this.spellbooks[cardData.name].add(card.id);
									}
								}
							})
							.catch((error) => {
								// There's no spellbook for this card, add an empty set so we don't request it again and return without error
								if (error.response?.status === 404) {
									this.spellbooks[cardData.name] = new Set();
								} else console.error("Error fetching spellbook:", error);
							});
					}
				});

				document.addEventListener("wheel", this.mouseWheel, { passive: false });
				document.addEventListener("keydown", this.keyDown, { capture: true });
				document.addEventListener("keyup", this.keyUp, { capture: true });
				this.display = true;
			} else this.close();
		});
		this.$root.$on("closecardpopup", () => {
			this.close();
		});
	},
	methods: {
		close() {
			this.display = false;
			this.displayCardText = false;
			this.cleanupEventHandlers();
		},
		hasPendingData(cardID) {
			return this.$cardCache.get(cardID)?.status === "pending";
		},
		additionalData(cardID) {
			return this.$cardCache.get(cardID);
		},
		nextPart() {
			if (this.relatedCards.length === 0) return;
			this.currentPart = (this.currentPart + 1) % (this.relatedCards.length + 1);
		},
		previousPart() {
			if (this.relatedCards.length === 0) return;
			this.currentPart = (this.currentPart - 1) % (this.relatedCards.length + 1);
			while (this.currentPart < 0) this.currentPart += this.relatedCards.length + 1;
		},
		mouseWheel(event) {
			if (this.relatedCards.length > 0) {
				if (Date.now() - this.lastScroll > scrollCooldown) {
					if (event.deltaY > 0) this.nextPart();
					else this.previousPart();
					this.lastScroll = Date.now();
				}
				event.stopPropagation();
				event.preventDefault();
			}
		},
		keyDown(event) {
			switch (event.key) {
				case "Escape":
					this.close();
					break;
				case "ArrowUp":
					this.previousPart();
					break;
				case "ArrowDown":
					this.nextPart();
					break;
				case "Alt":
					this.displayCardText = event.altKey;
					this.$forceUpdate();
					break;
				default:
					// Ignore this event
					return;
			}
			// We handled it.
			event.stopPropagation();
			event.preventDefault();
		},
		keyUp(event) {
			switch (event.key) {
				case "Alt":
					this.displayCardText = event.altKey;
					this.$forceUpdate();
					break;
				default:
					// Ignore this event
					return;
			}
			// We handled it.
			event.stopPropagation();
			event.preventDefault();
		},
		negMod(x, n) {
			return ((x % n) + n) % n;
		},
		cleanupEventHandlers() {
			document.removeEventListener("wheel", this.mouseWheel);
			document.removeEventListener("keydown", this.keyDown, { capture: true });
			document.removeEventListener("keyup", this.keyUp, { capture: true });
		},
	},
	computed: {
		cardAdditionalData() {
			return this.$cardCache.get(this.card.id);
		},
		relatedCards() {
			let r = [];
			if (this.$cardCache.get(this.card?.id)?.all_parts?.length > 0)
				for (let card of this.$cardCache.get(this.card?.id).all_parts) {
					if (card.id !== this.card.id) r.push(this.$cardCache.get(card.id));
				}
			if (this.spellbooks[this.card?.name]?.size > 0)
				for (const cid of this.spellbooks[this.card.name]) r.push(this.$cardCache.get(cid));
			return r;
		},
	},
};
</script>

<style scoped>
.card-popup {
	--image-height: calc(min(70vh, 90vw * 1.4 * 0.5));

	position: fixed;
	top: 15vh;
	height: var(--image-height);
	z-index: 900;
	pointer-events: none;
	filter: drop-shadow(0 0 0.5vw #000000);
}

.card-popup.right {
	right: 3.5vw;
	direction: rtl;
}

.card-popup.left {
	left: 3.5vw;
}

.zoom-enter-active,
.zoom-leave-active {
	transition: all 0.15s ease;
}

.zoom-enter,
.zoom-leave-to {
	opacity: 0;
}
.zoom-enter.left,
.zoom-leave-to.left {
	left: -5vw;
}

.zoom-enter.right,
.zoom-leave-to.right {
	right: -5vw;
}

.carousel {
	position: relative;
	width: 50vw;
	height: var(--image-height);
}

.carousel > .carousel-item {
	position: absolute;
	z-index: -2000;
	transition: all 0.4s ease-out;
	transform-origin: left center;
	backface-visibility: hidden;
}

.right .carousel > .carousel-item {
	transform-origin: right center;
}

.carousel .carousel-item.carousel-selected {
	z-index: 0;
}

.carousel .carousel-item.before {
	z-index: -1000;
	transform: translateY(-25%) scale(70%);
	opacity: 0.8;
}

.carousel .carousel-item.after {
	z-index: -1000;
	transform: translateY(25%) scale(70%);
	opacity: 0.8;
}

.carousel .carousel-item.before-hidden {
	z-index: -2000;
	transform: translateY(-50%) scale(1%);
	opacity: 0;
}

.carousel .carousel-item.after-hidden {
	z-index: -2000;
	transform: translateY(50%) scale(1%);
	opacity: 0;
}

.card-popup >>> .card-individual-image > img {
	width: auto;
	height: var(--image-height);
	aspect-ratio: 100/140;
	background-image: url("../assets/img/cardback.png");
	background-size: cover;
}

.related-card {
	width: calc(0.71 * var(--image-height));
	height: var(--image-height);
	background: url("../assets/img/cardback.png");
	background-size: 100%;
	border-radius: 3%;
}

.related-card .related-card-image {
	width: auto;
	height: var(--image-height);
	border-radius: 3%;
	background-image: url("../assets/img/cardback.png");
	background-size: cover;
}

.all-parts {
	position: absolute;
	top: 50%;
	transform: translateY(-50%);

	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	gap: 0.5vh;

	min-height: 50%;
	max-height: 95%;
	width: 2em;
	margin: 0 auto 0 auto;
	padding: 2.5em 0 2.5em 0;
	background: #222;
	border: 2px solid #aaa;

	color: #aaa;
}

.right .all-parts {
	right: -2em;

	border-left: 0;
	border-radius: 0 1em 1em 0;
}

.left .all-parts {
	left: -2em;
	text-align: left;

	border-right: 0;
	border-radius: 1em 0 0 1em;
}

.all-parts > * {
	transition: all 0.1s;
}

.all-parts .carousel-selected {
	text-shadow: 0 0 6px white;
	color: white;
}

.all-parts .fa-angle-up,
.all-parts .fa-angle-down {
	color: #666;
}

.mouse-hint {
	position: absolute;
	color: #666;
	text-shadow: 0px -1px 0px rgba(0, 0, 0, 0.7);
	top: 0.8em;
	left: 50%;
	transform: translateX(-50%);
	display: flex;
	gap: 0.25em;
}

.alt-hint {
	position: absolute;
	color: #666;
	text-shadow: 0px -1px 0px rgba(0, 0, 0, 0.7);
	bottom: 0.8em;
	left: 50%;
	transform: translateX(-50%);
	display: flex;
	gap: 0.25em;
}

.alt-card-text {
	position: absolute;
	top: 0;
	bottom: 0;
	left: 0;
	right: 0;
	z-index: 10;
}
</style>
