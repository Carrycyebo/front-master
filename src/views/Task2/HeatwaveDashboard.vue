<template>
    <div class="heatwave-dashboard">
        <!-- 主视图容器 -->
        <div class="main-view">
            <!-- 地图视图 -->
            <div class="map-view">
                <div id="map-container" ref="mapContainer"></div>
                <div class="map-controls">
                    <el-button-group>
                        <el-button @click="toggleHeatmap" :type="showHeatmap? 'primary' : ''">
                            {{ showHeatmap? '关闭热力图' : '显示热力图' }}
                        </el-button>
                        <el-button @click="showCorrelationView =!showCorrelationView">
                            {{ showCorrelationView? '隐藏关联分析' : '显示关联分析' }}
                        </el-button>
                    </el-button-group>
                    <div class="legend-container">
                        <div v-if="showHeatmap" class="legend heatmap-legend">
                            <h4>热浪{{ heatmapType === 'density'? '密度' : '强度' }}热力图</h4>
                            <div class="gradient-bar"></div>
                            <div class="legend-labels">
                                <span>低</span>
                                <span>高</span>
                            </div>
                        </div>
                        <div v-else class="legend intensity-legend">
                            <h4>热浪强度</h4>
                            <div v-for="(item, index) in intensityRanges" :key="index" class="legend-item">
                                <div class="color-box" :style="{ backgroundColor: item.color }"></div>
                                <span>{{ item.label }}</span>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- 关联分析视图 -->
                <div v-if="showCorrelationView" class="correlation-view">
                    <el-tabs v-model="activeCorrelationTab" type="card">
                        <el-tab-pane label="多维变量相关性图" name="correlation">
                            <div class="correlation-chart" ref="correlationChart"></div>
                        </el-tab-pane>
                        <el-tab-pane label="增强型平行坐标图" name="parallel">
                            <div class="parallel-chart" ref="parallelChart"></div>
                        </el-tab-pane>
                        <el-tab-pane label="因子贡献叠加图" name="contribution">
                            <div class="contribution-chart" ref="contributionChart"></div>
                        </el-tab-pane>
                        <el-tab-pane label="多变量指纹雷达" name="radar">
                            <div class="radar-chart" ref="radarChart"></div>
                        </el-tab-pane>
                        <el-tab-pane label="环境因子关联弦图" name="chord">
                            <div class="chord-chart" ref="chordChart"></div>
                        </el-tab-pane>
                        <el-tab-pane label="驱动因子占比分析玫瑰图" name="rose">
                            <div class="rose-chart" ref="roseChart"></div>
                        </el-tab-pane>
                    </el-tabs>
                </div>
            </div>

            <!-- 控制面板 -->
            <div class="control-panel">
                <el-collapse v-model="activeControls">
                    <el-collapse-item title="时间筛选" name="time">
                        <el-date-picker
                                v-model="timeRange"
                                type="daterange"
                                range-separator="至"
                                start-placeholder="开始日期"
                                end-placeholder="结束日期"
                                value-format="YYYY-MM-DD"
                                @change="filterEvents"
                        />
                    </el-collapse-item>

                    <el-collapse-item title="强度筛选" name="intensity">
                        <div class="slider-control">
                            <span>持续时间 (天)</span>
                            <el-slider
                                    v-model="minDuration"
                                    :min="1"
                                    :max="30"
                                    :step="1"
                                    @change="filterEvents"
                            />
                        </div>
                        <div class="slider-control">
                            <span>累计强度</span>
                            <el-slider
                                    v-model="minCumulativeIntensity"
                                    :min="0"
                                    :max="2000"
                                    :step="10"
                                    @change="filterEvents"
                            />
                        </div>
                        <div class="slider-control">
                            <span>最大强度</span>
                            <el-slider
                                    v-model="minMaxIntensity"
                                    :min="0"
                                    :max="50"
                                    :step="1"
                                    @change="filterEvents"
                            />
                        </div>
                    </el-collapse-item>

                    <el-collapse-item title="热力图设置" name="heatmap" v-if="showHeatmap">
                        <el-radio-group v-model="heatmapType" size="small" @change="updateHeatmap">
                            <el-radio-button label="density">事件密度</el-radio-button>
                            <el-radio-button label="intensity">强度分布</el-radio-button>
                        </el-radio-group>
                        <div class="slider-control">
                            <span>热力半径</span>
                            <el-slider
                                    v-model="heatmapRadius"
                                    :min="10"
                                    :max="50"
                                    :step="5"
                                    @change="updateHeatmap"
                            />
                        </div>
                    </el-collapse-item>

                    <el-collapse-item title="分析变量" name="variables">
                        <el-checkbox-group v-model="selectedVariables">
                            <el-checkbox v-for="varName in variableOptions"
                                         :key="varName"
                                         :label="varName">
                                {{ varName }}
                            </el-checkbox>
                        </el-checkbox-group>
                    </el-collapse-item>
                </el-collapse>
            </div>
        </div>
        <div v-if="isLoading" class="loading-overlay">
            数据加载中...
        </div>
        <div v-if="loadError" class="error-message">
            加载错误: {{ loadError }}
        </div>
    </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import 'leaflet.heat/dist/leaflet-heat.js';
import * as echarts from 'echarts';
import { ElMessage } from 'element-plus';
import Papa from 'papaparse'; // 用于解析CSV文件

export default {
    name: 'HeatwaveDashboard',
    data() {
        return {
            // 地图相关
            map: null,
            geoJsonLayer: null,
            heatLayer: null,
            showHeatmap: false,
            heatmapType: 'density',
            heatmapRadius: 25,
            heatmapGradient: { 0.4: 'blue', 0.6: 'cyan', 0.8: 'lime', 1.0: 'red' },

            // 图表相关
            correlationChart: null,
            parallelChart: null,
            contributionChart: null,
            radarChart: null,
            chordChart: null,
            roseChart: null,
            showCorrelationView: true,
            activeCorrelationTab: 'correlation',
            selectedVariables: [
                'duration','max_anomaly', 'cumulative_anomaly',
               'mean_wind', 'heat_flux_trend', 'vortex_coverage_ratio'
            ],
            variableOptions: [
                'duration','max_anomaly', 'cumulative_anomaly',
               'mean_wind', 'wind_anomaly','mean_heat_flux',
                'heat_flux_trend', 'vortex_count', 'vortex_coverage_ratio',
                'cumulative_salinity_anomaly', 'centroid_change_rate',
               'start_date', 'end_date', 'event_id', 'daily_info', 'geometry'
            ],

            // 数据筛选
            allEvents: [],
            filteredEvents: [],
            timeRange: ['2020-06-01', '2020-08-31'],
            minDuration: 3,
            minCumulativeIntensity: 100,
            minMaxIntensity: 1,

            // UI控制
            activeControls: ['time', 'intensity'],
            intensityRanges: [
                { min: 0, max: 100, color: '#4CAF50', label: '低 (<100)' },
                { min: 100, max: 300, color: '#FFC107', label: '中 (100-300)' },
                { min: 300, max: Infinity, color: '#F44336', label: '高 (>300)' }
            ],

            // 新添加的图表数据
            correlationMatrix: [],
            parallelData: [],
            contributionData: [],
            radarData: [],
            chordData: [],
            roseData: [],

            // 加载状态
            isLoading: false,
            loadError: null,

            // 新增：确保 geojsonFeatures 被定义
            geojsonFeatures: []
        };
    },
    mounted() {
        this.isLoading = true;
        this.initMap();
        this.loadData()
           .then(() => {
                console.log('数据加载成功');
                this.filterEvents();
                this.initCharts();
                this.isLoading = false;
            })
           .catch(error => {
                console.error('全局捕获到错误:', error);
                this.loadError = error.message;
                this.isLoading = false;
            });
    },
    methods: {

        fixAndParseDailyInfo(input) {
  if (!input || typeof input !== 'string') return [];

  try {
    // 步骤 1: 替换非法括号为 JSON 合法的中括号
    let cleaned = input.replace(/\(/g, '[').replace(/\)/g, ']');

    // 步骤 2: 给属性名加双引号
    cleaned = cleaned.replace(/([a-zA-Z_][a-zA-Z0-9_]*):/g, '"$1":');

    // 步骤 3: 替换单引号为双引号
    cleaned = cleaned.replace(/'/g, '"');

    // 步骤 4: 移除多余的尾随逗号（只作用于数组结尾）
    cleaned = cleaned.replace(/,(?=\s*\])/g, '');

    // 步骤 5: 尝试解析
    return JSON.parse(cleaned);
  } catch (e) {
    console.error('无法修复并解析 daily_info:', e.message);
    return [];
  }
},
        async loadData() {
    try {
        const csvPath = '/data/eventually_heatwave.csv';
        console.log('尝试加载 CSV 文件:', csvPath);
        const csvResponse = await fetch(csvPath);
        if (!csvResponse.ok) {
            throw new Error(`CSV 文件加载失败，状态码: ${csvResponse.status}`);
        }
        const csvText = await csvResponse.text();
        const csvData = Papa.parse(csvText, { header: true }).data;

        const geojsonPath = '/data/final_heatwaves.geojson';
        console.log('尝试加载 GeoJSON 文件:', geojsonPath);
        const geojsonResponse = await fetch(geojsonPath);
        if (!geojsonResponse.ok) {
            throw new Error(`GeoJSON 文件加载失败，状态码: ${geojsonResponse.status}`);
        }
        const geojsonData = await geojsonResponse.json();

        this.allEvents = csvData.map(row => {
  const props = {};
  this.variableOptions.forEach(varName => {
    if (varName === 'start_date' || varName === 'end_date') {
      const dateStr = row[varName];
      props[varName] = dateStr ? new Date(dateStr.split('T')[0]) : null;
    } else if (['duration', 'max_anomaly', 'cumulative_anomaly', 'centroid_change_rate',
                'vortex_count', 'vortex_coverage_ratio', 'cumulative_salinity_anomaly',
                'mean_wind', 'wind_anomaly', 'mean_heat_flux', 'heat_flux_trend'].includes(varName)) {
      props[varName] = Number(row[varName]);
    } else if (varName === 'daily_info') {
      const rawData = row[varName];
      props[varName] = this.fixAndParseDailyInfo(rawData);
    } else {
      props[varName] = row[varName];
    }
  });
  return { properties: props };
});

// 处理GeoJSON中的日期
                this.geojsonFeatures = geojsonData.features.map(feature => {
                    if (feature.properties.start_date) {
                        feature.properties.start_date = new Date(feature.properties.start_date.split('T')[0]);
                    }
                    if (feature.properties.end_date) {
                        feature.properties.end_date = new Date(feature.properties.end_date.split('T')[0]);
                    }
                    return feature;
                });

                console.log('成功加载的事件:', this.allEvents);
                console.log('geojsonFeatures:', this.geojsonFeatures);
                ElMessage.success(`成功加载 ${this.allEvents.length} 个热浪事件`);
            } catch (error) {
                console.error('数据加载失败:', error);
                ElMessage.error('数据加载失败，请检查控制台获取详细信息');
                throw error;
            }
        },

        initMap() {
            try {
                this.map = L.map('map-container', {
                    center: [30, 140],
                    zoom: 4,
                    preferCanvas: true
                });

                L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                    attribution: '&copy; OpenStreetMap contributors'
                }).addTo(this.map);

                this.addGeoJsonLayer();
                console.log('地图初始化成功');
            } catch (error) {
                console.error('地图初始化失败:', error);
            }
        },

        addGeoJsonLayer() {
            if (this.geoJsonLayer) {
                this.map.removeLayer(this.geoJsonLayer);
            }

            console.log('geojsonFeatures:', this.geojsonFeatures);
            this.geoJsonLayer = L.geoJSON(this.geojsonFeatures, {
                style: (feature) => ({
                    color: this.getIntensityColor(feature.properties.cumulative_anomaly),
                    weight: 2,
                    opacity: 0.8,
                    fillOpacity: 0.2
                }),
                onEachFeature: (feature, layer) => {
                    layer.bindPopup(this.createPopupContent(feature.properties));
                    layer.on('click', (e) => {
                        e.originalEvent.stopPropagation();
                    });
                }
            }).addTo(this.map);

            try {
                const bounds = this.geoJsonLayer.getBounds();
                if (bounds.isValid()) {
                    this.map.fitBounds(bounds, { padding: [50, 50] });
                } else {
                    console.warn('无效的边界，使用默认视图');
                    this.map.setView([30, 140], 4);
                }
            } catch (e) {
                console.warn('设置地图边界失败:', e);
                this.map.setView([30, 140], 4);
            }
        },

       // 在 filterEvents 方法中添加调试信息
filterEvents() {
   const [startDateStr, endDateStr] = this.timeRange;
            const startDate = new Date(startDateStr);
            const endDate = new Date(endDateStr);
    this.filteredEvents = this.allEvents.filter(event => {
        const props = event.properties;
        const eventStartDate = props.start_date;
                const eventEndDate = props.end_date;

                // 添加日期有效性检查
                if (!(eventStartDate instanceof Date) || isNaN(eventStartDate.getTime()) ||
                    !(eventEndDate instanceof Date) || isNaN(eventEndDate.getTime())) {
                    console.warn('无效的日期:', props.event_id, eventStartDate, eventEndDate);
                    return false;
                }return (
             props.duration >= this.minDuration &&
                    props.cumulative_anomaly >= this.minCumulativeIntensity &&
                    props.max_anomaly >= this.minMaxIntensity &&
                    eventStartDate >= startDate &&
                    eventEndDate <= endDate
        );
    });
    this.renderEvents();
     console.log('过滤后事件:', this.filteredEvents);

        },

        renderEvents() {
            if (this.geoJsonLayer) {
                this.map.removeLayer(this.geoJsonLayer);
            }
            if (this.heatLayer) {
                this.map.removeLayer(this.heatLayer);
            }

            if (this.showHeatmap) {
                this.renderHeatmap();
            } else {
                this.addGeoJsonLayer();
            }
        },

        renderHeatmap() {
    const heatData = this.filteredEvents.flatMap(event => {
        const props = event.properties;
        let dailyInfo = [];

        try {
            if (typeof props.daily_info === 'string') {
                dailyInfo = JSON.parse(props.daily_info);
            } else if (Array.isArray(props.daily_info)) {
                dailyInfo = props.daily_info;
            }
        } catch (e) {
            console.warn('daily_info 解析失败:', props.event_id, e);
        }

        const intensity = props.cumulative_anomaly;

        return dailyInfo.map(day => {
            const weight = this.heatmapType === 'intensity'
                ? day.intensity / intensity * 2
                : 1;

            return [day.centroid?.lat ?? 0, day.centroid?.lon ?? 0, weight];
        });
    });

    if (this.map && heatData.length > 0) {
        if (this.heatLayer) {
            this.map.removeLayer(this.heatLayer);
        }
        this.heatLayer = L.heatLayer(heatData, {
            radius: this.heatmapRadius,
            blur: 15,
            gradient: this.heatmapGradient,
            maxZoom: 17
        }).addTo(this.map);
    } else {
        console.warn('无有效热力点数据');
    }
},

        toggleHeatmap() {
            this.showHeatmap =!this.showHeatmap;
            this.renderEvents();
        },

        updateHeatmap() {
            if (this.showHeatmap) {
                this.renderHeatmap();
            }
        },

        getIntensityColor(intensity) {
            const range = this.intensityRanges.find(r =>
                intensity >= r.min && intensity < r.max
            );
            return range? range.color : '#999';
        },

        safeGetNumber(value, fixed = 1) {
            return (value!== undefined && value!== null)? value.toFixed(fixed) : 'N/A';
        },

        createPopupContent(props) {
            console.log('传入的 props 对象:', props);
            const { event_id, start_date, end_date, duration, max_anomaly, cumulative_anomaly,
                    mean_wind, wind_anomaly, mean_heat_flux, heat_flux_trend,
                    vortex_count, vortex_coverage_ratio, cumulative_salinity_anomaly, centroid_change_rate } = props;

              const formatDate = (date) => {
                if (!date || !(date instanceof Date) || isNaN(date.getTime())) {
                    return '未知日期';
                }
                return date.toLocaleDateString();
            };
           return `
                <div class="event-popup">
                    <h4>热浪事件 #${event_id || '未知'}</h4>
                    <div class="popup-grid">
                        <div>日期:</div><div>${formatDate(start_date)} - ${formatDate(end_date)}</div>
                        <div>持续时间:</div><div>${this.safeGetNumber(duration, 0)} 天</div>
                        <div>最大异常:</div><div>${this.safeGetNumber(max_anomaly)}°C</div>
                        <div>累计异常:</div><div>${this.safeGetNumber(cumulative_anomaly)}°C·天</div>
                        <div>平均风速:</div><div>${this.safeGetNumber(mean_wind)} m/s</div>
                        <div>风速异常:</div><div>${this.safeGetNumber(wind_anomaly)} m/s</div>
                        <div>平均热通量:</div><div>${this.safeGetNumber(mean_heat_flux)} W/m²</div>
                        <div>热通量趋势:</div><div>${this.safeGetNumber(heat_flux_trend)} W/m²·天</div>
                        <div>涡旋频次:</div><div>${this.safeGetNumber(vortex_count, 0)}</div>
                        <div>涡旋覆盖率:</div><div>${this.safeGetNumber(vortex_coverage_ratio * 100, 1)}%</div>
                        <div>累计盐度异常:</div><div>${this.safeGetNumber(cumulative_salinity_anomaly)} psu·天</div>
                        <div>质心变化率:</div><div>${this.safeGetNumber(centroid_change_rate)} km/天</div>
                    </div>
                </div>
            `;
        },

        initCharts() {
            try {
                this.correlationChart = echarts.init(this.$refs.correlationChart);
                this.parallelChart = echarts.init(this.$refs.parallelChart);
                this.contributionChart = echarts.init(this.$refs.contributionChart);
                this.radarChart = echarts.init(this.$refs.radarChart);
                this.chordChart = echarts.init(this.$refs.chordChart);
                this.roseChart = echarts.init(this.$refs.roseChart);
                this.updateCharts();
                console.log('图表初始化成功');
            } catch (error) {
                console.error('图表初始化失败:', error);
            }
        },

        updateCharts() {
            this.updateCorrelationChart();
            this.updateParallelChart();
            this.updateContributionChart();
            this.updateRadarChart();
            this.updateChordChart();
            this.updateRoseChart();
        },

        updateCorrelationChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const correlationMatrix = [];
            const numVariables = this.selectedVariables.length;
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
                    formatter: (params) => `${this.selectedVariables[params.data[0]]} 和 ${this.selectedVariables[params.data[1]]} 的相关性: ${params.data[2].toFixed(2)}`
                },
                xAxis: {
                    type: 'category',
                    data: this.selectedVariables
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
                    left: 'center',
                    bottom: '15%',
                    inRange: {
                        color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
                    }
                },
                series: [
                    {
                        name: '相关性',
                        type: 'heatmap',
                        data: correlationMatrix.map((row, i) =>
                            row.map((value, j) => [i, j, value])
                        ).flat(),
                        label: {
                            show: true,
                            formatter: (params) => params.value.toFixed(2)
                        }
                    }
                ]
            };

            this.correlationChart.setOption(option);
        },

        calculateCorrelation(x, y) {
            const n = x.length;
            if (n === 0) return 0;
            const sumX = x.reduce((acc, val) => acc + val, 0);
            const sumY = y.reduce((acc, val) => acc + val, 0);
            const sumXY = x.reduce((acc, val, i) => acc + val * y[i], 0);
            const sumX2 = x.reduce((acc, val) => acc + val * val, 0);
            const sumY2 = y.reduce((acc, val) => acc + val * val, 0);
            const numerator = n * sumXY - sumX * sumY;
            const denominator = Math.sqrt((n * sumX2 - sumX * sumX) * (n * sumY2 - sumY * sumY));
            return denominator === 0? 0 : numerator / denominator;
        },

        updateParallelChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const dimensions = this.selectedVariables.map(varName => ({
                name: varName
            }));

            const option = {
                parallel: {
                    dimensions: dimensions,
                    axisExpandable: true,
                    axisExpandCenter: 0,
                    axisExpandCount: 3,
                    axisExpandWidth: 50,
                    axisExpandTriggerOn: 'click'
                },
                series: [
                    {
                        name: '平行坐标图',
                        type: 'parallel',
                        data: data
                    }
                ]
            };

            this.parallelChart.setOption(option);
        },

        updateContributionChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const sumData = data.reduce((acc, row) => {
                row.forEach((val, i) => {
                    acc[i] = (acc[i] || 0) + val;
                });
                return acc;
            }, []);
            const total = sumData.reduce((acc, val) => acc + val, 0);
            const contributionData = sumData.map(val => (val / total) * 100);

            const option = {
                tooltip: {
                    trigger: 'item',
                    formatter: '{a} <br/>{b}: {c}%'
                },
                xAxis: {
                    type: 'category',
                    data: this.selectedVariables
                },
                yAxis: {
                    type: 'value',
                    name: '贡献百分比'
                },
                series: [
                    {
                        name: '因子贡献',
                        type: 'bar',
                        data: contributionData
                    }
                ]
            };

            this.contributionChart.setOption(option);
        },

        updateRadarChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const indicators = this.selectedVariables.map(varName => ({
                name: varName,
                max: Math.max(...data.map(row => row[this.selectedVariables.indexOf(varName)]))
            }));

            const option = {
                radar: {
                    indicator: indicators
                },
                series: [
                    {
                        name: '多变量指纹雷达',
                        type: 'radar',
                        data: data
                    }
                ]
            };

            this.radarChart.setOption(option);
        },

        updateChordChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const correlationMatrix = [];
            const numVariables = this.selectedVariables.length;
            for (let i = 0; i < numVariables; i++) {
                correlationMatrix[i] = [];
                for (let j = 0; j < numVariables; j++) {
                    const x = data.map(row => row[i]);
                    const y = data.map(row => row[j]);
                    correlationMatrix[i][j] = this.calculateCorrelation(x, y);
                }
            }

            const nodes = this.selectedVariables.map(varName => ({
                name: varName
            }));
            const links = [];
            for (let i = 0; i < numVariables; i++) {
                for (let j = i + 1; j < numVariables; j++) {
                    links.push({
                        source: i,
                        target: j,
                        value: correlationMatrix[i][j]
                    });
                }
            }

            const option = {
                tooltip: {
                    trigger: 'item',
                    formatter: (params) => {
                        if (params.dataType === 'edge') {
                            return `${nodes[params.data.source].name} 和 ${nodes[params.data.target].name} 的关联值: ${params.data.value.toFixed(2)}`;
                        }
                        return params.name;
                    }
                },
                series: [
                    {
                        type: 'chord',
                        data: nodes,
                        links: links
                    }
                ]
            };

            this.chordChart.setOption(option);
        },

        updateRoseChart() {
            const data = this.filteredEvents.map(event =>
                this.selectedVariables.map(varName => event.properties[varName])
            );
            const sumData = data.reduce((acc, row) => {
                row.forEach((val, i) => {
                    acc[i] = (acc[i] || 0) + val;
                });
                return acc;
            }, []);
            const total = sumData.reduce((acc, val) => acc + val, 0);
            const roseData = sumData.map((val, i) => ({
                value: (val / total) * 100,
                name: this.selectedVariables[i]
            }));

            const option = {
                tooltip: {
                    trigger: 'item',
                    formatter: '{a} <br/>{b}: {c}%'
                },
                legend: {
                    orient: 'vertical',
                    left: 'left'
                },
                polar: {
                    center: ['50%', '54%']
                },
                series: [
                    {
                        name: '驱动因子占比',
                        type: 'pie',
                        radius: [20, 140],
                        center: ['50%', '50%'],
                        roseType: 'area',
                        data: roseData
                    }
                ]
            };

            this.roseChart.setOption(option);
        }
    }
};
</script>

<style scoped>
.heatwave-dashboard {
    display: flex;
    height: 100vh;
}

.main-view {
    flex: 1;
    display: flex;
}

.map-view {
    flex: 3;
    position: relative;
}

#map-container {
    height: 100%;
}

.map-controls {
    position: absolute;
    top: 10px;
    left: 10px;
    z-index: 1000;
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.legend-container {
    background-color: rgba(255, 255, 255, 0.8);
    padding: 10px;
    border-radius: 5px;
}

.legend {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.gradient-bar {
    height: 20px;
    background: linear-gradient(to right, blue, cyan, lime, red);
}

.legend-labels {
    display: flex;
    justify-content: space-between;
}

.color-box {
    width: 20px;
    height: 20px;
    display: inline-block;
    margin-right: 5px;
}

.control-panel {
    flex: 1;
    padding: 10px;
    overflow-y: auto;
}

.slider-control {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 10px;
}

.loading-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(255, 255, 255, 0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px;
    z-index: 1001;
}

.error-message {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    padding: 10px;
    background-color: #f44336;
    color: white;
    text-align: center;
    z-index: 1001;
}

.event-popup {
    min-width: 200px;
}

.popup-grid {
    display: grid;
    grid-template-columns: 1fr 2fr;
    gap: 5px;
}
</style>