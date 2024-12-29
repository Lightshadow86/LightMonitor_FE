<script setup>
import {computed, onMounted, provide, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import axios from "axios";
import { Message } from "@arco-design/web-vue";
import StatsCard from "@/components/StatsCard.vue";
import {formatBytes, formatTimeStamp, formatUptime, calculateRemainingDays} from '@/utils/utils'

const socketURL = ref('')
const apiURL = ref('')
const Title = ref('')
const SubTitle = ref('')

const theme = window.localStorage.getItem('theme') || 'light'
const dark = ref(theme !== 'light')

const handleChangeDark = () => {
  dark.value = !dark.value

  if (dark.value) {
    window.localStorage.setItem('theme','dark')
    document.body.setAttribute('arco-theme', 'dark')
  } else {
    // 恢复亮色主题
    window.localStorage.setItem('theme','light')
    document.body.removeAttribute('arco-theme');
   }
}

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }

  return data.value.filter(item => item.Host.Region === selectArea.value)
})

const hosts = computed(() => {
  if (type.value === 'all') {
    return host.value
  } else if (type.value === 'online') {
    return host.value.filter(item => item.status)
  } else {
    return host.value.filter(item => !item.status)
  }
})

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0
  let disks_used = 0
  let disks_total = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetOutSpeed
    bandwidth_down += item.State.NetInSpeed
    traffic_up += item.State.NetOutTransfer
    traffic_down += item.State.NetInTransfer
    disks_used += item.State.DiskUsed
    disks_total += item.Host.DiskTotal
    traffic_down += item.State.NetInTransfer
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down,
    disks_used: disks_used,
    disks_total: disks_total,
    traffic_up: traffic_up,
    traffic_down: traffic_down
  }
})

const handleChangeType = (newType) => {
  type.value = newType; // 更新 type ref 的值
};

provide('handleChangeType', handleChangeType);

let socket = null // 定义全局 WebSocket 变量

/**
 * 从服务器获取配置信息
 */
const fetchConfig = async () => {
  try {
    // 使用 axios 发起 GET 请求，获取配置文件内容
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
    Title.value = res.data.Title
    SubTitle.value = res.data.SubTitle
  } catch (e) {
    // 捕获异常并显示错误消息
    Message.error('获取配置失败') // 显示错误提示
  }
}

/**
 * 初始化 WebSocket 连接
 */
const initSocket = async () => {
  // 首先获取配置信息，确保 WebSocket 服务器地址已加载
  await fetchConfig()

  // 使用获取的 socketURL 创建 WebSocket 连接
  socket = new WebSocket(socketURL.value)

  // 定义 WebSocket 收到消息的事件处理器
  socket.onmessage = function (event) {
    try {
      // 获取 WebSocket 消息的内容
      const message = event.data
      // 直接解析为 JSON 对象
      const res = JSON.parse(message)

      if (res != null) {
        // 从消息中提取区域信息，并生成去重后的区域名称列表
        area.value = Array.from(new Set(res.Servers.map(item => item.Host.Region)))
      }

      // 获取服务器的时间戳
      const serverTimestamp = res.Timestamp

      // 遍历消息中的主机数据，处理状态和统计信息
      data.value = res.Servers.map((host) => {
        // 检查是否已有该主机的图表数据记录，若没有则初始化
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],     // CPU 使用率记录
            mem: [],     // 内存使用记录
            net_in: [],  // 网络输入速率记录
            net_out: []  // 网络输出速率记录
          }
        }

        // 如果图表数据已达到 2 条，保留最新的 1 条记录
        if (charts.value[host.Host.Name].cpu.length == 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        // 将新数据追加到图表记录中
        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetOutSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetInSpeed])

        // 返回主机的处理后信息，包括状态信息
        return {
          ...host,
          status: (serverTimestamp - host.TimeStamp <= 10) ? 1 : 0 // 判断主机状态是否在线，使用服务器时间戳
        }
      })
    } catch (error) {
      // 捕获消息解析错误并打印错误信息
      console.error('解析 WebSocket 消息时出错:', error)
    }
  }

  // WebSocket 连接关闭时的回调
  socket.onclose = function () {
    // 显示警告提示并尝试重新连接
    Message.warning('WebSocket已断连，正在重连中...')
    initSocket() // 重新调用初始化方法
  }
}

onMounted(async() => {
  if (dark.value) {
    document.body.setAttribute('arco-theme', 'dark')
  }

  await initSocket()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'danger';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

// 删除主机
const deleteVisible = ref(false);
const Token = ref('');
const deleteHostName = ref('');

// 删除操作显示
const handleShowDelete = (name) => {
  Token.value = window.localStorage.getItem('Token') || '';
  deleteHostName.value = name;
  deleteVisible.value = true;
};

// 删除主机
const handleDeleteHost = async () => {
  try {
    const res = await axios.post(apiURL.value, {
      "Token": Token.value,
      "Name": deleteHostName.value,
      "Action": "Delete"
    });
    if (res.status === 200) {
      Message.success(res.data);
    } else {
      Message.error(res.data);
    }
    window.localStorage.setItem('Token', Token.value);
    deleteVisible.value = false;
  } catch (e) {
    Message.error(e.response ? e.response.data : '删除失败，请检查网络连接');
  }
};

// 关闭删除模态框
const handleClose = () => {
  deleteVisible.value = false;
};

const hostInfo = ref({});

const editHostName = ref('');
const Name = ref('');
const NodeToken = ref('');
const Region = ref('');
const City = ref('');
const addVisible = ref(false);
const editVisible = ref(false);
const isAdd = ref(false);

// 编辑主机信息
const handleShowEdit = (name, region, city) => {
  editVisible.value = true;
  isAdd.value = false;
  editHostName.value = name;
  Name.value = name;
  Region.value = region || '';
  City.value = city || '';
  Token.value = window.localStorage.getItem('Token') || '';
};

const handleShowAdd = () => {
  addVisible.value = true;
  isAdd.value = true;
  Name.value = '';
  NodeToken.value = '';
  Region.value = '';
  City.value = '';
  Token.value = window.localStorage.getItem('Token') || '';
};

// 更新或添加主机
const HandleConsole = async () => {
  if (isAdd.value === true) {
    if (!Name.value || !NodeToken.value || !Region.value) {
    Message.error('信息填写不完整');
    return;
    }
  }


  try {
    const action = isAdd.value ? 'Add' : 'Update';
    let payload = {}

    if (action === 'Add') {
      payload = {
      "Token": Token.value,
      "Action": action,
      "Name": Name.value,
      "NodeToken": NodeToken.value,
      "Region": Region.value,
      "City": City.value,
      };
    } else {
      payload = {
      "Token": Token.value,
      "Action": action,
      "Name": Name.value,
      "Region": Region.value,
      "City": City.value,
      };}


    if (action === 'Update') {
      payload.OriginName = Name.value;
    }

    const res = await axios.post(apiURL.value, payload);
    if (res.status === 200) {
    Message.success(res.data);
    if (isAdd.value) {
      addVisible.value = false;
    } else {
      editVisible.value = false;
    }
    isAdd.value = false;
    } else {
      Message.error(res.data);
    }
    window.localStorage.setItem('Token', Token.value);
  } catch (e) {
    Message.error(e.response ? e.response.data : '操作失败，请检查网络连接');
  }
};


// 关闭编辑模态框
const handleEditClose = () => {
  editVisible.value = false
}

</script>

<style scoped>
  .chart-margin {
    margin-bottom: 20px;
  }
</style>
<template>
  <div class="max-container">
    <div class="header">
      <a class="logo" href="#">
        <span>{{ Title }}</span>
        <small v-if="SubTitle" style="font-weight: 400; opacity: .8">｜ {{ SubTitle }}</small>
      </a>
       <!-- 编辑主机信息模态框 -->
      <div class="add-theme-buttons">
      <a-button class="add-btn" type="primary" shape="circle" @click="handleShowAdd">
        <template #icon>
          <icon-plus />
        </template>
      </a-button>
      <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">
        <template #icon>
          <icon-sun-fill v-if="!dark" />
          <icon-moon-fill v-else />
        </template>
      </a-button>
      </div>
      <a-modal v-model:visible="addVisible" :footer="false" :hide-title="true" width="360px">
        <div class="modal-title">
          <span>添加节点</span>
        </div>
        <div class="modal-content">
          <a-input v-model="Name" placeholder="主机名称" style="margin-bottom: 10px;"></a-input>
          <a-input v-model="NodeToken" placeholder="节点令牌" style="margin-bottom: 10px;"></a-input>
          <a-input v-model="Region" placeholder="区域" style="margin-bottom: 10px;"></a-input>
          <a-input v-model="City" placeholder="城市" style="margin-bottom: 10px;"></a-input>
          <a-input-password v-model="Token" placeholder="管理密钥"></a-input-password>
        </div>
        <div class="modal-action">
          <a-button type="primary" :long="true" @click="HandleConsole">确认添加</a-button>
        </div>
      </a-modal>
      <a-modal v-model:visible="editVisible" :footer="false" :hide-title="true" width="360px">
        <div class="modal-title">
          <span>更新节点</span>
        </div>
        <div class="modal-content">
          <a-input v-model="Name" placeholder="主机名称" style="margin-bottom: 10px;"></a-input>
          <a-input v-model="Region" placeholder="区域" style="margin-bottom: 10px;"></a-input>
          <a-input v-model="City" placeholder="城市" style="margin-bottom: 10px;"></a-input>
          <a-input-password v-model="Token" placeholder="管理密钥"></a-input-password>
        </div>
        <div class="modal-action">
          <a-button type="primary" :long="true" @click="HandleConsole">确认更新</a-button>
        </div>
      </a-modal>
    </div>
    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        全部地区
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>
    <StatsCard :type="type" :stats="stats" />
    <div class="monitor-card">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in hosts" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Region.replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status ? '在线' : '离线'}}</span>
            <span style="margin-left: 6px;">
              {{ (item && item.TimeStamp && item.Host && item.Host.BootTime) ? formatUptime(item.TimeStamp - item.Host.BootTime) : '-' }}
            </span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">系统</div>
          <div class="monitor-item-value">
            {{
              item.Host.Platform
                ? (item.Host.Platform.match(/Windows Server (\d+)/i)
                   ? `Windows Server ${RegExp.$1}`
                   : item.Host.Platform.match(/Windows (\d+)/i)
                     ? `Windows ${RegExp.$1}`
                     : item.Host.Platform).trim().replace(/\s+/g, ' ')
                : ''
            }}
          </div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">运行内存</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="disk">
          <div class="monitor-item-title">硬盘</div>
          <div class="monitor-item-value">{{(item.State.DiskUsed / item.Host.DiskTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">网络↓|↑</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="packets">
          <div class="monitor-item-title">数据包↓|↑</div>
          <div class="monitor-item-value">{{`${item.State.PacketsRecvRate}/s | ${item.State.PacketsSentRate}/s`}}</div>
        </div>
        <div class="connections">
          <div class="monitor-item-title">连接数TCP|UDP</div>
          <div class="monitor-item-value">{{`${item.State.TCPConections} | ${item.State.UDPConnections}`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">负载1|5|15</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <div class="processes">
          <div class="monitor-item-title">进程数</div>
          <div class="monitor-item-value">{{item.State.Processes}}</div>
        </div>
<!--        超过一列的限制了，重要性不高。如果启用会出现在第二行-->
<!--        <div class="uptime">-->
<!--          <div class="monitor-item-title">上报时间</div>-->
<!--          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>-->
<!--        </div>-->
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">地区</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Region.replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Region.replace('UK', 'GB')}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">城市</div>
                  <div class="value">{{item.Host.City}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">系统</div>
                  <div class="value">{{item.Host.Platform}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">版本</div>
                  <div class="value">{{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">架构</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div v-if="item.Host.Virtualization" class="detail-item">
                  <div class="name">虚拟化</div>
                  <div class="value">{{item.Host.Virtualization}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU占用</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">物理内存</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatBytes(item.State.MemUsed)}} / {{formatBytes(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟内存</div>
                  <div class="value">{{formatBytes(item.State.SwapUsed)}} / {{formatBytes(item.Host.SwapTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">硬盘</div>
                  <div class="value">{{formatBytes(item.State.DiskUsed)}} / {{formatBytes(item.Host.DiskTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">网络↓|↑</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">数据包↓|↑</div>
                  <div class="value">{{`${item.State.PacketsRecvRate}/s | ${item.State.PacketsSentRate}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">连接数↓|↑</div>
                  <div class="value">{{`${item.State.TCPConections} | ${item.State.UDPConnections}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">负载1|5|15</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">流量↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetOutTransfer)}} | {{formatBytes(item.State.NetInTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">启动时间</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">上报时间</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :xl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :xl="12">
                  <CPU ref="cpuRef" class="chart-margin" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :xl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" class="chart-margin" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :xl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :xl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
        <div class="edit-btn" @click.stop="handleShowEdit(item.Host.Name, item.Host.Region, item.Host.City)">
          <icon-edit />
        </div>
        <div class="delete-btn" @click.stop="handleShowDelete(item.Host.Name)">
          <icon-delete />
        </div>
        <a-modal v-model:visible="deleteVisible" :footer="false" :hide-title="true" width="360px">
          <div class="modal-title">
            <span>删除主机</span>
            <a-button @click="handleClose">
              <template #icon>
                <icon-close/>
              </template>
            </a-button>
          </div>
          <div class="modal-content">
            <a-input-password v-model="Token" placeholder="请输入管理密钥"></a-input-password>
            <div class="tips">提示：删除后无法恢复，请确定后再删除操作</div>
          </div>
          <div class="modal-action">
            <a-button type="primary" status="danger" :long="true" @click="handleDeleteHost">确认删除</a-button>
          </div>
        </a-modal>

        <!-- 编辑主机信息模态框 -->
        <a-modal v-model:visible="editVisible" :footer="false" :hide-title="true" width="360px">
          <div class="modal-title">
            <span>编辑主机信息</span>
            <a-button @click="handleEditClose">
              <template #icon>
                <icon-close/>
              </template>
            </a-button>
          </div>
          <div class="modal-content">
            <a-input v-model="editHostName" placeholder="主机名称" style="margin-bottom: 10px;"></a-input>
            <a-input v-model="Region" placeholder="区域" style="margin-bottom: 10px;"></a-input>
            <a-input v-model="City" placeholder="城市" style="margin-bottom: 10px;"></a-input>
            <a-input-password v-model="Token" placeholder="管理密钥"></a-input-password>
          </div>
          <div class="modal-action">
            <a-button type="primary" :long="true" @click="HandleConsole">确认操作</a-button>
          </div>
        </a-modal>
      </div>
    </div>
  </div>
<div class="footer" style="margin-top: 30px">代码开源在 <a href="https://github.com/akile-network/akile_monitor">GitHub v0.0.2</a></div>
<div class="footer" style="margin-bottom: 30px">Copyright © 2023-{{new Date().getFullYear()}} Akile LTD.</div>
</template>

<style lang="scss">
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    border: 1px solid #eeeeee!important;
    background-color: #ffffff!important;
    color: #333333!important;
  }
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: none!important;
      }

      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px!important;
      background: rgba(22, 131, 255, 0.13)!important;

      &:hover {
        background: rgba(22, 131, 255, 0.19)!important;
      }

      .arco-icon {
        color: #1673ff!important;
      }
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      background: #ff161620;
      border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: .15s background-color ease-in-out;

      &:hover {
        background: #ff161630;
      }

      .arco-icon {
        color: #ff1616;
      }
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      flex: 1 1 auto;  /* 自适应宽度 */
      min-width: 225px; /* 设置最小宽度 */
      white-space: nowrap; /* 防止内容换行 */
      margin-right: 10px; /* 设置元素间距，避免元素紧挨着 */
    }

    .name .title {
      margin-bottom: 5px;
      display: flex;
      align-items: center;
      font-weight: 600;
      font-size: 16px;
      white-space: nowrap;  /* 防止标题换行 */
    }

    .name .status {
      display: flex;
      align-items: center;
      white-space: nowrap; /* 防止状态文本换行 */
    }

    .name .status::before {
      margin-right: 10px;
      position: relative;
      display: block;
      content: '';
      width: 8px;
      height: 8px;
      border-radius: 12px;
      background-color: #fb2c36;
    }

    .name .status.online::before {
      background-color: #00c951;
    }

    .name .status span {
      font-size: 13px;
      opacity: 0.6;
    }

    .container {
      display: flex;
      flex-wrap: nowrap; /* 保证不换行 */
      justify-content: flex-start; /* 元素左对齐 */
      align-items: flex-start; /* 元素顶部对齐 */
      gap: 100px; /* 设置最小间隔，确保元素不紧挨着 */
    }

    .platform,
    .cpu,
    .mem,
    .disk,
    .average,
    .processes,
    .network,
    .connections,
    .packets,
    .uptime {
      display: inline-block;
      vertical-align: top;
      flex: 1 1 auto; /* 使宽度根据内容自适应 */
      margin-right: 10px; /* 设置元素间距，避免元素紧挨着 */
      white-space: nowrap; /* 防止内容换行 */
    }

    .platform { min-width: 150px; }
    .cpu { min-width: 75px; }
    .mem { min-width: 75px; }
    .disk { min-width: 75px; }
    .average { min-width: 140px; }
    .processes { min-width: 60px; }
    .network { min-width: 175px; }
    .connections { min-width: 100px; }
    .packets { min-width: 100px; }
    .uptime { min-width: 150px; }


    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .arco-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22,93,255)!important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  line-height: 22px;
  text-align: center;
  color: #565656;

  a {
    color: rgba(var(--primary-6));
  }
}

.modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 20px;
}

.modal-content {
  margin-bottom: 20px;
  .tips {
    font-size: 12px;
    color: #333333;
    margin-top: 10px;
  }
}

body[arco-theme='dark'] {
  background-color: #111111;

  .arco-modal {
    background-color: #0e0e0e;
    border: 1px solid rgba(255,255,255,0.05);
  }

  .header {
    .logo {
      span,
      small {
        color: #ffffff;
      }
    }

    .theme-btn {
      border: 1px solid #333333!important;
      background-color: #000000!important;
      color: #ffffff!important;
    }
  }

  .area-tabs {
    .area-tab-item {
      border-color: #333333;
      background: #000000;
      color: #ffffff;
      box-shadow: none;

      &.is-active {
        background: #005fe705;
        color: #005fe7;
        border: 1px solid #005fe7;
      }
    }
  }

  .footer {
    color: #ffffffAA;
  }

  .monitor-card {
    .monitor-item {
      border: 1px solid rgb(255 255 255 / 16%);
      box-shadow: none;
      background-color: #000000;
      color: #ffffff;

      &:hover {
        background-color: #101010;
      }

      .detail {
        border-color: #333333AA;
        .detail-item-list {
          .detail-item {
            .name {
              color: #999999;
            }
          }
        }
      }
    }
  }

}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}

.add-theme-buttons {
  display: flex;
  align-items: center;
  gap: 5px;
}

.add-btn {
  background-color: #ffffff !important;
  border-color: #ececec !important;
  color: #333333 !important;
}

.add-btn:hover{
  background-color: rgba(0,95,231,0.1) !important;
}

.theme-btn {
  border: 1px solid #eee !important; /* 设置边框颜色 */
  background-color: #ffffff !important; /* 设置背景颜色 */
  color: #333 !important; /* 设置文本颜色 */

}
</style>