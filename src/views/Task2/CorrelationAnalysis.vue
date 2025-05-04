<template>
  <div class="correlation-view">
    <el-tabs v-model="activeCorrelationTab" type="card">
      <!-- 多维变量相关性图 -->
      <el-tab-pane label="多维变量相关性图" name="correlation">
        <div class="correlation-chart" ref="correlationChart"></div>
      </el-tab-pane>

      <!-- 因子贡献叠加图 -->
      <el-tab-pane label="因子贡献叠加图" name="contribution">
        <div class="contribution-chart" ref="contributionChart"></div>
      </el-tab-pane>
      
      <!-- 多变量指纹雷达图 -->
      <el-tab-pane label="多变量指纹雷达" name="radar">
        <div class="radar-chart" ref="radarChart"></div>
      </el-tab-pane>
      
      <!-- 驱动因子占比分析玫瑰图 -->
      <el-tab-pane label="驱动因子占比分析玫瑰图" name="rose">
        <div class="rose-chart" ref="roseChart"></div>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import Papa from 'papaparse';

export default {
  props: ['events'],
  name: 'CorrelationAnalysis',
  data() {
    return {
      activeCorrelationTab: 'correlation', // 默认显示第一个 tab 图表
      selectedVariables: [
        'duration',
        'max_anomaly',
        'cumulative_anomaly',
        'mean_wind',
        'heat_flux_trend',
        'vortex_coverage_ratio'
      ],
      variableOptions: [
        'duration',
        'max_anomaly',
        'cumulative_anomaly',
        'mean_wind',
        'wind_anomaly',
        'mean_heat_flux',
        'heat_flux_trend',
        'vortex_count',
        'vortex_coverage_ratio',
        'cumulative_salinity_anomaly',
        'centroid_change_rate',
        'start_date',
        'end_date',
        'event_id',
        'daily_info',
        'geometry'
      ],
      allEvents: [],
      filteredEvents: [],
      radarSeriesData: [] // 存储雷达图系列数据
    };
  },
  mounted() {
    this.loadData().then(() => {
      this.filterEvents();
      
      // 延迟初始化以确保 DOM 渲染完成
      this.$nextTick(() => {
        setTimeout(() => {
          this.initCurrentChart();
          this.resizeAllCharts();
        }, 100);
      });
    });

    window.addEventListener('resize', this.resizeAllCharts);
  },
  watch: {
    // 监听 tab 切换，延迟加载对应图表
    activeCorrelationTab(newVal) {
      this.$nextTick(() => {
        setTimeout(() => {
          if (newVal === 'correlation') {
            this.initCorrelationChart();
          } else if (newVal === 'contribution') {
            this.initContributionChart();
          } else if (newVal === 'radar') {
            this.initRadarChart();
          } else if (newVal === 'rose') {
            this.initRoseChart();
          }
        }, 100);
      });
    }
  },
  methods: {
    async loadData() {
      try {
        const csvPath = '/data/eventually_heatwave.csv';
        const csvResponse = await fetch(csvPath);
        if (!csvResponse.ok) throw new Error(`CSV 加载失败: ${csvResponse.status}`);
        const csvText = await csvResponse.text();
        const csvData = Papa.parse(csvText, { header: true }).data;

        this.allEvents = csvData
          .filter(row => typeof row === 'object' && row !== null)
          .map(row => {
            const props = {};
            this.variableOptions.forEach(varName => {
              if (['duration', 'max_anomaly', 'cumulative_anomaly'].includes(varName)) {
                props[varName] = Number(row[varName]);
              } else if (varName === 'start_date' || varName === 'end_date') {
                props[varName] = row[varName] ? new Date(row[varName]) : null;
              } else {
                props[varName] = row[varName];
              }
            });
            return { properties: props };
          });

        console.log('成功加载的数据:', this.allEvents);
      } catch (error) {
        console.error('数据加载失败:', error);
      }
    },
    filterEvents() {
      this.filteredEvents = this.allEvents.filter(event => event.properties.duration >= 3);
      console.log('筛选后的数据:', this.filteredEvents);
      
      // 根据热浪强度分组
      this.groupEventsByIntensity();
    },
    groupEventsByIntensity() {
      // 根据最大热强度将事件分为高强度和低强度两组
      const maxAnomalies = this.filteredEvents.map(e => e.properties.max_anomaly);
      const medianAnomaly = this.calculateMedian(maxAnomalies);
      
      this.radarSeriesData = [
        {
          name: '高强度热浪',
          events: this.filteredEvents.filter(e => e.properties.max_anomaly >= medianAnomaly),
          color: '#c23531'
        },
        {
          name: '低强度热浪',
          events: this.filteredEvents.filter(e => e.properties.max_anomaly < medianAnomaly),
          color: '#2f4554'
        }
      ];
    },
    calculateMedian(values) {
      if (!values.length) return 0;
      const sorted = [...values].sort((a, b) => a - b);
      const middle = Math.floor(sorted.length / 2);
      return sorted.length % 2 === 0 
        ? (sorted[middle - 1] + sorted[middle]) / 2 
        : sorted[middle];
    },
    initCurrentChart() {
      if (this.activeCorrelationTab === 'correlation' && this.$refs.correlationChart) {
        this.initCorrelationChart();
      } else if (this.activeCorrelationTab === 'contribution' && this.$refs.contributionChart) {
        this.initContributionChart();
      } else if (this.activeCorrelationTab === 'radar' && this.$refs.radarChart) {
        this.initRadarChart();
      } else if (this.activeCorrelationTab === 'rose' && this.$refs.roseChart) {
        this.initRoseChart();
      }
    },
    initCorrelationChart() {
      if (!this.filteredEvents.length) return;

      const chartDom = this.$refs.correlationChart;
      if (!chartDom) return;

      // 如果已有实例，先销毁再重建
      if (this.correlationChart) {
        this.correlationChart.dispose();
        this.correlationChart = null;
      }

      this.correlationChart = echarts.init(chartDom);

      const data = this.filteredEvents.map(event =>
        this.selectedVariables.map(varName => Number(event.properties[varName]))
      );

      const numVariables = this.selectedVariables.length;
      const correlationMatrix = [];

      for (let i = 0; i < numVariables; i++) {
        correlationMatrix[i] = [];
        for (let j = 0; j < numVariables; j++) {
          const x = data.map(row => row[i]);
          const y = data.map(row => row[j]);
          correlationMatrix[i][j] = this.calculateCorrelation(x, y);
        }
      }

      const option = {
        tooltip: {
          trigger: 'item',
          formatter: params => {
            const val = params.data[2];
            return `${this.selectedVariables[params.data[0]]} 和 ${
              this.selectedVariables[params.data[1]]
            }: ${typeof val === 'number' && !isNaN(val) ? val.toFixed(2) : '-'}`;
          }
        },
        xAxis: {
          type: 'category',
          data: this.selectedVariables,
          axisLabel: { rotate: 45 }
        },
        yAxis: {
          type: 'category',
          data: this.selectedVariables
        },
        visualMap: {
          min: -1,
          max: 1,
          calculable: true,
          orient: 'horizontal',
          bottom: '15%',
          inRange: {
            color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
          }
        },
        series: [{
          name: '相关性矩阵',
          type: 'heatmap',
          data: correlationMatrix
            .map((row, i) => row.map((val, j) => [i, j, val]))
            .flat(),
          label: {
            show: true,
            formatter: params => {
              const val = params.value[2]; // 从数组中取出数值
              return typeof val === 'number' && !isNaN(val) ? val.toFixed(2) : '-';
            }
          }
        }]
      };

      this.correlationChart.setOption(option);
    },
    initContributionChart() {
      if (!this.filteredEvents.length) return;

      const chartDom = this.$refs.contributionChart;
      if (!chartDom) return;

      if (this.contributionChart) {
        this.contributionChart.dispose();
        this.contributionChart = null;
      }

      this.contributionChart = echarts.init(chartDom);

      const data = this.filteredEvents.map(event =>
        this.selectedVariables.map(varName => Number(event.properties[varName]))
      );

      const sumData = data.reduce((acc, row) => {
        row.forEach((val, i) => acc[i] = (acc[i] || 0) + val);
        return acc;
      }, []);

      const total = sumData.reduce((a, b) => a + b, 0);
      const percentData = sumData.map(val => ((val / total) * 100).toFixed(1));

      const option = {
        tooltip: { trigger: 'item', formatter: '{b}: {c}%' },
        xAxis: {
          type: 'category',
          data: this.selectedVariables,
          axisLabel: { rotate: 45 }
        },
        yAxis: { type: 'value', name: '贡献百分比 (%)' },
        series: [{ name: '因子贡献', type: 'bar', data: percentData }]
      };

      this.contributionChart.setOption(option);
    },
    initRadarChart() {
      if (!this.filteredEvents.length || !this.radarSeriesData.length) return;

      const chartDom = this.$refs.radarChart;
      if (!chartDom) return;

      if (this.radarChart) {
        this.radarChart.dispose();
        this.radarChart = null;
      }

      this.radarChart = echarts.init(chartDom);

      // 计算每个变量的平均值和标准化值
      const indicators = this.selectedVariables.map(varName => {
        const values = this.filteredEvents.map(e => Number(e.properties[varName]));
        const maxValue = Math.max(...values.filter(v => !isNaN(v)));
        const minValue = Math.min(...values.filter(v => !isNaN(v)));
        
        return {
          name: varName,
          max: maxValue === minValue ? 1 : maxValue,
          min: minValue
        };
      });

      // 准备雷达图系列数据
      const seriesData = this.radarSeriesData.map(group => {
        // 计算每组事件各变量的平均值
        const avgValues = this.selectedVariables.map(varName => {
          const validEvents = group.events.filter(e => !isNaN(Number(e.properties[varName])));
          if (validEvents.length === 0) return 0;
          
          const sum = validEvents.reduce((acc, e) => acc + Number(e.properties[varName]), 0);
          return sum / validEvents.length;
        });
        
        // 标准化处理
        const normalizedValues = avgValues.map((val, i) => {
          const indicator = indicators[i];
          if (indicator.max === indicator.min) return 0.5;
          return (val - indicator.min) / (indicator.max - indicator.min);
        });

        return {
          value: normalizedValues,
          name: group.name,
          symbol: 'circle',
          symbolSize: 6,
          lineStyle: {
            width: 2,
            type: 'solid'
          },
          areaStyle: {
            opacity: 0.3
          },
          itemStyle: {
            color: group.color
          }
        };
      });

      // 计算变量间的互信息强度（简化版，实际应用中应使用更精确的互信息计算）
      const mutualInfoMatrix = this.calculateMutualInfoMatrix();

      // 雷达图配置
      const option = {
        title: {
          text: '多变量指纹雷达图',
          subtext: '展示不同强度热浪事件的特征指纹',
          left: 'center'
        },
        tooltip: {
          trigger: 'item',
          formatter: params => {
            const index = params.dataIndex;
            const varName = this.selectedVariables[index];
            const value = params.value[index];
            const rawValue = (value * (indicators[index].max - indicators[index].min) + indicators[index].min).toFixed(2);
            return `${params.seriesName}<br>${varName}: ${rawValue} (标准化: ${value.toFixed(2)})`;
          }
        },
        legend: {
          data: this.radarSeriesData.map(g => g.name),
          bottom: 10
        },
        radar: {
          indicator: indicators.map(ind => ({
            name: ind.name,
            max: 1, // 标准化后的最大值
            min: 0  // 标准化后的最小值
          })),
          radius: '65%',
          splitNumber: 5,
          axisName: {
            color: '#333',
            fontSize: 12,
            formatter: (name, indicator) => {
              // 显示变量名和互信息强度
              const miStrength = mutualInfoMatrix[indicator.index] 
                ? mutualInfoMatrix[indicator.index].reduce((a, b) => a + b, 0) / (mutualInfoMatrix.length - 1)
                : 0;
              return `${name}\nMI: ${miStrength.toFixed(2)}`;
            }
          },
          splitArea: {
            show: true,
            areaStyle: {
              color: ['rgba(255, 255, 255, 0.5)', 'rgba(200, 200, 200, 0.3)']
            }
          },
          axisLine: {
            lineStyle: {
              color: 'rgba(100, 100, 100, 0.5)'
            }
          },
          splitLine: {
            lineStyle: {
              color: 'rgba(100, 100, 100, 0.5)'
            }
          }
        },
        series: [{
          name: '热浪事件指纹',
          type: 'radar',
          emphasis: {
            lineStyle: {
              width: 4
            }
          },
          data: seriesData
        }]
      };

      this.radarChart.setOption(option);
      
      // 添加点击事件监听
      this.radarChart.on('click', params => {
        if (params.componentType === 'radar') {
          const clickedVar = this.selectedVariables[params.dataIndex];
          console.log('点击了变量:', clickedVar);
          // 这里可以添加切换变量集的逻辑
        }
      });
    },
    calculateMutualInfoMatrix() {
      // 简化版的互信息计算，实际应用中应使用更精确的算法
      const numVars = this.selectedVariables.length;
      const matrix = Array(numVars).fill().map(() => Array(numVars).fill(0));
      
      // 使用之前计算的相关性矩阵作为简化版的互信息强度
      const data = this.filteredEvents.map(event =>
        this.selectedVariables.map(varName => Number(event.properties[varName]))
      );
      
      for (let i = 0; i < numVars; i++) {
        for (let j = 0; j < numVars; j++) {
          if (i === j) {
            matrix[i][j] = 1; // 自信息设为1
          } else {
            const x = data.map(row => row[i]);
            const y = data.map(row => row[j]);
            const corr = Math.abs(this.calculateCorrelation(x, y));
            matrix[i][j] = corr; // 使用绝对值相关性作为简化互信息
          }
        }
      }
      
      return matrix;
    },
    initRoseChart() {
      if (!this.filteredEvents.length || !this.radarSeriesData.length) return;

      const chartDom = this.$refs.roseChart;
      if (!chartDom) return;

      // 如果已有实例，先销毁再重建
      if (this.roseChart) {
        this.roseChart.dispose();
        this.roseChart = null;
      }

      this.roseChart = echarts.init(chartDom);

      // 准备玫瑰图数据 - 为每组热浪事件创建单独的系列
      const series = this.radarSeriesData.map(group => {
        // 计算该组事件中各变量的平均贡献
        const avgValues = this.selectedVariables.map(varName => {
          const validEvents = group.events.filter(e => !isNaN(Number(e.properties[varName])));
          if (validEvents.length === 0) return 0;
          
          const sum = validEvents.reduce((acc, e) => acc + Number(e.properties[varName]), 0);
          return sum / validEvents.length;
        });
        
        // 计算贡献百分比
        const total = avgValues.reduce((a, b) => a + b, 0);
        const percentData = avgValues.map((val, i) => ({
          value: total > 0 ? (val / total) * 100 : 0,
          name: this.selectedVariables[i]
        }));
        
        return {
          name: group.name,
          type: 'pie',
          radius: [group.name === '高强度热浪' ? '30%' : '50%', '70%'],
          center: ['50%', '50%'],
          roseType: 'area',
          itemStyle: {
            color: group.color
          },
          label: {
            show: true,
            formatter: '{b}: {c}%'
          },
          labelLine: {
            length: 10,
            length2: 10
          },
          emphasis: {
            itemStyle: {
              shadowBlur: 10,
              shadowOffsetX: 0,
              shadowColor: 'rgba(0, 0, 0, 0.5)'
            }
          },
          data: percentData
        };
      });

      const option = {
        title: {
          text: '驱动因子占比分析',
          subtext: '不同强度热浪事件的驱动因子贡献',
          left: 'center'
        },
        tooltip: {
          trigger: 'item',
          formatter: '{a}<br/>{b}: {c}%'
        },
        legend: {
          top: 'bottom',
          data: this.selectedVariables
        },
        toolbox: {
          show: true,
          feature: {
            mark: { show: true },
            dataView: { show: true, readOnly: false },
            restore: { show: true },
            saveAsImage: { show: true }
          }
        },
        series: series
      };

      this.roseChart.setOption(option);
    },
    calculateCorrelation(x, y) {
      const validData = x.map((xi, i) => ({ xi, yi: y[i] }))
                         .filter(d => typeof d.xi === 'number' && typeof d.yi === 'number');

      if (validData.length < 2) return 0;

      const sumX = validData.reduce((a, d) => a + d.xi, 0);
      const sumY = validData.reduce((a, d) => a + d.yi, 0);
      const sumXY = validData.reduce((a, d) => a + d.xi * d.yi, 0);
      const sumX2 = validData.reduce((a, d) => a + d.xi * d.xi, 0);
      const sumY2 = validData.reduce((a, d) => a + d.yi * d.yi, 0);

      const n = validData.length;
      const numerator = n * sumXY - sumX * sumY;
      const denominator = Math.sqrt((n * sumX2 - sumX * sumX) * (n * sumY2 - sumY * sumY));
      return denominator === 0 ? 0 : numerator / denominator;
    },
    resizeAllCharts() {
      if (this.correlationChart) this.correlationChart.resize();
      if (this.contributionChart) this.contributionChart.resize();
      if (this.radarChart) this.radarChart.resize();
      if (this.roseChart) this.roseChart.resize();
    },
    beforeUnmount() {
      if (this.correlationChart) {
        this.correlationChart.dispose();
        this.correlationChart = null;
      }
      if (this.contributionChart) {
        this.contributionChart.dispose();
        this.contributionChart = null;
      }
      if (this.radarChart) {
        this.radarChart.dispose();
        this.radarChart = null;
      }
      if (this.roseChart) {
        this.roseChart.dispose();
        this.roseChart = null;
      }
      window.removeEventListener('resize', this.resizeAllCharts);
    }
  }
};
</script>

<style scoped>
.correlation-view {
  padding: 20px;
  height: calc(100vh - 60px);
  overflow: auto;
}

.correlation-chart,
.contribution-chart,
.radar-chart,
.rose-chart {
  height: 600px;
  width: 100%;
  min-height: 500px;
}
</style>