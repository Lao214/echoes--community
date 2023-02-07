<template>
	<view class="detail">
		<view class="container">
				<view v-if="loadState">
					<u-skeleton loading :animate="true" rows="3" title></u-skeleton>
				</view>
				<view v-else>
					<view class="title">{{ datailObj.title }}</view>
					<view class="userinfo">
						<view class="avatar">
							<image :src="getAvatar(datailObj)"  mode="aspectFill"></image>
						</view>
						<view class="text">
							<view class="name">{{getNickname(datailObj)}}</view>
							<view class="small">
								<uni-dateformat :date="datailObj.publish_date" format="yyyy年MM月dd hh:mm" :threshold="[60000,3600000*24*30]"></uni-dateformat>· 发布于{{ datailObj.province }}
							</view>
						</view>
					</view>
					<view class="content">
						<u-parse :content="datailObj.content" :tagStyle="tagStyle"></u-parse>
					</view>
					
					<view class="like">
						<view class="btn" :class="datailObj.isLike ? 'active': ''" @click="clickLike">
							<text class="iconfont icon-dianzan_kuai"></text>
							<text v-if="datailObj.like_count">{{ datailObj.like_count }}</text>
						</view>								
						<view class="users">	
							<template v-for="item in likeUserArr">
								<image v-if="item.user_id[0].avatar_file" :src="getAvatar(item)" mode="aspectFill" ></image>
							</template>
						</view>				
						<view class="text"><text class="num">{{ datailObj.view_count }}</text>人看过</view>
					</view>
					
				</view>
			
			
		</view>
	
		<view class="comment">
			<view style="padding-bottom: 50rpx;" v-if="!commentList.length && noComment">
				<u-empty mode="comment"	icon="https://cdn.uviewui.com/uview/empty/comment.png"></u-empty>
			</view>
			
			<view class="content" v-if="commentList.length">
				<view class="item" v-for="item in commentList">
					<comment-item @removeEnv="P_removeEnv" :item="item"></comment-item>						
				</view>
			</view>
			
		</view>
		
		
		<comment-frame :commentObj="commentObj" @commentEnv="P_commentEnv"></comment-frame>
	</view>
</template>

<script>
	import pageJson from '@/pages.json'
	import {getAvatar, getNickname, likeFun} from '../../utils/tools.js'
	import {store, mutations} from '@/uni_modules/uni-id-pages/common/store.js'
	const db = uniCloud.database()
	const utilsObj = uniCloud.importObject("utilsObj",{customUI: true})
	export default {
		data() {
			return {
				artid: "",
				loadState: true,
				tagStyle: {
					p: "line-height:1.7em;font-size:17px;padding-bottom: 10rpx",
					img: "margin:10rpx 0"
				},
				datailObj: null,
				likeUserArr: [],
				commentObj: {
					article_id: "",
					comment_type: 0
				},
				commentList: [],
				noComment: false
			};
		},
		onLoad(e) {
			if(!e.id) {
				this.errFun()
				return
			}
			this.artid = e.id
			this.commentObj.article_id = e.id
			this.getData()
			this.readUpdate()
			this.getLikeUser()
			this.getComment()
		},
		methods: {
			// 评论成功的回调函数
			P_commentEnv(e) {
				this.commentList.unshift({
					...this.commentObj,
					...e,
					user_id: [store.userInfo]
				})
			},
			// 删除评论回调函数
			P_removeEnv(e) {
				let index = this.commentList.findIndex(item => {
					return item._id  == e.id
				})
				this.commentList.splice(index,1)
			},
			// 获取评论列表
			async getComment() {
				let commentTemp = db.collection("quanzi_comment").where(`article_id == "${this.artid}" && comment_type==0`).orderBy("comment_date desc").limit(10).getTemp()
				// console.log(commentTemp)
				let userTemp = db.collection("uni-id-users").field("_id,username,nickname,avatar_file").getTemp()
				// console.log(userTemp)
				let res = db.collection(commentTemp,userTemp).get()
				
				let idArr = res.result.data.map(item => {
					return item._id
				})
				
				let replyArr = await db.collection("quanzi_comment").where({
					reply_comment_id: db.command.in(idArr)
				}).groupBy('reply_comment_id')
				.groupField('count(*) as totalReply').get()
				
				res.result.data.forEach(item => {
					let index = replyArr.result.data.findIndex(find => {
						return find.reply_comment_id == item._id
					})
					if(index > -1) {
						item.totalReply = replyArr.result.data[index].totalReply
					}
				})
				
				if(res.result.data == 0) {
					this.noComment = true
				}
				this.commentList = res.result.data
			},
			getLikeUser() {
				let likeTemp = db.collection("quanzi_like").where(`article_id == "${this.artid}"`).getTemp()
				let userTemp = db.collection("uni-id-users").field("_id,avatar_file").getTemp()
				db.collection(likeTemp,userTemp).orderBy("publish_date desc").limit(5).get().then(res => {
					this.likeUserArr = res.result.data
				})
			},
			errFun() {
				uni.showToast({
					title: '参数有误',
					icon: 'none'
				})
				setTimeout(() => {
					uni.reLaunch({
						url: "/pages/index/index"
					})
				}, 1000)
			},
			//修改阅读量
			readUpdate() {
				utilsObj.operation("quanzi_article", "view_count", this.artid, 1).then(res => {
					
				})
			},
			getData() {
				let artTemp = db.collection("quanzi_article").where(`_id=='${this.artid}'`).getTemp()
				let userTemp = db.collection("uni-id-users").field("_id,username,avatar_file,nickname").getTemp()
				
				let likeTemp = db.collection("quanzi_like").where(`article_id=='${this.artid}' && user_id==$cloudEnv_uid`).getTemp()
				
				let tempArr = [artTemp,userTemp]
				
				if(store.hasLogin) { tempArr.push(likeTemp) }
				
				db.collection(...tempArr).get({
					getOne: true
				}).then(res => {
					if(!res.result.data) {
						this.errFun()
						return
					}
					// let isLike = res.result.data._id.quanzi_like.length ? true : false
					// res.result.data.isLike = isLike
					let isLike
					// 登录判断
					if(store.hasLogin) {
						isLike = res.result.data._id.quanzi_like.length ? true : false
					}
					res.result.data.isLike = isLike
					this.datailObj = res.result.data
					this.loadState = false
					uni.setNavigationBarTitle({
						title:this.datailObj.title
					})
				}).catch(err => {
					this.errFun()
					return
				})
			},
			// 点赞
			async clickLike() {
				
				if(!store.hasLogin) {
					uni.showModal({
						title: "登录才可点赞,是否确认登录",
						success: res => {
							uni.navigateTo({
								url: "/" + pageJson.uniIdRouter.loginPage
							})
						}
					})
				}
				
				let time = Date.now()
				// 如果没有likeTime  这里会自动创建的
				if(time - this.likeTime < 2000) {
					uni.showToast({
						title: '操作太频繁,请稍后...',
						icon: 'none'
					})
					return
				}
				
				this.likeTime = time
				
				this.datailObj.isLike ? this.datailObj.like_count-- : this.datailObj.like_count++
				this.datailObj.isLike = !this.datailObj.isLike
				
				likeFun(this.artid)
			},

			getAvatar,
			getNickname
		}
	}
</script>

<style lang="scss">
.detail{
	background: #f8f8f8;
	min-height: calc(100vh - var(--window-top));	
	.container{
		padding:30rpx;	
		background: #fff;
		.title{
			font-size: 46rpx;
			color:#333;
			line-height: 1.4em;
			font-weight: 600;
		}
		.userinfo{
			padding:20rpx 0 50rpx;
			display: flex;
			align-items: center;
			.avatar{
				width: 60rpx;
				height: 60rpx;
				padding-right: 15rpx;
				image{
					width: 100%;
					height: 100%;
					border-radius: 50%;
				}
			}
			.text{
				font-size: 28rpx;
				color:#555;
				.small{
					font-size: 20rpx;
					color:#999;
				}
			}
		}
		.content{
			
		}
		.like{
			display: flex;
			flex-direction: column;
			align-items: center;
			padding:80rpx 50rpx 50rpx;
			.btn{
				width: 260rpx;
				height: 120rpx;
				background: #e4e4e4;
				border-radius: 100rpx;
				color:#fff;
				display: flex;
				justify-content: center;
				align-items: center;
				flex-direction: column;
				font-size: 28rpx;
				.iconfont{
					font-size: 50rpx;
				}
				&.active{
					background: #0199FE;
				}
			}
			.text{
				font-size: 26rpx;
				color:#666;				
				.num{
					color:#0199FE
				}
				.spot{
					padding:0 10rpx;
				}
			}
			.users{
				display: flex;
				justify-content: center;
				padding:30rpx 0;
				image{
					width: 50rpx;
					height: 50rpx;
					border-radius: 50%;
					border:3px solid #fff;
					margin-left:-20rpx;
				}
			}
		}
		
	}

	.comment{
		padding:30rpx;
		padding-bottom:120rpx;
		.item{
			padding:10rpx 0;
		}
	}

}
</style>