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
import * as echarts from 'echarts/core';
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
import { nextTick, ref, reactive, watch  } from 'vue';
import { useI18n } from 'vue-i18n';

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
  cpu: Array,
  mem: Array,
  gpu: Array,
  fps: Array,
  disk: Array,
  network: Array,
  procPerf: Array,
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

  const din = [];
  const dout = [];
  props.network.map((obj,index) => {
    if (index !== 0) {
      din.push((obj.bytes_in - props.network[index-1].bytes_in) / 1024)
      dout.push((obj.bytes_out - props.network[index-1].bytes_out) / 1024)
    } else {
      din.push(0)
      dout.push(0)
    }
  });

  const syscpu = props.cpu.map((obj) => {
          return obj.total_load;
        });

  const sysmem = props.mem.map((obj) => {
          return obj.used_memory / 1024 * 16;
        });

  const dr = [];
  const dw = [];
  props.disk.map((obj,index) => {
    if (index !== 0) {
      dr.push((obj.data_read - props.disk[index-1].data_read) / (1024))
      dw.push((obj.data_written - props.disk[index-1].data_written) / (1024))
    } else {
      dr.push(0)
      dw.push(0)
    }
  });

  const fps = props.fps.map((obj) => {
          return obj.fps;
        });

  const gpu_device = [];
  const gpu_render = [];
  const gpu_tiler = [];

  props.gpu.map((obj) => {
    gpu_device.push(obj.device_utilization);
    gpu_render.push(obj.renderer_utilization);
    gpu_tiler.push(obj.tiler_utilization);
  });

  const perfCpu = props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.cpuUsage) {
            return obj.proc_perf.cpuUsage;
          }
        });

  const perfMem = props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.memResidentSize) {
            return obj.proc_perf.memResidentSize / (1024 * 1024);
          }
        });
  

  result.push({
    NetworkIN: sum(din),
    NetworkOUT: sum(dout),
    SysCPU: average(syscpu),
    SysMEM: average(sysmem),
    Fps: average(fps),
    gpu_device: average(gpu_device),
    gpu_render: average(gpu_render),
    gpu_tiler: average( gpu_tiler),
    DiskRead: sum(dr),
    DiskWrite: sum(dw),
    PerfCpu: average(perfCpu),
    PerfMem: average(perfMem),
  });
  return result
}

const getDiskDataGroup = () => {
  const result = [];
  const res = [];
  if (props.disk.length > 0) {
      const dr = [];
      const dw = [];
      props.disk.map((obj,index) => {
        if (index !== 0) {
          dr.push((obj.data_read - props.disk[index-1].data_read) / (1024))
          dw.push((obj.data_written - props.disk[index-1].data_written) / (1024))
        } else {
          dr.push(0)
          dw.push(0)
        }
      });
      result.push({
        type: 'line',
        name: `sys_disk_dr`,
        data: dr,
        showSymbol: false,
        boundaryGap: false,
      });
      result.push({
        type: 'line',
        name: `sys_disk_dw`,
        data: dw,
        showSymbol: false,
        boundaryGap: false,
      });
  }
  result.map((obj) => {
        if (obj.name === 'sys_disk_dr' || obj.name === 'sys_disk_dw') {
          res.push(obj)
        }
      });
  console.log(result)
  return res;
};

const getNetworkDataGroup = () => {
  const result = [];
  const res = [];
  if (props.network.length > 0) {
      const din = [];
      const dout = [];
      props.network.map((obj,index) => {
        if (index !== 0) {
          din.push((obj.bytes_in - props.network[index-1].bytes_in) / 1024)
          dout.push((obj.bytes_out - props.network[index-1].bytes_out) / 1024)
        } else {
          din.push(0)
          dout.push(0)
        }
      });
      result.push({
        type: 'line',
        name: `Bytes In`,
        data: din,
        showSymbol: false,
        boundaryGap: false,
      });
      result.push({
        type: 'line',
        name: `Bytes Out`,
        data: dout,
        showSymbol: false,
        boundaryGap: false,
      });
  }
  result.map((obj) => {
        if (obj.name === 'Bytes In' || obj.name === 'Bytes Out') {
          res.push(obj)
        }
      });
  console.log(result)
  return res;
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
      text: 'System CPU',
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
      boundaryGap: false,
      type: 'category',
      data: props.cpu.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
    series: [
      {
        type: 'line',
        data: props.cpu.map((obj) => {
          return obj.total_load;
        }),
        showSymbol: false,
        //areaStyle: {},
        boundaryGap: false,
      },
    ],
  };
  chart.setOption(option);
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
    },
    grid: { top: '30%', left: '13%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: [
        'Used Memory',
      ],
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.mem.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
        name: 'Used Memory',
        type: 'line',
        data: props.mem.map((obj) => {
          return obj.used_memory / 1024 * 16;
        }),
        showSymbol: false,
        boundaryGap: false,
      },
    ]
  };
  chart.setOption(option);
};
const printGpu = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysGpuChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysGpuChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#5470c6', '#91cc75', '#ee6666'],
    title: {
      text: 'System GPU',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '25%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: ['Device Utilization', 'Renderer Utilization', 'Tiler Utilization'],
      formatter: function(name) {
        if (name === 'Device Utilization') {
          return '设备整体使用率';
        } else if (name === 'Renderer Utilization') {
          return 'GPU渲染使用率';
        } else if (name === 'Tiler Utilization') {
          return 'GPU Tiler使用率';
        }
      }
    },
    xAxis: {
      data: props.gpu.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
    yAxis: [{ name: '(%)', min: 0 }],
    series: [
      {
        name: 'Device Utilization',
        type: 'line',
        data: props.gpu.map((obj) => {
          return obj.device_utilization;
        }),
      },
      {
        name: 'Renderer Utilization',
        type: 'line',
        data: props.gpu.map((obj) => {
          return obj.renderer_utilization;
        }),
      },
      {
        name: 'Tiler Utilization',
        type: 'line',
        data: props.gpu.map((obj) => {
          return obj.tiler_utilization;
        }),
      },
    ],
  };
  chart.setOption(option);
};
const printFps = () => {
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
      text: 'System FPS',
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
      data: props.fps.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
        data: props.fps.map((obj) => {
          return obj.fps;
        }),
        showSymbol: false,
      },
    ],
  };
  chart.setOption(option);
};
const printDisk = () => {
  let chart = echarts.getInstanceByDom(
    document.getElementById(
      `${props.rid}-${props.cid}-${props.did}-` + `sysDiskChart`
    )
  );
  if (chart == null) {
    chart = echarts.init(
      document.getElementById(
        `${props.rid}-${props.cid}-${props.did}-` + `sysDiskChart`
      )
    );
  }
  chart.resize();
  const option = {
    color: ['#5470c6', '#91cc75', '#fac858', '#ee6666', '#73c0de', '#409EFF'],
    title: {
      text: 'System Disk',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '25%', left: '18%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: ['sys_disk_dr', 'sys_disk_dw'],
      formatter: function(name) {
        if (name === 'sys_disk_dr') {
          return '每秒读取数据量';
        } else if (name === 'sys_disk_dw') {
          return '每秒写入数据量';
        }
      }
    },
    xAxis: {
      data: props.disk.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
    yAxis: [{ name: `${$t('perf.byteData')}(KB)`, min: 0 }],
    series: getDiskDataGroup(),
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
      text: 'System Network',
      textStyle: {
        color: '#606266',
      },
      x: 'center',
      y: 'top',
    },
    tooltip: {
      trigger: 'axis',
    },
    grid: { top: '25%', left: '18%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: ['Bytes In', 'Bytes Out', 'Packets In', 'Packets Out'],
      formatter: function(name) {
        if (name === 'Bytes In') {
          return '接收';
        } else if (name === 'Bytes Out') {
          return '发送';
        }
      }
    },
    xAxis: {
      data: props.network.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
      data: props.procPerf.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
        data: props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.cpuUsage) {
            return obj.proc_perf.cpuUsage;
          }
        }),
        showSymbol: false,
        //areaStyle: {},
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
    grid: { top: '30%', left: '13%' },
    toolbox: {
      feature: {
        saveAsImage: { show: true, title: 'Save' },
      },
    },
    legend: {
      top: '8%',
      data: [
        'Mem Anon',
        'Mem Resident Size',
        'Phys Footprint',
      ],
      formatter: function(name) {
        if (name === 'Mem Anon') {
          return '匿名内存消耗';
        } else if (name === 'Mem Resident Size') {
          return '实际总内存消耗';
        } else if (name === 'Phys Footprint') {
          return '实际物理内存消耗'
        }
      }
    },
    xAxis: {
      boundaryGap: false,
      type: 'category',
      data: props.procPerf.map((obj) => {
        return moment(new Date(obj.timestamp * 1000)).format('HH:mm:ss');
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
        name: 'Mem Anon',
        type: 'line',
        data: props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.memAnon) {
            return obj.proc_perf.memAnon / (1024 * 1024);
          }
        }),
        showSymbol: false,
        boundaryGap: false,
      },
      {
        name: 'Mem Resident Size',
        type: 'line',
        data: props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.memResidentSize) {
            return obj.proc_perf.memResidentSize / (1024 * 1024);
          }
        }),
        showSymbol: false,
        boundaryGap: false,
      },
      {
        name: 'Phys Footprint',
        type: 'line',
        data: props.procPerf.map((obj) => {
          if (obj.proc_perf && obj.proc_perf.physFootprint) {
            return obj.proc_perf.physFootprint / (1024 * 1024);
          }
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
  printMem,
  printGpu,
  printFps,
  printDisk,
  printNetwork,
  printPerfCpu,
  printPerfMem,
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
    });
  } else {
    nextTick(() => {
      const memChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysMemChart`
        )
      );
      memChart.resize();
      const cpuChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysCpuChart`
        )
      );
      cpuChart.resize();
      const gpuChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysGpuChart`
        )
      );
      gpuChart.resize();
      const fpsChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysFpsChart`
        )
      );
      fpsChart.resize();
      const diskChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysDiskChart`
        )
      );
      diskChart.resize();
      const networkChart = echarts.getInstanceByDom(
        document.getElementById(
          `${props.rid}-${props.cid}-${props.did}-` + `sysNetworkChart`
        )
      );
      networkChart.resize();
    });
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
            <div class="card-value" :style="{ color: '#FF6600' }">{{ getAnalyze()[0].PerfCpu }}%</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">系统平均内存(MB)</div>
            <div class="card-value" :style="{ color: '#FFA500' }">{{ getAnalyze()[0].SysMEM }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">APP平均内存(MB)</div>
            <div class="card-value" :style="{ color: '#FFA500' }">{{ getAnalyze()[0].PerfMem }}</div>
          </el-card>
        </div>
        <div class="row">
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">网络总上行(KB)</div>
            <div class="card-value" :style="{ color: '#33CC33' }">{{ getAnalyze()[0].NetworkOUT }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">网络总下行(KB)</div>
            <div class="card-value" :style="{ color: '#33CC33' }">{{ getAnalyze()[0].NetworkIN }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">磁盘写入(KB)</div>
            <div class="card-value" :style="{ color: '#0099CC' }">{{ getAnalyze()[0].DiskWrite }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">磁盘读取(KB)</div>
            <div class="card-value" :style="{ color: '#0099CC' }">{{ getAnalyze()[0].DiskRead }}</div>
          </el-card>
        </div>
        <div class="row">
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">平均GPU设备使用率(%)</div>
            <div class="card-value" :style="{ color: '#00CED1' }">{{ getAnalyze()[0].gpu_device }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">平均GPU渲染使用率(%)</div>
            <div class="card-value" :style="{ color: '#00CED1' }">{{ getAnalyze()[0].gpu_render }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">平均GPU-Tiler使用率(%)</div>
            <div class="card-value" :style="{ color: '#00CED1' }">{{ getAnalyze()[0].gpu_tiler }}</div>
          </el-card>
          <el-card class="card" :body-style="{ padding: '20px' }" shadow="hover">
            <div class="card-title">平均FPS</div>
            <div class="card-value" :style="{ color: '#800080' }">{{ getAnalyze()[0].Fps }}</div>
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
              v-loading="cpu.length === 0"
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
              v-loading="cpu.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysMemChart'"
              v-loading="mem.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysGpuChart'"
              v-loading="gpu.length === 0"
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
              v-loading="fps.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
        <el-col :span="24">
          <el-card style="margin-top: 10px">
            <div
              :id="rid + '-' + cid + '-' + did + '-' + 'sysDiskChart'"
              v-loading="disk.length === 0"
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
              v-loading="network.length === 0"
              :element-loading-text="$t('perf.emptyData')"
              element-loading-spinner="el-icon-box"
              style="width: 100%; height: 350px"
            ></div>
          </el-card>
        </el-col>
      </el-row>
    </el-tab-pane>
    <el-tab-pane label="Process PerfMon">
      <el-card style="margin-top: 10px">
        <div
          :id="rid + '-' + cid + '-' + did + '-' + 'perfCpuChart'"
          v-loading="procPerf.length === 0"
          :element-loading-text="$t('perf.emptyData')"
          element-loading-spinner="el-icon-box"
          style="width: 100%; height: 350px"
        ></div>
      </el-card>
      <el-card style="margin-top: 10px">
        <div
          :id="rid + '-' + cid + '-' + did + '-' + 'perfMemChart'"
          v-loading="procPerf.length === 0"
          :element-loading-text="$t('perf.emptyData')"
          element-loading-spinner="el-icon-box"
          style="width: 100%; height: 350px"
        ></div>
      </el-card>
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
