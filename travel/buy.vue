<template>
<view id="buy">
	<!--日历表-->
	<view class="calendar">
		<uni-calendar
		:date="date"
		:insert="true"
		:start-date="startDate"
		:end-date="endDate"
		:showMonth="false"
		:selected="plans"
		@change="selectPlan"
		@monthSwitch="changeMonth"
		 />
	</view>
	
	<!--价格-->
	<view class="box plan-box">
		<uni-section :padding="true" title="价格类型及人数" :sub-title="activeData.data && (activeData.data.date ? activeData.data.date + '出发（余位:'+activeData.data.remain+'）' : '请选择出发日期')" type="line"></uni-section>
		<view class="form-wrap prices" v-if="activeData.data">
			<view class="form-item price-item" v-for="(item,index) in activeData.data.prices" :key="item.price_id">
				<view class="price-name ellipsis2">
					{{item.name}}
				</view>
				<view class="price-view">
					<text class="price-color">&yen;{{item.price}}</text>
					<text class="unit">/人</text>
				</view>
				<view class="set-volume">
					<view class="sub" :class="{'disabled':getVolumeByPriceId(item.price_id)<1}" @click="setVolume(item,-1)"><text class="ext-icon">&#xe6f2;</text></view>
					<input class="num" type="number" min="1" :value="getVolumeByPriceId(item.price_id)" disabled="true">
					<view class="add" @click="setVolume(item,1)"><text class="ext-icon">&#xe650;</text></view>
				</view>
			</view>
		</view>
	</view>
	
	<!--联系人-->
	<view class="box contact-box">
		<uni-section :padding="true" title="订单联系人" type="line"></uni-section>
		
		<view class="form-wrap">
			<view class="form-item">
				<view class="item-label">联系人</view>
				<view class="item-value">
					<input type="text" v-model="contact.name" placeholder="必填，请填写">
				</view>
			</view>
			
			
			<view class="form-item">
				<view class="item-label">手机号</view>
				<view class="item-value">
					<input type="text" v-model="contact.mobile" placeholder="必填，请填写">
				</view>
			</view>
		</view>
	</view>
	
	
	<!--出行人-->
	<view class="box person-box">
		<uni-section :padding="true" title="出行人名单" :showArrow="true" type="line">
			<template v-slot:right>
				<view class="fs-13 second-color" @click="selectPersons">点击选择<text class="green">{{totalPerson}}位</text>出行人</view>
			</template>
		</uni-section>
		
		<view class="empty" v-if="persons.length==0"><view class="ext-icon fs-20 mb-5">&#xe606;</view>需选择{{totalPerson}}位出行人</view>
		
		<view class="persons" v-if="persons.length>0">
			<view class="item" v-for="(item,index) in persons" :key="item.id">
				<view class="title">
					<text class="name">{{item.name}}</text>
				</view>
				<view class="detail">
					<text class="mr-10">{{item.card_type_name}}</text>
					<text>{{item.card_no}}</text>
				</view>
			</view>
			
		</view>
	</view>
	
	<!--保险-->
	<view class="box insure-box">
		<uni-section :padding="true" title="旅游意外险" :showArrow="true" type="line">
			<template v-slot:right>
				<view class="fs-13 second-color" @click="toggleInsureList">查看保费标准</view>
			</template>
		</uni-section>
		<view class="flexs items-center pd-10">
			<view class="flex fs-14 pl-10">选择投保旅游意外险</view>
			<view class="flex t-right">
				<switch :checked="insureFlag" @change="setInsureFlag" color="#FFE000" style="transform: scale(0.7);" />
			</view>
		</view>
	</view>
	
	<!--订单备注-->
	<view class="box remark-box">
		<uni-section :padding="true" title="订单备注" type="line"></uni-section>
		<view class="form-wrap">
			<textarea v-model="remark" placeholder-style="color:#bbbbbb" placeholder="如有需要，请填写" />
		</view> 
	</view>
	
	<!--底部栏-->
	<view class="foot-bar">
		<view class="describe flexs items-center">
			<view>合计<text class=" ml-3 second-color">{{totalPerson}}</text><text class="fs-10 ml-3 grey">人</text></view>
			<view class="flex ml-10">金额<text class="amount price-color fs-16 ml-5"><text class="fs-12 mr-3">&yen;</text>{{totalAmount}}</text></view>
		</view>
		<view class="btns">
			<view class="btn-item">
				<button type="primary" :disabled="disabledSubmit" class="btn" @click="createOrder">提交订单</button>
			</view>
		</view>
	</view>
	
	<uni-popup ref="insureList" @change="showInsureList=!showInsureList">
		<view class="insure-list">
			<uni-section :padding="true" title="保费标准" type="line">
				<template v-slot:right>
					<text class="close ext-icon" @click="toggleInsureList">&#xe65e;</text>
				</template>
			</uni-section>
			<view class="items">
				<view class="item under-line">
					<view class="day bold">行程天数</view>
					<view class="age bold">游客年龄</view>
					<view class="price bold">保费</view>
				</view>
				<view class="item" v-for="(item,index) in insureList" :key="index" :class="{'under-line':index<insureList.length-1}">
					<view class="day">{{item.start_day}}~{{item.end_day}}天</view>
					<view class="age">{{item.start_age}}~{{item.end_age}}岁</view>
					<view class="price"><text class="red mr-5">{{item.member_fee}}</text>元/人</view>
				</view>
			</view>
		</view>
	</uni-popup>
	
</view>
</template>

<script>
	import common from "@/static/js/common.js";
	import Calendar from '@/components/uni-calendar/util.js';
	export default {
		name: "detail",
		data() {
			return {
				id: '',
				date:'',
				year:'',
				month:'',
				
				startDate:'',
				endDate:'',
				
				plans:[],
				
				//联系方式
				contact:{
					name:'',
					mobile:'',
				},
				
				//保险信息
				insureList:[],
				insureFlag:true,
				showInsureList:false,
				
				activeVolume:[],
				activeData:{data:{remain:0}},
				
				totalPerson:0,
				totalAmount:0,
				
				
				//出行人
				persons:[],
				
				onSubmit:false,
				
				remark:''
				
			};
		},
		
		onLoad:function(e) {
			if(common.checkLogin()){
				this.init(e);
			}
		},
		
		onShow(){
			this.buildPersons();
		},
		
		onPullDownRefresh(){
			this.getTravelPlans();
		},
		
		computed:{
			disabledSubmit:function(){
				if(this.onSubmit){					return true;
				}else{
					return false;
				}
			}
		},

		methods: {
			init(e){
				this.cale = new Calendar()
				this.id=e.id;
				this.date=e.date;
				this.year=this.cale.getDate(this.date).year;
				this.month=this.cale.getDate(this.date).month;
				
				this.getOrderPrepareInfo();
				this.getTravelPlans();
			},
			
			//设置保险保记
			setInsureFlag(e){
				this.insureFlag=e.detail.value;
			},
			
			//切换保费保准显示
			toggleInsureList(){
				if(this.showInsureList){
					this.$refs.insureList.close();
				}else{
					this.$refs.insureList.open();
				}
			},
			
			
			//整理出行人
			buildPersons(){
				let personMaps=uni.getStorageSync('selectPersons');
				let persons=[];
				for(var i=0;i<this.totalPerson;i++){
					if(personMaps[i]){
						persons.push(personMaps[i]);
					}else{
						break;
					}
				}
				this.persons=persons;
			},
			
			//获取交易前置信息
			getOrderPrepareInfo() {
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:'travel',
						action:'getOrderPrepareInfo',
						id:that.id,
						data:that.date,
						token:uni.getStorageSync('token')
					},
					success:(res)=> {
						if (res.data.code == 1) {
							let data = res.data.data;
							this.contact=data.contact;
							this.insureList=data.insureList;
						}
					}
				})
			},
			
			//获取团期
			getTravelPlans() {
				let that = this;
				uni.request({
					header:this.requestHeader,
					url:this.apiUrl, 
					data: {
						model:'travel',
						action:'getTravelPlans',
						id: that.id,
						year:that.year,
						month:that.month,
					},
					success:(res)=> {
						uni.stopPullDownRefresh();
						if (res.data.code == 1) {
							that.plans=res.data.data;
							if(that.plans.length>0){
								that.startDate=that.plans[0].date;
								that.endDate=that.plans[that.plans.length-1].date;
								that.setActive();
								if(!that.date){
									that.date=that.plans[0].date;
									that.activeData=that.plans[0];
								}
							}
						} else {
							uni.stopPullDownRefresh();
							uni.showToast({
								title: '获取团期失败',
								icon:"none",
							});
						}
					}
				})
			},
			
			setActive(){
				let data = this.plans.find(plan => plan.date == this.date);
				if(!data){
					return false;
				}
				this.activeData=data;
			},
			
			//选择月份
			changeMonth(e){
				this.year=e.year;
				this.month=e.month > 9 ? e.month : '0' + e.month;
				this.getTravelPlans();
			},
			
			//选择团期
			selectPlan(e){
				this.date=e.fulldate;
				this.setActive();
			},
			
			//根据价格ID获取数量
			getVolumeByPriceId(price_id){
				let curr=this.activeVolume.find(it => it.price_id == price_id);
				if(curr){
					return curr.volume;
				}else{
					return 0;
				}
			},
			
			//变更购买项数量
			setVolume(item,volume){
				let that=this;
				let price_id=item.price_id;
				let price=item.price;
				let single=item.single;
				let oldVolume=that.getVolumeByPriceId(price_id);
				let newVolume=oldVolume+volume;
				if(newVolume<0 || isNaN(newVolume)){
					newVolume=0;
				}
				if(newVolume == oldVolume){
					return false;
				}
				
				
				let curr=that.activeVolume.find(it => it.price_id == price_id);
				if(!curr){
					that.activeVolume.push({
						price_id:price_id,
						single:single,
						price:price,
						volume:newVolume
					});
				}else{
					let volumes=[];
					that.activeVolume.map(it=>{
						if(it.price_id==price_id){
							it.volume=newVolume;
						}
						volumes.push(it);
					})
					that.activeVolume=volumes;
				}
				
			},
			
			//计算总价
			calcTotalAmount(){
				let totalPerson=0;
				let totalAmount=0;
				this.activeVolume.forEach(item=>{
					totalAmount=common.add(totalAmount,common.mul(item.volume,item.price));
					if(item.single!=1){
						totalPerson=common.add(totalPerson,item.volume);
					}
				})
				this.totalPerson=totalPerson;
				this.totalAmount=totalAmount;
				
				if(this.totalPerson > this.activeData.data.remain){
					uni.showToast({
						title:'最多可预订'+this.activeData.data.remain+'人',
						icon:'none'
					})
				}
				
				this.buildPersons();
			},
	
			//选择出行人
			selectPersons(){
				let ids='';
				let idList=[];
				
				if(this.totalPerson==0){
					this.toast('请先选择人数');
					return false;
				}
				
				this.persons.forEach(item=>{
					idList.push(item.id);
				})
				
				ids=idList.join('|');
				uni.navigateTo({url:'/pages/member/persons?select='+this.totalPerson+'&id='+ids})
			},
			
			//整理订单参数
			buildOrderParams(){
				
				//出行人
				let person_ids=[];
				this.persons.map(item=>{
					person_ids.push(item.id);
				})
				
				//价格
				let price_arr=[];
				this.activeVolume.map(item=>{
					if(item.volume>0){
						price_arr.push({
							price_id:item.price_id,
							volume:item.volume
						});
					}
				})
				
				let params={
					'id':this.id,
					'date':this.date,
					'items':price_arr,
					'name':this.contact.name,
					'mobile':this.contact.mobile,
					'persons':person_ids,
					'insurance':this.insureFlag ? 1 : 0,
					'remark':this.remark
				};
				
				return params;
			},
			
			//检查订单参数
			checkOrderParams(params){
				if(this.totalPerson==0){
					this.toast('请选择出行人数');
					return false;
				}
				if(!this.contact.name){
					this.toast('请填写联系人');
					return false;
				}
				if(!this.contact.mobile){
					this.toast('请填写联系手机号');
					return false;
				}
				if(params.persons.length!=this.totalPerson){
					this.toast('请选择' + this.totalPerson + '位出行人');
					return false;
				}
				return true;
			},
			
			//创建订单
			createOrder(){
				let that=this;
				let params=this.buildOrderParams();
				if(!this.checkOrderParams(params)){
					return false;
				}
				
				params.model="travel";
				params.action="createOrder";
				params.token=that.token;
				
				that.onSubmit=true;
				uni.showLoading({
					title:'订单提交中..'
				});
				
				uni.request({
					header: {'json': 1},
					method:"POST",
					url:this.apiUrl,
					data:params,
					success:(res)=> {
						uni.hideLoading();
						that.onSubmit=false;
						if (res.data.code == 1) {
							let _data=res.data.data;
							let order_sn=_data.order_sn;
							uni.redirectTo({url:'./orderPay?order_sn='+order_sn});
						}else{
							that.toast(res.data.msg);
						
						}
					},
					fail() {
						that.toast('订单提交失败')
					}
				})
			}
			
		},
		watch:{
			activeVolume(){
				this.calcTotalAmount();
			}
		}
	};
</script>
<style lang="scss">
@import "@/static/css/travel-buy.scss";
#buy {
	padding-bottom: 120rpx;
	
	//底部栏
	.foot-bar .btns{
		flex: 1;
		max-width: 300rpx;
		min-width: 300rpx;
	}
}
</style>
