<template>
  <GridLayout
    class="h-100 w-100"
    :row-height="50"
    :layout="gridLayout"
    :vertical-compact="false"
    :margin="[5, 5]"
    :is-resizable="!getLayoutLocked"
    :is-draggable="!getLayoutLocked"
    @layout-updated="layoutUpdatedEvent"
  >
    <GridItem
      :i="gridLayoutTradeHistory.i"
      :x="0"
      :w="12"
      :h="12"
      :min-w="3"
      :min-h="4"
      drag-allow-from=".card-header"
    >
      <DraggableContainer header="Closed Trades">
        <TradeList
          class="trade-history"
          :trades="closedTrades"
          title="Trade history"
          empty-text="No closed trades so far."
        />
      </DraggableContainer>
    </GridItem>
    <GridItem
      v-if="detailTradeId"
      :i="gridLayoutTradeDetail.i"
      :x="0"
      :y="12"
      :w="12"
      :h="8"
      :min-h="4"
      drag-allow-from=".card-header"
    >
      <DraggableContainer header="Trade Detail">
        <TradeDetail :trade="tradeDetail"> </TradeDetail>
      </DraggableContainer>
    </GridItem>
  </GridLayout>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import { namespace } from 'vuex-class';
import { GridLayout, GridItem, GridItemData } from 'vue-grid-layout';

import DraggableContainer from '@/components/layout/DraggableContainer.vue';
import TradeDetail from '@/components/ftbot/TradeDetail.vue';
import TradeList from '@/components/ftbot/TradeList.vue';

import { Trade } from '@/types';
import { BotStoreGetters } from '@/store/modules/ftbot';
import { TradeLayout, findGridLayout, LayoutGetters, LayoutActions } from '@/store/modules/layout';

const ftbot = namespace('ftbot');
const layoutNs = namespace('layout');

@Component({
  components: {
    DraggableContainer,
    GridItem,
    GridLayout,
    Performance,
    TradeDetail,
    TradeList,
  },
})
export default class ClosedTrades extends Vue {
  @ftbot.Getter [BotStoreGetters.detailTradeId]!: number;

  @ftbot.Getter [BotStoreGetters.closedTrades]!: Trade[];

  @ftbot.Getter [BotStoreGetters.tradeDetail]!: Trade;

  @layoutNs.Getter [LayoutGetters.getTradingLayout]!: GridItemData[];

  @layoutNs.Action [LayoutActions.setTradingLayout];

  @layoutNs.Getter [LayoutGetters.getLayoutLocked]: boolean;

  get gridLayout(): GridItemData[] {
    return this.getTradingLayout;
  }

  get gridLayoutMultiPane(): GridItemData {
    return findGridLayout(this.gridLayout, TradeLayout.multiPane);
  }

  get gridLayoutTradeHistory(): GridItemData {
    return findGridLayout(this.gridLayout, TradeLayout.tradeHistory);
  }

  get gridLayoutTradeDetail(): GridItemData {
    return findGridLayout(this.gridLayout, TradeLayout.tradeDetail);
  }

  layoutUpdatedEvent(newLayout) {
    this.setTradingLayout(newLayout);
  }
}
</script>

<style scoped></style>
