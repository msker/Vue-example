<template>
<view class="main orderDetail" id="mallConfirmOrder">
	
	<!--收货地址-->
	<view class="white-box receive-info mt-10" @click="receiveEmpty?toReceiveEdit():toReceives()">
		<view class="icon"><text class="ext-icon">&#xe610;</text></view>
		<view class="info">
			<view class="title" v-if="receiveInfo.receive_name">
				<text class="name">{{receiveInfo.receive_name}}</text>
				<text class="phone">{{receiveInfo.receive_phone}}</text>
			</view>
			<view class="detail" v-if="receiveInfo.receive_name">
				<view class="address">{{receiveInfo.province_name}}{{receiveInfo.receive_address}}</view>
			</view>
			
			<view class="receive-empty" v-if="!receiveInfo.receive_name">
				新增收货地址
			</view>
		</view>
		
		<view class="action"><text class="ext-icon">&#xe9a8;</text></view>
	</view>
	
	<!--购买商品项-->
	<view class="white-box buy-items" v-if="isLoading">
		<view class="loading">
			<uni-load-more status="loading" :iconSize="16"></uni-load-more>
		</view>
	</view>
	
	<template v-for="group in buyGroups">
	<view class="white-box cells buy-items" v-if="!isLoading">
		<view class="title">购买商品</view>
		<view class="item" v-for="item in group.buyItems" :key="item.sku_id" @click="toUrl('./detail?id='+item.goods_id)">
			<view class="body">
				<view class="pic"><img :src="item.main_pic"></view>
				<view class="detail">
					<view class="goods-name ellipsis2">{{item.goods_name}}</view>
					<view class="sku">{{item.sku_name}}</view>
					<view class="price-volume">
						<text class="price"><text class="fs-12 mr-3">&yen;</text>{{item.sku_price}}</text>
						<text class="volume">×{{item.volume}}</text>
					</view>
					<view class="desc" v-if="buildItemTips(item).tips!=''"  :class="{danger:buildItemTips(item).isError}">
						<text class="ext-icon mr-3">&#xe606;</text>{{buildItemTips(item).tips}}
					</view>
				</view>
			</view>
		</view>
		<view class="cell remark-cell">
			<view class="label">订单备注</view>
			<view class="value">
				<input type="text" class="remark" v-model="group.remark" placeholder="选填，如需告知我们请填写">
			</view>
		</view>
	</view>
	</template>
	
	<!--金额-->
	<view class="white-box cells">
		<view class="cell">
			<text class="label">商品总价</text>
			<text class="value"><text>&yen;</text>{{totalAmount}}</text>
		</view>
		<view class="cell pay-amount">
			<text class="label">订单金额</text>
			<text class="value"><text>&yen;</text>{{payAmount}}</text>
		</view>
	</view>
	
	
	<!--底部栏-->
	<view class="foot-bar">
		<view class="describe">
			实付款：<text class="amount price-color fs-16"><text class="fs-12 mr-3">&yen;</text>{{payAmount}}</text>
		</view>
		<view class="btns">
			<view class="btn-item">
				<button type="primary" :disabled="disabledBuy" class="btn" @click="createOrder">提交订单</button>
			</view>
		</view>
	</view>
</view>
</template>

<script>
	import common from "@/static/js/common.js";
	export default {
		name: "confirmOrder",
		data() {
			return {
				//来源
				mallBuyFrom:uni.getStorageSync('mallBuyFrom'),
				
				//购买项
				isLoading:false,
				isSubmit:false,
				buyGroups:[],
				disabledBuy:false,
				cartIds:[],
				
				//收货地址
				receiveEmpty:true,
				receive_id:'',
				receiveInfo:{},
				
				//金额
				totalAmount:0,//商品总额
				reachLimitMax:0,//最大满额（满减券）
				payAmount:0,//实付金额
				
				
				//错误信息
				isError:false,
				errorMessage:'',
			};
		},
		onLoad(e) {
			if(common.checkLogin()){
				this.getReceiveDetail();
				this.getBuyItems();
			}
		},
		methods: {
			
			//获取购买项商品信息
			getBuyItems(){
				let that=this;
				//从缓存中获取购买项
				let buyItemsCache=uni.getStorageSync('mallBuyItems');
				if(!buyItemsCache){
					uni.redirectTo({url:'./cart'});
					return false;
				}
				
				buyItemsCache=JSON.parse(buyItemsCache);
				
				//购物车ID
				that.cartIds=buyItemsCache.map((item)=>{
					return item.cart_id;
				})
				
				
				//SKU ID
				let skuIds=[];
				skuIds=buyItemsCache.map((item)=>{
					return item.sku_id;
				})
				
				
				//获取商品信息
				that.isLoading=true;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl,
					data:{
						token:uni.getStorageSync('token'),
						model:"mall",
						action:"getBuyItems",
						sku_ids:JSON.stringify(skuIds)
					},
					success:(res)=> {
						if (res.data.code == 1) {
							let _datas=res.data.data;
							let newDatas=[];
							let lastSupplierId='';
							
							let buyGroups=[];
							let items=[];
							
							_datas.forEach(function(data){
								if(data.supplier_id!=lastSupplierId){
									items=[];
								}
								data.volume=buyItemsCache.find((item)=>item.sku_id == data.sku_id).volume;
								data.amount=common.mul(data.volume,data.sku_price);
								items.push(data);
								
								if(data.supplier_id!=lastSupplierId){
									buyGroups.push({
										groupTitle:data.supplier_name,
										supplier_id:data.supplier_id,
										buyItems:items,
										remark:''
									})
									lastSupplierId=data.supplier_id;
								}
							})
							
							that.buyGroups=buyGroups;
							
							//计算金额
							that.calcTotalAmount();
								
							that.isLoading=false;
						}else{
							uni.showToast({
								title: res.data.msg || '获取购买商品失败',
								icon:"none"
							});
						}
					}
				})
			},
			
			//整理购买项Tips
			buildItemTips(item){
				let tips='';
				let isError=false;
				if(item.goods_status!=1){
					tips='该商品已下架';
					isError=true;
					this.disabledBuy=true;
				}
				else if(item.sku_status!=1){
					tips='该规格不可购买';
					isError=true;
					this.disabledBuy=true;
				}
				else if(parseInt(item.sku_stock)<parseInt(item.volume)){
					tips='库存不足，仅剩'+item.sku_stock+'件';
					isError=true;
					this.disabledBuy=true;
				}
				
				return {
					tips:tips,
					isError:isError
				};
			},
			
			//计算总价
			calcTotalAmount(){
				let that=this;
				let totalAmount=0;
				let reachLimitMax=0;
				that.buyGroups.forEach((group)=>{
					group.buyItems.forEach((item)=>{
						totalAmount=common.add(totalAmount,common.mul(item.sku_price,item.volume));
					})
				})
				
				that.totalAmount=totalAmount;
				that.reachLimitMax=reachLimitMax;
			},
			
			//计算实付
			calcPayAmount(){
				let that=this;
				that.payAmount=that.totalAmount;
			},
			
			//获取收货地址详情
			getReceiveDetail(){
				let that=this;
				
				//先从缓存读取
				let receiveInfoCache=uni.getStorageSync('receiveInfoCache');
				if(receiveInfoCache){
					that.setReceiveInfo(receiveInfoCache);
					return false;
				}else{
					uni.request({
						header:this.requestHeader,
						url:this.apiUrl,
						data:{
							token:uni.getStorageSync('token'),
							model:'mall',
							action:'getReceiveDetail'
						},
						success:(res)=> {
							if (res.data.code == 1) {
								let _data=res.data.data;
								if(_data){
									that.setReceiveInfo(_data);
									uni.setStorageSync('receiveInfoCache',_data);
								}
							}else{
								uni.showToast({
									title: res.data.msg,
									icon:"none"
								});
							}
						}
					})
				}
			},
			
			//设置收货信息
			setReceiveInfo(receiveInfo){
				let that=this;
				//receiveInfo=JSON.parse(receiveInfo);
				that.receiveInfo=receiveInfo;
				that.receive_id=that.receiveInfo.receive_id;
				that.receiveEmpty=false;
			},
			
			//选择收货地址
			toReceives(){
				uni.navigateTo({url:'/pages/member/receives?select=1&receive_id='+this.receive_id})
			},
			
			//新增收货地址
			toReceiveEdit(){
				uni.navigateTo({url:'/pages/member/receiveEdit?select=1'});
			},
			
			//获取金额字符长度
			getAmountLength(item){
				return item.balance.toString().length;
			},
			
			//组织订单参数
			buildOrderParams(){
				let that=this;
				
				//收货地址
				let receiveInfo=that.receiveInfo;
				let receive={
					province_id:receiveInfo.province_id,
					province_name:receiveInfo.province_name,
					receive_address:receiveInfo.receive_address,
					receive_name:receiveInfo.receive_name,
					receive_phone:receiveInfo.receive_phone,
				}
				
				//购买商品项
				let buyGroups=[];
				that.buyGroups.forEach((group)=>{
					buyGroups.push({
						supplier_id:group.supplier_id,
						remark:group.remark,
						buy_items:group.buyItems.map((item)=>{
							return {
								'sku_id':item.sku_id,
								'volume':item.volume,
								'sku_price':item.sku_price
							}
						})
					})
				})
				
				//组合参数
				let params={
					receive:receive,
					buy_groups:buyGroups,
					cart_ids:that.cartIds,
					total_amount:that.totalAmount,
					pay_amount:that.payAmount,
					buy_from:that.mallBuyFrom
				}
				
				return params;
			},
			
			//检查订单参数
			checkOrderParams(params){
				if(!params.receive.receive_phone){
					this.setError('请选择收货地址');
					return false;
				}
				if(params.buy_groups.length==0){
					this.setError('缺少购买项');
					return false;
				}
				if(params.total_amount<=0){
					this.setError('订单金额非法');
					return false;
				}
				this.isError=false;
				this.errorMessage='';
			},
			
			//设置错误
			setError(errorMessage){
				this.isError=true;
				this.errorMessage=errorMessage;
			},
			
			//创建订单
			createOrder(){
				let that=this;
				if(that.isSubmit==true){
					uni.showToast({
						title: '请勿重复提交',
						icon:"none"
					});
					return false;
				}
				
				let params=that.buildOrderParams();
				params=JSON.parse(JSON.stringify(params));
				
				that.checkOrderParams(params);
				if(that.isError){
					uni.showToast({
						title: that.errorMessage
					});
					return false;
				}
				
				
				that.isSubmit=true;
				that.disabledBuy=true;
				uni.showLoading({
					title: '正在提交订单'					
				});
				params.token=uni.getStorageSync('token');
				params.model="mall";
				params.action="orderCreate";
				uni.request({
					header: {'json': 1},
					method:"POST",
					url:this.apiUrl,
					data:params,
					success:(res)=> {
						uni.hideLoading();
						if (res.data.code == 1) {
							let _data=res.data.data;
							let mainSn=_data.main_sn;
							that.clearBuyItemsCache();
							uni.redirectTo({url:'./orderPay?order_sn='+mainSn});
						}else{
							that.isSubmit=false;
							that.disabledBuy=false;
							uni.showToast({
								title: res.data.msg,icon:"none"
							});
						
						}
					},
				})
			},
			
			//清空购买项缓存
			clearBuyItemsCache(){
				uni.removeStorageSync('mallBuyItems');
				uni.removeStorageSync('mallBuyFrom');
			}
		},
		
		watch:{
			totalAmount(){
				this.calcPayAmount();
			},
		},
	};
</script>
<style lang="scss">
@import "@/static/css/order-detail.scss";
@import '@/static/css/mall-receive.scss';
</style>