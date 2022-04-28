<template>
	<view class="wrap" id="detail">
		
		<view class="main-pics">
			<swiper style="height: 100vw;" :indicator-dots="true" indicator-color="#000" indicator-active-color="#FFE000" @change="picChange">
				<swiper-item v-for="(item,index) in goodsInfo.main_pics" :key="index" show-dots="false" @click="previewImg">
					<image :src="item" :id="'img'+index" :preview="1"></image>
				</swiper-item>
			</swiper>
			
		</view>

		<!--基础信息-->
		<view class="basic-info">
			<!--特别信息-->
			<view class="special-info" v-if="hasSpecial">
				<view class="fs-12" :class="specialInfo.class_name">
					<view v-if="specialInfo.icon!=''" class="ext-icon mr-3 bold d-inline" v-html="specialInfo.icon"></view>
					<view v-html="specialInfo.text" class="d-inline"></view>
				</view>
			</view>
			
			<view class="flexs items-center">
				<view class="flex price t-left price-color bold">
					<text class="fs-12 mr-3">&yen;</text>
					<text class="fs-20">{{goodsInfo.price}}</text>
				</view>
				<view class="flex t-right">
					<view class="commission-tag" v-if="goodsInfo.commission>0">
						<text class="tag">返佣</text>
						<text class="value">&yen;{{goodsInfo.commission}}</text>
					</view>
				</view>
			</view>
			

			<view class="goods-name">
				{{goodsInfo.goods_name}}
			</view>

			<view class="feature">
				{{goodsInfo.feature}}
			</view>

			<view class="sku-select-view" @click="showChangeSku(0)">
				<view class="sku-view ellipsis" v-if="activeSku.id">
					已选：{{activeSku.name}}，{{activeSku.volume}}件
				</view>
				<view class="sku-view ellipsis" v-if="!activeSku.id">
					请选择规格
				</view>
				<view class="sku-select" v-if="canBuy">
					<text class="txt">选择更多</text>
					<text class="ext-icon">&#xe9a8;</text>
				</view>
			</view>
		</view>

		<!--详细描述-->
		<view class="content">
			<h3 class="title">商品详情</h3>
			<view class="deliver">
				<view class="item"><text class="label">分类</text><text class="value">{{goodsInfo.cname}}</text></view>
				<view class="item"><text class="label">供应商</text><text class="value">{{goodsInfo.supplier_name}}</text></view>
				<view class="item"><text class="label">发货地</text><text class="value">{{goodsInfo.deliver_name}}</text></view>
			</view>

			<view class="description" v-if="goodsInfo.content!=''" v-html="goodsInfo.content">
			</view>
			<view class="content-pics" v-if="goodsInfo.content_pics.length>0">
				<image :src="item" v-for="(item,index) in goodsInfo.content_pics" :key="index" mode="widthFix" @click="previewContentImg(index)"></image>
			</view>
		</view>

		<view class="foot-bar">
			<view class="links">
				<view class="link" @click="toUrl('./index')">
					<text class="ext-icon icon">&#xe601;</text>
					<text class="txt">首页</text>
				</view>
				<!-- <view class="link" @click="showShareMenus">
					<text class="ext-icon icon">&#xe62c;</text>
					<text class="txt">分享</text>
				</view> -->
				<view class="link" @click="showService=true">
					<text class="ext-icon icon">&#xe6df;</text>
					<text class="txt">客服</text>
				</view>
				<view class="link cart-link" @click="toUrl('./cart')">
					<uni-badge class="badge" :text="cartCount.count" size="small" type="error" v-if="cartCount.count>0"></uni-badge>
					<text class="ext-icon icon">&#xe60e;</text>
					<text class="txt">购物车</text>
				</view>
			</view>
			
			<view class="btns">
				<view class="btn-item">
					<button type="default" class="btn" @click="showChangeSku(1)" :disabled="!canBuy">加入购物车</button>
				</view>
				<view class="btn-item">
					<button type="default" class="btn" @click="showChangeSku(2)" :disabled="!canBuy">立即购买</button>
				</view>
			</view>
		</view>

		<!--选择SKU-->
		<uni-popup ref="popup" type="bottom">
			<view class="change-sku-wrap">
				<view class="close" @click="$refs.popup.close()"><text class="ext-icon">&#xe633;</text></view>
		
				<view class="select-sku" v-if="activeSku.id">
					<view class="price-wrap">&yen;<text class="price">{{activeSku.price}}</text></view>
					<view class="name-wrap">已选择<text class="name">{{activeSku.name}}<text class="ml-3 mr-3">x</text>{{activeSku.volume}}</text></view>
				</view>
		
				<view class="all-sku">
					<view class="title">规格</view>
					<view class="skus">
						<text class="sku" v-for="(item,index) in skus" :key="index" :class="{active:item.sku_id==activeSku.id,disabled:item.sku_stock==0}" @click="changeSku(item)">
							{{item.sku_name}}
						</text>
					</view>
				</view>
		
				<view class="set-volume">
					<view class="title">数量</view>
					<view class="volume">
						<text class="sub" :class="{'disabled':activeSku.volume<=1}" @click="changeVolume(-1)"><text class="ext-icon">&#xe6f2;</text></text>
						<input class="num" type="number" min="1" v-model="activeSku.volume">
						<text class="add" @click="changeVolume(1)"><text class="ext-icon">&#xe650;</text></text>
					</view>
				</view>
		
				<view class="btns">
					<view class="btn-item" v-if="!changeSkuFlag || changeSkuFlag==1"><button type="primary" class="btn" @click="selectSku(1)">加入购物车</button></view>
					<view class="btn-item" v-if="!changeSkuFlag || changeSkuFlag==2"><button type="primary" class="btn" @click="selectSku(2)">立即购买</button></view>
				</view>
			</view>
		</uni-popup>
		
		<!--客服-->
		<service :show="showService" @hide="showService=false" />
	</view>
</template>

<script>
	import common from "@/static/js/common.js";
	export default {
		name: "detail",
		data() {
			return {
				currTime:'',
				
				//分享
				shareInfo: {},
				
				id: '',
				showShareMenu: false,
				showShareGuide: false,
				shareMenus: ['分享产品','一键生成海报'],
				goodsInfo: {
					goods_name:'--',
					feature:'--',
					main_pics: [],
					content_pics: [],
					url: '/'
				},

				//商品SKUs
				skus: [],

				//选择sku
				selectSkuText:'确定',
				changeSkuFlag: 0,
				activeSku: {
					id: '',
					name: '',
					price: '',
					volume: 1,
					stock:0
				},
				
				//购物车计数
				cartCount:{
					total_volume:0,
					count:0
				},
				
				showService:false,
				
				hasSpecial:false,
				specialInfo:{
					icon:'',
					text:'',
					class_name:'',
				},
				
				canBuy:true,
				
			};
		},
		onLoad:function(e) {
			this.id=e.id;
			e.source_uid && uni.setStorageSync('source_uid',e.source_uid);//渠道ID
			
			this.getDetail();
			this.getCartCount();
		},
		
		//下拉
		onPullDownRefresh(){
			this.getDetail();
		},
		
		//分享
		onShareAppMessage() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},
		onShareTimeline() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},

		methods: {
			countdown(){
				let that=this;
				this.currTime=common.addTime(this.currTime,1);
				setTimeout(function(){
					that.countdown();
				},1000);
			},
			
			//特殊信息
			buildSpecialInfo(data){
				let is_seckill=data.is_seckill;
				let start_time=data.start_time;
				let end_time=data.end_time;
				let text='';
				let icon='';
				let class_name='';
				let buy_name=is_seckill ? '秒杀' : '抢购';
				
				if(is_seckill!=1) return false;
				
				this.canBuy=true;
				if(start_time && start_time>this.currTime){
					text='距离' + buy_name + '开始还有<label class="orange ml-3 mr-3">' + common.countdown(this.currTime,start_time) + '</label>';
					icon='&#xe623;';
					this.canBuy=false;
				}
				else if(end_time && end_time<=this.currTime){
					text='当前商品已于<label class="orange ml-3 mr-3">' + end_time + '</label>停止'+buy_name;
					icon='&#xe63a;';
					this.canBuy=false;
				}else if(is_seckill){
					text='正在'+(end_time ? '限时' : '') + buy_name + (end_time ? '，距离结束还有' + '<label class="orange ml-3 mr-3">' + common.countdown(this.currTime,end_time) + '</label>' : '');
					icon='&#xe623;';
				}
				if(text){
					this.hasSpecial=true;
					this.specialInfo={
						text:text,
						class_name:class_name,
						icon:icon
					}
				}
			},

			//获取购物车计数
			getCartCount() {
				let that = this;
				common.getCartCount().then((res)=>{
					that.cartCount=res;
				});
			},

			//获取商品详情
			getDetail() {
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:'mall',
						action:'getGoodsDetailByApp',
						goods_id: that.id,
						token:uni.getStorageSync('token')
					},
					success:(res)=> {
						uni.stopPullDownRefresh();
						if (res.data.code == 1) {
							let _data = res.data.data;
							that.goodsInfo = _data;
							that.skus = _data.skus;
					
							//SKU默认选择
							if (that.skus.length > 0) {
								that.activeSku = {
									id: that.skus[0].sku_id,
									name: that.skus[0].sku_name,
									price: that.skus[0].sku_price,
									stock:that.skus[0].sku_stock,
									volume: 1
								}
							}
							that.shareInfo={
								title:that.goodsInfo.goods_name,
								desc:that.goodsInfo.feature,
								pic:that.goodsInfo.main_pic
							};
							
							that.currTime=that.goodsInfo.system_time;
							that.countdown();
							that.buildSpecialInfo(that.goodsInfo);
					
						} else {
							uni.stopPullDownRefresh();
							
							uni.showModal({
								title:"提示",
								content:res.data.msg,
								showCancel:false,
								success:function(res){
									if (res.confirm) {
									    that.backs();
									}
								}
							})
						}
					}
				})
			},

			//返回上页
			backs() {
				uni.navigateBack();
			},
			
			//显示SKU选择器
			showChangeSku(flag) {
				
				if(!this.canBuy) return false;
				
				let that = this;
				//检查登陆状态
				if (!common.checkLogin()) {
					return false;
				}

				this.changeSkuFlag = flag;
				this.$refs.popup.open();
			},

			//更改SKU
			changeSku(item) {
				this.activeSku = {
					id: item.sku_id,
					name: item.sku_name,
					price: item.sku_price,
					stock:item.sku_stock,
					volume: this.activeSku.volume
				};
			},

			//变更数量
			changeVolume(num) {
				let volume = parseInt(this.activeSku.volume);
				volume += parseInt(num);
				if (volume <= 0) {
					volume = 1;
				}
				this.activeSku.volume = volume;
			},

			//确定选择SKU
			selectSku(flag) {
				if (flag == 1) {
					this.cartItemAdd();
				} else if (flag == 2) {
					this.toConfirmOrder();
				}else{
					this.$refs.popup.close();
				}
			},
			

			//加入购物车
			cartItemAdd() {
				let that = this;
				if(!that.activeSku.id){
					uni.showToast({
						title: '请选择购买规格'
					});
					return false;
				}
				if(that.activeSku.stock==0){
					uni.showToast({
						title: '该规格无库存',
						icon:"none"
					});
					return false;
				}
				uni.request({
					header:this.requestHeader,
					method:"POST",
					url:this.apiUrl,
					data:{
						token:uni.getStorageSync('token'),
						model:"mall",
						action:"cartItemAdd",
						sku_id:that.activeSku.id,
						volume:that.activeSku.volume
					},
					success:(res)=> {
						if(res.data.code!=1){
							uni.showToast({
								title: '操作失败：'+res.data.msg,
								icon:"none",
								mask: true,
							});
						}else{
							that.$refs.popup.close();
							that.cartCount=res.data.data;
							that.setActiveCartIds(res.data.data.cart_id);
						}
					}
				})
			},
			
			//设置购物车选中
			setActiveCartIds(cart_id){
				let mallCartIds=uni.getStorageSync('mallCartIds');
				let activeCartIds=[];
				if(mallCartIds){
					activeCartIds=JSON.parse(mallCartIds);
				}
				activeCartIds.push(cart_id + '');
				activeCartIds=[...new Set(activeCartIds)];
				uni.setStorageSync('mallCartIds', JSON.stringify(activeCartIds));
			},

			//去购买
			toConfirmOrder() {
				let that = this;
				if(!that.activeSku.id){
					uni.showToast({
						title: '请选择购买规格',
						icon:"none"
					});
					return false;
				}
				if(that.activeSku.stock==0){
					uni.showToast({
						title: '该规格无库存',
						icon:"none"
					});
					return false;
				}
				let buyItems = [
					{
						sku_id: that.activeSku.id,
						volume: that.activeSku.volume,
					}
				];
				let buyItemsString = JSON.stringify(buyItems);
				uni.setStorageSync('mallBuyItems', buyItemsString);
				uni.setStorageSync('mallBuyFrom', 'goods');
				this.$refs.popup.close()
				this.toUrl("./confirmOrder");
			},
			
			//图片切换
			picChange(e){
				this.currPic=e.detail.current+1;
			},
			
			//预览封面图片
			previewImg(){
				uni.previewImage({
					current:this.currPic,
					urls: this.goodsInfo.main_pics,
					longPressActions: {
						itemList: ['发送给朋友', '保存图片', '收藏'],
					}
				});
			},
			//预览内容图片
			previewContentImg(index){
				uni.previewImage({
					current:index,
					urls: this.goodsInfo.content_pics,
					longPressActions: {
						itemList: ['发送给朋友', '保存图片', '收藏'],
					}
				});
			},
		},
		watch: {
			changeSkuFlag(val){
				if(val==1){
					this.selectSkuText='加入购物车';
				}else if(val==2){
					this.selectSkuText='现在去购买';
				}else{
					this.selectSkuText='确定';
				}
			},
			
			currTime(){
				this.buildSpecialInfo(this.goodsInfo);
			}
		}
	};
</script>
<style lang="scss">
	
	
	
	#detail {
		padding-bottom: 120rpx;

		* {
			box-sizing: border-box;
		}

		//价格颜色
		.price-color {
			color: $price-color;
		}

		//主图
		.main-pics {
			position: relative;
			image {
				width: 100%;
				height: 100vw;
				overflow: hidden;
			}

			.fiexd-link {
				position: absolute;
				lett: 0;
				top: 0;
				width: 100%;
				box-sizing: border-box;
				padding: 24rpx;

				text {
					i {
						display: inline-block;
						border-radius: 50%;
						width: 60rpx;
						height: 60rpx;
						line-height: 60rpx;
						overflow: hidden;
						background-color: rgba(0, 0, 0, 0.4);
						color: #fff;
						text-align: center;
						font-size: 36rpx;
					}
				}

				.back {
					float: left;
				}

				.share {
					float: right;
				}
			}
		}

		//基础信息
		.basic-info {
			background-color: #fff;
			padding: 24rpx;

			.special-info{
				background-color: $primary-color;
				text-align: center;
				// border-radius: 30rpx;
				padding: 12rpx;
				margin-top: -24rpx;
				margin-left: -24rpx;
				margin-right: -24rpx;
				margin-bottom: 24rpx;
			}

			.goods-name {
				font-weight: 500;
				font-size: 32rpx;
				line-height: 44rpx;
				margin-top: 24rpx;
			}

			.feature {
				line-height: 36rpx;
				margin-top: 12rpx;
				color: #999999;
				font-size: 24rpx;
			}

			.sku-select-view {
				color: #333333;
				padding: 0 20rpx;
				margin-top: 24rpx;
				border-radius: 12rpx;
				background-color: #F9F9F9;
				height: 33px;
				line-height: 33px;
				font-size: 26rpx;
				display: flex;
				align-items: center;

				.sku-view {
					flex: 1;
				}

				.sku-select {
					flex: 1;
					max-width: 160rpx;
					text-align: right;

					.txt,
					i {
						font-size: 24rpx;
						transform: scale(0.916);
						display: inline-block;
					}

					.txt {
						color: #9585bf;
					}
				}

			}
		}

		//详细描述
		.content {
			background-color: #fff;
			padding: 24rpx;
			margin-top: 24rpx;

			.title {
				font-weight: bold;
				font-size: 32rpx;
				margin-top: 10rpx;
				margin-bottom: 24rpx;
			}

			.deliver {
				font-size: 24rpx;
				margin-bottom: 24rpx;

				.item {
					display: flex;
					padding: 0 20rpx;
					line-height: 60rpx;

					.label {
						min-width: 100rpx;
						padding-right: 20rpx;
						color: #999999;
					}

					.value {
						padding-left: 20rpx;
					}
				}
			}

			.description {
				line-height: 44rpx;
				font-size: 28rpx;
				margin-bottom: 24rpx;
			}

			.content-pics {
				text-align: center;

				img {
					max-width: 100%;
				}
			}

		}
	}

	//选择SKU
	.uni-popup{
		z-index: 10000 !important;
	}
	.change-sku-wrap {
		box-sizing: border-box;
		width: 100%;
		background-color:#fff;
		padding: 40rpx 24rpx 40rpx 24rpx;
		margin:0 auto;
		border-radius:10px 10px 0 0;
		position: relative;

		.btns {
			.btn {
				border-radius: 60rpx;
				border: none;
				font-size: 28rpx;
				line-height: 72rpx;
				background-color: $primary-color;
			}
		}

		.title {
			font-weight: bold;
			font-size: 14px;
			padding: 12rpx 0;
		}

		.close {
			position: absolute;
			top: 40rpx;
			right: 20rpx;
			font-size: 36rpx;
		}
		
		.select-sku{
			margin-bottom: 24rpx;
			.price-wrap{
				color:$price-color;
				font-size: 24rpx;
				.price{
					font-size: 40rpx;
				}
			}
			.name-wrap{
				margin-top: 16rpx;
				font-size: 24rpx;
				color: #999;
				.name{
					color: #333;
					margin-left: 6rpx;
				}
			}
		}

		.all-sku {
			margin-bottom: 16rpx;

			.title {
				margin-bottom: 24rpx;
			}

			.skus {
				.sku {
					display: inline-block;
					border-radius: 26rpx;
					height: 50rpx;
					line-height: 50rpx;
					font-size: 26rpx;
					margin-right: 24rpx;
					padding: 0 24rpx;
					border: 2rpx solid #E2E2E2;
					margin-bottom: 24rpx;
					&.active {
						color: $second-color;
						border-color: $second-color;
					}
					&.disabled {
						color: #bbb;
						border-color: #e2e2e2;
						background-color: rgba(0, 0, 0, 0.05);
						opacity: 0.7;
					}
				}
			}

		}

		.set-volume {
			display: flex;
			align-items: center;
			margin-bottom: 40rpx;

			.title {
				flex: auto;
			}

			.volume {
				flex: auto;
				max-width: 240rpx;
				text-align: right;
				display: flex;
				align-items: center;

				.add,
				.sub,
				.num {
					max-width: 80rpx;
					min-width: 80rpx;
					display: inline-block;
					flex: 1;
					text-align: center;
					height: 48rpx;
					line-height: 48rpx;
				}

				.add,
				.sub {
					max-width: 80rpx;
					font-size: 24rpx;
					font-weight: bold;

					&:active {
						color: $primary-color;
					}

					&.disabled {
						opacity: 0.3;
					}
				}

				.num {
					width: auto;
					border-radius: 4rpx;
					background-color: #F1F1F1;
					font-size: 28rpx;
					border: none;
				}
			}
		}
	}
</style>
