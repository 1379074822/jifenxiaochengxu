<template>
	<view class="container">
		<view class="header">
			<image class="logo" src="/static/logo.png"></image>
			<text class="title">计分助手</text>
			<text class="subtitle">多人游戏计分神器</text>
		</view>
		
		<view class="button-list">
			<view class="button-item" @click="startMultiGame">
				<view class="button-icon">👥</view>
				<view class="button-content">
					<text class="button-title">开始计分 - 多人模式</text>
					<text class="button-desc">创建房间，邀请朋友一起计分</text>
				</view>
				<text class="button-arrow">></text>
			</view>
			
			<view class="button-item" @click="startSingleGame">
				<view class="button-icon">👤</view>
				<view class="button-content">
					<text class="button-title">开始计分 - 单人模式</text>
					<text class="button-desc">房主计分，适合单人记录</text>
				</view>
				<text class="button-arrow">></text>
			</view>
			
			<view class="button-item" @click="scanToJoin">
				<view class="button-icon">📱</view>
				<view class="button-content">
					<text class="button-title">扫码加入房间</text>
					<text class="button-desc">扫描二维码快速加入游戏</text>
				</view>
				<text class="button-arrow">></text>
			</view>
			
			<view class="button-item" @click="showQRCode">
				<view class="button-icon">🔗</view>
				<view class="button-content">
					<text class="button-title">小程序二维码</text>
					<text class="button-desc">分享小程序给朋友</text>
				</view>
				<text class="button-arrow">></text>
			</view>
		</view>
		
		<!-- 二维码弹窗 -->
		<view class="qr-modal" v-if="showQRModal" @click="closeQRModal">
			<view class="qr-content" @click.stop>
				<view class="qr-header">
					<text class="qr-title">小程序二维码</text>
					<text class="close-btn" @click="closeQRModal">×</text>
				</view>
				<view class="qr-image">
					<text class="qr-placeholder">小程序二维码</text>
				</view>
				<text class="qr-tip">长按保存图片，分享给朋友</text>
			</view>
		</view>
	</view>
</template>

<script>
	import CloudAPI from '@/utils/cloud.js'
	
	export default {
		data() {
			return {
				showQRModal: false
			}
		},
		onLoad() {
			console.log('首页加载完成');
		},
		methods: {
			// 检查用户登录状态
			async checkLogin() {
				console.log('首页: 开始检查登录状态');
				try {
					const userInfo = await CloudAPI.getCurrentUser();
					console.log('首页: 获取到用户信息', userInfo);
					const isLogin = !!userInfo;
					console.log('首页: 登录状态', isLogin);
					return isLogin;
				} catch (error) {
					console.error('首页: 检查登录状态失败:', error);
					return false;
				}
			},
			
			// 跳转到登录页面
			goToLogin() {
				uni.showModal({
					title: '提示',
					content: '请先登录后再使用该功能',
					success: (res) => {
						if (res.confirm) {
							uni.switchTab({
								url: '/pages/profile/profile'
							});
						}
					}
				});
			},
			
			// 开始多人游戏
			async startMultiGame() {
				const isLogin = await this.checkLogin();
				if (!isLogin) {
					this.goToLogin();
					return;
				}
				
				uni.showLoading({
					title: '创建房间中...'
				});
				
				try {
					const room = await CloudAPI.createRoom('multi');
					uni.hideLoading();
					
					uni.navigateTo({
						url: `/pages/room/room?roomId=${room._id}&roomCode=${room.roomCode}&mode=multi&isOwner=true`
					});
				} catch (error) {
					uni.hideLoading();
					console.error('创建房间失败:', error);
					uni.showToast({
						title: error.message || '创建房间失败',
						icon: 'none'
					});
				}
			},
			
			// 开始单人游戏  
			async startSingleGame() {
				const isLogin = await this.checkLogin();
				if (!isLogin) {
					this.goToLogin();
					return;
				}
				
				uni.showLoading({
					title: '创建房间中...'
				});
				
				try {
					const room = await CloudAPI.createRoom('single');
					uni.hideLoading();
					
					uni.navigateTo({
						url: `/pages/room/room?roomId=${room._id}&roomCode=${room.roomCode}&mode=single&isOwner=true`
					});
				} catch (error) {
					uni.hideLoading();
					console.error('创建房间失败:', error);
					uni.showToast({
						title: error.message || '创建房间失败',
						icon: 'none'
					});
				}
			},
			
			// 扫码加入房间
			scanToJoin() {
				uni.scanCode({
					success: (res) => {
						console.log('扫码结果：', res);
						// 解析二维码，如果是房间码则加入房间
						const roomCode = this.parseRoomCode(res.result);
						if (roomCode) {
							this.joinRoom(roomCode);
						} else {
							uni.showToast({
								title: '无效的房间码',
								icon: 'none'
							});
						}
					},
					fail: (err) => {
						console.log('扫码失败：', err);
						uni.showToast({
							title: '扫码失败',
							icon: 'none'
						});
					}
				});
			},
			
			// 解析房间码
			parseRoomCode(code) {
				// 检查是否为6位数字房间码
				if (/^\d{6}$/.test(code)) {
					return code;
				}
				return null;
			},
			
			// 加入房间
			async joinRoom(roomCode) {
				const isLogin = await this.checkLogin();
				if (!isLogin) {
					// 存储要加入的房间码，登录后自动加入
					uni.setStorageSync('pendingRoomCode', roomCode);
					this.goToLogin();
					return;
				}
				
				uni.showLoading({
					title: '加入房间中...'
				});
				
				try {
					const room = await CloudAPI.joinRoom(roomCode);
					uni.hideLoading();
					
					uni.navigateTo({
						url: `/pages/room/room?roomId=${room._id}&roomCode=${room.roomCode}&mode=${room.mode}&isOwner=false`
					});
				} catch (error) {
					uni.hideLoading();
					console.error('加入房间失败:', error);
					uni.showToast({
						title: error.message || '加入房间失败',
						icon: 'none'
					});
				}
			},
			
			// 显示小程序二维码
			showQRCode() {
				this.showQRModal = true;
			},
			
			// 关闭二维码弹窗
			closeQRModal() {
				this.showQRModal = false;
			}
		}
	}
</script>

<style scoped>
	.container {
		min-height: 100vh;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		padding: 0 40rpx;
	}
	
	.header {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 80rpx 0 60rpx;
	}
	
	.logo {
		width: 120rpx;
		height: 120rpx;
		border-radius: 60rpx;
		margin-bottom: 30rpx;
		box-shadow: 0 8rpx 25rpx rgba(0, 0, 0, 0.15);
	}
	
	.title {
		font-size: 48rpx;
		font-weight: bold;
		color: #FFFFFF;
		margin-bottom: 15rpx;
		text-shadow: 0 2rpx 4rpx rgba(0, 0, 0, 0.3);
	}
	
	.subtitle {
		font-size: 28rpx;
		color: rgba(255, 255, 255, 0.8);
	}
	
	.button-list {
		flex: 1;
	}
	
	.button-item {
		display: flex;
		align-items: center;
		background: rgba(255, 255, 255, 0.95);
		border-radius: 20rpx;
		padding: 30rpx;
		margin-bottom: 25rpx;
		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.1);
		transition: all 0.3s ease;
	}
	
	.button-item:active {
		transform: scale(0.98);
		box-shadow: 0 2rpx 10rpx rgba(0, 0, 0, 0.15);
	}
	
	.button-icon {
		font-size: 48rpx;
		margin-right: 25rpx;
		width: 60rpx;
		text-align: center;
	}
	
	.button-content {
		flex: 1;
		display: flex;
		flex-direction: column;
	}
	
	.button-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #333333;
		margin-bottom: 8rpx;
	}
	
	.button-desc {
		font-size: 24rpx;
		color: #666666;
		line-height: 1.4;
	}
	
	.button-arrow {
		font-size: 32rpx;
		color: #999999;
		margin-left: 15rpx;
	}
	
	/* 二维码弹窗样式 */
	.qr-modal {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.5);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
	}
	
	.qr-content {
		background: #FFFFFF;
		border-radius: 20rpx;
		padding: 40rpx;
		margin: 0 60rpx;
		max-width: 600rpx;
		width: 100%;
	}
	
	.qr-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 40rpx;
	}
	
	.qr-title {
		font-size: 32rpx;
		font-weight: 600;
		color: #333333;
	}
	
	.close-btn {
		font-size: 48rpx;
		color: #999999;
		line-height: 1;
	}
	
	.qr-image {
		width: 400rpx;
		height: 400rpx;
		background: #F5F5F5;
		border-radius: 15rpx;
		display: flex;
		align-items: center;
		justify-content: center;
		margin: 0 auto 30rpx;
		border: 2rpx dashed #CCCCCC;
	}
	
	.qr-placeholder {
		font-size: 28rpx;
		color: #999999;
	}
	
	.qr-tip {
		text-align: center;
		font-size: 24rpx;
		color: #666666;
		display: block;
	}
</style> 