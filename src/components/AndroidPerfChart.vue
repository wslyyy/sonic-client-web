<script setup>
/*
 *   sonic-client-web  Front end of Sonic cloud real machine platform.
 *   Copyright (C) 2022 SonicCloudOrg
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU Affero General Public License as published
 *   by the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 *   GNU Affero General Public License for more details.
 *
 *   You should have received a copy of the GNU Affero General Public License
 *   along with this program.  If not, see <https://www.gnu.org/licenses/>.
 */
import moment from 'moment/moment';
import { useI18n } from 'vue-i18n';
import * as echarts from 'echarts/core';
import 'vue3-video-play/dist/style.css';
import { videoPlay } from 'vue3-video-play';
import {
  TitleComponent,
  ToolboxComponent,
  TooltipComponent,
  GridComponent,
  LegendComponent,
  DataZoomComponent,
} from 'echarts/components';
import { LineChart } from 'echarts/charts';
import { CanvasRenderer } from 'echarts/renderers';
import { nextTick, ref, reactive, watch } from 'vue';
import { subMenuProps } from 'element-plus';

echarts.use([
  DataZoomComponent,
  ToolboxComponent,
  GridComponent,
  LegendComponent,
  LineChart,
  CanvasRenderer,
  TitleComponent,
  TooltipComponent,
]);

const { t: $t } = useI18n();
const props = defineProps({
  rid: String,
  cid: Number,
  did: Number,
  sysCpu: Array,
  sysMem: Array,
  sysNetwork: Array,
  procCpu: Array,
  procMem: Array,
  procFps: Array,
  procThread: Array,
});
const currentTime = ref(0);
const recordUrl = ref('');
const videoPlayRef = ref();
var Media
const videoOptions = reactive({
  width: '100%',
  height: 'auto',
  controlBtns: ['audioTrack', 'quality', 'volume', 'fullScreen', 'speedRate'],
  currentTime: currentTime.value,
});

watch(currentTime, () => {
  setTime(currentTime)
}, {deep:true})

function calculateTimeDifference(time1, time2) {
  var timeArr1 = time1.split(":");
  var timeArr2 = time2.split(":");
  
  var hours1 = parseInt(timeArr1[0]);
  var minutes1 = parseInt(timeArr1[1]);
  var seconds1 = parseInt(timeArr1[2]);
  
  var hours2 = parseInt(timeArr2[0]);
  var minutes2 = parseInt(timeArr2[1]);
  var seconds2 = parseInt(timeArr2[2]);
  
  var totalSeconds1 = hours1 * 3600 + minutes1 * 60 + seconds1;
  var totalSeconds2 = hours2 * 3600 + minutes2 * 60 + seconds2;
  
  return Math.abs(totalSeconds1 - totalSeconds2);
}

const setTime = (time) => {
  //videoPlayRef.value.pause()
  Media = document.getElementById("media")
  Media.currentTime = time.value
  //videoOptions.currentTime = time.value
  videoPlayRef.value.play()
}
recordUrl.value = "http://192.168.88.3:3333/server/api/folder/recordFiles/666.mp4"
//currentTime.value = 40;

const getAnalyze = () => {
  const result = [];
  const average = arr => arr.length>0 ? Math.round(arr.reduce((acc, val) => acc + val, 0) / arr.length): 0;
  const sum = arr => Math.round(arr.reduce((acc, val) => acc + val, 0));
  const tx = [];
  const rx = [];
  props.sysNetwork.map((obj,index) => {
        if (index !== 0) {
          tx.push((obj['wlan0'].tx - props.sysNetwork[index-1]['wlan0'].tx) / 1024);
          rx.push((obj['wlan0'].rx - props.sysNetwork[index-1]['wlan0'].rx) / 1024);
        } else {
          tx.push(0);
          rx.push(0);
        }
  });
  const syscpu = props.sysCpu.map((obj) => {
            if (obj.cpu) {
              return obj.cpu['cpuUsage'];
            }
            return 0;
  });

  const sysmem = props.sysMem.map((obj) => {
          return obj.memUsage / (1024 * 1024);
  });

  result.push({
    NetworkTX: sum(tx),
    NetworkRX: sum(rx),
    SysCPU: average(syscpu),
    SysMEM: average(sysmem),
  });
  return result
}

const getNetworkTimeStamp = () => {
  const result = [];
  if (props.sysNetwork.length > 0) {
    for (const i in props.sysNetwork[0]) {
      if (i) {
        props.sysNetwork.map((obj) => {
          result.push(moment(new Date(obj[i].timeStamp)).format('HH:mm:ss'));
        });
        break;
      }
    }
  }
  return result;
};

const getNetworkDataGroup = () => {
  const result = [];
  const res = [];
  if (props.sysNetwork.length > 0) {
    for (const i in props.sysNetwork[0]) {
      const tx = [];
      const rx = [];
      props.sysNetwork.map((obj,index) => {
        if (index !== 0) {
          tx.push((obj[i].tx - props.sysNetwork[index-1][i].tx) / 1024)
          rx.push((obj[i].rx - props.sysNetwork[index-1][i].rx) / 1024)
        } else {
          tx.push(0)
          rx.push(0)
        }
      });
      result.push({
        type: 'line',
        name: `${i}_tx`,
        data: tx,
        showSymbol: false,
        boundaryGap: false,
      });
      result.push({
        type: 'line',
        name: `${i}_rx`,
        data: rx,
        showSymbol: false,
        boundaryGap: false,
      });
    }
  }
  result.map((obj) => {
        if (obj.name === 'wlan0_tx' || obj.name === 'wlan0_rx') {
          res.push(obj)
        }
      });
  console.log(getAnalyze())
  return res;
};
const getCpuDataGroup = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0].cpu) {
      if (i !== 'timeStamp') {
        result.push({
          type: 'line',
          name: i,
          data: props.sysCpu.map((obj) => {
            if (obj.cpu) {
              return obj.cpu[i];
            }
            return 0;
          }),
          showSymbol: false,
          areaStyle: {},
          boundaryGap: false,
        });
      }
    }
  }
  return result;
};
const getwsyCpuDataGroup = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0].cpu) {
      if (i === 'cpuUsage') {
        result.push({
          type: 'line',
          name: i,
          data: props.sysCpu.map((obj) => {
            if (obj.cpu) {
              return obj.cpu[i];
            }
            return 0;
          }),
          showSymbol: false,
          areaStyle: {},
          boundaryGap: false,
        });
      }
    }
  }
  return result;
};
const getCpuDataLegend = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0].cpu) {
      if (i !== 'timeStamp') {
        result.push(i);
      }
    }
  }
  return result;
};
const getwsyCpuDataLegend = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0].cpu) {
      if (i === 'cpuUsage') {
        result.push(i);
      }
    }
  }
  return result;
};
const getCpuLegend = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0]) {
      if (i !== 'cpu') {
        result.push(i);
      }
    }
  }
  return result;
};
const getCpuGroup = () => {
  const result = [];
  if (props.sysCpu.length > 0) {
    for (const i in props.sysCpu[0]) {
      if (i !== 'cpu') {
        result.push({
          type: 'line',
          name: i,
          data: props.sysCpu.map((obj) => {
            if (obj[i]) {
              return obj[i].cpuUsage;
            }
            return 0;
          }),
          showSymbol: false,
          areaStyle: {},
          boundaryGap: false,
        });
      }
    }
  }
  return result;
};
// analyze
/*
const printAnalyze = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `analyzeChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `analyzeChart`
      )
    );
  }
  chart.resize();
  const option = {
    title: {
      text: 'Analyze',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      position(pos, params, dom, rect, size) {
        const obj = { top: 60 };
        obj[['left', 'right'][+(pos[0] < size.viewSize[0] / 2)]] = 5;
        return obj;
      },
      valueFormatter: (value) => `${value.toFixed(3)} %`,
    },
    legend: {
      top: '8%',
      data: getwsyCpuDataLegend(),
    },
    grid: { top: '28%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.sysCpu.map((obj) => {
        return moment(new Date(obj.cpu.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.totalCpu')}(%)`, min: 0 }],
    series: getwsyCpuDataGroup(),
  };
  chart.setOption(option);
  chart.off("click");
  chart.on('click', function(params){
            // 这里设置视频跳转的秒数
            console.log(props.sysCpu[0].cpu.timeStamp);
            var start = moment(new Date(props.sysCpu[0].cpu.timeStamp)).format('HH:mm:ss');
            console.log(start);
            console.log(params.name);
            var difference = calculateTimeDifference(params.name, start);
            console.log(difference + " 秒");
            //var num = Math.floor(Math.random() * (30 - 10 + 1) + 10)
            currentTime.value = difference
            //videoOptions.currentTime = 10
            console.log(videoOptions)
        })
};
*/

const printSingleCpu = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysSingleCpuChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysSingleCpuChart`
      )
    );
  }
  chart.resize();
  const option = {
    title: {
      text: 'System Single-Core CPU',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      position(pos, params, dom, rect, size) {
        const obj = { top: 60 };
        obj[['left', 'right'][+(pos[0] < size.viewSize[0] / 2)]] = 5;
        return obj;
      },
      valueFormatter: (value) => `${value.toFixed(3)} %`,
    },
    legend: {
      top: '8%',
      data: getCpuLegend(),
    },
    grid: { top: '26%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.sysCpu.map((obj) => {
        return moment(new Date(obj.cpu.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.singleCpu')}(%)`, min: 0 }],
    series: getCpuGroup(),
  };
  chart.setOption(option);
};
const printCpu = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysCpuChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysCpuChart`
      )
    );
  }
  chart.resize();
  const option = {
    title: {
      text: 'System Total CPU',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      position(pos, params, dom, rect, size) {
        const obj = { top: 60 };
        obj[['left', 'right'][+(pos[0] < size.viewSize[0] / 2)]] = 5;
        return obj;
      },
      valueFormatter: (value) => `${value.toFixed(3)} %`,
    },
    legend: {
      top: '8%',
      data: getwsyCpuDataLegend(),
    },
    grid: { top: '28%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.sysCpu.map((obj) => {
        return moment(new Date(obj.cpu.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.totalCpu')}(%)`, min: 0 }],
    series: getwsyCpuDataGroup(),
  };
  chart.setOption(option);
  chart.off("click");
  chart.on('click', function(params){
            // 这里设置视频跳转的秒数
            console.log(props.sysCpu[0].cpu.timeStamp)
            console.log(params.name)
        })
};
const printMem = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysMemChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysMemChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#5470c6', '#91cc75', '#fac858', '#ee6666', '#73c0de', '#409EFF'],
    title: {
      text: 'System Memory',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      valueFormatter: (value) => `${value.toFixed(3)}`,
    },
    grid: { top: '30%', left: '20%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: [
        'Mem Usage',
      ],
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.sysMem.map((obj) => {
        return moment(new Date(obj.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.memUsage')}(MB)`, min: 0 }],
    series: [
      {
        name: 'Mem Usage',
        type: 'line',
        data: props.sysMem.map((obj) => {
          return obj.memUsage / (1024 * 1024);
        }),
        showSymbol: false,
        areaStyle: {},
        boundaryGap: false,
      },
    ],
  };
  chart.setOption(option);
};
const printProcFps = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysFpsChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysFpsChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#67C23A'],
    title: {
      text: 'Process FPS',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '15%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      data: props.procFps.map((obj) => {
        return moment(new Date(obj.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: 'FPS', min: 0 }],
    series: [
      {
        type: 'line',
        data: props.procFps.map((obj) => {
          return obj.fps;
        }),
        showSymbol: false,
      },
    ],
  };
  chart.setOption(option);
};
const printProcThread = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `procThreadChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `procThreadChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#ee6666'],
    title: {
      text: 'Process Thread Count',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '15%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      data: props.procThread.map((obj) => {
        return moment(new Date(obj.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: 'Count', min: 0 }],
    series: [
      {
        type: 'line',
        data: props.procThread.map((obj) => {
          return obj.threadCount;
        }),
        showSymbol: false,
      },
    ],
  };
  chart.setOption(option);
};
const printNetwork = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysNetworkChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysNetworkChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#5470c6', '#91cc75', '#fac858', '#ee6666', '#73c0de', '#409EFF'],
    title: {
      text: 'System Network(仅WIFI)',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      valueFormatter: (value) => `${value.toFixed(3)}`,
    },
    legend: {
      top: '8%',
      data: ['wlan0_tx', 'wlan0_rx'],
      formatter: function(name) {
        if (name === 'wlan0_tx') {
          return '发送';
        } else if (name === 'wlan0_rx') {
          return '接收';
        }
      }
    },
    grid: { top: '20%', left: '18%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      data: getNetworkTimeStamp(),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.network')}(KB)`, min: 0 }],
    series: getNetworkDataGroup(),
  };
  chart.setOption(option);
};
const printPerfCpu = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `perfCpuChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `perfCpuChart`
      )
    );
  }
  chart.resize();
  const option = {
    title: {
      text: 'Process CPU',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
      valueFormatter: (value) => `${value.toFixed(3)} %`,
    },
    grid: { top: '15%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.procCpu.map((obj) => {
        return moment(new Date(obj.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    yAxis: [{ name: `${$t('perf.procCpu')}(%)`, min: 0 }],
    series: [
      {
        type: 'line',
        data: props.procCpu.map((obj) => {
          return obj.cpuUtilization;
        }),
        showSymbol: false,
        areaStyle: {},
        boundaryGap: false,
      },
    ],
  };
  chart.setOption(option);
};
const printPerfMem = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `perfMemChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `perfMemChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#5470c6', '#91cc75', '#fac858', '#ee6666', '#73c0de', '#409EFF'],
    title: {
      text: 'Process Memory',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '20%', left: '20%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.procMem.map((obj) => {
        return moment(new Date(obj.timeStamp)).format('HH:mm:ss');
      }),
    },
    dataZoom: [
      {
        show: true,
        realtime: true,
        start: 0,
        end: 100,
        xAxisIndex: [0, 1],
      },
    ],
    legend: {
      top: '8%',
      data: ['Phy RSS(物理内存使用量)', 'VM RSS(虚拟内存使用量)', 'Total PSS(总共共享内存使用量)'],
    },
    yAxis: [{ name: `${$t('perf.memUsage')}(MB)`, min: 0 }],
    series: [
      {
        name: 'Phy RSS(物理内存使用量)',
        type: 'line',
        data: props.procMem.map((obj) => {
          return obj.phyRSS / 1024;
        }),
        showSymbol: false,
        boundaryGap: false,
      },
      {
        name: 'VM RSS(虚拟内存使用量)',
        type: 'line',
        data: props.procMem.map((obj) => {
          return obj.vmRSS / 1024;
        }),
        showSymbol: false,
        boundaryGap: false,
      },
      {
        name: 'Total PSS(总共共享内存使用量)',
        type: 'line',
        data: props.procMem.map((obj) => {
          return obj.totalPSS / 1024;
        }),
        showSymbol: false,
        boundaryGap: false,
      },
    ],
  };
  chart.setOption(option);
};
defineExpose({
  printCpu,
  //printSingleCpu,
  printMem,
  printNetwork,
  printPerfCpu,
  printPerfMem,
  printProcFps,
  printProcThread,
  //printAnalyze,
});
const switchTab = (e) => {
  if (e.index == 2) {
    nextTick(() => {
      const memChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `perfMemChart`
        )
      );
      memChart.resize();
      const cpuChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `perfCpuChart`
        )
      );
      cpuChart.resize();
      const fpsChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysFpsChart`
        )
      );
      fpsChart.resize();
      const threadChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `procThreadChart`
        )
      );
      threadChart.resize();
    });
  } else if (e.index == 1) {
    nextTick(() => {
      const memChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysMemChart`
        )
      );
      memChart.resize();
      /*
      const cpuChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysSingleCpuChart`
        )
      );
      cpuChart.resize();
      */
      const cpuChart2 = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysCpuChart`
        )
      );
      cpuChart2.resize();
      const networkChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysNetworkChart`
        )
      );
      networkChart.resize();
    });
  } else {
    /*
    nextTick(() => {
      const analyzeChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `analyzeChart`
        )
      )
      analyzeChart.resize();
    });
    */
  }
};
</script>

<template>
  <el-tabs
    style="margin-top: 10px"
    stretch
    type="border-card"
    @tab-click="switchTab"
  >
  <el-tab-pane label="PerfMon Analyze">
      <div class="dashboard">
        <div class="row">
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">系统平均CPU使用率(%)</div>
            <div class="card-value" :style="{ color: '#FF6600' }">{{ getAnalyze()[0].SysCPU }}%</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">APP平均CPU使用率(%)</div>
            <div class="card-value" :style="{ color: '#FF6600' }">-</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">系统平均内存(MB)</div>
            <div class="card-value" :style="{ color: '#FFA500' }">{{ getAnalyze()[0].SysMEM }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">APP平均内存(MB)</div>
            <div class="card-value" :style="{ color: '#FFA500' }">-</div>
          </el-card>
        </div>
        <div class="row">
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">网络总上行(KB)</div>
            <div class="card-value" :style="{ color: '#33CC33' }">{{ getAnalyze()[0].NetworkTX }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">网络总下行(KB)</div>
            <div class="card-value" :style="{ color: '#33CC33' }">{{ getAnalyze()[0].NetworkRX }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">平均FPS</div>
            <div class="card-value" :style="{ color: '#0099CC' }">-</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">指标建设中...</div>
            <div class="card-value" :style="{ color: '#0099CC' }">-</div>
          </el-card>
        </div>
        </div>
      <!--
      <el-row :gutter="10">
          <el-col :span="24">
            <div class="container">
          <el-card class="left-card">
            <div v-if="recordUrl.length > 0" class="flex-center" style="width: 100%; height: 500px">
                  <video-play id="media" ref= "videoPlayRef" v-bind="videoOptions" :src="recordUrl" />
                </div>
                <el-empty
                  v-else
                  :description="$t('resultDetailTS.page.onRecording')"
                ></el-empty>
          </el-card>
          <el-card class="right-card">
          <div
              :id="rid + '-' + cid + '-' + did + '-' + 'analyzeChart'"
              v-loading="sysCpu.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 450px"
              ></div>
          </el-card>
        </div>
          </el-col>
        </el-row>
        -->
    </el-tab-pane>
    <el-tab-pane label="System PerfMon">
      <el-row :gutter="10">
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysCpuChart'"
              v-loading="sysCpu.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <!--
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysSingleCpuChart'"
              v-loading="sysCpu.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 450px"
            ></div>
          </el-card>
        </el-col>
        -->
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysMemChart'"
              v-loading="sysMem.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysNetworkChart'"
              v-loading="sysNetwork.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
      </el-row>
    </el-tab-pane>
    <el-tab-pane label="Process PerfMon">
      <el-row :gutter="10">
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'perfCpuChart'"
              v-loading="procCpu.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'perfMemChart'"
              v-loading="procMem.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysFpsChart'"
              v-loading="procFps.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'procThreadChart'"
              v-loading="procThread.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
      </el-row>
    </el-tab-pane>
  </el-tabs>
</template>

<style scoped>
.dashboard {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.row {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
}
.card {
  width: 190px;
  margin: 0 10px;
  text-align: center;
}

.card-title {
  font-size: 14px;
  margin-bottom: 10px;
}

.card-value {
  font-size: 24px;
}

.container {
  display: flex;
}

.left-card {
  flex: 3;
  margin-right: 10px;
}

.right-card {
  flex: 7;
}
</style>
