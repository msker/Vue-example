<template>
<view class="wrap" id="cart">
	<uni-section :padding="true" title="已选商品" type="line" :navigateType="2">
		<template v-slot:right>
			<text class="ext-icon mr-3 orange" v-html="isEdit ? '&#xe64c;' : '&#xe639;'"></text>
			<text @click="cartEdit" class="fs-13" v-if="cartItems.length>0">{{isEdit?'完成':'编辑'}}</text>
		</template>
	</uni-section>
	
	<view class="empty" v-if="cartItems.length==0 && !isLoading">
		购物车空空如也~
	</view>
	
	
	<view class="cart-items" v-if="!isLoading">
					
		<evan-checkbox-group v-model="activeCartIds" @change="changeItem">
			<view class="item under-line" v-for="(item,index) in cartItems" :key="item.cart_id">
				<label class="uni-list-cell uni-list-cell-pd check-item" :disabled="item.sku_stock==0" :class="{disabled:item.sku_stock==0}">
					<evan-checkbox class="checker" :label="item.cart_id" primary-color="#F90" :iconSize="12"></evan-checkbox>
				</label>
				<view class="pic" @click="toUrl('./detail'+item.goods_id)">
					<image :src="item.main_pic" />
				</view>
				<view class="detail">
					<view class="goods-name ellipsis2" @click="toUrl('./detail?id='+item.goods_id)">{{item.goods_name}}</view>
					<view class="sku-name">
						<text class="sku">
							<text class="tag">{{item.sku_name}}</text>
						</text>
						<text class="tip" v-if="itemTip(item)!=''">
							<text class="txt"><text class="ext-icon mr-3">&#xe606;</text>{{itemTip(item)}}</text>
						</text>
					</view>
					<view class="price-volume">
						<view class="price price-color">
							<text class="fs-12 mr-3">&yen;</text>
							<text class="fs-14">{{item.sku_price}}</text>
						</view>
						<view class="set-volume" v-if="!isEdit">
							<view class="volume">
								<view class="sub" :class="{'disabled':item.volume<=1}" @click="changeItemVolume(item,-1)"><text class="ext-icon">&#xe6f2;</text></view>
								<input class="num" type="number" min="1" v-model="item.volume" @input="checkItemVolume(item)">
								<view class="add" @click="changeItemVolume(item,1)"><text class="ext-icon">&#xe650;</text></view>
							</view>
						</view>
						<view class="action" v-if="isEdit">
							<view class="act delete" @click="cartItemDelete(item)">删除</view>
						</view>
					</view>
					
				</view>
			</view>
		</evan-checkbox-group>
	</view>
	
	<view class="foot-bar" v-if="cartItems.length>0">
	
		<view class="check-all" @click="checkAllItem">
			<text class="check" :class="{'checked':isCheckAll==true}"></text>
			<text class="txt">全选</text>
		</view>
		
		<view class="describe">
			<text class="amount price-color fs-16"><text class="fs-12 mr-3">&yen;</text>{{cartTotal.amount}}</text>
		</view>
		
		<view class="btns">
			
			<view class="btn-item">
				<button type="primary" :disabled="cartTotal.total==0" class="btn" @click="toConfirmOrder">去结算<text class="fs-12">({{cartTotal.total}}件)</text></button>
			</view>
		</view>
	</view>
</view>
</template>

<script>
import common from "@/static/js/common.js";
export default {
	name: "index",
	data() {
		return {
			cartItems:[],
			modifyCartItem:{},
			activeCartIds:[],
			onRequest:false,
			cartTotal:{
				count:0,
				total:0,
				amount:0
			},
			isEdit:false,
			isCheckAll:false,
			
			isLoading:false,
			hasMore:true,
			contentText:{
				contentdown: "点击加载更多..",
				contentrefresh: "正在获取更多产品..",
				contentnomore: "已加载全部产品",
			},
		};
	},
	onLoad() {
		if(common.checkLogin()){
			this.getCartItems();
		}
	},
	
	//下拉
	onPullDownRefresh(){
		this.getCartItems();
	},
	methods: {
		//缓存选中购物车ID
		setActiveCartIdsCache(){
			uni.setStorageSync('mallCartIds', JSON.stringify(this.activeCartIds));
		},
		
		//缓存购买商品
		setBuyItemsCache(){
			let that=this;
			let items = [];
			that.cartItems.forEach(function(item){
				if(that.activeCartIds.indexOf(item.cart_id)!=-1){
					items.push({
						cart_id: item.cart_id,
						sku_id: item.sku_id,
						volume: item.volume,
					})
				}
			})
			let itemStr = JSON.stringify(items);
			uni.setStorageSync('mallBuyItems', itemStr);
			uni.setStorageSync('mallBuyFrom', 'cart');
		},
		
		//全选购物车
		checkAllItem(){
			let newActiveCartIds=[];
			this.isCheckAll=!this.isCheckAll;
			if(this.isCheckAll==true){
				this.cartItems.forEach(function(item){
					if(item.sku_stock>0){
						newActiveCartIds.push(item.cart_id);
					}
				})
				this.activeCartIds=newActiveCartIds;
			}
			this.activeCartIds=newActiveCartIds;
			this.setActiveCartIdsCache();
			this.getActiveItemsTotal();
		},
		
		//单个商品选择状态改变
		changeItem(e){
			
			this.activeCartIds=e;
			this.setActiveCartIdsCache();
			this.getActiveItemsTotal();
			
			if(this.activeCartIds.length!=this.cartItems.length){
				this.isCheckAll=false;
			}else{
				this.isCheckAll=true;
			}
		},
		
		//根据缓存设置默认选中项
		setActiveCardIds(){
			let that=this;
			let ids=uni.getStorageSync('mallCartIds');
			if(!ids){
				return false;
			}
			ids=JSON.parse(ids);
			that.activeCartIds=ids;
			that.getActiveItemsTotal();
		},
		
		//删除选中项中的ID
		deleteActiveCardId(cart_id){
			let newActiveCartIds=[];
			this.activeCartIds.forEach(function(id,index){
				if(cart_id!=id){
					newActiveCartIds.push(id);
				}
			})
			this.activeCartIds=newActiveCartIds;
			this.setActiveCartIdsCache();
		},
		
		//获取购物车商品列表
		getCartItems(){
			let that=this;
			this.isLoading=true;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl,
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					action:"getCartItems",
				},
				success:(res)=> {
					uni.stopPullDownRefresh();
					this.isLoading=false;
					if (res.data.code == 1) {
						that.cartItems=res.data.data;
						that.setActiveCardIds();
					}else{
						uni.showToast({
							title: '购物车内容获取失败',
							icon:"none"
						});
					}
				}
			})
		},
		
		//编辑购物车
		cartEdit(){
			this.isEdit=!this.isEdit;
		},
		
		
		
		//变更购买项数量
		changeItemVolume(item,volume){
			let that=this;
			
			let oldVolume=parseInt(item.volume);
			let newVolume=oldVolume+volume;
			if(newVolume<=0 || isNaN(newVolume)){
				newVolume=1;
			}
			if(newVolume == oldVolume){
				return false;
			}
			
			that.modifyCartItem=item;
			item.volume=newVolume;
		},
		
		//检查购买项数量
		checkItemVolume(item){
			let that=this;
			let volume=parseInt(item.volume);
			if(isNaN(volume) || volume<=0){
				item.volume=1;
			}
		},
		
		//修改购买项数量	
		cartItemModify(sku_id,volume) {
			let that = this;
			return new Promise((resolve,reject)=>{
				uni.request({
					header:this.requestHeader,
					method:"POST",
					url:this.apiUrl, 
					data:{
						token:uni.getStorageSync('token'),
						model:"mall",
						action:"cartItemAdd",
						sku_id:sku_id,
						volume:volume,
						is_modify:1
					},
					success:(res)=> {
						resolve();
						if(res.data.code!=1){
							uni.showToast({
								title: '操作失败：'+res.data.msg,
								icon:"none",
								mask: true,
							});
						}else{
							that.isShowChangeSku = false;
							that.cartCount=res.data.data;
						}
					}
				})
			})
		},
		
		//删除购买项
		cartItemDelete(item){
			let that=this;
			let cart_id=item.cart_id;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl,
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					action:"cartItemDelete",
					cart_id:cart_id
				},
				success:(res)=> {
					if (res.data.code == 1) {
						that.getCartItems();
						that.deleteActiveCardId(cart_id);
					}else{
						uni.showToast({
							title: '删除未成功',
							icon:"none"
						});
					}
				}
			})
		},
		
		//购买项提示
		itemTip(item){
			return common.sub(item.sku_stock,item.volume)<0?'库存不足':'';
		},
		
		//获取选项商品金额
		getActiveItemsTotal(item){
			let that=this;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl, 
				data:{
					token:uni.getStorageSync('token'),
					model:"mall",
					"action":"getCartTotal",
					"cart_ids":JSON.stringify(that.activeCartIds)
				},
				success:(res)=> {
					if (res.data.code == 1) {
						that.cartTotal=res.data.data;
					}else{
						uni.showToast({
							title: res.data.msg,
							icon:"none"
						});
					}
					that.onRequest=false;
				}
			})
		},
		
		//去购买
		toConfirmOrder() {
			let that = this;
			that.setBuyItemsCache();
			that.toUrl("./confirmOrder");
		}
	},
	
	watch:{
		'modifyCartItem':{
			handler:function(item){
				let that=this;
				if(that.onRequest){
					return false;
				}
				if(item.volume<=0){
					return false;
				}
				that.onRequest=true;
				that.cartItemModify(item.sku_id,item.volume).then((res)=>{
					that.getActiveItemsTotal();
				});
			},
			deep:true
		}
	}
};
</script>
<style lang="scss">
#cart{
	
	.foot-bar .describe{
		min-width: 240rpx;
	}
	
	.cart-items{
		margin-bottom: 110rpx;
		background-color: #fff;
		padding:0 24rpx;
		.item{
			padding: 24rpx 0;
			padding-left: 0;
			display: flex;
			.check-item{
				flex: 1;
				position: relative;
				max-width: 60rpx;
				height: 180rpx;
				box-sizing: border-box;
				display: flex;
				line-height: 180rpx;
				.checker{
					width: 120rpx;
					.evan-checkbox{
						height: 180rpx;
					}
				}
			}
			
			.pic{
				flex: 1;
				max-width: 180rpx;
				margin: 0 24rpx 0 0;
				image{
					width: 180rpx;
					height: 180rpx;
					border-radius: 12rpx;
				}
			}
			
			.detail{
				flex: 1;
				
				.goods-name{
					font-size: 28rpx;
					margin-bottom: 10rpx;
					// font-weight: bold;
					line-height: 40rpx;
					height: 80rpx;
				}
				
				.sku-name{
					margin-bottom: 14rpx;
					display: flex;
					.sku{
						flex: auto;
						.tag{
							display: inline-block;
							border-radius: 18rpx;
							background: #F6F6F6;
							height: 38rpx;
							line-height: 38rpx;
							padding: 0 16rpx;
							color: #999999;
							overflow: hidden;
							font-size: 24rpx;
							transform:scale(0.916) translateX(-10rpx);
						}
					}
					.tip{
						flex: 1;
						max-width: 140rpx;
						min-width: 140rpx;
						text-align: right;
						.txt{
							font-size: 24rpx;
							color: #F00;
							display: inline-block;
							font-size: 24rpx;
							transform:scale(0.916) ;
						}
					}
				}
				
				.price-volume{
					display: flex;
					
					.price{
						flex: 1;
					}
					
					.set-volume {
						max-width: 160rpx;
						flex: auto;
						align-items: center;
						.volume {
							width: 100%;
							text-align: right;
							display: flex;
							align-items: center;
							height: 36rpx;
							.add,
							.sub,
							.num {
								max-width: 40rpx;
								min-width: 40rpx;
								display: inline-block;
								flex: 1;
								text-align: center;
								height: 36rpx;
								line-height: 36rpx;
							}
					
							.add,
							.sub {
								font-size: 24rpx;
								&:active {
									color: $second-color;
								}
					
								&.disabled {
									opacity: 0.3;
								}
							}
							.sub{
								text-align: left;
							}
							.add{
								text-align: right;
							}
					
							.num {
								max-width: 80rpx;
								min-width: 80rpx;
								border-radius: 4rpx;
								background-color: #F1F1F1;
								font-size: 24rpx;
								border: none;
							}
						}
					}
					
					
					.action{
						flex: auto;
						text-align: right;
						.act{
							padding: 0 10rpx;
							min-width: 80rpx;
							display: inline-block;
							border:2rpx solid $price-color;
							color: $price-color;
							height: 36rpx;
							border-radius: 24rpx;
							font-size: 24rpx;
							text-align: center;
						}
					}
				}
			}
		}
	}
	
}
</style>