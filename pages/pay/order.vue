<template>
	<view class="to-pay-order">
		<u-divider text="商品信息"></u-divider>
		<view class="order">
			<view class="item" v-for="(item, index) in goodsList" :key="'a' + index">
				<view class="left">
					<image :src="item.pic" mode="aspectFill"></image>
				</view>
				<view class="content">
					<view class="title u-line-3">{{ item.goodsName }}</view>
					<view class="type">
						<text v-for="(item2, index2) in item.sku"
							:key="'b' + index2">{{ item2.optionName }}:{{ item2.optionValueName }}/</text>
						<text v-for="(item3, index3) in item.additions"
							:key="'c' + index3">{{ item3.pname }}:{{ item3.name }}/</text>
					</view>
				</view>
				<view class="right">
					<view class="price-score">
						<view v-if="item.price" class="item"><text>¥</text>{{item.price}}</view>
						<view v-if="item.score" class="item"><text>
								<image class="score-icon" src="/static/images/score.png"></image>
							</text>{{item.score}}</view>
					</view>
					<view class="number">x{{ item.number }}</view>
				</view>
			</view>
			<view class="total">
				<text>共 {{ goodsNumber }} 件商品 合计:</text>
				<view class="price-score" style="display: inline-flex;">
					<view v-if="goodsPrice" class="item"><text>¥</text>{{ goodsPrice }}</view>
					<view v-if="goodsScore" class="item"><text>
							<image class="score-icon" src="/static/images/score.png"></image>
						</text>{{ goodsScore }}</view>
				</view>
			</view>
		</view>
		<view v-if="orderInfo && orderInfo.couponUserList" class="coupon-box">
			<u-divider text="使用优惠券"></u-divider>
			<u-radio-group placement="column" v-model="couponId" iconPlacement="right" @change="couponChange">
				<u-radio name="" label="不使用优惠券" activeColor="red"></u-radio>
				<u-radio v-for="(item,index) in orderInfo.couponUserList" :key="index" :name="item.id"
					:label="item.nameExt" activeColor="red"></u-radio>
			</u-radio-group>
		</view>
		<u-divider text="配送地址"></u-divider>
		<u-cell v-if="!defaultAddress" icon="map" title="新增收货地址" isLink url="/pages/address/addSite"></u-cell>
		<u-cell v-else icon="map" :title="defaultAddress.info.linkMan + ' ' + defaultAddress.info.mobile"
			:label="defaultAddress.info.provinceStr + defaultAddress.info.cityStr + defaultAddress.info.areaStr + defaultAddress.info.address"
			isLink url="/pages/address/index"></u-cell>
		<view class="remark">
			<u-textarea v-model="remark" placeholder="订单备注(选填)"></u-textarea>
		</view>
		<view v-if="orderInfo">
			<u-divider text="合计"></u-divider>
			<u-cell v-if="orderInfo.amountTotle" title="商品金额" :value="'¥' + orderInfo.amountTotle" :arrow="false">
			</u-cell>
			<u-cell v-if="orderInfo.score" title="积分" :value="orderInfo.score" :arrow="false"></u-cell>
			<u-cell v-if="orderInfo.amountLogistics" title="运费" :value="'¥' + orderInfo.amountLogistics" :arrow="false">
			</u-cell>
			<u-cell v-if="orderInfo.freightScore" title="运费积分" :value="orderInfo.freightScore" :arrow="false"></u-cell>
			<u-cell v-if="orderInfo.couponAmount" title="优惠券" :value="'-¥' + orderInfo.couponAmount" :arrow="false">
			</u-cell>
			<u-cell v-if="orderInfo.amountReal" title="总计" :value="'¥' + orderInfo.amountReal" :arrow="false"></u-cell>
			<u-cell v-if="userAmount" title="账户可用余额" :value="'¥' + userAmount.balance" :arrow="false"></u-cell>
			<u-cell v-if="userAmount" title="账户可用积分" :value="userAmount.score" :arrow="false"></u-cell>
		</view>
		<view class="submit safe-area-inset-bottom">
			<u-button type="error" @click="submit" :disabled="!canSubmit">提交订单</u-button>
		</view>
	</view>
</template>

<script>
	const PAY = require('@/common/pay.js')
	export default {
		data() {
			return {
				mod: undefined,
				cartType: undefined,
				couponId: '',
				defaultAddress: undefined,
				goodsList: undefined,
				goodsNumber: 0,
				goodsPrice: 0,
				goodsScore: 0,
				remark: '',
				canSubmit: false,
				orderInfo: undefined,
				userAmount: undefined,
				goodsType: 0 // 0 自营商品； 1 京东vop商品
			};
		},
		onLoad(e) {
			this.mod = e.mod
			this.cartType = e.cartType
			if (e.mod == 'cart') {
				if (e.cartType == 'apifm') {
					this.readCartApifm()
				}
				if (e.cartType == 'jdvop') {
					this.goodsType = 1
					this.readCartJdVop()
				}
			}
			if (e.mod == 'buy') {
				this.goodsList = uni.getStorageSync('goodsList')
				this.goodsList.forEach(ele => {
					this.goodsNumber += ele.number
					this.goodsPrice += ele.price * ele.number
					this.goodsScore += ele.score * ele.number
					this.goodsType = ele.goodsType
				})
				this.calculatePrice()
			}
			this._userAmount()
		},
		onShow() {
			this._defaultAddress()
		},
		methods: {
			async _defaultAddress() {
				// https://www.yuque.com/apifm/nu0f75/udrcag
				const res = await this.$wxapi.defaultAddress(this.token)
				if (res.code == 0) {
					this.defaultAddress = res.data
					this.calculatePrice()
				}
			},
			couponChange() {
				this.calculatePrice()
			},
			async _userAmount() {
				// https://www.yuque.com/apifm/nu0f75/wrqkcb
				const res = await this.$wxapi.userAmount(this.token)
				if (res.code == 0) {
					this.userAmount = res.data
				}
			},
			async readCartApifm() {
				// https://www.yuque.com/apifm/nu0f75/awql14
				const res = await this.$wxapi.shippingCarInfo(this.token)
				if (res.code == 0) {
					let supplyType = ''
					let supplyTypeHasEmpty = false // 是否有空类型
					let supplyTypeCanBuy = true // 是否允许下单
					this.goodsList = []
					// .filter(ele => { return ele.selected })
					res.data.items.forEach(ele => {
						if (!ele.supplyType) {
							supplyTypeHasEmpty = true
						}
						if (!supplyType && ele.supplyType) {
							supplyType = ele.supplyType
						}
						if (supplyType && supplyType != ele.supplyType) {
							supplyTypeCanBuy = false
						}
						this.goodsList.push({
							key: ele.key,
							goodsId: ele.supplyType == 'jdJoycityPoints' ? ele.yyIdStr : ele.goodsId,
							goodsName: ele.name,
							number: ele.number,
							pic: ele.pic,
							price: ele.price,
							score: ele.score,
							sku: ele.sku, // optionId optionName optionValueId optionValueName
							additions: ele.additions, // id name pid pname price
						})
					})
					if (supplyTypeHasEmpty && supplyType) {
						supplyTypeCanBuy = false
					}
					if (!supplyTypeCanBuy) {
						uni.showToast({
							title: supplyType + '商品不能和其他商品一起下单',
							icon: 'none'
						})
						return
					}
					if (supplyType == 'jdJoycityPoints') {
						this.goodsType = 2
					} else {
						this.goodsType = 0
					}
					this.goodsNumber = res.data.number
					this.goodsPrice = res.data.price
					this.goodsScore = res.data.score
					this.calculatePrice()
				}
			},
			async readCartJdVop() {
				// https://www.yuque.com/apifm/nu0f75/gwat37
				const res = await this.$wxapi.jdvopCartInfo(this.token)
				if (res.code == 0) {
					this.goodsList = []
					// .filter(ele => { return ele.selected })
					res.data.items.forEach(ele => {
						this.goodsList.push({
							key: ele.key,
							goodsId: ele.goodsId,
							goodsName: ele.name,
							number: ele.number,
							pic: ele.pic,
							price: ele.price,
							score: 0,
							sku: ele.sku, // optionId optionName optionValueId optionValueName
							additions: ele.additions, // id name pid pname price
						})
					})
					this.goodsNumber = res.data.number
					this.goodsPrice = res.data.price
					this.goodsScore = 0
					this.calculatePrice()
				}
			},
			buildOrderParams(calculate) {
				const goodsJsonStr = []
				this.goodsList.forEach(g => {
					let propertyChildIds = ''
					if (g.sku && g.sku.length > 0) {
						g.sku.forEach(option => {
							propertyChildIds = propertyChildIds + ',' + option.optionId + ':' + option
								.optionValueId
						})
					}
					let goodsAdditionList = []
					if (g.additions && g.additions.length > 0) {
						g.additions.forEach(option => {
							goodsAdditionList.push({
								pid: option.pid,
								id: option.id
							})
						})
					}
					goodsJsonStr.push({
						goodsId: g.goodsId,
						number: g.number,
						propertyChildIds,
						goodsAdditionList
					})
				})
				const data = {
					token: this.token,
					goodsJsonStr: JSON.stringify(goodsJsonStr),
					remark: this.remark,
					calculate,
					goodsType: this.goodsType,
					couponId: this.couponId
				}
				if (this.defaultAddress) {
					data.provinceId = this.defaultAddress.info.provinceId
					data.cityId = this.defaultAddress.info.cityId
					data.districtId = this.defaultAddress.info.districtId
					data.streetId = this.defaultAddress.info.streetId
					data.address = this.defaultAddress.info.address
					data.linkMan = this.defaultAddress.info.linkMan
					data.mobile = this.defaultAddress.info.mobile
				}
				return data
			},
			async calculatePrice() {
				this.canSubmit = false
				if (!this.goodsList || this.goodsList.length == 0) {
					return
				}
				// https://www.yuque.com/apifm/nu0f75/qx4w98
				const res = await this.$wxapi.orderCreate(this.buildOrderParams(true))
				if (res.code == 0) {
					this.canSubmit = true
					if (res.data.couponUserList) {
						res.data.couponUserList.forEach(ele => {
							let moneyUnit = '元'
							if (ele.moneyType == 1) {
								moneyUnit = '%'
							}
							if (ele.moneyHreshold) {
								ele.nameExt = ele.name + ' [面值' + ele.money + moneyUnit + '，满' + ele
									.moneyHreshold + '元可用]'
							} else {
								ele.nameExt = ele.name + ' [面值' + ele.money + moneyUnit + ']'
							}
						})
					}
					this.orderInfo = res.data
				} else {
					uni.showToast({
						title: res.msg,
						icon: 'none'
					})
				}
			},
			async submit() {
				// 判断实名认证
				if (this.sysconfigMap.needIdCheck == '1') {
					const res = await this.$wxapi.userDetail(this.token)
					if (res.code == 0 && !res.data.base.isIdcardCheck) {
						uni.navigateTo({
							url: '/pages/my/idcheck',
						})
						this.canSubmit = true
						return
					}
				}
				this.canSubmit = false
				if (!this.goodsList || this.goodsList.length == 0) {
					this.canSubmit = true
					return
				}
				// 判断余额和积分是否足够
				if (!this.orderInfo || !this.userAmount) {
					uni.showToast({
						title: '请稍后',
						icon: 'none'
					})
					this.canSubmit = true
					return
				}
				if (this.userAmount.score < this.orderInfo.score) {
					uni.showToast({
						title: '可用积分不足',
						icon: 'none'
					})
					this.canSubmit = true
					return
				}
				// https://www.yuque.com/apifm/nu0f75/qx4w98
				const res = await this.$wxapi.orderCreate(this.buildOrderParams(false))
				if (res.code == 0) {
					uni.showToast({
						title: '下单成功'
					})
					if (this.mod == 'cart') {
						const keys = []
						this.goodsList.forEach(ele => {
							keys.push(ele.key)
						})
						if (this.cartType == 'apifm') {
							// https://www.yuque.com/apifm/nu0f75/pndgyc
							await this.$wxapi.shippingCarInfoRemoveItem(this.token, keys.join())
						}
						if (this.cartType == 'jdvop') {
							// https://www.yuque.com/apifm/nu0f75/syqlot
							await this.$wxapi.jdvopCartRemove(this.token, keys.join())
						}
					}
					this.pay(res.data)
				} else {
					uni.showToast({
						title: res.msg,
						icon: 'none'
					})
				}
			},
			async pay(orderInfo) {
				const needPay = (orderInfo.amountReal - this.userAmount.balance).toFixed(2)
				if (needPay <= 0) {
					// 直接调用支付接口
					const res = await this.$wxapi.orderPay(this.token, orderInfo.id)
					if (res.code != 0) {
						uni.showToast({
							title: res.msg,
							icon: 'none'
						})
						uni.redirectTo({
							url: "../order/index?status=0"
						})
					} else {
						uni.showToast({
							title: '支付成功'
						})
						uni.redirectTo({
							url: "../order/index?status=1"
						})
					}
				} else {
					// 发起在线支付
					PAY.pay('wxpay', {
						appid: getApp().globalData.wxpayOpenAppId
					}, needPay, '支付订单 ：' + orderInfo.id, '支付订单 ：' + orderInfo.id, {
						type: 0,
						id: orderInfo.id
					}, () => {
						uni.redirectTo({
							url: "../order/index?status=0"
						})
					}, () => {
						uni.redirectTo({
							url: "../order/index?status=0"
						})
					})
				}
			},
		}
	};
</script>

<style lang="scss" scoped>
	.to-pay-order {
		padding-bottom: 100rpx;

		.shop-section {
			margin-top: 32rpx;
		}
	}

	.order {
		width: 710rpx;
		background-color: #ffffff;
		margin: 20rpx auto;
		border-radius: 20rpx;
		box-sizing: border-box;
		padding: 20rpx;
		font-size: 28rpx;

		.top {
			display: flex;
			justify-content: space-between;

			.left {
				display: flex;
				align-items: center;

				.store {
					margin: 0 10rpx;
					font-size: 32rpx;
					font-weight: bold;
				}
			}

			.right {
				color: $u-warning-dark;
			}
		}

		.item {
			display: flex;
			margin: 20rpx 0 0;

			.left {
				margin-right: 20rpx;

				image {
					width: 200rpx;
					height: 200rpx;
					border-radius: 10rpx;
				}
			}

			.content {
				flex: 1;

				.title {
					font-size: 28rpx;
					line-height: 50rpx;
				}

				.type {
					margin: 10rpx 0;
					font-size: 24rpx;
					color: $u-tips-color;
				}

				.delivery-time {
					color: #e5d001;
					font-size: 24rpx;
				}
			}

			.right {
				margin-left: 10rpx;
				padding-top: 20rpx;
				text-align: right;

				.price {
					text {
						font-size: 26rpx;
					}

					font-size: 38rpx;
					color: #e64340;
				}

				.decimal {
					font-size: 24rpx;
					margin-top: 4rpx;
				}

				.number {
					color: $u-tips-color;
					font-size: 24rpx;
				}
			}
		}

		.total {
			margin-top: 20rpx;
			text-align: right;
			font-size: 24rpx;
		}

		.bottom {
			display: flex;
			margin-top: 40rpx;
			padding: 0 10rpx;
			justify-content: space-between;
			align-items: center;

			.btn {
				line-height: 52rpx;
				width: 160rpx;
				border-radius: 26rpx;
				border: 2rpx solid $u-border-color;
				font-size: 26rpx;
				text-align: center;
				color: $u-info-dark;
			}

			.evaluate {
				color: $u-warning-dark;
				border-color: $u-warning-dark;
			}
		}
	}

	.remark {
		padding: 32rpx;
	}

	.submit {
		position: fixed;
		width: 100vw;
		left: 0;
		bottom: 0;
		z-index: 9;
	}

	.coupon-box {
		padding: 0 32rpx;

		.u-radio {
			padding: 24rpx 0;
		}
	}
</style>
