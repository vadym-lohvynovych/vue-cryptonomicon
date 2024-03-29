<template>
  <div class="container mx-auto flex flex-col items-center bg-gray-100 p-4">
    <app-loader :loading="coinlistLoading" />

    <div class="container">
      <section>
        <add-ticker
          v-model="ticker"
          :addTicker="addTicker"
          :error="error"
          @loaded="coinlistLoading = false"
          @blur="clearError"
        />
        <div class="flex items-center">
          <app-filter v-model="filter" />
          <app-pagination
            :page="page"
            @prevPage="page--"
            @nextPage="page++"
            :isNextPageAvailable="isNextPageAvailable"
          />
        </div>
      </section>

      <template v-if="paginatedTickers.length">
        <hr class="w-full border-t border-gray-600 my-4" />
        <dl class="mt-5 grid grid-cols-1 gap-5 sm:grid-cols-3">
          <app-ticker
            v-for="t in paginatedTickers"
            :key="t.name"
            :price="formatPrice(t.price)"
            :name="t.name"
            :error="t.error"
            :selected="t === this.selected"
            @delete="onDelete(t)"
            @select="onSelect(t)"
          />
        </dl>
      </template>
      <hr class="w-full border-t border-gray-600 my-4" />
      <price-bars
        v-if="selected"
        :title="`${selected.name} - USD`"
        :prices="graph"
        @close="selected = null"
      />
    </div>
  </div>
</template>

<script>
import AddTicker from './AddTicker.vue';
import AppTicker from './AppTicker.vue';
import AppFilter from './AppFilter.vue';
import AppPagination from './AppPagination.vue';
import AppLoader from './AppLoader.vue';
import PriceBars from './PriceBars.vue';
import { nextTick } from '@vue/runtime-core';
import { subscribeToTicker, unsubscribeFromTicker } from './api/api';
import { PRICE_UPDATE, PRICE_ERROR } from './api/constants';
import { createStorageService } from './storageService';
import { urlService } from './urlService';

const ITEMS_PER_PAGE = 6;

const tickersStorage = createStorageService('tickers');

export default {
  name: 'App',
  data() {
    return {
      ticker: 'BTC',
      filter: '',
      page: 1,
      coinlistLoading: true,
      tickers: [],
      error: null,
      selected: null,
      graph: [],
      interval: null,
    };
  },
  components: {
    AddTicker,
    AppTicker,
    AppFilter,
    AppPagination,
    AppLoader,
    PriceBars,
  },
  mounted() {
    this.getSavedTickets();
    this.parseUrl();
  },
  unmounted() {
    this.tickers.forEach((t) => {
      unsubscribeFromTicker(t.name);
    });
  },
  methods: {
    addTicker() {
      if (this.tickerAlreadyAdded) {
        nextTick(() => {
          this.error = 'Такой тикер уже добавлен';
        });
      } else {
        const newTicker = {
          name: this.ticker.toUpperCase(),
          price: '-',
          error: false,
        };
        this.tickers = [...this.tickers, newTicker];
        this.subscribeToTickerUpdates(newTicker.name);
        this.ticker = '';
      }
    },
    getTickerByName(name) {
      const index = this.tickers.findIndex((t) => t.name === name);
      return this.tickers[index];
    },
    updateTickerPrice(name, price) {
      const ticker = this.getTickerByName(name);
      ticker.price = price;
      if (ticker === this.selected) {
        this.graph.push(price);
      }
    },
    setTickerError(name, error) {
      this.getTickerByName(name).error = error;
    },
    clearError() {
      this.error = null;
    },
    onSuggestionClick(ticker) {
      this.ticker = ticker;
      this.addTicker();
    },
    onSelect(t) {
      this.selected = t;
      this.graph = [];
    },
    onDelete(tickerToRemove) {
      this.tickers = this.tickers.filter((t) => t !== tickerToRemove);
      if (tickerToRemove === this.selected) {
        this.selected = null;
      }
      unsubscribeFromTicker(tickerToRemove.name);
    },
    parseUrl() {
      Object.assign(this, urlService.getSearchParams(['filter', 'page']));
    },
    getSavedTickets() {
      const savedTickers = tickersStorage.get();
      if (savedTickers) {
        this.tickers = savedTickers.map((t) => {
          t.price = '-';
          this.subscribeToTickerUpdates(t.name);
          return t;
        });
      }
    },
    subscribeToTickerUpdates(name) {
      subscribeToTicker(name, ({ type, value }) => {
        if (type === PRICE_UPDATE) {
          this.updateTickerPrice(name, value);
        } else if (type === PRICE_ERROR) {
          this.setTickerError(name, true);
        }
      });
    },
    formatPrice(price) {
      if (!price || typeof price !== 'number') {
        return '-';
      }
      return price > 1 ? price.toFixed(3) : price.toPrecision(3);
    },
  },
  computed: {
    startIndex() {
      return (this.page - 1) * ITEMS_PER_PAGE;
    },
    endIndex() {
      return this.page * ITEMS_PER_PAGE;
    },
    tickerAlreadyAdded() {
      return (
        this.ticker.length &&
        this.tickers.findIndex(
          (t) => t.name.toLowerCase() === this.ticker.toLowerCase()
        ) !== -1
      );
    },
    filteredTickers() {
      return this.tickers.filter((t) =>
        t.name.toLowerCase().includes(this.filter)
      );
    },
    paginatedTickers() {
      return this.filteredTickers.slice(this.startIndex, this.endIndex);
    },
    isNextPageAvailable() {
      return this.filteredTickers.length > this.endIndex;
    },
    pageState() {
      return { filter: this.filter, page: this.page };
    },
  },
  watch: {
    ticker() {
      this.clearError();
    },
    tickers() {
      tickersStorage.set(this.tickers);
    },
    paginatedTickers() {
      if (this.paginatedTickers.length === 0 && this.page > 1) {
        this.page -= 1;
      }
    },
    filter() {
      this.page = 1;
    },
    pageState(pageState) {
      urlService.setSearchParams(pageState);
    },
  },
};
</script>
