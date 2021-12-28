<template>
  <div>
    <b-form @submit="onSubmit" @reset="onReset">
      <b-form-group description="Start time will be from 00:00 and end time to 23:59">
        <label for="start_date">Start date</label>
        <b-form-datepicker
          id="start_date"
          v-model="form.start_date"
          :max="form.end_date"
          today-button
          reset-button
          close-button
          locale="en"
        ></b-form-datepicker>
        <label for="end_date">End date</label>
        <b-form-datepicker
          id="end_date"
          v-model="form.end_date"
          :min="form.start_date"
          today-button
          reset-button
          close-button
          locale="en"
        ></b-form-datepicker>
      </b-form-group>
      <b-form-group>
        <b-form-select v-model="form.pair" :options="pairs"></b-form-select>
      </b-form-group>
      <b-form-group>
        <b-button type="submit" variant="primary">Search</b-button>
        <b-button type="reset" variant="danger">Reset</b-button>
      </b-form-group>
    </b-form>
    <b-table class="table-sm" :items="performanceStats" :fields="tableFields"></b-table>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import { namespace } from 'vuex-class';
import { BotState, PerformanceEntry, PerformanceFilter } from '@/types';
import { formatPrice } from '@/shared/formatters';
import { BotStoreGetters } from '@/store/modules/ftbot';
import StoreModules from '@/store/storeSubModules';

const ftbot = namespace(StoreModules.ftbot);

@Component({})
export default class Performance extends Vue {
  @Prop({ required: true }) pairlist!: string[];

  // TODO: Verify type of PerformanceStats!
  @ftbot.Getter [BotStoreGetters.performanceStats]!: PerformanceEntry[];

  @ftbot.Getter [BotStoreGetters.botState]?: BotState;

  @ftbot.Action public getPerformance!: (
    // eslint-disable-next-line @typescript-eslint/no-unused-vars
    filter: PerformanceFilter,
  ) => Promise<PerformanceEntry>;

  form: PerformanceFilter = {
    // eslint-disable-next-line @typescript-eslint/camelcase
    start_date: '',
    // eslint-disable-next-line @typescript-eslint/camelcase
    end_date: '',
    pair: '',
  };

  get pairs() {
    return [
      { text: '- All pairs -', value: '' },
      ...this.pairlist.map((pair) => ({ text: pair, value: pair })),
    ];
  }

  get tableFields() {
    return [
      { key: 'pair', label: 'Pair' },
      { key: 'profit', label: 'Profit %' },
      {
        key: 'profit_abs',
        label: `Profit ${this.botState?.stake_currency}`,
        formatter: (v: number) => formatPrice(v, 5),
      },
      { key: 'count', label: 'Count' },
    ];
  }

  onReset() {
    // eslint-disable-next-line @typescript-eslint/camelcase
    this.form.start_date = '';
    // eslint-disable-next-line @typescript-eslint/camelcase
    this.form.end_date = '';
    this.form.pair = '';
    this.getPerformance(this.form);
  }

  onSubmit(event) {
    event.preventDefault();
    this.getPerformance(this.form);
  }
}
</script>
