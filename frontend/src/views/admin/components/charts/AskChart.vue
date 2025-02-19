<template>
  <div class="pr-4">
    <v-chart class="h-100" :option="option" :loading="props.loading" />
  </div>
</template>

<script setup lang="ts">
import { BarSeriesOption } from 'echarts';
import { BarChart } from 'echarts/charts';
import {
  BrushComponent,
  DatasetComponent,
  DataZoomComponent,
  GridComponent,
  LegendComponent,
  TitleComponent,
  ToolboxComponent,
  TooltipComponent,
} from 'echarts/components';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { computed, ref } from 'vue';
import VChart from 'vue-echarts';
import { useI18n } from 'vue-i18n';

import { useAppStore } from '@/store';
import { AskLogAggregation, UserRead } from '@/types/schema';

import { timeFormatter } from './helpers';
const { t } = useI18n();
const appStore = useAppStore();

use([
  TitleComponent,
  CanvasRenderer,
  BarChart,
  GridComponent,
  TooltipComponent,
  LegendComponent,
  DatasetComponent,
  DataZoomComponent,
  ToolboxComponent,
  BrushComponent,
]);

const props = defineProps<{
  loading: boolean;
  askStats: AskLogAggregation[];
  granularity: number;
  users?: UserRead[];
}>();

type AskStatRecord = {
  timestamp: number;
  count: number;
  users: string;
  totalAskTime: number;
  totalQueueingTime: number;
};

type AskDataset = {
  id: number;
  source: AskStatRecord[];
  // dimensions: any;
  type: string;
  model: string;
  name: string;
};

function makeDataset(askRecords: AskLogAggregation[]) {
  const dataset = [] as AskDataset[];

  // 对askRecords按照_id.meta.type和_id.meta.model进行聚合
  const askRecordsGroupByTypeAndModel = askRecords.reduce((acc, cur) => {
    const key = `${cur._id.meta.source}|${cur._id.meta.model}`;
    if (acc[key]) {
      acc[key].push(cur);
    } else {
      acc[key] = [cur];
    }
    return acc;
  }, {} as Record<string, AskLogAggregation[]>);

  Object.entries(askRecordsGroupByTypeAndModel).forEach(([key, value], idx) => {
    const [type, model] = key.split('|');
    const source = value.map((v) => {
      return {
        timestamp: new Date(v._id.start_time).getTime(),
        count: v.count,
        // userIds: v.user_ids || [],
        // findUsername 生成 string，超过5人则省略；格式：'user1, user2, user3, ... 等 x 人'
        users: v.user_ids
          ? v.user_ids.length > 5
            ? `${findUsername(v.user_ids[0])}, ${findUsername(
              v.user_ids[1],
            )}, ${findUsername(v.user_ids[2])}, ... and ${v.user_ids.length - 3} more`
            : v.user_ids.map((id) => findUsername(id)).join(', ')
          : '',
        totalAskTime: v.total_ask_time?.toFixed(2) || 0,
        totalQueueingTime: v.total_queueing_time?.toFixed(2) || 0,
      } as AskStatRecord;
    });
    dataset.push({
      id: idx,
      source,
      // 下面的非echarts配置，用于生成series
      type,
      model,
      name: `${t('sources.' + type)}-${t('models.' + model)}`,
    });
  });

  const allTimestamps = new Set<number>();
  dataset.forEach((d) => {
    d.source.forEach((s) => {
      allTimestamps.add(s.timestamp);
    });
  });
  // 遍历所有source, 补全其在allTimestamps中不存在的值
  dataset.forEach((d) => {
    const source = d.source;
    const timestamps = source.map((s) => s.timestamp);
    const missingTimestamps = Array.from(allTimestamps).filter((t) => !timestamps.includes(t));
    missingTimestamps.forEach((ts) => {
      source.push({
        timestamp: ts,
        count: 0,
        users: t('commons.empty'),
        totalAskTime: 0,
        totalQueueingTime: 0,
      });
    });
    source.sort((a, b) => a.timestamp - b.timestamp);
  });

  return dataset;
}

const dataset = computed(() => {
  if (props.askStats) {
    return makeDataset(props.askStats);
  } else {
    return [];
  }
});

const findUsername = (user_id: number) => {
  const user = props.users?.find((u) => u.id === user_id);
  return user?.username || user_id;
};

const isDark = computed(() => appStore.theme === 'dark');

const generateSeries = (lineColor: string, itemBorderColor: string, datasetIndex: number): BarSeriesOption => {
  return {
    type: 'bar',
    name: dataset.value[datasetIndex].name,
    datasetIndex,
    yAxisIndex: 0,
    encode: {
      x: 'timestamp',
      y: 'count',
      tooltip: ['count', 'totalAskTime', 'totalQueueingTime', 'users'],
    },
    stack: 'total',

    emphasis: {
      focus: 'series',
      itemStyle: {
        // color: lineColor,
        borderWidth: 2,
        borderColor: itemBorderColor,
      },
    },
  };
};

const series = computed(() => {
  // 从 dataset 中生成 series

  // const colors = [
  //   ['#9ce6aa', '#E8FFFB'], // green
  //   ['#F77234', '#FFE4BA'], // orange
  //   ['#F7B334', '#FFF4BA'],
  //   ['#9ce6aa', '#E8FFFB'],
  // ];
  const colorModelMap = {
    'rev|gpt_3_5': ['#9ce6aa', '#E8FFFB'],
    'rev|gpt_4': ['#F77234', '#FFE4BA'],
    'api|gpt_3_5': ['#F7B334', '#FFF4BA'],
    'api|gpt_4': ['#9ce6aa', '#E8FFFB'],
  };

  const getDatasetColors = (d: AskDataset) => {
    const colors = colorModelMap[`${d.type}|${d.model}` as keyof typeof colorModelMap];
    if (colors) {
      return colors;
    } else {
      const color = isDark.value ? '#fff' : '#000';
      return [color, color];
    }
  };

  return dataset.value.map((d, idx) => {
    const colors = getDatasetColors(d);
    return generateSeries(colors[0], colors[1], idx);
  });
});

const showDataZoom = ref(false);
const dataZoomOption = computed(() => {
  return showDataZoom.value
    ? [
      {
        type: 'slider',
        show: showDataZoom.value,
        xAxisIndex: 0,
        start: 0,
        end: 100,
        filterMode: 'filter',
      },
    ]
    : [];
});
const gridBottom = computed(() => {
  return showDataZoom.value ? '25%' : '5%';
});

const option = computed(() => {
  return {
    title: {
      text: t('commons.askRequestsCount'),
      left: 'center',
      top: '2.6%',
      textStyle: {
        color: isDark.value ? '#DDD' : '#4E5969',
        fontSize: 16,
        fontWeight: 500,
      },
    },
    grid: {
      left: '2.6%',
      right: '4',
      top: '40',
      bottom: gridBottom.value,
      containLabel: true,
    },
    dataset: dataset.value,
    xAxis: {
      type: 'time',
      axisLabel: {
        color: '#4E5969',
        formatter: (val: number) => timeFormatter(val, false),
        hideOverlap: true,
      },
      axisLine: {
        show: false,
      },
      axisTick: {
        show: false,
      },
      splitLine: {
        show: true,
        // interval: (idx: number) => {
        //   if (idx === 0) return false;
        //   if (idx === xAxis.value.length - 1) return false;
        //   return true;
        // },
        lineStyle: {
          // source: 'dashed',
          color: isDark.value ? '#2E2E30' : '#E5E8EF',
        },
      },
      axisPointer: {
        show: true,
        lineStyle: {
          color: '#23ADFF',
          width: 2,
        },
      },
    },
    yAxis: [
      {
        type: 'value',
        position: 'left',
        axisLine: {
          show: false,
        },
        axisLabel: {
          formatter(value: number, idx: number) {
            if (idx === 0) return String(value);
            return `${value}`;
          },
        },
        splitLine: {
          lineStyle: {
            type: 'dashed',
            color: isDark.value ? '#2E2E30' : '#E5E8EF',
          },
        },
      },
    ],
    tooltip: {
      trigger: 'axis',
      // formatter: '{a}: {b} {c} {d}'
    },

    series: series.value,

    legend: {
      show: true,
      orient: 'vertical',
      right: 10,
      top: 40,
      textStyle: {
        color: isDark.value ? '#EDEDED' : '#4E5969',
      }
    },

    toolbox: {
      feature: {
        myDataZoom: {
          show: true,
          title: 'DataZoom',
          icon: 'path://M0,0H12V2H0V0ZM0,14H12V16H0V14ZM0,6H12V8H0V6ZM0,10H12V12H0V10Z',
          onclick: () => {
            showDataZoom.value = !showDataZoom.value;
          },
        },
        restore: {},
        saveAsImage: {},
      },
    },
    dataZoom: dataZoomOption.value,
  };
});

// watchEffect(() => {
console.log('props', props.askStats);
// console.log('xAxis', xAxis.value);
// console.log('totalRequestsCountData', totalRequestsCountData.value);
// console.log('datasetSource', datasetSource.value);
// console.log('users', props.users)
//   console.log('option', option.value);
// });
</script>
