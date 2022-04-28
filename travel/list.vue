<template>
<view class="wrap" id="list">
	<view class="header fixed">
		<view class="public-search-bar items-center">
			<view class="search-box">
				<uni-search-bar @confirm="searchGoods" :keyword="keyword" bgColor="#ffffff" radius="30" cancelButton="none" placeholder="请填写搜索关键词"></uni-search-bar>
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
	
	<view class="travel-list">
		<view class="empty" v-if="list.length==0">暂无商品</view>
		<view class="items">
			<TravelListItem :data="list" />
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
import TravelListItem from '@/components/Common/TravelListItem.vue';
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
				cid:'',
				sort:'',
				day:'',
				price:''
			},
			filterValue:[[0],[0],[0]],
			filterData: []
		};
	},
	onLoad(e) {
		this.cid=e.cid || 0;
		this.keyword=e.keyword || '';
		e.source_uid && uni.setStorageSync('source_uid',e.source_uid);//渠道ID
		this.getTravelFilters();
		this.getList();
		
		let that=this;
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
		
		//获取分类
		getTravelFilters(){
			let that = this;
			uni.request({
				header:this.requestHeader,
				url:this.apiUrl, 
				data: {
					model:"travel",
					action:"getTravelFilters",
					cid:that.cid
				},
				success:(res)=> {
					that.filterData=res.data.data;
					that.shareInfo={
						title:'为您推荐'+res.data.cname + '相关旅游线路',
						desc:'赶紧看看吧>>',
					}
				}
			})
		},
		
		//筛选
		changeFilter(e) {
			let val=e.value;
			let cid=val[0][1];
			if(!cid){
				cid=val[0][0];
			}
			this.cid=cid;
			this.filter.cid=cid;
			this.filter.sort=val[1][0] || '';
			this.filter.day=val[2][0][0] || '';
			this.filter.price=val[2][1][0] || '';
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
					token:uni.getStorageSync('token'),
					model:"travel",
					action:"getTravelList",
					keyword:that.keyword,
					cid:that.cid,
					sort:that.filter.sort,
					day:that.filter.day,
					price:that.filter.price,
					page:that.pageNo
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
	},
	components: {
		ListFilter,
		TravelListItem
	},
};
</script>
<style lang="scss">
@import '@/static/css/travel-list.scss';
.travel-list{
	margin-top: 196rpx;
	/*  #ifdef H5 */
	margin-top: 196rpx;
	/*  #endif  */
}

.loading-btn{
	margin:0 0 20rpx 0;
}
</style>