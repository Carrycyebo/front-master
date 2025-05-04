<template>
  <div class="heatwave-vis">
    <!-- 控制面板 -->
    <div class="control-panel">
      <div class="control-row">
        <!-- 热力图控制区域 -->
        <div class="heatmap-control">
          <el-tooltip content="基于事件密度生成热力图" placement="top">
            <el-switch
              v-model="showHeatmap"
              active-text="热力图"
              @change="toggleHeatmap"
            />
          </el-tooltip>
          
          <div v-if="showHeatmap" class="heatmap-options">
            <el-radio-group v-model="heatmapType" size="small">
              <el-radio-button value="density">事件密度</el-radio-button>
              <el-radio-button value="intensity">强度分布</el-radio-button>
            </el-radio-group>
            
            <el-slider
              v-model="heatmapRadius"
              :min="10"
              :max="50"
              :step="5"
              label="热力半径"
              @change="updateHeatmap"
            />
          </div>
        </div>

        <!-- 分析面板按钮 -->
        <el-button 
          @click="toggleAnalysisPanel"
          type="primary" 
          class="analysis-toggle-btn"
        >
          {{ showAnalysisPanel ? '隐藏分析' : '显示分析' }}
        </el-button>
      </div>

      <el-date-picker
        v-model="timeRange"
        type="daterange"
        range-separator="至"
        start-placeholder="开始日期"
        end-placeholder="结束日期"
        value-format="YYYY-MM-DD"
        @change="filterEvents"
      />
      
      <div class="slider-container">
        <span>最小持续时间：{{ minDuration }}天</span>
        <el-slider
          v-model="minDuration"
          :min="1"
          :max="90"
          :step="1"
          @change="filterEvents"
        />
      </div>
      
      <div class="slider-container">
        <span>最小累计强度：{{ minCumulativeIntensity }}</span>
        <el-slider
          v-model="minCumulativeIntensity"
          :min="0"
          :max="2000"
          :step="5"
          @change="filterEvents"
        />
      </div>
      
      <div class="slider-container">
        <span>最小最大强度：{{ minMaxIntensity }}</span>
        <el-slider
          v-model="minMaxIntensity"
          :min="0"
          :max="50"
          :step="1"
          @change="filterEvents"
        />
      </div>
    </div>

    <!-- 地图容器 -->
    <div id="map-container"></div>

    <!-- 分析面板容器 -->
    <transition name="slide-fade">
      <div v-if="showAnalysisPanel" class="analysis-panel">
        <CorrelationAnalysis ref="analysisPanel" />
      </div>
    </transition>

    <!-- 图例容器 -->
    <div class="legend-container">
      <!-- 强度图例只在非热力图模式显示 -->
      <div v-if="!showHeatmap" class="legend intensity-legend">
        <h4>事件强度</h4>
        <div v-for="(item, index) in intensityRanges" :key="index" class="legend-item">
          <div class="color-box" :style="{ backgroundColor: item.color }"></div>
          <span>{{ item.label }}</span>
        </div>
      </div>

      <!-- 热力图例单独显示 -->
      <div v-if="showHeatmap" class="legend heat-legend">
        <h4>{{ heatLegendTitle }}</h4>
        <div class="legend-scale">
          <div class="gradient-bar"></div>
          <div class="scale-labels">
            <span>低</span>
            <span>高</span>
          </div>
        </div>
      </div>
    </div>
    
    <CacheMonitor :cacheManager="cacheManager" />
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import 'leaflet.heat/dist/leaflet-heat.js';
import CorrelationAnalysis from './CorrelationAnalysis.vue';


// 或如果使用统一导出方式：
import CacheManager,{HybridStrategy} from '@/lib/cacheManager';
import CacheMonitor from '@/components/CacheMonitor';

const GeoJSONFixer = {
  preprocess(str) {
    return str
      .replace(/'/g, '"')
      .replace(/None/g, 'null')
      .replace(/(\w+):/g, '"$1":')
      .replace(/(\d+\.\d+),\s*]/g, '$1]')
      .replace(/,(\s*[\]}])/g, '$1')
      .replace(/$/g, '[').replace(/$/g, ']')
      .replace(/(\d+),]/g, '$1]');
  },

  safeParse(str) {
    try {
      return JSON.parse(str);
    } catch (e) {
      const fixed = str
        .replace(/($$[^[]+?)([,$$]*)$/g, '$1]')
        .replace(/},]/g, '}]');
      return JSON.parse(fixed);
    }
  }
};

const INTENSITY_COLORS = {
  low: '#4CAF50',
  medium: '#FFC107',
  high: '#F44336'
};

export default {
  
  components: { CacheMonitor, CorrelationAnalysis },
  name: 'HeatwaveVisualization',
  data() {
    const defaultStart = new Date(2020, 5, 1);
    const defaultEnd = new Date(2020, 7, 31);
    return {
      showAnalysisPanel: false,
      cacheManager: new CacheManager(
        'HeatwaveCache',  // 缓存名称
        1,// IndexedDB 版本
        new HybridStrategy(100, 3600000 * 24 * 7)
      ),
         // 新增热力图相关数据
      showHeatmap: false,
      heatLayer: null,
      heatmapType: 'density', // density｜intensity
      heatmapRadius: 25,
      heatmapGradient: {
        0.4: 'blue',
        0.6: 'lime',
        0.9: 'red'
      },

      currentHighlight: null,
      isMapInitialized: false,
      map: null,
      geoJsonLayer: null,
      timeRange: [
        defaultStart.toISOString().split('T')[0],
        defaultEnd.toISOString().split('T')[0]
      ],
      minDuration: 3,
      minCumulativeIntensity: 100,
      minMaxIntensity: 1,
      allEvents: [],
      filteredEvents: [],
      intensityRanges: [
        { min: 0, max: 100, color: INTENSITY_COLORS.low, label: '低强度 (<100)' },
        { min: 100, max: 300, color: INTENSITY_COLORS.medium, label: '中等强度 (100-300)' },
        { min: 300, max: Infinity, color: INTENSITY_COLORS.high, label: '高强度 (>300)' }
      ]
    };
  },
  
  async mounted() {
    await this.cacheManager.init();
    try {
      await this.initHeatPlugin();
        // 先初始化缓存管理器
      await this.cacheManager.init(); 
      this.initMap();
      await this.loadData();
      this.filterEvents();
    } catch (error) {
      console.error('初始化失败:', error);
    }
  },
  computed: {
    heatLegendTitle() {
      return this.heatmapType === 'density' 
        ? '事件密度热力分布' 
        : '强度分布热力图';
    }
  },
  methods: {
    toggleAnalysisPanel() {
      this.showAnalysisPanel = !this.showAnalysisPanel;
      // 面板显示后重新渲染图表
      if (this.showAnalysisPanel) {
        this.$nextTick(() => {
          this.$refs.analysisPanel.initCurrentChart();
        });
      }
    },
    logOperation(type, params) {
      const entry = {
        type,
        timestamp: new Date().toISOString(),
        params: JSON.parse(JSON.stringify(params)),
        component: 'Taks2'
      };
      
      this.$store.commit('history/ADD_OPERATION', entry);
    },

    async initHeatPlugin() {
      if (typeof window !== 'undefined') {
        await import('leaflet.heat/dist/leaflet-heat.js');
      }
    },
    // 生成热力图数据点
    generateHeatmapData() {
      return this.filteredEvents.flatMap(event => {
        const baseIntensity = event.properties.cumulative_intensity;
        return event.properties.daily_info.map(day => {
          const weight = this.heatmapType === 'intensity' 
            ? day.intensity / baseIntensity
            : 1;
            
          return [
            day.centroid.lat,
            day.centroid.lon,
            weight * 0.5 // 强度权重系数
          ];
        });
      });
    },
    
     // 切换热力图显示
    // 修改后的切换热力图方法
    toggleHeatmap() {
      this.logOperation('开启热力图', { 
        status: this.showHeatmap 
      });
      if (this.showHeatmap) {
        // 移除原有热浪多边形
        if (this.geoJsonLayer) {
          this.map.removeLayer(this.geoJsonLayer);
          this.geoJsonLayer = null;
        }

        const points = this.generateHeatmapData();
        this.heatLayer = L.heatLayer(points, {
          radius: this.heatmapRadius,
          blur: 15,
          gradient: this.heatmapGradient,
          maxZoom: 9
        }).addTo(this.map);
        this.addHeatLegend();
      } else {
        // 关闭热力图时重新渲染多边形
        this.heatLayer?.remove();
        this.heatLayer = null;
        this.removeHeatLegend();
        this.renderEvents(); // 重新显示热浪多边形
      }
    },

     // 更新热力图参数
     updateHeatmap() {
      this.logOperation('HEATMAP_CONFIG', {
        radius: this.heatmapRadius,
        type: this.heatmapType
      });
      if (this.heatLayer) {
        this.heatLayer.setOptions({
          radius: this.heatmapRadius,
          gradient: this.heatmapGradient
        });
        this.heatLayer.redraw();
      }
    },

    // 修改addHeatLegend方法
    addHeatLegend() {
      this.removeHeatLegend(); // 先移除旧图例
      
      this.heatLegend = L.control({ position: 'bottomleft' });
      
      this.heatLegend.onAdd = () => {
        const div = L.DomUtil.create('div', 'heat-legend');
        div.innerHTML = `
          <h4>${this.heatLegendTitle}</h4>
          <div class="legend-scale">
            <div style="background:linear-gradient(to right, 
              ${this.heatmapGradient[0.4]}, 
              ${this.heatmapGradient[0.6]}, 
              ${this.heatmapGradient[0.9]})"></div>
            <span>低</span><span>高</span>
          </div>
        `;
        return div;
      };
      
      this.heatLegend.addTo(this.map);
    },
    // 添加watch监听
    watch: {
      heatmapType() {
        if (this.showHeatmap) {
          this.addHeatLegend(); // 切换类型时更新图例
        }
      }
    },
    // 移除图例
    removeHeatLegend() {
      this.heatLegend?.remove();
    },
    async loadData() {
      const tileKey = 'final_heatwaves'; // 缓存的键
      try {
      
        let data = await this.cacheManager.get(tileKey);
          // 增加缓存有效性检查
        if (data && !this.validateCache(data)) {
          console.log('缓存数据格式已过期');
          await this.cacheManager.db.delete(this.storeName, tileKey); // 删除无效缓存
          data = null;
        }
        if (!data) {
          console.log('缓存未命中，从data中加载数据...');
          const response = await fetch('/data/final_heatwaves.geojson');
          data = await response.json();

          // 将数据存储到缓存中
          await this.cacheManager.set(tileKey, data);
        } else {
          
          console.log('从缓存加载数据...');
        }

        this.allEvents = data.features.map(feature => {
          if (!feature?.properties) return null;
          const props = feature.properties;

          let dailyInfo = [];
          try {
            const rawStr = props.daily_info || '[]';
            const processed = GeoJSONFixer.preprocess(rawStr);
            const chunkPattern = /\{"date".*?\}(?=\s*,?\s*\{)/g;
            const chunks = processed.match(chunkPattern) || [];
            
            dailyInfo = chunks.map(chunk => {
              try {
                const completeChunk = chunk.replace(/(\])*$/, ']');
                return GeoJSONFixer.safeParse(completeChunk + ']');
              } catch (e) {
                const dateMatch = chunk.match(/"date": "(\d{4}-\d{2}-\d{2})"/);
                const centroidMatch = chunk.match(/"centroid": \{"lon": ([\d.]+), "lat": ([\d.]+)\}/);
                
                return dateMatch && centroidMatch ? {
                  date: dateMatch[1],
                  centroid: {
                    lon: parseFloat(centroidMatch[1]),
                    lat: parseFloat(centroidMatch[2])
                  }
                } : null;
              }
            }).filter(Boolean);

          } catch (e) {
            console.warn(`事件 ${props.event_id} daily_info 解析失败，使用备用方案:`, e);
            dailyInfo = this.fallbackParse(props.daily_info);
          }

          const cumulativeIntensity = Number(props.cumulative_anomaly) || 0;
          const maxIntensity = Number(props.max_anomaly) || 0;

          return {
            ...feature,
            properties: {
              event_id: props.event_id,
              start_date: new Date(props.start_date),
              duration: Number(props.duration) || 0,
              daily_info: dailyInfo,
              cumulative_intensity: cumulativeIntensity,
              max_intensity: maxIntensity,
              centroid: dailyInfo.length > 0 ? dailyInfo[0].centroid : null
            }
          };
        }).filter(event => 
          event?.properties?.daily_info?.length > 0
        );

        console.log('有效事件数:', this.allEvents.length);
        this.initMap();
        this.filterEvents();

      } catch (error) {
        console.error('数据加载失败:', error);
      }
    },
    validateCache(data) {
      return Object.prototype.hasOwnProperty.call(
        data?.features?.[0]?.properties || {}, 
        'daily_info'
      );
    },
    async _performCleanup() {
      const tx = this.db.transaction(this.storeName, 'readwrite');
      await this.strategy.cleanup(tx.objectStore(this.storeName));
    },
    fallbackParse(str) {
      const results = [];
      const pattern = /"date": "(\d{4}-\d{2}-\d{2})".*?"lon": ([\d.]+).*?"lat": ([\d.]+)/g;
      
      let match;
      while ((match = pattern.exec(str)) !== null) {
        results.push({
          date: match[1],
          centroid: {
            lon: parseFloat(match[2]),
            lat: parseFloat(match[3])
          }
        });
      }
      return results;
    },
 
    initMap() {
         // 在Task2.vue的initMap方法顶部添加
      function patchLeafletCanvas() {
        const originalInit = L.Canvas.prototype._initContainer;
        L.Canvas.prototype._initContainer = function () {
          originalInit.call(this);
          if (this._container && this._container.getContext) {
            this._ctx = this._container.getContext('2d', { 
              willReadFrequently: true 
            });
          }
        };
      }
      if (this.map) return;
      patchLeafletCanvas(); // 添加补丁
      const container = document.getElementById('map-container');
      container.style.width = '100%';
      container._leaflet_id = null;
      
      this.map = L.map('map-container', {
        zoomControl: false,
        preferCanvas: true,
        dragging: true,
      }).setView([30, 140], 4);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; OpenStreetMap contributors'
      }).addTo(this.map);
      
      this.map.on('touchstart', (e) => {
        if (e.originalEvent.touches.length === 1) {
          this.map.dragging.enable();
        }
      });
      
      L.control.zoom({ position: 'topright' }).addTo(this.map);
      this.isMapInitialized = true;
    },
    
    renderEvents() {
      if (this.showHeatmap) return; // 热力图模式下不渲染多边形
      if (!this.map || typeof this.map.addLayer !== 'function') {
        console.error('地图实例异常');
        return;
      }

      if (this.geoJsonLayer) {
        this.geoJsonLayer.remove();
        this.geoJsonLayer = null;
      }

      if (!this.filteredEvents?.length) return;

      this.geoJsonLayer = L.geoJSON(this.filteredEvents, {
        coordsToLatLng: (coords) => {
          if (Array.isArray(coords) && coords.length >= 2) {
            return L.latLng(coords[1], coords[0]);
          }
          return L.latLng(0, 0);
        },
        style: (feature) => ({
          color: this.getIntensityColor(feature.properties.cumulative_intensity),
          weight: 2,
          opacity: 0.8,
          fillOpacity: 0.2
        }),
        onEachFeature: (feature, layer) => {
          if (feature.properties) {
            layer.bindPopup(this.createPopupContent(feature.properties));
          }
          
          const originalStyle = {
            color: this.getIntensityColor(feature.properties.cumulative_intensity),
            weight: 2,
            opacity: 0.8
          };
          layer.setStyle(originalStyle);
          layer.originalStyle = originalStyle;

          layer.on('click', () => {
            if (this.currentHighlight) {
              this.currentHighlight.setStyle(this.currentHighlight.originalStyle);
            }
            
            layer.setStyle({
              color: layer.originalStyle.color,
              weight: 5,
              opacity: 1
            });
            
            this.currentHighlight = layer;
          });
          
          layer.on('mouseover', () => {
            layer.setStyle({
              color: layer.originalStyle.color,
              weight: 3,
              opacity: 1
            });
          });
          
          layer.on('mouseout', () => {
            if (this.currentHighlight !== layer) {
              layer.setStyle(layer.originalStyle);
            }
          });
        }
      }).addTo(this.map);

      this.safeFitBounds();
    },

    filterEvents() {
     
      this.$store.commit('history/ADD_OPERATION', {
        type: 'FILTER_UPDATE',
        timestamp: new Date().toISOString(),
        details: `热浪强度可视化中过滤条件更新: 
          时间范围 ${this.timeRange.join(' ~ ')}
          最小持续时间 ${this.minDuration}天
          最小累计强度 ${this.minCumulativeIntensity}
          最大强度阈值 ${this.minMaxIntensity}`
      });
       
        if (!this.allEvents.length) return;

        const [startDate, endDate] = this.timeRange.map(d => new Date(d));
        
        this.filteredEvents = this.allEvents.filter(event => {
          const props = event.properties;
          
          const duration = Number(props.duration) || 0;
          if (duration < this.minDuration) return false;
          if (props.cumulative_intensity < this.minCumulativeIntensity) return false;
          if (props.max_intensity < this.minMaxIntensity) return false;

          const eventStart = new Date(props.start_date);
          const eventEnd = new Date(eventStart);
          eventEnd.setDate(eventStart.getDate() + duration);
          
          return (
            eventEnd >= startDate && 
            eventStart <= endDate &&
            !isNaN(eventStart.getTime())
          );
        });

         // 热力图模式下不渲染多边形
        if (!this.showHeatmap) {
          this.renderEvents();
        }
        // 热力图模式下需要更新热力数据
        else if (this.heatLayer) {
          const points = this.generateHeatmapData();
          this.heatLayer.setLatLngs(points);
          this.heatLayer.redraw();
        }
    },

    getIntensityColor(intensity) {
      return this.intensityRanges.find(range => 
        intensity >= range.min && intensity < range.max
      )?.color || '#999';
    },

    createPopupContent(properties) {
      const centroid = properties.centroid;
      return `
      <div class="event-popup">
        <h4>事件 #${properties.event_id}</h4>
        <div class="popup-grid">
          <div>📅 开始日期:</div>
          <div>${properties.start_date.toLocaleDateString()}</div>
          <div>⏳ 持续时间:</div>
          <div>${properties.duration} 天</div>
          <div>🔥 累计强度:</div>
          <div>${properties.cumulative_intensity.toFixed(1)}</div>
          <div>💥 最大强度:</div>
          <div>${properties.max_intensity.toFixed(1)}</div>
          <div>📍 初始位置:</div>
          <div>${ centroid ? `${centroid.lat.toFixed(2)}°N, ${centroid.lon.toFixed(2)}°E` : '未知' }</div>
        </div>
      </div>
    `;
    },

    safeFitBounds() {
      if (this.geoJsonLayer) {
        const bounds = this.geoJsonLayer.getBounds();
        if (bounds.isValid()) {
          this.map.fitBounds(bounds, { padding: [30, 30] });
        }
      }
    }
  },
  
  beforeUnmount() {
    if (this.map) {
      this.map.eachLayer(layer => layer.remove());
      this.map.remove();
      this.map = null;
    }
    this.isMapInitialized = false;
  }
};
</script>

<style scoped>
.heatwave-vis {
  position: relative;
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 控制面板样式 */
.control-panel {
  padding: 12px;
  background: #f5f7fa;
  border-bottom: 1px solid #e6e6e6;
  z-index: 1000;
}

.control-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
  flex-wrap: wrap;
  gap: 12px;
}

.heatmap-control {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.heatmap-options {
  display: flex;
  flex-direction: column;
  gap: 8px;
  background: #fff;
  padding: 8px;
  border-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

.analysis-toggle-btn {
  margin-left: auto;
}

/* 地图容器 */
#map-container {
  flex: 1;
  width: 100%;
  position: relative;
}

/* 分析面板样式 */
.analysis-panel {
  position: absolute;
  top: 60px;
  right: 20px;
  width: 380px;
  height: calc(100% - 80px);
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  z-index: 1001;
  overflow: hidden;
}

/* 图例容器 */
.legend-container {
  position: absolute;
  bottom: 20px;
  right: 20px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.legend {
  background: rgba(255, 255, 255, 0.9);
  padding: 10px;
  border-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

.legend h4 {
  margin: 0 0 8px 0;
  font-size: 12px;
  color: #333;
}

.legend-item {
  display: flex;
  align-items: center;
  margin: 4px 0;
}

.color-box {
  width: 16px;
  height: 16px;
  margin-right: 8px;
  border: 1px solid #ddd;
}

.legend-scale {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.gradient-bar {
  height: 12px;
  width: 100%;
  background: linear-gradient(to right, #2b83ba, #abdda4, #ffffbf, #fdae61, #d7191c);
  border-radius: 2px;
}

.scale-labels {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: #666;
}

/* 过渡动画 */
.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.3s ease;
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateX(20px);
  opacity: 0;
}

/* 滑块容器样式 */
.slider-container {
  margin: 12px 0;
}

.slider-container span {
  display: block;
  margin-bottom: 4px;
  font-size: 12px;
  color: #666;
}

/* 响应式调整 */
@media (max-width: 768px) {
  .control-row {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .analysis-panel {
    width: 90%;
    right: 5%;
    height: 50%;
  }
  
  .legend-container {
    bottom: 10px;
    right: 10px;
    max-width: 40%;
  }
}
</style>