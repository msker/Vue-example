<template>
	<view class="wrap orderDetail" id="travelOrderDetail">
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
				</view>

				<!--付款信息-->
				<view class="white-box cells">
					<view class="title">付款信息</view>
					<view class="cell">
						<text class="label grey">团费金额</text>
						<text class="value"><text>&yen;</text>{{orderInfo.amount}}</text>
					</view>
					<view class="cell">
						<text class="label grey">保险金额</text>
						<text class="value"><text>&yen;</text>{{orderInfo.member_fee}}</text>
					</view>
					<view class="cell" v-if="orderInfo.pay_amount">
						<text class="label grey">订单总额</text>
						<text class="value bold price-color"><text>&yen;</text>{{orderInfo.total_amount}}</text>
					</view>
					<view class="cell" v-if="orderInfo.pay_time">
						<text class="label grey">付款时间</text>
						<text class="value">{{orderInfo.pay_time}}</text>
					</view>
					<view class="cell" v-if="orderInfo.pay_way">
						<text class="label grey">付款方式</text>
						<text class="value">{{orderInfo.pay_way.name}}</text>
					</view>
				</view>
				
				<!--退款信息-->
				<view class="white-box cells" v-if="orderInfo.refund_set">
					<view class="title">退款信息</view>
					<view class="cell">
						<text class="label grey">退款时间</text>
						<text class="value">{{orderInfo.refund_set.refund_time}}</text>
					</view>
					<view class="cell">
						<text class="label grey">退款金额</text>
						<text class="value"><text>&yen;</text>{{orderInfo.refund_set.refund_amount}}</text>
					</view>
					<view class="cell">
						<text class="label grey">退款状态</text>
						<text class="value">{{orderInfo.refund_set.refund_status}}</text>
					</view>
					<view class="cell">
						<text class="label grey">退款原因</text>
						<text class="value">{{orderInfo.refund_set.refund_cause}}</text>
					</view>
				</view>

			</view>

			<!--退款窗口-->
			<view>
				<uni-popup ref="refundPopup">
					<OrderRefund page="detail" type="mall" :item="refundItem" />
				</uni-popup>
			</view>

		</view>

		<!--底部栏-->
		<view class="action-bar">
			<OrderAction page="detail" :data="orderInfo" type="travel" style="width: 100%;" />
		</view>
	</view>
</template>
<script>
	import common from "@/static/js/common.js";
	import OrderAction from "@/components/Common/OrderAction.vue";
	import OrderRefund from "@/components/Common/OrderRefund.vue";
	import OrderDeliver from "@/components/Common/OrderDeliver.vue";
	export default {
		name: "orderPay",
		data() {
			return {
				orderSn: '',
				orderInfo: {
					actions: [],
					pay: {},
					refunds: [],
					status_set: {
						icon: '',
						name: ''
					},
				},
				buyItems: [],
				receive: {},

				showRefund: false,
				refundItem: {
					goods: {},
					amount: 0,
					pay_amount: 0,
					refund_amount: 0,
				},

				deliverData: {},
			};
		},
		onLoad(e) {
			let that=this;
			that.orderSn = e.order_sn;

			if (common.checkLogin()) {
				that.getOrderDetail();
			}
			//#ifdef H5 
			uni.$on('getOrderDetail',function(data){
				that.getOrderDetail(data);
			})
			//#endif
			
			//#ifdef H5 
			uni.$on('confirm_receive',function(data){
				that.confirm_receive(data);
			})
			//#endif
			
			//#ifdef H5 
			uni.$on('show_deliver',function(data){
				that.show_deliver(data);
			})
			//#endif
			
			//#ifdef H5 
			uni.$on('hide_deliver',function(){
				that.hide_deliver();
			})
			//#endif
			
			//#ifdef H5 
			uni.$on('showRefundWin',function(data){
				that.showRefundWin(data);
			})
			//#endif
			
			//#ifdef H5 
			uni.$on('hideRefundWin',function(){
				that.hideRefundWin();
			})
			//#endif
			 
		},
		
		//下拉
		onPullDownRefresh(){
			this.getOrderDetail();
		},
		
		methods: {

			//返回
			back() {
				uni.navigateBack();
			},

			//自定义回退
			goBack() {
				if (this.showRefund == true) {
					this.showRefund = false;
				}
			},

			//确认收货
			confirm_receive(order) {
				let order_sn = order.order_sn;
				let that = this;
				uni.request({
					header:this.requestHeader,
					url: this.apiUrl,
					data: {
						token:uni.getStorageSync('token'),
						model:"mall",
						action:"orderComplete",
						order_sn: order_sn,
						role:'member',
					},
					success: (res) => {
						if (res.data.code == 1) {
							that.getOrderDetail();
						} else {
							uni.showToast({
								title: res.data.msg,
								icon: "none"
							});
						}
					}
				})
			},

			//查看物流
			show_deliver(order) {
				this.deliverData = order.deliver;
				this.$refs.deliverPopup.open();
			},
			hide_deliver() {
				this.$refs.deliverPopup.close();
			},

			//显示退款窗口
			showRefundWin(item) {
				this.$refs.refundPopup.open();
				this.showRefund = true;
				this.refundItem = item;
			},

			//隐藏退款窗口
			hideRefundWin() {
				this.$refs.refundPopup.close();
			},

			//获取订单详情
			getOrderDetail() {
				let that = this;
				uni.request({
					header:this.requestHeader,
					url: this.apiUrl,
					data: {
						token:uni.getStorageSync('token'),
						model:"travel",
						action:"getOrderDetail",
						order_sn: that.orderSn,
						role:"member"
					},
					success: (res) => {
						uni.stopPullDownRefresh();
						if (res.data.code == 1) {
							let _data = res.data.data;
							that.orderInfo = _data;
						} else {
							uni.showToast({
								title: res.data.msg,
								icon: "none"
							});
						}
					}
				})
			},

		},
		components: {
			OrderAction,
			OrderRefund,
			OrderDeliver
		},
	};
</script>
<style lang="scss">
@import "@/static/css/order-detail.scss";
@import '@/static/css/travel-preview.scss';
</style>
