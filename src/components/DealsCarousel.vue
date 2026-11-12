<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import { useDeals } from '../composables/useDeals';
import { hasDiscount, mainPrice } from '../utils/deals';
import type { Deal } from '../utils/deals';
import DealImage from './DealImage.vue';

/**
 * Home block: the latest daily drop as a swipeable row of cards.
 * Every text, price, picture and link is taken from the Google Sheet.
 */
const emit = defineEmits<{
  (e: 'showAll'): void;
}>();

const { featuredDeals, groups, newTodayCount, title, endLabel } = useDeals();

const trackRef = ref<HTMLElement | null>(null);
const canScrollPrev = ref(false);
const canScrollNext = ref(false);

const headerNote = computed(() => {
  const parts: string[] = [];
  if (newTodayCount.value > 0) parts.push('New today');
  if (endLabel.value) parts.push(`${parts.length > 0 ? 'ends' : 'Ends'} ${endLabel.value}`);
  return parts.join(' · ');
});

// The picture area is drawn only when at least one of these deals has an image in the sheet
const hasImages = computed(() => featuredDeals.value.some((deal) => !!deal.image));

const totalDeals = computed(() => groups.value.reduce((sum, group) => sum + group.deals.length, 0));
const showAllLabel = computed(() =>
  totalDeals.value > featuredDeals.value.length ? 'Show more deals' : 'View all deals',
);

function priceLabel(deal: Deal): string {
  const price = mainPrice(deal);
  if (!price) return '';
  return hasDiscount(deal) ? `Sale price ${price}, regular price ${deal.regularPrice}` : `Price ${price}`;
}

function updateArrows() {
  const track = trackRef.value;
  if (!track) {
    canScrollPrev.value = false;
    canScrollNext.value = false;
    return;
  }
  canScrollPrev.value = track.scrollLeft > 4;
  canScrollNext.value = track.scrollLeft + track.clientWidth < track.scrollWidth - 4;
}

function scrollCards(direction: 1 | -1) {
  const track = trackRef.value;
  if (!track) return;
  const card = track.querySelector<HTMLElement>('.tt-deal-card');
  const gap = parseFloat(getComputedStyle(track).columnGap || '0') || 0;
  const step = card ? card.offsetWidth + gap : track.clientWidth * 0.85;
  track.scrollBy({ left: direction * step, behavior: 'smooth' });
}

watch(featuredDeals, () => {
  nextTick(() => {
    if (trackRef.value) trackRef.value.scrollLeft = 0;
    updateArrows();
  });
});

onMounted(() => {
  nextTick(updateArrows);
  window.addEventListener('resize', updateArrows);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', updateArrows);
});
</script>

<template>
  <section v-if="featuredDeals.length > 0" class="tt-deals-home" :aria-label="title">
    <div class="tt-deals-home-head">
      <span class="tt-deals-home-title">
        <span class="tt-deals-home-dot" aria-hidden="true"></span>
        {{ title }}
      </span>
      <span v-if="headerNote" class="tt-deals-home-note">{{ headerNote }}</span>
    </div>

    <div class="tt-deals-home-carousel" :class="{ 'no-images': !hasImages }">
      <div
        ref="trackRef"
        class="tt-deals-home-track"
        :class="{ 'is-single': featuredDeals.length === 1 }"
        @scroll.passive="updateArrows"
      >
        <article v-for="deal in featuredDeals" :key="deal.id" class="tt-deal-card">
          <DealImage v-if="hasImages" class="tt-deal-card-image" :src="deal.image" :alt="deal.name" :width="480" />

          <div class="tt-deal-card-body">
            <div class="tt-deal-card-title">{{ deal.name }}</div>
            <div v-if="deal.description" class="tt-deal-card-desc">{{ deal.description }}</div>

            <div v-if="mainPrice(deal)" class="tt-deal-card-prices" role="group" :aria-label="priceLabel(deal)">
              <span class="tt-deal-price" aria-hidden="true">{{ mainPrice(deal) }}</span>
              <span v-if="hasDiscount(deal)" class="tt-deal-old-price" aria-hidden="true">{{ deal.regularPrice }}</span>
            </div>
          </div>

          <a
            v-if="deal.link"
            class="tt-deal-card-cta"
            :href="deal.link"
            target="_blank"
            rel="noopener noreferrer"
          >
            <span>View Deal</span>
            <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
              <polyline points="15 3 21 3 21 9" />
              <line x1="10" y1="14" x2="21" y2="3" />
            </svg>
          </a>
        </article>
      </div>

      <button
        v-if="canScrollPrev"
        type="button"
        class="tt-deals-home-arrow is-prev"
        aria-label="Previous deal"
        @click="scrollCards(-1)"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <polyline points="15 18 9 12 15 6" />
        </svg>
      </button>

      <button
        v-if="canScrollNext"
        type="button"
        class="tt-deals-home-arrow is-next"
        aria-label="Next deal"
        @click="scrollCards(1)"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <polyline points="9 18 15 12 9 6" />
        </svg>
      </button>
    </div>

    <button type="button" class="tt-deals-home-more" @click="emit('showAll')">
      <span>{{ showAllLabel }}</span>
      <svg xmlns="http://www.w3.org/2000/svg" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
        <polyline points="6 9 12 15 18 9" />
      </svg>
    </button>
  </section>
</template>

<style lang="scss">
$deal-accent: #7b1f25;
$deal-accent-dark: #641920;

.tt-deals-home {
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 100%;
  min-width: 0;
  margin: 0;
  padding: 0;
  text-align: left;

  .tt-deals-home-head {
    display: flex;
    align-items: baseline;
    justify-content: space-between;
    gap: 10px;
    padding: 0 2px;
  }

  .tt-deals-home-title {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    min-width: 0;
    font-size: 11.5px;
    font-weight: 800;
    line-height: 1.3;
    letter-spacing: 0.7px;
    text-transform: uppercase;
    color: $deal-accent;
  }

  .tt-deals-home-dot {
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: $deal-accent;
    box-shadow: 0 0 0 0 rgba(123, 31, 37, 0.55);
    flex-shrink: 0;
    animation: ttDealsHomePulse 2s cubic-bezier(0.4, 0, 0.2, 1) infinite;
  }

  .tt-deals-home-note {
    flex-shrink: 0;
    font-size: 11.5px;
    font-weight: 500;
    line-height: 1.3;
    color: #64748b;
    white-space: nowrap;
  }

  .tt-deals-home-carousel {
    position: relative;
    min-width: 0;
  }

  .tt-deals-home-track {
    display: flex;
    align-items: stretch;
    gap: 10px;
    margin: 0 -16px;
    padding: 2px 16px 6px;
    overflow-x: auto;
    overflow-y: hidden;
    scroll-snap-type: x mandatory;
    scroll-padding: 0 16px;
    overscroll-behavior-x: contain;
    -webkit-overflow-scrolling: touch;
    scrollbar-width: none;

    &::-webkit-scrollbar {
      display: none;
    }
  }

  .tt-deal-card {
    display: flex;
    flex-direction: column;
    flex: 0 0 86%;
    min-width: 0;
    margin: 0;
    padding: 12px;
    background: #ffffff;
    border: 1px solid #e8ecf1;
    border-radius: 16px;
    box-shadow: 0 2px 8px rgba(15, 23, 42, 0.03);
    box-sizing: border-box;
    scroll-snap-align: start;
  }

  .tt-deals-home-track.is-single .tt-deal-card {
    flex-basis: 100%;
  }

  .tt-deal-card-image {
    width: 100%;
    height: 132px;
    margin-bottom: 10px;
    border-radius: 12px;
  }

  /* grows so that the button sits at the bottom of every card */
  .tt-deal-card-body {
    display: flex;
    flex-direction: column;
    flex: 1 1 auto;
    min-width: 0;
    padding-bottom: 10px;
  }

  .tt-deal-card-title {
    margin: 0;
    font-size: 14.5px;
    font-weight: 700;
    line-height: 1.3;
    letter-spacing: -0.15px;
    color: #0f172a;
    overflow-wrap: anywhere;
  }

  .tt-deal-card-desc {
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 3;
    line-clamp: 3;
    overflow: hidden;
    margin: 3px 0 0;
    font-size: 12.5px;
    font-weight: 400;
    line-height: 1.4;
    color: #475569;
    overflow-wrap: anywhere;
  }

  .tt-deal-card-prices {
    display: flex;
    align-items: baseline;
    flex-wrap: wrap;
    gap: 2px 8px;
    margin: 8px 0 0;
  }

  .tt-deal-price {
    font-size: 17px;
    font-weight: 800;
    line-height: 1.25;
    letter-spacing: -0.2px;
    color: $deal-accent;
    white-space: nowrap;
  }

  .tt-deal-old-price {
    font-size: 12.5px;
    font-weight: 500;
    line-height: 1.25;
    color: #94a3b8;
    text-decoration: line-through;
    white-space: nowrap;
  }

  .tt-deal-card-cta {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    width: 100%;
    margin: 0;
    padding: 10px 14px;
    background: $deal-accent;
    border: none;
    border-radius: 12px;
    box-sizing: border-box;
    color: #ffffff;
    font-family: inherit;
    font-size: 13.5px;
    font-weight: 700;
    line-height: 1.3;
    text-decoration: none;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover {
      background: $deal-accent-dark;
      color: #ffffff;
      text-decoration: none;
    }

    &:active {
      transform: scale(0.985);
    }

    &:focus-visible {
      outline: 2px solid $deal-accent;
      outline-offset: 2px;
    }

    svg {
      flex-shrink: 0;
    }
  }

  .tt-deals-home-arrow {
    position: absolute;
    /* vertical centre of the picture area */
    top: 80px;
    z-index: 2;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 34px;
    height: 34px;
    margin: 0;
    padding: 0;
    transform: translateY(-50%);
    background: $deal-accent;
    border: 2px solid #ffffff;
    border-radius: 50%;
    box-shadow: 0 4px 12px rgba(15, 23, 42, 0.22);
    color: #ffffff;
    cursor: pointer;
    transition: background 0.15s ease;

    &.is-prev {
      left: -8px;
    }

    &.is-next {
      right: -8px;
    }

    &:hover {
      background: $deal-accent-dark;
    }

    &:focus-visible {
      outline: 2px solid $deal-accent;
      outline-offset: 2px;
    }
  }

  /* no pictures in this drop: the arrows sit next to the title and description */
  .tt-deals-home-carousel.no-images .tt-deals-home-arrow {
    top: 46px;
  }

  .tt-deals-home-more {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 7px;
    width: 100%;
    margin: 0;
    padding: 9px 14px;
    background: #ffffff;
    border: 1px solid #e3c9cb;
    border-radius: 12px;
    box-sizing: border-box;
    color: $deal-accent;
    font-family: inherit;
    font-size: 13px;
    font-weight: 700;
    line-height: 1.3;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease;

    &:hover {
      background: #fbf4f4;
      border-color: #d1a9ac;
    }

    &:focus-visible {
      outline: 2px solid $deal-accent;
      outline-offset: 2px;
    }

    svg {
      flex-shrink: 0;
    }
  }
}

@keyframes ttDealsHomePulse {
  0% {
    box-shadow: 0 0 0 0 rgba(123, 31, 37, 0.55);
  }
  70% {
    box-shadow: 0 0 0 6px rgba(123, 31, 37, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(123, 31, 37, 0);
  }
}

@media (prefers-reduced-motion: reduce) {
  .tt-deals-home .tt-deals-home-dot {
    animation: none;
  }
}
</style>
