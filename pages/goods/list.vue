<template>
	<view>
		<u-sticky bgColor="#FFFFFF">
			<view class="search">
				<u-search placeholder="输入关键词搜索" v-model="kw" :showAction="false" @search="search">
				</u-search>
				<!--  #ifndef  H5 || MP-KUAISHOU -->
				<view class="scan" @click="searchscan">
					<u-icon name="scan" size="48rpx"></u-icon>
				</view>
				<!--  #endif -->
			</view>
			<u-subsection activeColor="#e64340" :list="tabs" :current="activetab" @change="paixu"></u-subsection>
		</u-sticky>
		<page-box-empty v-if="!goods || goods.length == 0" title="暂无商品" sub-title="当前类目下无法帮你找到合适的商品" :show-btn="true" />
		<view v-if="showmod == 0" class="goodslist">
			<view v-for="(item, index) in goods" :key="index" class="list" @click="toDetailsTap">
				<image :src="item.pic" class="image" mode="aspectFill" lazy-load="true" @click="goDetail(item)" />
				<view class="r">
					<view class="goods-title u-line-3 pt16" @click="goDetail(item)">
						<u-tag v-if="item.supplyType == 'vop_jd' || item.supplyType == 'jdJoycityPoints'" text="京东自营" bgColor="#e64340" borderColor="#e64340" size="mini" class="goods-title-tag"></u-tag>
						<text class="goods-title">{{ item.name }}</text>
					</view>
					<u-text v-if="item.characteristic" class="goods-title" :text="item.characteristic" size="28rpx"
						color="#c95060"></u-text>
					<view class="price-score">
						<view v-if="item.minPrice" class="item"><text>¥</text>{{item.minPrice}}</view>
						<view v-if="item.minScore" class="item"><text><image class="score-icon" src="/static/images/score.png"></image></text>{{item.minScore}}</view>
					</view>
				</view>
				
			</view>
		</view>
		<view v-else>
			<view class="goods-container">
				<view v-for="(item, index) in goods" :key="index" class="goods-box" bindtap="toDetailsTap">
					<view class="img-box">
						<image :src="item.pic" class="image" mode="aspectFill" lazy-load="true" @click="goDetail(item)" />
					</view>
					<view class="goods-title u-line-3 pt16" @click="goDetail(item)">
						<u-tag v-if="item.supplyType == 'vop_jd' || item.supplyType == 'jdJoycityPoints'" text="京东自营" bgColor="#e64340" borderColor="#e64340" size="mini" class="goods-title-tag"></u-tag>
						<text class="goods-title">{{ item.name }}</text>
					</view>
					<u-text v-if="item.characteristic" class="goods-title" :text="item.characteristic" size="28rpx"
						color="#c95060"></u-text>
					<view class="price-score">
						<view v-if="item.minPrice" class="item"><text>¥</text>{{item.minPrice}}</view>
						<view v-if="item.minScore" class="item"><text><image class="score-icon" src="/static/images/score.png"></image></text>{{item.minScore}}</view>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				activetab: 0,
				kw: '',
				orderBy: '',
				categoryId: '',
				page: 1,
				tabs: [
					{
						code: '',
						name: '综合',
					},
					{
						code: 'addedDown',
						name: '新品',
					},
					{
						code: 'ordersDown',
						name: '销售'
					},
					{
						code: 'priceUp',
						name: '价格'
					},
				],
				goods: undefined,
				showmod: 1
			}
		},
		created() {

		},
		mounted() {

		},
		onReady() {

		},
		onLoad(e) {
			this.kw = e.kw ? e.kw : ''
			this.categoryId = e.categoryId ? e.categoryId : ''
			this._goods()
		},
		onShow() {

		},
		onShareAppMessage(e) {
			return {
				title: '"' + this.sysconfigMap.mallName + '" ' + this.sysconfigMap.share_profile,
				path: '/pages/index/index?inviter_id=' + this.uid
			}
		},
		onReachBottom() {
			this.page += 1
			this._goods()
		},
		methods: {
			search(v) {
				this.kw = v
				this.page = 1
				this._goods()
			},
			searchscan() {
				uni.scanCode({
					scanType: ['barCode', 'qrCode', 'datamatrix', 'pdf417'],
					success: res => {
						this.kw = res.result
						this.page = 1
						this._goods()
					}
				})
			},
			paixu(item) {
				this.activetab = item
				this.orderBy = this.tabs[item].code
				this.page = 1
				this._goods()
			},
			async _goods() {
				// https://www.yuque.com/apifm/nu0f75/wg5t98
				uni.showLoading({
					title: ''
				})
				const res = await this.$wxapi.goodsv2({
					page: this.page,
					k: this.kw,
					orderBy: this.orderBy ? this.orderBy : '',
					categoryId: this.categoryId ? this.categoryId : '',
				})
				uni.hideLoading()
				if (res.code == 0) {
					if (this.page == 1) {
						this.goods = res.data.result
					} else {
						this.goods = this.goods.concat(res.data.result)
					}
				} else {
					if (this.page == 1) {
						this.goods = null
					} else {
						uni.showToast({
							title: '没有更多了～',
							icon: 'none'
						})
					}
				}
			},
			goDetail(item) {
				uni.navigateTo({
					url: '/pages/goods/detail?id=' + item.id
				})
			}
		}
	}
</script>
<style scoped lang="scss">
	.search {
		padding: 8rpx;
		display: flex;
		align-items: center;
		.scan {
			padding: 0 16rpx;
		}
	}
	.goodslist {
		.list {
			margin: 20rpx;
			border-radius: 16rpx;
			display: flex;
			padding: 20rpx;
			.image {
				width: 260rpx;
				height: 260rpx;
				flex-shrink: 0;
				border-radius: 16rpx;
			}
			.r {
				margin-left: 32rpx;
				.goods-title {
					color: #333;
					font-size: 28rpx;
				}
				.label {
					color: #e64340;
					font-size: 24rpx;
					display: flex;
					align-items: flex-start;
					margin-top: 8rpx;
					text {
						margin-left: 8rpx;
					}
				}
			}
			.count-down {
				background: rgba(250, 195, 198, 0.3);
				border-radius: 5rpx;
				font-size: 14rpx;
				color: red;
				font-weight: 400;
				padding: 6rpx 16rpx;
				margin-top: 6rpx;
				text-align: center;
				border-radius: 10rpx;
			}
		}
		.price-box {
			display: flex;
			justify-content: space-between;
			align-items: center;
			.msbtn {
				width: 170rpx;
				height: 60rpx;
				background: linear-gradient(156deg, #FF7863 0%, #FF211A 100%);
				border-radius: 34rpx;
				border: none !important;
				line-height: 60rpx !important;
				font-size: 13px !important;
			}
			.price {
				color: #e64340;
				font-size: 40rpx;
				margin-top: 12rpx;
				padding-right: 32rpx;
				text {
					margin-left: 16rpx;
					color: #666666;
					font-size: 26rpx;
					text-decoration: line-through;
				}
			}
		}
	}
	.goods-container {
		display: flex;
		justify-content: space-between;
		flex-wrap: wrap;
		box-sizing: content-box;
		padding: 0 24rpx;
		.goods-box {
			width: 339rpx;
			background-color: #fff;
			overflow: hidden;
			margin-top: 24rpx;
			border-radius: 5px;
			border: 1px solid #D1D1D1;
			padding-bottom: 10rpx;
			.img-box {
				width: 339rpx;
				height: 339rpx;
				overflow: hidden;
				image {
					width: 339rpx;
					height: 339rpx;
				}
			}
			.goods-title {
				padding: 0 4rpx;
			}
			.goods-price-container {
				display: flex;
				align-items: baseline;
			}
			.goods-price {
				overflow: hidden;
				font-size: 34rpx;
				color: #F20C32;
				margin-left: 24rpx;
			}
			.goods-price2 {
				overflow: hidden;
				font-size: 26rpx;
				color: #aaa;
				text-decoration: line-through;
				margin-left: 20rpx;
			}
		}
	}
</style>
