<template>
<view class="wrap orderDetail" id="travelOrderPay">
	<view class="wrapper">
		<view class="content">
			<!--头部-->
			<view class="order-top">
				<view class="title">
					<view class="icon"><view class="ext-icon" v-html="orderInfo.status_set[2]"></view></view>
					<view class="text">
						{{orderInfo.status_set[0]}}
						<text class="fs-12 ml-3" v-if="orderInfo.status_describle">（{{orderInfo.status_describle}})</text>
					</view>
				</view>
			</view>
			
			<!--产品预览-->
			<view class="travel-preview white-box">
				<view class="img">
					<image :src="orderInfo.pic" class="img"></image>
				</view>
				
				<view class="detail">
				  <view class="title ellipsis2">{{orderInfo.title}}</view>
				  <view class="other">
					<view>出发日期：<text>{{orderInfo.true_date}}</text></view>
				  </view>
				</view>
			</view>
	
			
			<!--订单信息-->
			<view class="white-box cells">
				<view class="title">订单信息</view>
				<view class="cell">
					<text class="label grey">订单编号</text>
					<text class="value">{{orderInfo.order_sn}}</text>
				</view>
				<view class="cell">
					<text class="label grey">下单时间</text>
					<text class="value">{{orderInfo.create_time}}</text>
				</view>
				<view class="cell">
					<text class="label grey">出行人数</text>
					<text class="value">{{orderInfo.person}}</text>
				</view>
				<view class="cell">
					<text class="label grey">团费金额</text>
					<text class="value"><text>&yen;</text>{{orderInfo.amount}}</text>
				</view>
				<view class="cell">
					<text class="label grey">保险金额</text>
					<text class="value"><text>&yen;</text>{{orderInfo.member_fee}}</text>
				</view>
				<view class="cell pay-amount">
					<text class="label grey">实付金额</text>
					<text class="value"><text>&yen;</text>{{orderInfo.total_amount}}</text>
				</view>
				<view class="cell" v-if="orderInfo.remark!=''">
					<text class="label grey">订单备注</text>
					<text class="value">{{orderInfo.remark}}</text>
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
				<button type="primary" :disabled="disabled" class="btn" @click="pay">立即付款&nbsp;&yen;{{orderInfo.total_amount}}</button>
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
			order_sn:'',
			tradeNo:'',
			orderInfo:{
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
		this.order_sn=e.order_sn
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
						let first=';'
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
					model:"travel",
					action:"getPayOrder",
					order_sn:that.order_sn,
				},
				success:(res)=> {
					uni.stopPullDownRefresh();
					if (res.data.code == 1) {
						that.orderInfo=res.data.data;
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
					let payRes=await that.requestPay();
					if(!payRes.pay_params){
						that.toast('支付请求失败')
						return false;
					}
					uni.requestPayment({
					    timeStamp: ''+payRes.pay_params.timeStamp,
					    nonceStr: payRes.pay_params.nonceStr,
					    package: payRes.pay_params.package,
					    signType: 'MD5',
					    paySign: payRes.pay_params.paySign,
					    success() {
					       that.payResult();
					    },
					    fail: (res) => {
							if(res.errMsg=='requestPayment:fail cancel'){
								that.toast('您已取消支付')
							}else{
								that.toast('支付失败')
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
						model:"travel",
						action:"orderPay",
						order_sn:that.order_sn,
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
							that.toast(res.data.msg);
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
					model:"travel",
					action:"payResult",
					trade_no:that.tradeNo
				},
				success:(res)=> {
					uni.hideLoading();
					if (res.data.code == 1) {
						that.paySuccess();
					}else{
						that.toast(res.data.msg);
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
						uni.setStorageSync('orderType', 'travel');
						uni.setStorageSync('orderStatus', '');
						that.toUrl('/pages/order/list','redirect');
					},1000);
				}
			});
		}
		
	}
};
</script>
<style lang="scss">
@import '@/static/css/order-detail.scss';
@import '@/static/css/travel-preview.scss';
@import '@/static/css/pay-ways.scss';
</style>