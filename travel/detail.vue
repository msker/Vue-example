<template>
	<view class="wrap" id="detail">
		
		<view class="main-pics">
			<swiper style="height: calc(100vw * 0.6666);" :indicator-dots="true" indicator-color="#000" indicator-active-color="#FFE000" @change="picChange">
				<swiper-item v-for="(item,index) in goodsInfo.main_pics" :key="index" show-dots="false" @click="previewImg">
					<image :src="item" :id="'img'+index" :preview="1"></image>
				</swiper-item>
			</swiper>
		</view>

		<!--基础信息-->
		<view class="basic-info">
			<view class="flexs items-center">
				<view class="flex price t-left price-color bold">
					<text class="fs-12 mr-3">&yen;</text>
					<text class="fs-20 mr-3">{{goodsInfo.price}}</text>
					<text class="fs-12 grey no-bold">起</text>
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

			<view class="feature" v-html="goodsInfo.feature"></view>
		</view>
		
		<view class="content">
			<uni-section title="最近团期" type="line" subTitle="全部团期" :subUrl="'/pages/travel/buy?id=' + id" :showArrow="true"></uni-section>
			<view class="plans">
				<view class="item" v-for="(item,index) in goodsInfo.plans" :key="index" @click="toUrl('/pages/travel/buy?id=' + id + '&date='+item.value)">
					<view class="date">{{item.date}}</view>
					<view class="price">&yen;{{item.price}}</view>
				</view>
			</view>
		</view>

		<!--详细描述-->
		<view class="content">
			<uni-section title="商品详情" type="line"></uni-section>
			
			<template v-if="goodsInfo.trip_pics && goodsInfo.trip_pics.length>0">
				<view class="sub-title">线路行程</view>
				<view class="trip-pic" v-for="(item,index) in goodsInfo.trip_pics" :key="index">
					<image :src="item" mode="widthFix" @click="previewContentImg(index)"></image>
				</view>
			</template>
			
			
			<template v-if="goodsInfo.fee_in!=''">
				<view class="sub-title">费用包含</view>
				<mp-html class="description" :content="goodsInfo.fee_in" />
			</template>
			
			
			<template v-if="goodsInfo.fee_not_in!=''">
				<view class="sub-title">费用不含</view>
				<mp-html class="description" :content="goodsInfo.fee_not_in" />
			</template>
			
			<template v-if="goodsInfo.shopping!=''">
				<view class="sub-title">购物项目</view>
				<mp-html class="description" :content="goodsInfo.shopping" />
			</template>
			
			
			<template v-if="goodsInfo.notice!=''">
				<view class="sub-title">预订须知</view>
				<mp-html class="description" :content="goodsInfo.notice" />
			</template>
			
		</view>

		<view class="foot-bar">
			<view class="links">
				<view class="link" @click="toUrl('/pages/index')">
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
			</view>
			
			<view class="btns">
				<view class="btn-item">
					<button type="default" class="btn" @click="toBuy()">立即预订</button>
				</view>
			</view>
		</view>
		
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
				shareInfo: {},
				id: '',
				goodsInfo: {},
				showService:false
			};
		},
		onLoad:function(e) {
			this.id=e.id;
			e.source_uid && uni.setStorageSync('source_uid',e.source_uid);//渠道ID
			this.getDetail();
		},
		//下拉
		onPullDownRefresh(){
			this.getDetail();
		},
		onShareAppMessage() {
			this.shareInfo.path=common.getFullUrl(1);
			return this.shareInfo;
		},
		
		//分享
		onShareAppMessage() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},
		onShareTimeline() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},

		methods: {
			//获取商品详情
			getDetail() {
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:'travel',
						action:'getTravelDetail',
						id: that.id,
						token:uni.getStorageSync('token'),
					},
					success:(res)=> {
						uni.stopPullDownRefresh();
						if (res.data.code == 1) {
							let _data = res.data.data;
							that.goodsInfo = _data;
							
							that.shareInfo={
								title:that.goodsInfo.goods_name,
								desc:that.goodsInfo.feature,
								pic:that.goodsInfo.main_pic
							};
					
						} else {
							uni.stopPullDownRefresh();
							uni.showToast({
								title: '商品信息获取失败',
								icon:"none",
							});
						}
					}
				})
			},

			
			//立即购买
			toBuy(date){
				this.toUrl("./buy?id=" + this.id + ( date ? "&date=" + date : ""));
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
					urls: this.goodsInfo.trip_pics,
					longPressActions: {
						itemList: ['发送给朋友', '保存图片', '收藏'],
					}
				});
			},
		}
	};
</script>
<style lang="scss">
	@import "@/static/css/travel-detail.scss";
	#detail {
		padding-bottom: 120rpx;
		
		//底部栏
		.foot-bar .btns{
			flex: 1;
			max-width: 300rpx;
			min-width: 300rpx;
		}
	}
</style>
