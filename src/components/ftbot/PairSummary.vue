<template>
  <!-- <b-table class="table-sm" :items="combinedPairList" :fields="tableFields"> </b-table> -->
  <div>
    <div class="controls">
      <b-dropdown size="sm" text="Filter" class="sort m-2">
        <b-dropdown-item-button @click="setFilterType('all_pairs')">
          All pairs</b-dropdown-item-button
        >
        <b-dropdown-item-button @click="setFilterType('no_orders')"
          >Only without open orders</b-dropdown-item-button
        >
        <b-dropdown-item-button @click="setFilterType('open_orders')"
          >Only with open orders</b-dropdown-item-button
        >
        <b-dropdown-item-button @click="setFilterType('with_supertrend')"
          >With Supertrend</b-dropdown-item-button
        >
        <b-dropdown-item-button @click="setFilterType('without_supertrend')"
          >Without Supertrend</b-dropdown-item-button
        >
      </b-dropdown>
      <b-dropdown size="sm" text="Sort" class="sort m-2">
        <b-dropdown-item-button @click="setSortBy('name')">
          By name
          <span v-if="sortBy === 'name'" v-html="sortSymbol"
        /></b-dropdown-item-button>
        <b-dropdown-item-button @click="setSortBy('profit')"
          >By Profit <span v-if="sortBy === 'profit'" v-html="sortSymbol"
        /></b-dropdown-item-button>
        <b-dropdown-item-button @click="setSortBy('volume')"
          >By Volume <span v-if="sortBy === 'volume'" v-html="sortSymbol"
        /></b-dropdown-item-button>
        <b-dropdown-item-button @click="setSortBy('timestamp')"
          >By Open Time <span v-if="sortBy === 'timestamp'" v-html="sortSymbol"
        /></b-dropdown-item-button>
      </b-dropdown>
    </div>

    <b-list-group>
      <b-list-group-item
        v-for="comb in combinedPairList"
        :key="comb.id"
        button
        class="d-flex justify-content-between align-items-center py-1"
        :active="comb.pair === selectedPair"
        :title="`${comb.pair}`"
        @click="setSelectedPair(comb.pair, comb.id)"
      >
        <div>
          {{ comb.pair }}
          <span v-if="comb.locks" :title="comb.lockReason"> &#128274; </span>
        </div>

        <TradeProfit v-if="comb.trade && !backtestMode" :trade="comb.trade" />
        <ProfitPill v-if="backtestMode && comb.tradeCount > 0" :profit-ratio="comb.profit" />
      </b-list-group-item>
    </b-list-group>
  </div>
</template>

<script lang="ts">
import { formatPercent, timestampms } from '@/shared/formatters';
import { BotStoreGetters } from '@/store/modules/ftbot';
import { Lock, Trade } from '@/types';
import { Component, Prop, Vue } from 'vue-property-decorator';
import { namespace } from 'vuex-class';
import SortAscendingIcon from 'vue-material-design-icons/SortAscending.vue';
import SortDescendingIcon from 'vue-material-design-icons/SortDescending.vue';
import TradeProfit from '@/components/ftbot/TradeProfit.vue';
import ProfitPill from '@/components/general/ProfitPill.vue';
import StoreModules from '@/store/storeSubModules';

const ftbot = namespace(StoreModules.ftbot);

interface CombinedPairList {
  id: number;
  pair: string;
  lockReason: string;
  profitString: string;
  trade?: Trade;
  locks?: Lock;
  profit: number;
  profitAbs: number;
  openTimestamp: number;
  superTrendDirection: number;
  volume: null | number;
}

const sortAscSymbol = '&#8595;';
const sortDescSymbol = '&#8593;';

@Component({ components: { SortAscendingIcon, SortDescendingIcon, TradeProfit, ProfitPill } })
export default class PairSummary extends Vue {
  @Prop({ required: true }) pairlist!: string[];

  @Prop({ required: false, default: () => [] }) currentLocks!: Lock[];

  @Prop({ required: true }) trades!: Trade[];

  /** Sort method, "normal" (sorts by open trades > pairlist -> locks) or "profit" */
  // @Prop({ required: false, default: 'name' }) sortBy!: string;

  // @Prop({ required: false, default: 'asc' }) sortMethod!: string;

  @Prop({ required: false, default: false }) backtestMode!: boolean;

  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  @ftbot.Action setSelectedPair!: (pair: string, id: string) => void;

  @ftbot.Getter [BotStoreGetters.selectedPair]!: string;

  filterType = 'open_orders';

  sortBy = 'name';

  sortMethod = 'asc';

  sortSymbol = sortAscSymbol;

  timestampms = timestampms;

  formatPercent = formatPercent;

  get combinedPairList() {
    const comb: CombinedPairList[] = [];

    if (this.filterType === 'all_pairs') {
      this.pairlist.forEach((pair, index) => {
        const trades: Trade[] = this.trades.filter((el) => el.pair === pair);
        const allLocks = this.currentLocks.filter((el) => el.pair === pair);
        let lockReason = '';
        let locks;
        const id = index;
        const openTimestamp = 0;

        // Sort to have longer timeframe in front
        allLocks.sort((a, b) => (a.lock_end_timestamp > b.lock_end_timestamp ? -1 : 1));
        if (allLocks.length > 0) {
          [locks] = allLocks;
          lockReason = `${timestampms(locks.lock_end_timestamp)} - ${locks.reason}`;
        }
        let profitString = '';
        let profit = 0;
        let profitAbs = 0;
        let superTrendDirection = 0;

        trades.forEach((trade) => {
          profit += trade.profit_ratio;
          profitAbs += trade.profit_abs;
        });
        const tradeCount = trades.length;
        const trade = tradeCount ? trades[0] : undefined;
        const volume = trades.length ? trades[trades.length - 1].last_candle.volume : null;
        if (trades.length > 0) {
          profitString = `Current profit: ${formatPercent(profit)}`;
        }
        if (trade) {
          profitString += `\nOpen since: ${timestampms(trade.open_timestamp)}`;
          superTrendDirection = trade.last_candle.SUPERTd || 0;
          // TODO: find a way to get volume from all pairs
        }

        comb.push({
          id,
          pair,
          trade,
          locks,
          lockReason,
          profitString,
          profit,
          profitAbs,
          openTimestamp,
          superTrendDirection,
          volume,
        });
      });
    } else if (this.filterType === 'no_orders') {
      this.pairlist.forEach((pair, index) => {
        const trades: Trade[] = this.trades.filter((el) => el.pair === pair);
        const allLocks = this.currentLocks.filter((el) => el.pair === pair);
        let lockReason = '';
        let locks;
        const id = index;
        const openTimestamp = 0;
        const trade = undefined;
        const profitString = '';
        const profit = 0;
        const profitAbs = 0;
        const superTrendDirection = 0;
        const volume = trades.length ? trades[trades.length - 1].last_candle.volume : null;

        // Sort to have longer timeframe in front
        allLocks.sort((a, b) => (a.lock_end_timestamp > b.lock_end_timestamp ? -1 : 1));
        if (allLocks.length > 0) {
          [locks] = allLocks;
          lockReason = `${timestampms(locks.lock_end_timestamp)} - ${locks.reason}`;
        }
        if (!trades.length) {
          comb.push({
            id,
            pair,
            trade,
            locks,
            lockReason,
            profitString,
            profit,
            profitAbs,
            openTimestamp,
            superTrendDirection,
            volume,
          });
        }
      });
    } else if (['open_orders', 'with_supertrend', 'without_supertrend'].includes(this.filterType)) {
      this.trades.forEach((trade) => {
        let profitString = '';
        const {
          trade_id: id,
          pair,
          profit_ratio: profit,
          profit_abs: profitAbs,
          open_timestamp: openTimestamp,
        } = trade;

        const superTrendDirection = trade.last_candle.SUPERTd || 0;
        const { volume } = trade.last_candle;
        const allLocks = this.currentLocks.filter((el) => el.pair === pair);
        let lockReason = '';
        let locks;
        let addItem = false;

        // Sort to have longer timeframe in front
        allLocks.sort((a, b) => (a.lock_end_timestamp > b.lock_end_timestamp ? -1 : 1));
        if (allLocks.length > 0) {
          [locks] = allLocks;
          lockReason = `${timestampms(locks.lock_end_timestamp)} - ${locks.reason}`;
        }
        profitString += `\nOpen since: ${timestampms(trade.open_timestamp)}`;

        if (this.filterType === 'open_orders') {
          addItem = true;
        } else if (this.filterType === 'with_supertrend' && superTrendDirection > 1) {
          addItem = true;
        } else if (this.filterType === 'without_supertrend' && superTrendDirection <= 1) {
          addItem = true;
        }

        if (addItem) {
          comb.push({
            id,
            pair,
            locks,
            lockReason,
            trade,
            profitString,
            profit,
            profitAbs,
            openTimestamp,
            superTrendDirection,
            volume,
          });
        }
      });
    }
    console.log('comb:', comb);
    if (this.sortBy === 'name') {
      comb.sort((a, b) => {
        if (a.pair > b.pair) {
          return this.sortMethod === 'asc' ? 1 : -1;
        }
        return this.sortMethod === 'asc' ? -1 : 1;
      });
    } else if (this.sortBy === 'profit') {
      comb.sort((a, b) => {
        if (a.profit > b.profit) {
          return this.sortMethod === 'asc' ? 1 : -1;
        }
        return this.sortMethod === 'asc' ? -1 : 1;
      });
    } else if (this.sortBy === 'volume') {
      comb.sort((a, b) => {
        if (a.profit > b.profit) {
          return this.sortMethod === 'asc' ? 1 : -1;
        }
        return this.sortMethod === 'asc' ? -1 : 1;
      });
    } else if (this.sortBy === 'timestamp') {
      comb.sort((a, b) => {
        if (a.openTimestamp > b.openTimestamp) {
          return this.sortMethod === 'asc' ? 1 : -1;
        }
        return this.sortMethod === 'asc' ? -1 : 1;
      });
    }
    return comb;
  }

  get tableFields() {
    return [
      { key: 'pair', label: 'Pair' },
      {
        key: 'locks.lock_end_timestamp',
        label: 'Lock',
        formatter: (value) => (value ? `${timestampms(value)}` : ''),
      },
      {
        key: 'trade.current_profit',
        label: 'Position',
        formatter: (value) => formatPercent(value, 3),
      },
    ];
  }

  setFilterType(type) {
    this.filterType = type;
  }

  setSortBy(by) {
    if (this.sortBy === by) {
      this.sortMethod = this.sortMethod === 'asc' ? 'desc' : 'asc';
    }
    this.sortSymbol = this.sortMethod === 'asc' ? sortAscSymbol : sortDescSymbol;
    this.sortBy = by;
  }
}
</script>

<style scoped>
.list-group {
  text-align: left;
}

.controls {
  display: flex;
  flex-direction: row;
}

.sort {
  align-self: end;
  padding-right: 14px;
}
</style>
