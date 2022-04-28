<template>
	<view class="wrap orderDetail" id="orderDetail">
		<view class="wrapper">
			<view class="content">
				<!--头部-->
				<view class="order-top">
					<view class="title">
						<view class="icon">
							<view class="ext-icon" v-html="orderInfo.status_set.icon"></view>
						</view>
						<view class="text">
							{{orderInfo.status_set.name}}
							<text class="fs-12 ml-3" v-if="orderInfo.status_set.describle">（{{orderInfo.status_set.describle}})</text>
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

				<!--购买商品项-->
				<template v-for="order in orderInfo.orders">
					<view class="white-box cells buy-items" :key="order.order_sn">
						<view class="title">
							<text class="supplier">
								<text class="ext-icon icon">&#xe6d8;</text>{{order.supplier.supplier_name}}
							</text>
							<text class="status"
								:class="order.order_status_set.color">{{order.order_status_set.name}}</text>
						</view>
						<view class="item" v-for="item in order.items" :key="item.sku_id">
							<view class="body" @click="toUrl('./detail?id='+item.goods_id)">
								<view class="pic"><img :src="item.goods.main_pic"></view>
								<view class="detail">
									<view class="goods-name">{{item.goods.goods_name}}</view>
									<view class="sku">{{item.sku_name}}</view>
									<view class="price-volume">
										<text class="price"><text
												class="fs-12 mr-3">&yen;</text>{{item.item_price}}</text>
										<text class="volume">×{{item.item_volume}}</text>
									</view>
								</view>
							</view>

							<view class="item-foot" v-if="item.actions.length>0 || item.refund">
							
								<view class="status">
									<template v-if="item.refund">
										<text class="fs-10"
											:class="item.refund_status_set.color">{{item.refund_status_set.name}}</text>
										<text class="fs-10"
											v-if="(item.refund.refund_status==3 || item.refund.refund_status==4) && item.refund.check_message!=''">{{item.refund.check_message}}</text>
									</template>
								</view>
								<OrderAction v-if="item.actions.length>0" page="detail" type="mall" class="small-action" :data="item" />
							</view>
						</view>

						<view class="cell remark-cell" v-if="order.order_remark">
							<view class="label grey">订单备注</view>
							<view class="value">
								<text class="remark-text">{{order.order_remark}}</text>
							</view>
						</view>
						<view class="action-bar" v-if="order.actions.length>0">
							<OrderAction page="detail" :data="order" type="mall" style="width: 100%;" />
						</view>
					</view>
				</template>


				<!--订单信息-->
				<view class="white-box cells">
					<view class="title">订单信息</view>
					<view class="cell">
						<text class="label grey">订单编号</text>
						<text class="value">{{orderInfo.main_sn}}</text>
					</view>
					<view class="cell">
						<text class="label grey">下单时间</text>
						<text class="value">{{orderInfo.create_time}}</text>
					</view>
					<view class="cell" v-if="orderInfo.pay && orderInfo.pay.pay_status==2">
						<text class="label grey">付款时间</text>
						<text class="value">{{orderInfo.pay.pay_time}}</text>
					</view>
					<view class="cell" v-if="orderInfo.pay && orderInfo.pay.pay_status==2">
						<text class="label grey">付款方式</text>
						<text class="value">{{orderInfo.pay.pay_way_name}}</text>
					</view>
				</view>

				<!--付款信息-->
				<view class="white-box cells">
					<view class="title">付款信息</view>
					<view class="cell">
						<text class="label grey">订单总额</text>
						<text class="value"><text>&yen;</text>{{orderInfo.amount}}</text>
					</view>
					<view class="cell">
						<text class="label grey">付款金额</text>
						<text class="value bold price-color"><text>&yen;</text>{{orderInfo.pay_amount}}</text>
					</view>
					<view class="cell" v-if="orderInfo.refund_amount>0">
						<text class="label grey">退款金额</text>
						<text class="value"><text>&yen;</text>{{orderInfo.refund_amount}}</text>
					</view>
				</view>

			</view>

			<!--退款窗口-->
			<view>
				<uni-popup ref="refundPopup">
					<OrderRefund page="detail" type="mall" :item="refundItem" />
				</uni-popup>
			</view>

			<view>
				<uni-popup ref="deliverPopup" type="center">
					<OrderDeliver page="detail" :data="deliverData" />
				</uni-popup>
			</view>

		</view>

		<!--底部栏-->
		<view class="action-bar">
			<OrderAction page="detail" :data="orderInfo" type="mall" style="width: 100%;" />
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
						model:"mall",
						action:"getOrderDetail",
						main_sn: that.orderSn,
						role:"member"
					},
					success: (res) => {
						uni.stopPullDownRefresh();
						if (res.data.code == 1) {
							let _data = res.data.data;
							that.orderInfo = _data;
							that.receive = _data['receive'];
							that.buyItems = _data['items'];
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
@import '@/static/css/mall-receive.scss';
.buy-items{
	.action-bar{
		position: relative;
		box-shadow: none;
	}
}

</style>
