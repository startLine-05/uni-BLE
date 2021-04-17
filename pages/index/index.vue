<template>
	<view>
		<view class="log">
			<view>展示log日志(可滑动查看)：</view>
			<view>
				<scroll-view scroll-y="true" class="scroll-list">
					<text>{{textLog}}</text>
				</scroll-view>
			</view>
		</view>
		<view class="scan-btn">
			<view class="btn">
				<u-button type="primary" @click="startClear">清空log日志</u-button>
			</view>
			<view class="btn">
				<u-button type="primary" @click="startScan">扫描蓝牙设备</u-button>
			</view>
		</view>

		<view class="devices_summary">已发现 {{devices.length}} 个外围设备：</view>
		<scroll-view class="device_list" scroll-y scroll-with-animation>
			<view v-for="(item, index) in devices" :key="index" :data-device-id="item.deviceId" :data-name="item.name || item.localName" 
			@tap="createBLEConnection" class="device_item" hover-class="device_item_hover">
			  <view style="font-size: 16px; color: #333;">{{item.name}}</view>
			  <view style="font-size: 10px">信号强度: {{item.RSSI}}dBm ({{utils.max(0, item.RSSI + 100)}}%)</view>
			  <view style="font-size: 10px">UUID: {{item.deviceId}}</view>
			  <view style="font-size: 10px">Service数量: {{utils.len(item.advertisServiceUUIDs)}}</view>
			</view>
		</scroll-view>
		
		<block v-if="connected">
		<!-- <block v-if="true"> -->
			<view  class="linkd_devices">
				<view>已连接的设备:{{name}}</view>
				<view>已连接的设备ID:{{devId}}</view>
			</view>
			
			<view class="linkd_btn">
				<view class="btn">
					<u-button  type="primary" @click="show = true">开启{{name}}主服务</u-button>
				</view>
				<view class="btn">
					<u-button  type="primary" @click="closeBLEConnection">关闭{{name}}服务连接</u-button>
				</view>
			</view>
		</block>
		
		
		<u-action-sheet :list="list" v-model="show" @click="toLink"></u-action-sheet>
	</view>
</template>

<script module="utils" lang="wxs" src="./utils.wxs"></script>

<script>
// pages/index/index.js
var app = getApp();
// var utils = require("@/utils/BLE.js");
import utils from '@/utils/BLE.js'
//查数据的重复
function inArray(arr, key, val) {
	for (let i = 0; i < arr.length; i++) {
		if (arr[i][key] === val) {
		  return i;
		}
	}
	return -1;
}

export default {
	data() {
		return {
			discoveryStarted:false,//初始化蓝牙
			textLog: "", //打印日志
			isopen: false, //是否已经开启蓝牙
			devices: [], //设备数组
			connected: false, //是否创建链接 
			name: "", //设备名称
			devId: "", //设备uuid
			//服务的uuid
			list: [],
			show: false//是否显示服务菜单栏
		  
		};
	},
	onLoad: function (options) {
		uni.$on('close',()=>{
			this.closeBLEConnection()
		},)
	},
	onUnload: function () {
		console.log("生命周期函数--监听页面卸载");
		this.closeBluetoothAdapter(); //关闭蓝牙模块，使其进入未初始化状态。
	},
	methods: {
		//清空log日志
		startClear: function () {
		  this.textLog = '';
		},
		/**
		 * 流程：
		 * 1.先初始化蓝牙适配器，
		 * 2.获取本机蓝牙适配器的状态，
		 * 3.开始搜索，当停止搜索以后在开始搜索，就会触发蓝牙是配置状态变化的事件，
		 * 4.搜索完成以后获取所有已经发现的蓝牙设备，就可以将devices中的设备Array取出来，
		 * 5.然后就可以得到所有已经连接的设备了
		 */
		startScan: function () {
		  this.discoveryStarted = false;
		  if (this.isopen) {
			//如果已初始化小程序蓝牙模块，则直接执行扫描
			this.getBluetoothAdapterState();
		  } else {
			this.openBluetoothAdapter();
		  }
		},
		//初始化小程序蓝牙模块
		openBluetoothAdapter: function () {
			uni.openBluetoothAdapter({
				success: (res) => {
				  this.textLog = this.textLog + "打开蓝牙适配器成功！\n"
				  this.isopen = true
				  this.getBluetoothAdapterState();
				},
				fail: (err) => {
					this.$u.toast('蓝牙开关未开启');
					this.textLog = this.textLog + "蓝牙开关未开启 \n";
				}
			}); 
			//监听蓝牙适配器状态变化事件
			uni.onBluetoothAdapterStateChange((res) =>{
				console.log('onBluetoothAdapterStateChange', res);
				const isDvailable = res.available; //蓝牙适配器是否可用
				if (isDvailable) {
					this.getBluetoothAdapterState();
				} else {
					this.stopBluetoothDevicesDiscovery(); //停止搜索
					this.devices = []
					this.$u.toast('蓝牙开关未开启');
				}
			});
		},
		//关闭蓝牙模块，使其进入未初始化状态。
		closeBluetoothAdapter: function () {
			uni.closeBluetoothAdapter();
			this.discoveryStarted = false;
		},
		//获取本机蓝牙适配器状态
		getBluetoothAdapterState: function () {
		  uni.getBluetoothAdapterState({
			success:  (res) => {
				var isDiscov = res.discovering; //是否正在搜索设备
				var isDvailable = res.available; //蓝牙适配器是否可用
				if (isDvailable) {
				this.textLog = this.textLog + "本机蓝牙适配器状态：可用 \n";
					if (!isDiscov) {
						this.startBluetoothDevicesDiscovery();
					} else {
						this.textLog = this.textLog + "已在搜索设备 \n";
					}
				}
			}
		  });
		},
		//开始扫描附近的蓝牙外围设备。
		//注意，该操作比较耗费系统资源，请在搜索并连接到设备后调用 stop 方法停止搜索。
		startBluetoothDevicesDiscovery: function () {
			if (this.discoveryStarted) {
				return;
			}
		
			this.discoveryStarted = true;
			uni.showLoading({
				title: '正在扫描..'
			});
		
			this.textLog = this.textLog + "正在扫描..\n";
			
			setTimeout(function () {
				uni.hideLoading(); //隐藏loading
			}, 3000);
			
			//开始扫描
			uni.startBluetoothDevicesDiscovery({
				// 要搜索但蓝牙设备主 service 的 uuid 列表。某些蓝牙设备会广播自己的主 service 的 uuid。
				// 如果设置此参数，则只搜索广播包有对应 uuid 的主服务的蓝牙设备。建议主要通过该参数过滤掉周边不需要处理的其他蓝牙设备
				services: [],
				//是否允许重复上报同一设备, 如果允许重复上报，则onDeviceFound 方法会多次上报同一设备，但是 RSSI(信号) 值会有不同 
				// 默认值 false
				// allowDuplicatesKey: false,
				success: (res) => {
					this.textLog = this.textLog + "扫描附近的蓝牙外围设备成功，准备监听寻找新设备" + "\n";
					this.onBluetoothDeviceFound(); //监听寻找到新设备的事件
				}
			});
		},
		//停止搜寻附近的蓝牙外围设备。若已经找到需要的蓝牙设备并不需要继续搜索时，建议调用该接口停止蓝牙搜索。
		stopBluetoothDevicesDiscovery: function () {
			this.textLog = this.textLog + "停止搜寻附近的蓝牙外围设备 \n";
			uni.stopBluetoothDevicesDiscovery();
		},
		//监听寻找到新设备的事件
		onBluetoothDeviceFound: function () {
		  var that = this;
			
			uni.onBluetoothDeviceFound((res) => {
				res.devices.forEach( (device) =>{
					// console.log(device)
					//去掉名字未知的蓝牙设备
					if (!device.name && !device.localName) {
						return;
					}
					this.devices.push(device)
				});
			});
		},
		//连接低功耗蓝牙设备。
		createBLEConnection: function (e) {
			// console.log(e)
			const devId = e.currentTarget.dataset.deviceId; //设备UUID
			const name = e.currentTarget.dataset.name; //设备名
			
			(this.connected) && this.closeBLEConnection();  //配对之前先断开已连接设备

			this.textLog = this.textLog + "正在连接，请稍后..\n";

			uni.showLoading({
				title: '连接中...'
			});

			
			uni.createBLEConnection({
				deviceId: devId,
				success:  (res) => {
					uni.hideLoading(); //隐藏loading

					this.textLog = this.textLog + "配对成功,获取服务..\n";
					this.devId = devId
					this.name = name
					this.connected = true
					this.getBLEDeviceServices(devId);
				},
				fail: (err) => {
					uni.hideLoading(); //隐藏loading

					this.textLog = this.textLog + "连接失败，错误码：" + err.errCode + "\n";

					if (err.errCode === 10012) {
					  this.$u.toast('连接超时,请重试!"');
					} else if (err.errCode === 10013) {
					   this.$u.toast('连接失败,蓝牙地址无效!"');
					} else {
					   this.$u.toast('连接失败,请重试!"');
					 // + err.errCode10003原因多种：蓝牙设备未开启或异常导致无法连接;蓝牙设备被占用或者上次蓝牙连接未断开导致无法连接
					}
					this.closeBLEConnection();
				}
			});
			this.stopBluetoothDevicesDiscovery(); //停止搜索
		},
		//断开与低功耗蓝牙设备的连接
		closeBLEConnection: function () {
			uni.closeBLEConnection({
				deviceId: this.devId
			});
			this.devId = '';
			this.name = '';
			this.list = [];
			this.connected = false;
		},
		//获取蓝牙设备所有 service（服务）
		getBLEDeviceServices: function (devId) {
			var that = this;
				uni.getBLEDeviceServices({
				deviceId: devId,
				success: (res)=> {
					this.list = res.services.map(v => {
						if(v.isPrimary){
							return{
								text:'蓝牙主服务ID',
								subText:v.uuid,
							}
						}
					})
					this.show = true
					this.textLog = this.textLog + `开启${this.name}主服务`
				}
			});
		},
		//跳转到详情页面
		toLink(index){
			const { name, devId, list} = this
			const serviceId = list[index].subText
			
			//这里要格式
			this.$u.route({
				url: 'pages/BLE-Link/index',
				params: {
					name,
					deviceId:devId,
					serviceId,
				}
			})
		},
	}
};
</script>
<style>
/* pages/index/index.wxss */
.log {
	margin: 20rpx;
}
.btn{
	margin-bottom: 20rpx;
}
.scroll-list {
  max-height: 300rpx;
}
.scan-btn {
	padding: 30rpx;
}


.devices_summary {
  padding: 10rpx;
  font-size: 32rpx;
}
.device_list {
  height: 400rpx;
  margin: 50px 5%;
  margin-top: 0;
  border: 1px solid #EEE;
  border-radius: 5px;
  width: auto;
}
.device_item {
  border-bottom: 1px solid #EEE;
  padding: 10px;
  color: #666;
}
.device_item_hover {
  background-color: rgba(0, 0, 0, .1);
}

.linkd_devices{
	margin: 20rpx;
	font-size: 28rpx;
	color: #007AFF;
	font-weight: bold;
}
.linkd_btn{
	padding: 30rpx;
}
</style>
