<template>
  <!-- <b-table class="table-sm" :items="combinedPairList" :fields="tableFields"> </b-table> -->
  <div>
    <div class="controls">
      <b-dropdown size="sm" text="Sort" class="sort m-2">
        <b-dropdown-item-button @click="setSortBy('name')">
          By name
          <span v-if="sortBy === 'name'" v-html="sortSymbol"
        /></b-dropdown-item-button>
        <b-dropdown-item-button @click="setSortBy('profit')"
          >By Profit <span v-if="sortBy === 'profit'" v-html="sortSymbol"
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

const ftbot = namespace('ftbot');

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

  sortBy = 'name';

  sortMethod = 'asc';

  sortSymbol = sortAscSymbol;

  timestampms = timestampms;

  formatPercent = formatPercent;

  get combinedPairList() {
    const comb: CombinedPairList[] = [];

    this.trades.forEach((trade) => {
      let profitString = '';
      const {
        trade_id: id,
        pair,
        profit_ratio: profit,
        profit_abs: profitAbs,
        open_timestamp: openTimestamp,
      } = trade;
      const allLocks = this.currentLocks.filter((el) => el.pair === pair);
      let lockReason = '';
      let locks;

      // Sort to have longer timeframe in front
      allLocks.sort((a, b) => (a.lock_end_timestamp > b.lock_end_timestamp ? -1 : 1));
      if (allLocks.length > 0) {
        [locks] = allLocks;
        lockReason = `${timestampms(locks.lock_end_timestamp)} - ${locks.reason}`;
      }
      profitString += `\nOpen since: ${timestampms(trade.open_timestamp)}`;
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
      });
    });
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
  flex-direction: column;
}

.sort {
  align-self: end;
  padding-right: 14px;
}
</style>
