<template>
  <div class="row flex-grow-1 chart-wrapper">
    <v-chart v-if="hasData" ref="candleChart" :theme="theme" autoresize manual-update />
  </div>
</template>

<script lang="ts">
// @ts-nocheck
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import { Trade, PairHistory, PlotConfig } from '@/types';
import randomColor from '@/shared/randomColor';
import { roundTimeframe } from '@/shared/timemath';
import ECharts from 'vue-echarts';

import { use } from 'echarts/core';
import { EChartsOption, SeriesOption, ScatterSeriesOption } from 'echarts';
import { CanvasRenderer } from 'echarts/renderers';
import {
  CandlestickChart,
  LineChart,
  BarChart,
  ScatterChart,
  EffectScatterChart,
} from 'echarts/charts';
import {
  AxisPointerComponent,
  CalendarComponent,
  DatasetComponent,
  DataZoomComponent,
  GridComponent,
  LegendComponent,
  TimelineComponent,
  TitleComponent,
  ToolboxComponent,
  TooltipComponent,
  VisualMapComponent,
  VisualMapPiecewiseComponent,
} from 'echarts/components';

use([
  AxisPointerComponent,
  CalendarComponent,
  DatasetComponent,
  DataZoomComponent,
  GridComponent,
  LegendComponent,
  TimelineComponent,
  TitleComponent,
  ToolboxComponent,
  TooltipComponent,
  VisualMapComponent,
  VisualMapPiecewiseComponent,

  CandlestickChart,
  BarChart,
  LineChart,
  ScatterChart,
  EffectScatterChart,
  CanvasRenderer,
]);

// Chart default options
const MARGINLEFT = '5.5%';
const MARGINRIGHT = '1%';
const NAMEGAP = 55;
const SUBPLOTHEIGHT = 8; // Value in %

// Binance colors
const upColor = '#26A69A';
const upBorderColor = '#26A69A';
const downColor = '#EF5350';
const downBorderColor = '#EF5350';

const buySignalColor = '#00ff26';
const shortEntrySignalColor = '#00ff26';
const sellSignalColor = '#faba25';
const shortexitSignalColor = '#faba25';
const tradeBuyColor = '#FFFE06';
const tradeSellColor = '#00FF35';
const tradeBorderColor = '#212F3D';

@Component({
  components: { 'v-chart': ECharts },
})
export default class CandleChart extends Vue {
  $refs!: {
    candleChart: typeof ECharts;
  };

  @Prop({ required: false, default: [] }) readonly trades!: Array<Trade>;

  @Prop({ required: true }) readonly dataset!: PairHistory;

  @Prop({ default: true }) readonly useUTC!: boolean;

  @Prop({ required: true }) plotConfig!: PlotConfig;

  @Prop({ default: 'dark' }) theme!: string;

  buyData = [] as Array<number>[];

  sellData = [] as Array<number>[];

  chartOptions: EChartsOption = {};

  activeSeries: object | null;

  @Watch('dataset')
  datasetChanged() {
    this.updateChart();
  }

  @Watch('plotConfig')
  plotConfigChanged() {
    this.initializeChartOptions();
  }

  get strategy() {
    return this.dataset ? this.dataset.strategy : '';
  }

  get pair() {
    return this.dataset ? this.dataset.pair : '';
  }

  get timeframe() {
    return this.dataset ? this.dataset.timeframe : '';
  }

  get timeframems() {
    return this.dataset ? this.dataset.timeframe_ms : 0;
  }

  get datasetColumns() {
    return this.dataset ? this.dataset.columns : [];
  }

  get hasData() {
    return this.dataset !== null && typeof this.dataset === 'object';
  }

  get filteredTrades() {
    return this.trades.filter((item: Trade) => item.pair === this.pair);
  }

  mounted() {
    this.initializeChartOptions();
  }

  get chartTitle() {
    return `${this.strategy} - ${this.pair} - ${this.timeframe}`;
  }

  initializeChartOptions() {
    function numberWithCommas(x) {
      return x.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ',');
    }

    const indicatorsNames = Object.keys(this.plotConfig.main_plot).concat(
      ...Object.keys(this.plotConfig.subplots).map((o) => [
        ...Object.keys(this.plotConfig.subplots[o]),
      ]),
    );

    this.chartOptions = {
      title: [
        {
          // text: this.chartTitle,
          show: false,
        },
      ],
      backgroundColor: 'rgba(0, 0, 0, 0)',
      useUTC: this.useUTC,
      animation: false,
      legend: {
        // Initial legend, further entries are pushed to the below list
        data: ['Candles', 'Volume', 'Buy', 'Sell'],
        right: '1%',
      },
      tooltip: {
        show: true,
        trigger: 'axis',
        backgroundColor: 'rgba(80,80,80,0.7)',
        borderWidth: 0,
        textStyle: {
          color: '#fff',
        },
        axisPointer: {
          type: 'cross',
          lineStyle: {
            color: '#cccccc',
            width: 1,
            opacity: 1,
          },
        },
        // positioning copied from https://echarts.apache.org/en/option.html#tooltip.position
        position(pos, params, dom, rect, size) {
          // tooltip will be fixed on the right if mouse hovering on the left,
          // and on the left if hovering on the right.
          const obj = { top: 60 };
          obj[['left', 'right'][+(pos[0] < size.viewSize[0] / 2)]] = 5;
          return obj;
        },
        formatter(params) {
          function toLabelValueHtml(label, value) {
            return `<div><span style="font-weight: 600;">${label}: </span><span>${value}</span></div>`;
          }

          const SPACES = '&#160;&#160;';
          const date = params[0].axisValueLabel;
          const sectionsHtml: [string] = [''];
          const Candles = params.find((p) => p.seriesName === 'Candles');
          const Volume = params.find((p) => p.seriesName === 'Volume');
          const Buy = params.find((p) => p.seriesName === 'Buy');
          const Sell = params.find((p) => p.seriesName === 'Sell');
          const TradesList = params.filter((p) => p.seriesName === 'Trades');
          const TradesCloseList = params.filter((p) => p.seriesName === 'Trades Close');
          const Indicators = params.filter((p) => indicatorsNames.indexOf(p.seriesName) > -1);
          const buyTag = (Candles || Volume || Buy || Sell).data[14] || params[0].data[8];

          if (Volume) {
            sectionsHtml.push(toLabelValueHtml('Volume', numberWithCommas(Volume.data[5])));
          }
          if (Candles) {
            sectionsHtml.push(
              `<div style="display: flex; flex-direction: column;">${toLabelValueHtml(
                'Candles',
                '',
              )}<div style="display: flex; flex-direction: column;">
              <span>${SPACES}open = ${Candles.data[1]}</span>
              <span>${SPACES}highest = ${Candles.data[2]}</span>
              <span>${SPACES}lowest = ${Candles.data[3]}</span>
              <span>${SPACES}close = ${Candles.data[4]}</span>
              </div></div>`,
            );
          }
          if (Buy && Buy.data[11]) {
            sectionsHtml.push(toLabelValueHtml('Buy', numberWithCommas(Buy.data[11])));
          }
          if (Sell && Sell.data[12]) {
            sectionsHtml.push(toLabelValueHtml('Sell', numberWithCommas(Sell.data[12])));
          }
          if (TradesList.length) {
            sectionsHtml.push(
              `<div style="display: flex; flex-direction: column;">${toLabelValueHtml(
                'Trades',
                '',
              )}<div style="display: flex; flex-direction: column;">
              ${TradesList.map((t) => {
                const trade = t.data[2];
                const { tradeId } = trade;
                const value = t.data[1];

                return `<span>${SPACES}#${tradeId} = ${value}</span>`;
              }).join('')}
            
              </div></div>`,
            );
          }
          if (TradesCloseList.length) {
            sectionsHtml.push(
              `<div style="display: flex; flex-direction: column;">${toLabelValueHtml(
                'Trades Close',
                '',
              )}<div style="display: flex; flex-direction: column;">
              ${TradesCloseList.map((t) => {
                const trade = t.data[2];
                const { tradeId } = trade;
                const value = t.data[1];

                return `<span>${SPACES}#${tradeId} = ${value}</span>`;
              }).join('')}
            
              </div></div>`,
            );
          }
          console.log('Indicators:', Indicators);
          if (Indicators.length) {
            sectionsHtml.push(
              `<div style="display: flex; flex-direction: column;">${toLabelValueHtml(
                'Indicators',
                '',
              )}<div style="display: flex; flex-direction: column;">
              ${Indicators.map((indicator, index) => {
                const name = indicator.seriesName;
                const value = indicator.data[6 + index];

                return `<span>${SPACES}${name} = ${value}</span>`;
              }).join('')}
            
              </div></div>`,
            );
          }

          const html = `
            <div style="display: flex; flex-direction: column; text-align: left;">
              ${toLabelValueHtml('Date', date)}
              ${buyTag ? toLabelValueHtml('Buy Tag', buyTag) : ''}
              ${sectionsHtml.join('\n')}
            </div>
          `;
          return html;
        },
      },
      axisPointer: {
        link: [{ xAxisIndex: 'all' }],
        label: {
          backgroundColor: '#777',
        },
      },
      xAxis: [
        {
          type: 'time',
          scale: true,
          boundaryGap: false,
          axisLine: { onZero: false },
          axisTick: { show: true },
          axisLabel: { show: true },
          axisPointer: {
            label: { show: false },
          },
          position: 'top',
          splitLine: { show: false },
          splitNumber: 20,
          min: 'dataMin',
          max: 'dataMax',
        },
        {
          type: 'time',
          gridIndex: 1,
          scale: true,
          boundaryGap: false,
          axisLine: { onZero: false },
          axisTick: { show: false },
          axisLabel: { show: false },
          axisPointer: {
            label: { show: false },
          },
          splitLine: { show: false },
          splitNumber: 20,
          min: 'dataMin',
          max: 'dataMax',
        },
      ],
      yAxis: [
        {
          scale: true,
        },
        {
          scale: true,
          gridIndex: 1,
          splitNumber: 2,
          name: 'volume',
          nameLocation: 'middle',
          // position: 'right',
          nameGap: NAMEGAP,
          axisLabel: { show: false },
          axisLine: { show: false },
          axisTick: { show: false },
          splitLine: { show: false },
        },
      ],
      dataZoom: [
        // Start values are recalculated once the data is known
        {
          type: 'inside',
          xAxisIndex: [0, 1],
          start: 80,
          end: 100,
        },
        {
          show: true,
          xAxisIndex: [0, 1],
          type: 'slider',
          bottom: 10,
          start: 80,
          end: 100,
        },
      ],
      visualMap: {
        //  TODO: this would allow to colorize volume bars (if we'd want this)
        //  Needs green / red indicator column in data.
        show: true,
        seriesIndex: 1,
        dimension: 5,
        pieces: [
          {
            max: 500000.0,
            color: downColor,
          },
          {
            min: 500000.0,
            color: upColor,
          },
        ],
      },
    };

    this.updateChart(true);
  }

  updateChart(initial = false) {
    if (!this.hasData) {
      return;
    }
    if (this.chartOptions?.title) {
      this.chartOptions.title[0].text = this.chartTitle;
    }
    const colDate = this.dataset.columns.findIndex((el) => el === '__date_ts');
    const colOpen = this.dataset.columns.findIndex((el) => el === 'open');
    const colHigh = this.dataset.columns.findIndex((el) => el === 'high');
    const colLow = this.dataset.columns.findIndex((el) => el === 'low');
    const colClose = this.dataset.columns.findIndex((el) => el === 'close');
    const colVolume = this.dataset.columns.findIndex((el) => el === 'volume');
    // TODO: Remove below *signal_open after December 2021
    const colBuyData = this.dataset.columns.findIndex(
      (el) =>
        el === '_buy_signal_open' ||
        el === '_buy_signal_close' ||
        el === '_enter_long_signal_close',
    );
    const colSellData = this.dataset.columns.findIndex(
      (el) =>
        el === '_sell_signal_open' ||
        el === '_sell_signal_close' ||
        el === '_exit_long_signal_close',
    );

    const hasShorts =
      this.dataset.enter_short_signals &&
      this.dataset.enter_short_signals > 0 &&
      this.dataset.exit_short_signals &&
      this.dataset.exit_short_signals > 0;
    const colShortEntryData = this.dataset.columns.findIndex(
      (el) => el === '_enter_short_signal_close',
    );
    const colShortExitData = this.dataset.columns.findIndex(
      (el) => el === '_exit_short_signal_close',
    );
    console.log('short_exit', colShortExitData);

    const subplotCount =
      'subplots' in this.plotConfig ? Object.keys(this.plotConfig.subplots).length + 1 : 1;

    if (this.chartOptions?.dataZoom && Array.isArray(this.chartOptions?.dataZoom)) {
      // Only set zoom once ...
      if (initial) {
        const startingZoom = (1 - 250 / this.dataset.length) * 100;
        this.chartOptions.dataZoom.forEach((el, i) => {
          if (this.chartOptions && this.chartOptions.dataZoom) {
            this.chartOptions.dataZoom[i].start = startingZoom;
          }
        });
      } else {
        // Remove start/end settings after chart initialization to avoid chart resetting
        this.chartOptions.dataZoom.forEach((el, i) => {
          if (this.chartOptions && this.chartOptions.dataZoom) {
            delete this.chartOptions.dataZoom[i].start;
            delete this.chartOptions.dataZoom[i].end;
          }
        });
      }
    }

    const options: EChartsOption = {
      dataset: {
        source: this.dataset.data,
      },
      grid: [
        {
          left: MARGINLEFT,
          right: MARGINRIGHT,
          // Grid Layout from bottom to top
          bottom: `${subplotCount * SUBPLOTHEIGHT + 2}%`,
        },
        {
          // Volume
          left: MARGINLEFT,
          right: MARGINRIGHT,
          // Grid Layout from bottom to top
          bottom: `${subplotCount * SUBPLOTHEIGHT}%`,
          height: `${SUBPLOTHEIGHT}%`,
        },
      ],
      series: [
        {
          name: 'Candles',
          type: 'candlestick',
          barWidth: '80%',
          itemStyle: {
            color: upColor,
            color0: downColor,
            borderColor: upBorderColor,
            borderColor0: downBorderColor,
          },
          encode: {
            x: colDate,
            // open, close, low, high
            y: [colOpen, colClose, colLow, colHigh],
          },
        },
        {
          name: 'Volume',
          type: 'bar',
          xAxisIndex: 1,
          yAxisIndex: 1,
          itemStyle: {
            color: '#777777',
          },
          large: true,
          encode: {
            x: colDate,
            y: colVolume,
          },
        },
        {
          name: 'Buy',
          type: 'scatter',
          symbol: 'triangle',
          symbolSize: 12,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            opacity: 1,
            borderColor: tradeBorderColor,
            borderWidth: 1.4,
            color: buySignalColor,
          },
          encode: {
            x: colDate,
            y: colBuyData,
          },
        },
        {
          name: 'Sell',
          type: 'scatter',
          symbol: 'diamond',
          symbolSize: 8,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            color: sellSignalColor,
          },
          encode: {
            x: colDate,
            y: colSellData,
          },
        },
      ],
    };

    if (hasShorts) {
      // Add short support
      if (!Array.isArray(this.chartOptions.legend) && this.chartOptions.legend?.data) {
        this.chartOptions.legend.data.push('Short');
        this.chartOptions.legend.data.push('Short exit');
      }
      if (Array.isArray(options.series)) {
        options.series.push({
          name: 'Short',
          type: 'scatter',
          symbol: 'pin',
          symbolSize: 10,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            color: shortEntrySignalColor,
          },
          encode: {
            x: colDate,
            y: colShortEntryData,
          },
        });
        options.series.push({
          name: 'Short exit',
          type: 'scatter',
          symbol: 'pin',
          symbolSize: 8,
          xAxisIndex: 0,
          yAxisIndex: 0,
          itemStyle: {
            color: shortexitSignalColor,
          },
          encode: {
            x: colDate,
            y: colShortExitData,
          },
        });
      }
    }

    // Merge this into original data
    Object.assign(this.chartOptions, options);

    if ('main_plot' in this.plotConfig) {
      Object.entries(this.plotConfig.main_plot).forEach(([key, value]) => {
        const col = this.dataset.columns.findIndex((el) => el === key);
        if (col > 1) {
          if (!Array.isArray(this.chartOptions.legend) && this.chartOptions.legend?.data) {
            this.chartOptions.legend.data.push(key);
          }
          const sp: SeriesOption = {
            name: key,
            type: value.type || 'line',
            xAxisIndex: 0,
            yAxisIndex: 0,
            itemStyle: {
              color: value.color,
            },
            encode: {
              x: colDate,
              y: col,
            },
            showSymbol: false,
          };
          if (Array.isArray(this.chartOptions.series)) {
            this.chartOptions.series.push(sp);
          }
        } else {
          console.log(`element ${key} for main plot not found in columns.`);
        }
      });
    }

    // START Subplots
    if ('subplots' in this.plotConfig) {
      let plotIndex = 2;
      Object.entries(this.plotConfig.subplots).forEach(([key, value]) => {
        // define yaxis

        // Subplots are added from bottom to top - only the "bottom-most" plot stays at the bottom.
        // const currGridIdx = totalSubplots - plotIndex > 1 ? totalSubplots - plotIndex : plotIndex;
        const currGridIdx = plotIndex;
        if (Array.isArray(this.chartOptions.yAxis) && this.chartOptions.yAxis.length <= plotIndex) {
          this.chartOptions.yAxis.push({
            scale: true,
            gridIndex: currGridIdx,
            name: key,
            nameLocation: 'middle',
            nameGap: NAMEGAP,
            axisLabel: { show: true },
            axisLine: { show: false },
            axisTick: { show: false },
            splitLine: { show: false },
          });
        }
        if (Array.isArray(this.chartOptions.xAxis) && this.chartOptions.xAxis.length <= plotIndex) {
          this.chartOptions.xAxis.push({
            type: 'time',
            scale: true,
            gridIndex: currGridIdx,
            boundaryGap: false,
            axisLine: { onZero: false },
            axisTick: { show: false },
            axisLabel: { show: false },
            axisPointer: {
              label: { show: false },
            },
            splitLine: { show: false },
            splitNumber: 20,
          });
        }
        if (Array.isArray(this.chartOptions.dataZoom)) {
          this.chartOptions.dataZoom.forEach((el) =>
            el.xAxisIndex && Array.isArray(el.xAxisIndex) ? el.xAxisIndex.push(plotIndex) : null,
          );
        }
        if (this.chartOptions.grid && Array.isArray(this.chartOptions.grid)) {
          this.chartOptions.grid.push({
            left: MARGINLEFT,
            right: MARGINRIGHT,
            bottom: `${(subplotCount - plotIndex + 1) * SUBPLOTHEIGHT}%`,
            height: `${SUBPLOTHEIGHT}%`,
          });
        }
        Object.entries(value).forEach(([sk, sv]) => {
          // entries per subplot
          const col = this.dataset.columns.findIndex((el) => el === sk);
          if (col > 0) {
            if (!Array.isArray(this.chartOptions.legend) && this.chartOptions.legend?.data) {
              this.chartOptions.legend.data.push(sk);
            }
            const sp: SeriesOption = {
              name: sk,
              type: sv.type || 'line',
              xAxisIndex: plotIndex,
              yAxisIndex: plotIndex,
              itemStyle: {
                color: sv.color || randomColor(),
              },
              encode: {
                x: colDate,
                y: col,
              },
              showSymbol: false,
            };
            if (this.chartOptions.series && Array.isArray(this.chartOptions.series)) {
              this.chartOptions.series.push(sp);
            }
          } else {
            console.log(`element ${sk} was not found in the columns.`);
          }
        });

        plotIndex += 1;
      });
    }
    // END Subplots
    // Last subplot should show xAxis labels
    // if (options.xAxis && Array.isArray(options.xAxis)) {
    //   options.xAxis[options.xAxis.length - 1].axisLabel.show = true;
    //   options.xAxis[options.xAxis.length - 1].axisTick.show = true;
    // }
    if (this.chartOptions.grid && Array.isArray(this.chartOptions.grid)) {
      // Last subplot is bottom
      this.chartOptions.grid[this.chartOptions.grid.length - 1].bottom = '50px';
      delete this.chartOptions.grid[this.chartOptions.grid.length - 1].top;
    }
    const { trades, tradesClose } = this.getTradeEntries();
    const notClosedTrades = trades.filter(
      (t) => !tradesClose.find((_t) => _t[2].tradeId === t[2].tradeId),
    );

    const name = 'Trades';
    const nameClose = 'Trades Close';
    if (!Array.isArray(this.chartOptions.legend) && this.chartOptions.legend?.data) {
      this.chartOptions.legend.data.push(name);
    }
    const sp: ScatterSeriesOption | (number | string | object) = {
      name,
      type: 'scatter',
      xAxisIndex: 0,
      yAxisIndex: 0,
      symbolSize: 15,
      itemStyle: {
        opacity: 1,
        borderColor: tradeBorderColor,
        borderWidth: 1.4,
        color: tradeBuyColor,
      },
      data: trades.filter((t) => !notClosedTrades.find((_t) => _t[2].tradeId === t[2].tradeId)),
    };
    if (Array.isArray(this.chartOptions?.series)) {
      this.chartOptions.series.push(sp);
    }
    if (!Array.isArray(this.chartOptions.legend) && this.chartOptions.legend?.data) {
      this.chartOptions.legend.data.push(nameClose);
    }
    const closeSeries: ScatterSeriesOption | (number | string | object) = {
      name: nameClose,
      type: 'scatter',
      xAxisIndex: 0,
      yAxisIndex: 0,
      symbolSize: 15,
      itemStyle: {
        opacity: 1,
        borderColor: tradeBorderColor,
        borderWidth: 1.4,
        color: tradeSellColor,
      },
      data: tradesClose,
    };
    const notCloseSeries: ScatterSeriesOption | (number | string | object) = {
      name,
      type: 'effectScatter',
      xAxisIndex: 0,
      yAxisIndex: 0,
      animation: true,
      colorBy: 'series',
      symbolSize: 10,
      rippleEffect: {
        period: 6,
        scale: 4,
      },
      itemStyle: {
        opacity: 1,
        borderColor: tradeBorderColor,
        borderWidth: 1.4,
        color: tradeBuyColor,
      },
      data: notClosedTrades,
    };
    if (this.chartOptions.series && Array.isArray(this.chartOptions.series)) {
      this.chartOptions.series.push(closeSeries);
    }
    this.chartOptions.series.push(notCloseSeries);

    // eslint-disable-next-line no-restricted-syntax
    for (const trade of trades) {
      const data = [trade];
      const { tradeId } = trade[2];
      const foundTradeClose = tradesClose.find((t) => t[2].tradeId === tradeId);

      if (foundTradeClose) {
        data.push(foundTradeClose);

        const trades: ScatterSeriesOption | (number | string | object) = {
          name: 'Connected Trades',
          type: 'line',
          symbol: 'none',
          lineStyle: {
            type: 'solid',
          },
          data,
        };
        this.chartOptions.series.push(trades);
      }
    }

    this.$refs.candleChart.setOption(this.chartOptions);
  }

  /** Return trade entries for charting */
  getTradeEntries() {
    const trades: (string | number | object)[][] = [];
    const tradesClose: (string | number | object)[][] = [];
    for (let i = 0, len = this.filteredTrades.length; i < len; i += 1) {
      const trade: Trade = this.filteredTrades[i];
      if (
        trade.open_timestamp >= this.dataset.data_start_ts &&
        trade.open_timestamp <= this.dataset.data_stop_ts
      ) {
        trades.push([
          roundTimeframe(this.timeframems, trade.open_timestamp),
          trade.open_rate,
          { tradeId: trade.trade_id, buyTag: trade.buy_tag },
        ]);
      }
      if (
        trade.close_timestamp !== undefined &&
        trade.close_timestamp < this.dataset.data_stop_ts &&
        trade.close_timestamp > this.dataset.data_start_ts
      ) {
        if (trade.close_date !== undefined && trade.close_rate !== undefined) {
          tradesClose.push([
            roundTimeframe(this.timeframems, trade.close_timestamp),
            trade.close_rate,
            { tradeId: trade.trade_id, buyTag: trade.buy_tag },
          ]);
        }
      }
    }
    return { trades, tradesClose };
  }

  // createSignalData(colDate: number, colOpen: number, colBuy: number, colSell: number): void {
  // Calculate Buy and sell Series
  // if (!this.signalsCalculated) {
  //   // Generate Buy and sell array (using open rate to display marker)
  //   for (let i = 0, len = this.dataset.data.length; i < len; i += 1) {
  //     if (this.dataset.data[i][colBuy] === 1) {
  //       this.buyData.push([this.dataset.data[i][colDate], this.dataset.data[i][colOpen]]);
  //     }
  //     if (this.dataset.data[i][colSell] === 1) {
  //       this.sellData.push([this.dataset.data[i][colDate], this.dataset.data[i][colOpen]]);
  //     }
  //   }
  //   this.signalsCalculated = true;
  // }
  // }

  @Watch('useUTC')
  useUTCChanged() {
    this.initializeChartOptions();
  }
}
</script>

<style scoped>
.chart-wrapper {
  width: 100%;
  height: 100%;
}
.echarts {
  width: 100%;
  min-height: 200px;
  /* TODO: height calculation is not working correctly - uses min-height for now */
  /* height: 600px; */
  height: 100%;
}
</style>
