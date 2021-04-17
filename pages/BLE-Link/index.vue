<template>
	<view>
		<view class="BLE-detail">
			<view>当前连接的蓝牙设备信息</view>
			<view>设备名：{{name}} </view>
			<view>设备ID：{{deviceId}} </view>
			<view>服务ID：{{serviceId}} </view>
		</view>

		<view class="card">
			<view>展示log日志(可滑动查看)：</view>
			<scroll-view scroll-y="true" class="text-box list">
			  <text>{{textLog}}</text>
			</scroll-view>  
		</view>

		<!-- 底部操作 -->
		<view class="option">
			<view class="btn">
				<u-input cursor-spacing="20" v-model="orderInputStr"  border placeholder="请输入字符串指令" />
			</view>
			<!--选择要发送的特征值id-->
			<view class="btn">
				<u-button  type="primary" @click="show = true">发送</u-button>
			</view>	
			<view class="btn">
				<u-button  type="primary" @click="startClear">清空日志</u-button>
			</view>
			<view class="btn">
				<u-button  type="primary" @click="closeBLEConnection">断开连接</u-button>
			</view>
		</view>
			
		<u-action-sheet :list="list" v-model="show" @click="send"></u-action-sheet>
		<u-toast ref="uToast" />
	</view>
</template>

<script>
import utils from '@/utils/BLE.js'
export default {
	data() {
		return {
			textLog: "", //日志
			deviceId: "", //设备id
			name: "", //设备名字
			serviceId: "", //服务id
			list:[], //订阅的特征值
			show:false,
			orderInputStr: "" //指令
		};
	},
	onLoad: function (options) {
		// console.log(options)
		//接收参数
		this.deviceId = options.deviceId
		this.name = options.name
		this.serviceId = options.serviceId
		this.textLog = this.textLog + "设备名=" + options.name + "\n设备UUID=" + options.deviceId + "\n服务UUID=" + options.serviceId + "\n";
		//获取特征值
		this.getBLEDeviceCharacteristics();
	},
	//蓝牙连接需要保持常亮
	onShow: function () {
		if (uni.setKeepScreenOn) {
			uni.setKeepScreenOn({
				keepScreenOn: true,
				success: function (res) {//console.log('保持屏幕常亮')
				}
			});
		}
	},
	methods: {
		//清空log日志
		startClear: function () {
			this.textLog = '';
		},
		//返回蓝牙是否正处于链接状态
		onBLEConnectionStateChange: function (onFailCallback) {
		  uni.onBLEConnectionStateChange(function (res) {
			// 该方法回调中可以用于处理连接意外断开等异常情况
			console.log(`device ${res.deviceId} state has changed, connected: ${res.connected}`);
			return res.connected;
		  });
		},
		//断开与低功耗蓝牙设备的连接
		closeBLEConnection: function () {
			uni.closeBLEConnection({
				deviceId: this.deviceId
			});
			this.connected = false
			this.$refs.uToast.show({
				title: '连接已断开',
				back:true,
				callback:()=>{
					uni.$emit('close')
				}
			})
		},
		//获取蓝牙设备某个服务中的所有 characteristic（特征值）
		getBLEDeviceCharacteristics: function (order) {
			// console.log('获取服务ID')
			var that = this;
			uni.getBLEDeviceCharacteristics({
				deviceId: this.deviceId,
				serviceId: this.serviceId,
				success: (res) => {
					console.log(res);
					for (let i = 0; i < res.characteristics.length; i++) {
						let item = res.characteristics[i];
						//读操作
						if (item.properties.read) {
							//该特征值是否支持 read 操作
							this.textLog = this.textLog + "该特征值支持 read 操作:" + item.uuid + "\n";
						}
						//写操作
						if (item.properties.write) {
							//该特征值是否支持 write 操作
							this.textLog = this.textLog  + "该特征值支持 write 操作:" + item.uuid + "\n";
							//可写uuid
							//可写的不一定有 notify 
							this.list.push({text:'特征值ID',subText:item.uuid,})
						}
						//该特征值是否支持 notify或indicate 操作
						if (item.properties.notify || item.properties.indicate) {
							this.textLog = this.textLog  +  "该特征值支持 notify 操作:" + item.uuid + "\n";
							//启动 notify 功能
							this.notifyBLECharacteristicValueChange(item.uuid);
						}
					}
				}
		  }); // that.onBLECharacteristicValueChange();   //监听特征值变化
		},
		//启用低功耗蓝牙设备特征值变化时的 notify 功能，订阅特征值。
		//注意：必须设备的特征值支持notify或者indicate才可以成功调用，具体参照 characteristic 的 properties 属性
		notifyBLECharacteristicValueChange: function (characteristicId) {
			var that = this;
			uni.notifyBLECharacteristicValueChange({
				state: true,
				// 启用 notify 功能
				deviceId: this.deviceId,
				serviceId: this.serviceId,
				characteristicId,
				success: (res) => {
					this.textLog = this.textLog  + "notify启动成功" + res.errMsg + "\n";
					this.onBLECharacteristicValueChange(); //监听特征值变化
				},
				fail: (err) =>{
					this.$u.toast(`notify启动失败${err}`);
				}
			});
		},
		//监听低功耗蓝牙设备的特征值变化。必须先启用notify接口才能接收到设备推送的notification。
		onBLECharacteristicValueChange: function () {
			uni.onBLECharacteristicValueChange((res) => {
				console.log('特征值变化');
				console.log(res);
				//转16进制字符串
				let resValue = utils.ab2hext(res.value); //16进制字符串
				//转字符串
				let resValueStr = utils.hexToString(resValue);
				this.textLog = this.textLog + "成功获取：" + resValueStr + "\n";
			});
		},
		//发送指令
		send(index){
			this.writeBLECharacteristicValue(this.list[index].subText)
		},
		//向低功耗蓝牙设备特征值中写入二进制数据。
		//注意：必须设备的特征值支持write才可以成功调用，具体参照 characteristic 的 properties 属性
		writeBLECharacteristicValue: function (characteristicId) {
			//指令处理
			let order = utils.stringToBytes(this.orderInputStr);
			let byteLength = order.byteLength;
			this.textLog = this.textLog + "当前执行指令的字节长度:" + byteLength + "\n";
			uni.writeBLECharacteristicValue({
				deviceId: this.deviceId,
				serviceId: this.serviceId,
				characteristicId,
				// 这里的value是ArrayBuffer类型 注意长度不要超过20
				value: order.slice(0, 20),
				success: (res)=> {
					// if (byteLength > 20) {
					// 	this.writeBLECharacteristicValue(order.slice(20, byteLength));
					// }
					this.textLog = this.textLog + "写入成功：" + res.errMsg + "\n";
				},
				fail: (res)=> {
					this.textLog = this.textLog + "写入失败" + res.errMsg + "\n";
				}
			});
		}
	}
};
</script>
<style>
	.btn{
		margin-bottom: 40rpx;
	}
	.BLE-detail{
		padding: 30rpx;
		font-size: 32rpx;
		color: #007AFF;
		font-weight: bold;
	}
	.card {
		border-radius: 10rpx;
		background-color: #fff;
		margin: 20rpx;
		padding: 20rpx;
		box-shadow: 0 2rpx 4rpx rgba(0,0,0,.3);
	}
	.list {
		max-height: 45vh;
	}
	.option{
		padding: 30rpx;
	}
</style>

