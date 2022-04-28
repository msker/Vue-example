<template>
<view class="wrap orderDetail" id="mallOrderPay">
	<view class="wrapper">
		<view class="content">
			<!--头部-->
			<view class="order-top">
				<view class="title">
					<view class="icon"><view class="ext-icon" v-html="payData.status_set.icon"></view></view>
					<view class="text">
						{{payData.status_set.name}}
						<text class="fs-12 ml-3" v-if="payData.status_set.describle">（{{payData.status_set.describle}})</text>
					</view>
				</view>
			</view>
			
			<!--收货地址-->
			<view class="receive-info white-box">
				<view class="icon"><text class="ext-icon">&#xe610;</text></view>
				<view class="info">
					<view class="title" v-if="receive.receive_name">
						<text class="name">{{receive.receive_name}}</text>
						<text class="phone">{{receive.receive_phone}}</text>
					</view>
					<view class="detail" v-if="receive.receive_name">
						<view class="address">{{receive.province_name}}{{receive.receive_address}}</view>
					</view>
				</view>
			</view>
	
			
			<!--订单信息-->
			<view class="white-box cells">
				<view class="title">订单信息</view>
				<view class="cell">
					<text class="label grey">订单编号</text>
					<text class="value">{{payData.main_sn}}</text>
				</view>
				<view class="cell">
					<text class="label grey">下单时间</text>
					<text class="value">{{payData.create_time}}</text>
				</view>
				<view class="cell">
					<text class="label grey">购买数量</text>
					<text class="value">{{payData.volume}}</text>
				</view>
				<view class="cell">
					<text class="label grey">订单总价</text>
					<text class="value"><text>&yen;</text>{{payData.amount}}</text>
				</view>
				<view class="cell pay-amount">
					<text class="label grey">实付金额</text>
					<text class="value"><text>&yen;</text>{{payData.pay_amount}}</text>
				</view>
				<view class="cell" v-if="payData.remark!=''">
					<text class="label grey">订单备注</text>
					<text class="value">{{payData.remark}}</text>
				</view>
			</view>
			
			
			<!--付款方式-->
			<view class="white-box pay-ways">
				<radio-group v-model="payWay" type="radio" default-item-class="uncheck" selected-item-class="checked" :radio-required="true">
				<label class="item" v-for="(item,index) in payWays" :key="index" :class="{disabled:item.open!=1}">
					
					<view class="detail">
						<view class="icon" :style="'color:'+item.color"><view class="ext-icon" v-html="item.icon"></view></view>
						<view class="name">
							{{item.name}}
							<text class="desc">{{item.describe}}</text>
						</view>
					</view>
					
					<view class="check">
						<radio :value="index" :disabled="item.open!=1" :checked="item.default==1" />
					</view>
				</label>
				</radio-group>
			</view>

		</view>
	</view>
	
	<!--底部栏-->
	<view class="foot-bar">
		<view class="btns">
			<view class="btn-item">
				<button type="primary" :disabled="disabled" class="btn" @click="pay">立即付款&nbsp;&yen;{{payData.pay_amount}}</button>
			</view>
		</view>
	</view>
</view>
</template>
<script>
import common from "@/static/js/common.js";
export default {
	name: "orderPay",
	data() {
		return {
			mainSn:'',
			tradeNo:'',
			payData:{
				status_set:{
					icon:'',
					name:''
				}
			},
			receive:{},
			payWays:[],
			payWay:'',
			disabled:false,
			wxcode:'',
			platform:'',
		};
	},
	onLoad(e) {
		this.mainSn=e.order_sn
		this.getPlatForm();
		if(common.checkLogin()){
			this.getPayOrder();
		}
	},
	
	//下拉
	onPullDownRefresh(){
		this.getPayOrder();
	},
	
	methods: {
		getPlatForm(){
			let platform='wxmp';
			this.platform=platform;
		},
		
		
		//获取付款方式
		getpayWays(){
			let that=this;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl,
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					action:"getPayWays",
					platform:that.platform
				},
				success:(res)=> {
					if (res.data.code == 1) {
						let _datas=res.data.data;
						that.payWays=_datas;
						
						//默认选中
						let first='';
						let i=0;
						Object.keys(_datas).forEach((key)=>{
							i++;
							if(i==1){
								first=key;
							}
							if(_datas[key].default==1){
								that.payWay=key;
								return false;
							}
						})
						if(!that.payWay){
							that.payWay=first;
						}
					}else{
						uni.showToast({
							title: '支付方式获取失败',
							icon:"none"
						});
					}
				}
			})
		},
		
		//获取支付订单
		getPayOrder(){
			let that=this;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl,
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					action:"getPayOrder",
					main_sn:that.mainSn,
				},
				success:(res)=> {
					uni.stopPullDownRefresh();
					if (res.data.code == 1) {
						let _datas=res.data.data;
						that.payData=_datas.payData;
						that.receive=_datas.receive;
						if(that.payData.status!=1){
							uni.redirectTo({url:'./orderDetail'+that.payData.main_sn});
						}
						that.receive=_datas['receive'];
						
						that.getpayWays();
					}else{
						uni.showToast({
							title: res.data.msg,
							icon:"none"
						});
					}
				}
			})
		},
		
		
		//支付
		pay(){
			let that=this;
			
			//微信小程序
			if(that.payWay=="wxmp"){
				that.wxmp();
			}else{
				uni.showModal({
					title:"很抱歉",
					content:"暂未开通此支付方式",
					showCancel:false
				})
			}
		},
		
		
		//微信支付(小程序)
		wxmp(){
			let that=this;
			uni.login({
				async success(res) {
					that.wxcode=res.code;
					let payData=await that.requestPay();
					if(!payData.pay_params){
						return false;
					}
					uni.requestPayment({
					    timeStamp: ''+payData.pay_params.timeStamp,
					    nonceStr: payData.pay_params.nonceStr,
					    package: payData.pay_params.package,
					    signType: 'MD5',
					    paySign: payData.pay_params.paySign,
					    success() {
					       that.payResult();
					    },
					    fail: (res) => {
							console.log(res)
							if(res.errMsg=='requestPayment:fail cancel'){
								uni.showToast({
								    title: "您已取消支付",
									icon:"none"
								})
							}else{
								uni.showToast({
								    title: "支付失败",
									icon:"none"
								})
							}
					    }
					})
				}
			})
			
		},
		
		//请求支付
		requestPay(){
			let that=this;
			that.disabled=true;
			return new Promise((resolve,reject) => {
				uni.showLoading({
					title:'正在请求付款'
				})
				uni.request({
					header:{json:1},
					method:"POST",
					url:this.apiUrl,
					data:{
						token:uni.getStorageSync('token'),
						model:"mall",
						action:"orderPay",
						main_sn:that.mainSn,
						pay_way:that.payWay,
						wxcode:that.wxcode,
					},
					success:(res)=> {
						uni.hideLoading();
						that.disabled=false;
						if (res.data.code == 1) {
							that.tradeNo=res.data.data.trade_no;
							resolve(res.data.data);
						}else{
							reject();
							uni.showToast({
								title: res.data.msg,
								icon:"none"
							});
						}
					},
					fail:(res)=>{
						uni.hideLoading();
						reject();
					}
				})
			});
		},
		
		
		//查询付款结果
		payResult(){
			let that=this;
			console.log(that.tradeNo)
			uni.showLoading({
				title: '确认付款结果'
			});
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl,
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					action:"payResult",
					trade_no:that.tradeNo
				},
				success:(res)=> {
					uni.hideLoading();
					if (res.data.code == 1) {
						that.paySuccess();
					}else{
						uni.showToast({
							title: res.data.msg,
							icon:"none"
						});
					}
				}
			})
		},
		
		
		//付款完成
		paySuccess(){
			let that=this;
			uni.showToast({
				title: '付款成功',
				success:()=>{
					setTimeout(function(){
						uni.setStorageSync('orderType', 'mall');
						uni.setStorageSync('orderStatus', '');
						that.toUrl('/pages/order/list','redirect');
					},1000);
				}
			});
		}
		
	},
};
</script>
<style lang="scss">
@import '@/static/css/order-detail.scss';
@import '@/static/css/mall-receive.scss';
@import '@/static/css/pay-ways.scss';
</style>