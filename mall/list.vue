<template>
<view class="wrap" id="list">
	<view class="header fixed">
		<view class="public-search-bar items-center">
			<view class="search-box">
				<uni-search-bar @confirm="searchGoods" :keyword="keyword" bgColor="#ffffff" radius="30" cancelButton="none" placeholder="请填写搜索关键词"></uni-search-bar>
			</view>
	
			<view class="right-link right-icon" @click="toUrl('./cart')">
				<label class="ext-icon fs-20">&#xe60e;</label>
			</view>
		</view>
		
		<view class="filter-bar">
			<ListFilter
			  :filterData="filterData"
			  :defaultSelected="filterValue"
				:updateMenuName="false"
				:top="46"
				@confirm="changeFilter"
			/>
		</view>
	</view>
	
	<view class="mall-list">
		<view class="empty" v-if="list.length==0">暂无商品</view>
		<view class="items">
			<view @click="toDetail(item.goods_id)" class="item under-line" v-for="(item,index) in list" :key="index">
				<view class="inner">
					<view class="pic">
						<img :src="item.main_pic">
					</view>
					<view class="detail">
						<view class="name ellipsis2">{{item.goods_name}}</view>
						<view class="other ellipsis">
							{{item.feature}}
						</view>
						<view class="foot flexs items-center">
							<view class="t-left flex">
								<text class="price-color fs-12 mr-3">&yen;</text>
								<text class="price-color">{{item.price}}</text>
								<text class="grey fs-12 ml-3">起</text>
							</view>
							<view class="t-right flex fs-12">
								<text class="fs-12 grey" v-if="!item.commission">已售 {{item.volume}}</text>
								<view class="commission-tag" v-if="item.commission>0">
									<text class="tag">返佣</text>
									<text class="value">&yen;{{item.commission}}</text>
								</view>
							</view>
						</view>
					</view>
				</view>
			</view>
		</view>
	</view>
	
	<!--LoadMore-->
	<view class="list-load-view" v-if="list.length>0">
		<view class="loading-btn" :class="{'disabled':!hasMore || isLoading}" @click="hasMore && !isLoading?loadMore():false">
			<uni-load-more :status="isLoading?'loading':(!hasMore?'noMore':'more')" :contentText="contentText" :iconSize="16"></uni-load-more>
		</view>
	</view>
	
</view>
</template>

<script>
import common from "@/static/js/common.js";
import ListFilter from '@/components/Common/ListFilter.vue';
export default {
	name: "list",
	data() {
		return {
			shareInfo:{},
			
			isLoading:false,
			hasMore:true,
			contentText:{
				contentdown: "点击加载更多..",
				contentrefresh: "正在获取更多产品..",
				contentnomore: "已加载全部产品",
			},
			
			
			pageNo:1,
			keyword:'',
			cid:'',
			list:[],
			
			
			filter:{
				flag:'',
				sort:''
			},
			filterValue:[[0],[0],[0]],
			filterData: [
				{
					"name": "分类",
					"icon": "&#xe616;",
					"type": "hierarchy",
					defaultActive:2,
					"submenu": []
				},
				{
					"name": "排序方式",
					"icon": "&#xe6da;",
					"type": "hierarchy",
					"submenu": [
						{
							"name": "默认排序",
							"value": ""
						},
						{
							"name": "价格从低到高",
							"value": 1,
						},
						{
							"name": "价格从高到低",
							"value": 2
						},
						{
							"name": "销量从低到高",
							"value": 1
						},
						{
							"name": "销量从高到低",
							"value": 2
						}
					]
				},
				{
					"name": "筛选",
					"icon": "&#xe646;",
					"type": "hierarchy",
					"submenu": [
						{
							"name": "不限",
							"value": ''
						},
						{
							"name": "推荐商品",
							"value": 1
						},
					]
				},
			],
		};
	},
	onLoad(e) {
		this.cid=e.cid || 0;
		this.keyword=e.keyword || '';
		e.source_uid && uni.setStorageSync('source_uid',e.source_uid);//渠道ID
		
		this.getList();
		
		let that=this;
		common.getMallSetting().then((res)=>{
			let catesArr=Object.keys(res.cates).map((i,k)=>{
				return {index:k,value:res.cates[i].cid,name:res.cates[i].name,icon:res.cates[i].icon}
			})
			
			catesArr.splice(0,-1,{
				icon:"&#xe616;",
				"name": "默认分类",
				"value": "",
			})
			
			if(that.cid){
				let currCate = catesArr.find(cate => cate.value == that.cid);
				uni.setNavigationBarTitle({
				   title:currCate.name
				});
				that.filterValue=[[currCate.index+1],[0],[0]];
				that.shareInfo.title='为您推荐'+currCate.name+'相关商品';
			}
			
			that.filterData[0].submenu=catesArr;
		});
	},
	
	//下拉
	onPullDownRefresh(){
		this.pageNo=1;
		this.getList();
	},
	
	//分享
	onShareAppMessage() {
		return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
	},
	onShareTimeline() {
		return common.share(this.shareInfo.title,this.shareInfo.desc,this.shareInfo.pic);
	},
	
	methods: {
		//筛选
		changeFilter(e) {
			var val=e.value;
			this.cid=val[0][0];
			this.filter.sort=val[1][0];
			this.filter.flag=val[2][0];
			this.pageNo=1;
			this.getList();
			
		},
		
		//加载更多
		loadMore(){
			this.pageNo++;
			this.getList();
		},
		
		//获取商品列表
		getList(){
			let that = this;
			that.isLoading=true;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl, 
				data: {
					model:"mall",
					action:"getGoodsByApp",
					cid:that.cid,
					keyword:that.keyword,
					flag:that.filter.flag,
					sort:that.filter.sort,
					page:that.pageNo,
					token:uni.getStorageSync('token')
				},
				success:(res)=> {
					uni.stopPullDownRefresh();
					that.isLoading=false;
					if (res.data.code == 1) {
						if(that.pageNo==1){
							that.list=res.data.data;
						}else{
							that.list=JSON.parse(JSON.stringify(that.list)).concat(res.data.data);
						}
						
						if(res.data.data.length<10){
							that.hasMore=false;
						}
						
					}
				}
			})
		},
		
		//搜索产品
		searchGoods(e){
			this.keyword=e.value;
			this.pageNo=1;
			this.getList();
		},
		
		//到商品商情
		toDetail(id) {
			uni.navigateTo({url:'./detail?id='+id});
		},
	},
	components: {
		ListFilter
	},
};
</script>
<style lang="scss">
@import '@/static/css/mall-list.scss';
#list{
	&,*{
		box-sizing: border-box;
	}
	.mall-list{
		margin-top: 196rpx;
		/*  #ifdef H5 */
		margin-top: 196rpx;
		/*  #endif  */
	}
	
	.loading-btn{
		margin:0 0 20rpx 0;
	}
}
</style>