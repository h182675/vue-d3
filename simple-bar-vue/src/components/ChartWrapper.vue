<template>
  <div>
    <BarChart v-if="chartData.length" :chart-data="chartData" />  
    <select v-model="countryCode">
      <option>USA</option>
      <option>GER</option>
      <option>GRE</option>
    </select>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import axios from 'axios';
import { csvParse } from 'd3-dsv';
import BarChart from '@/components/BarChart.vue';

interface medalData {
  year: number,
  city: string,
  bronzeEvents: Set<string>,
  silverEvents: Set<string>,
  goldEvents: Set<string>,
  totalMedals: number,
}

export default Vue.extend({
  components: {
    BarChart,
  },
  data() {
    return {
      countryCode: 'USA',
      rawData: [] as any[],
    }
  },
  mounted() {
    this.getData('@/assets/summer.csv');
  },
  methods: {
    /**
     * Fetch data from csv
     */
    getData(path: string): void {
      axios.get('summer.csv').then(res => {
        const parsed = csvParse(res.data);
        this.rawData = parsed;
      });
    },
    /**
     * Total the medal count for a given data point
     */
    getTotalMedals(data: medalData): number {
      const { bronzeEvents, silverEvents, goldEvents } = data;
      return bronzeEvents.size + silverEvents.size + goldEvents.size;
    },
    /**
     * Data point factory
     */
    medalDataFactory(year: number, city: string): medalData {
      return {
        year,
        city,
        bronzeEvents: new Set(),
        silverEvents: new Set(),
        goldEvents: new Set(),
        totalMedals: 0,
      };
    },
  },
  computed: {
    /** Format data, group by year, filter by country */
    chartData(): medalData[] {
      if (this.rawData.length) {
        const filteredByCountry: any[] = this.rawData.filter(d => d.Country === this.countryCode);

        // the dataset contains a medal for each athlete on a team event. Need to reduce them down to medal winning events
        const groupedByYear: any = filteredByCountry.reduce((groupedData, d) => {
          if (!groupedData[d.Year]) {
            groupedData[d.Year] = this.medalDataFactory(d.Year, d.City);
          }

          switch (d.Medal.toLowerCase()) {
            case 'bronze':
              groupedData[d.Year].bronzeEvents.add(d.Event);
              break;
            case 'silver':
              groupedData[d.Year].silverEvents.add(d.Event);
              break;
            case 'gold':
              groupedData[d.Year].goldEvents.add(d.Event);
              break;
            default:
              break;
          }

          return groupedData;
        }, {});

        const years: string[] = Object.keys(groupedByYear).sort();
        return years.map((year: string): medalData => ({
          ...groupedByYear[year],
          totalMedals: this.getTotalMedals(groupedByYear[year]),
        }));
      }

      return [];
    },
  },
});
</script>
