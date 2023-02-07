<template>
	<view class="reply">
		<view class="top">
			<comment-item :closeBtn="true" :childState="true" :item="replyItem"></comment-item>
		</view>
		<view class="list">
			<view class="row" v-for="item in childReplyArr">
				<comment-item @removeEnv="removeComment" :childState="true" :item="item"></comment-item>
			</view>
		</view>
		<view>
			<comment-frame @commentEnv="commentEnv" :commentObj="commentObj" :placeholder="`回复: ${getNickname(this.replyItem)}`"></comment-frame>
		</view>
	</view>
</template>

<script>
	import {getNickname,getAvatar} from '../../utils/tools.js'
	const db = uniCloud.database()
	export default {
		data() {
			return {
				replyItem: null,
				commentObj: {
					article_id: "",
					comment_type: 1,
					reply_user_id: "",
					reply_comment_id: "" 
				},
				childReplyArr:[]
			};
		},
		onLoad(e) {
			let replyItem = uni.getStorageSync("replyItem")
			if(!replyItem) {
				uni.navigateBack()
			}
			this.replyItem = replyItem || {}
			this.commentObj.article_id = this.replyItem.article_id
			this.commentObj.reply_user_id = this.replyItem.user_id[0]._id
			this.commentObj.reply_comment_id = this.replyItem._id
			this.getComment()
		},
		onUnload(){
			uni.removeStorageSync("replyItem")
		},
		methods: {
			getNickname,getAvatar,
			// 获取评论列表
			getComment() {
				let commentTemp = db.collection("quanzi_comment").where(`article == "${this.replyItem.article_id}" && comment_type==1 && reply_comment_id == "${this.replyItem._id}"`).orderBy("comment_date desc").limit(5).getTemp()
				let userTemp = db.collection("uni-id-users").field("_id,username,nickname,avatar_file").getTemp()
				
				db.collection(commentTemp,userTemp).get().then(res => {
					// if(res.result.data == 0) this.noComment = true
					this.childReplyArr = res.result.data
				})
			},
			// 评论成功后回调
			commentEnv() {
				this.childReplyArr = []
				this.getComment()
			}
		}
	}
</script>

<style lang="scss">
	.reply {
		.top {
			padding:30rpx;
			border-bottom: 15rpx solid #eee ;
		}
		.list {
			padding: 30rpx;
			padding-bottom: 120rpx;
			.row {
				padding-bottom: 15rpx;
			}
		}
	}
</style>
