<template>
	<view class="wrap mall-index">
		<view class="public-search-bar items-center searchBar">

			<view class="logo">
				<image src="@/static/img/logo-img.png" mode="widthFix"></image>
			</view>
			
			<view class="search-box">
				<uni-search-bar @confirm="searchGoods" :keyword="keyword" bgColor="#ffffff" radius="30" cancelButton="none" placeholder="请填写搜索关键词"></uni-search-bar>
			</view>

			<view class="right-link" @click="toUrl('/pages/mall/cart')">
				<label class="ext-icon">&#xe60e;</label>
				<text class="txt fs-11">购物车</text>
				<uni-badge class="badge" :text="cartCount.count" size="small" type="error" v-if="cartCount.count>0"></uni-badge>
			</view>
		</view>

		<!--Banners-->
		<view class="banner-wrap">
			<swiper :indicator-dots="true" indicator-color="#000" indicator-active-color="#FFE000" :auto="true" :interval='5000' class="banners">
				<swiper-item class="banner" v-for="(item,index) in banners" :key="index">
					<image @click="toUrl(item.banner_url)" :src="item.banner_pic" />
				</swiper-item>
			</swiper>
		</view>

		<!--navs-->
		<view class="navs">
			<view @click="toUrl('/pages/mall/list?cid='+item.cid)" class="item" v-for="(item,index) in navs" :key="index">
				<view class="inner">
					<view class="ext-icon icon" v-html="item.icon"></view>
					<text class="txt">{{item.name}}</text>
				</view>
			</view>
		</view>
		
		
		<view class="mall-seckill-list" v-if="seckillGoods.length>0">
			<uni-section title="秒杀专区" :padding="true" icon="&#xe689;" :subTitle="seckillGoods.length>3 ? '向左滑浏览更多' : ''"></uni-section>
			
			<scroll-view scroll-x="true" scroll-with-animation="true" class="scroll-X">
			<view class="items">
				<view @click="toDetail(item.goods_id)" class="item" v-for="(item,index) in seckillGoods" :key="index">
					<view class="inner">
						<view class="pic">
							<img :src="item.main_pic">
						</view>
						<view class="detail">
							<view class="name ellipsis2">{{item.goods_name}}</view>
							<view class="foot">
								<text class="price"><text class="fs-14 mr-3">&yen;</text>{{item.price}}</text>
								<!-- <text class="o-price"><text class="fs-14 mr-3">&yen;</text>{{item.market_price}}</text> -->
							</view>
						</view>
					</view>
				</view>
			</view>
			</scroll-view>
		</view>

		<!--Ads-->
		<view class="ads" v-if="ads.length>0">
			<view @click="toUrl(item.ad_url)" class="item" v-for="(item,index) in ads" :key="index">
				<view class="inner">
					<view class="pic"><img :src="item.ad_pic"></view>
				</view>
			</view>
		</view>

		<view class="mall-recommend-list">
			<uni-section title="热门推荐" :padding="true" icon="&#xe66c;"></uni-section>
			<view class="items">
				<view @click="toDetail(item.goods_id)" class="item" v-for="(item,index) in recommendGoods" :key="index">
					<view class="inner">
						<view class="pic">
							<img :src="item.main_pic">
						</view>
						<view class="detail">
							<view class="name ellipsis2">{{item.goods_name}}</view>
							<view class="other" v-if="item.commission">
								<view class="commission-tag">
									<text class="tag">返佣</text>
									<text class="value">&yen;{{item.commission}}</text>
								</view>
							</view>
							<view class="foot">
								<text class="price"><text class="fs-14 mr-3">&yen;</text>{{item.price}}</text>
							</view>
						</view>
					</view>
				</view>
			</view>
		</view>
		
		<!--tabbar-->
		<tabbar curr='mall' :bars="bars"></tabbar>
	</view>
</template>

<script>
	import common from "@/static/js/common.js";
	export default {
		name: "index",
		data() {
			return {
				shareInfo: {},
				
				bars:this.tabbars,
				keyword: '',
				banners: [],
				navs: [],
				ads: [],
				recommendGoods: [],
				seckillGoods:[],
				seller_id:uni.getStorageSync('seller_id'),
				
				//购物车计数
				cartCount:{
					total_volume:0,
					count:0
				},
			};
		},
		onLoad(e) {
			let that=this;
			e.source_uid && uni.setStorageSync('source_uid',e.source_uid);//渠道ID
			
			if(common.checkLogin('none')){
				common.getCartCount().then(res=>{
					this.cartCount=res;
				})
			}
			
			common.getMallSetting().then((res)=>{
				that.setMallSetting(res)
			});
			
			that.getSeckillGoods();
			that.getRecommendGoods();
		},
		
		//分享
		onShareAppMessage() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},
		onShareTimeline() {
			return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
		},
		
		methods: {
			
			//初始化数据
			setMallSetting(setting){
				this.banners = setting.banners || [];
				this.navs = setting.cates || [];
				this.ads = setting.ads || [];
				this.shareInfo={
					title:setting.mall_name,
					desc:setting.share_desc,
					pic:setting.share_pic,
				}
			},
			
			//获取推荐产品列表
			getSeckillGoods(){
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:"mall",
						action:"getGoodsByApp",
						flag:4,
						sort:4,
						rows:10,
						token:uni.getStorageSync('token')
					},
					success:(res)=> {
						if (res.data.code == 1) {
							that.seckillGoods = res.data.data;
						}
					}
				})
			},
			
			//获取推荐产品列表
			getRecommendGoods(){
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:"mall",
						action:"getGoodsByApp",
						flag:0,
						sort:4,
						rows:10,
						token:uni.getStorageSync('token')
					},
					success:(res)=> {
						if (res.data.code == 1) {
							that.recommendGoods = res.data.data;
						}
					}
				})
			},
			
			//到商品商情
			toDetail(id) {
				uni.navigateTo({url:'/pages/mall/detail?id='+id});
			},
			
			//搜索产品
			searchGoods(e){
				uni.navigateTo({url:'/pages/mall/list?keyword='+e.value});
			}
		}
	};
</script>
<style lang="scss">
	@import '@/static/css/index.scss';
	@import '@/static/css/mall-recommend-list.scss';
	@import '@/static/css/mall-seckill.scss';
	.mall-index{
		padding-bottom: 140rpx;
	}
</style>
