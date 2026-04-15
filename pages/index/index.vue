<template>
	<view class="content">
		<!-- 顶部总存款显示 -->
		<view class="total-amount">
			<text class="total-label">总存款</text>
			<text class="total-value">¥{{totalAmount.toFixed(2)}}</text>
		</view>

		<!-- 操作入口 -->
		<view class="actions-section">
			<button type="primary" @click="showAddAccountDialog">添加账户</button>
			<button type="default" @click="showEditAmountDialog">修改金额</button>
		</view>

		<!-- 账户列表 -->
		<view class="accounts-section">
			<text class="section-title">账户列表</text>
			<view class="accounts-list">
				<view v-for="(account, index) in accounts" :key="index" class="account-item">
					<view class="account-info">
						<text class="account-name">{{account.name}}</text>
						<text class="account-amount">¥{{account.amount.toFixed(2)}}</text>
					</view>
					<view class="account-actions">
						<button size="mini" type="primary" @click="showEditAccountDialog(index)">编辑</button>
						<button size="mini" type="warn" @click="confirmDeleteAccount(index)">删除</button>
					</view>
				</view>
				<view v-if="accounts.length === 0" class="empty-account">
					<text>暂无账户，请添加账户</text>
				</view>
			</view>
		</view>

		<!-- 存款变化图 -->
		<view class="chart-section" v-if="moneyHistory.length > 0">
			<view class="chart-header">
				<text class="section-title">存款变化趋势</text>
				<view class="time-range-selector">
					<picker :range="timeRanges" :value="selectedTimeRangeIndex" @change="handleTimeRangeChange">
						<view class="time-range-button">
							{{timeRanges[selectedTimeRangeIndex]}}
						</view>
					</picker>
				</view>
			</view>
			<view class="chart-container">
				<canvas canvas-id="moneyChart" class="chart"></canvas>
			</view>
		</view>

		<!-- 历史记录 -->
		<view class="history-section" v-if="filteredMoneyHistory.length > 0">
			<text class="section-title">历史记录</text>
			<view class="history-list">
				<view v-for="(item, index) in filteredMoneyHistory" :key="index" class="history-item">
					<view class="history-info">
						<text class="history-date">{{item.date}}</text>
						<text class="history-amount">¥{{item.amount.toFixed(2)}}</text>
					</view>
					<view class="history-actions">
						<button size="mini" type="primary" @click="editHistory(index)">编辑</button>
						<button size="mini" type="warn" @click="deleteHistory(index)">删除</button>
					</view>
				</view>
			</view>
		</view>

		<!-- 添加账户对话框 -->
		<view v-if="showAddDialog" class="dialog-overlay" @click="closeAllDialogs">
			<view class="dialog-content" @click.stop>
				<text class="dialog-title">添加账户</text>
				<view class="form-item">
					<text>账户名称</text>
					<input v-model="newAccount.name" placeholder="请输入账户名称" />
				</view>
				<view class="form-item">
					<text>初始金额</text>
					<input v-model.number="newAccount.amount" type="number" placeholder="请输入初始金额" />
				</view>
				<view class="dialog-actions">
					<button @click="closeAllDialogs">取消</button>
					<button type="primary" @click="confirmAddAccount">确认</button>
				</view>
			</view>
		</view>

		<!-- 编辑账户对话框 -->
		<view v-if="showEditDialog" class="dialog-overlay" @click="closeAllDialogs">
			<view class="dialog-content" @click.stop>
				<text class="dialog-title">编辑账户</text>
				<view class="form-item">
					<text>账户名称</text>
					<input v-model="editAccount.name" placeholder="请输入账户名称" />
				</view>
				<view class="form-item">
					<text>账户金额</text>
					<input v-model.number="editAccount.amount" type="number" placeholder="请输入账户金额" />
				</view>
				<view class="dialog-actions">
					<button @click="closeAllDialogs">取消</button>
					<button type="primary" @click="confirmEditAccount">确认</button>
				</view>
			</view>
		</view>

		<!-- 修改金额对话框 -->
		<view v-if="showAmountDialog" class="dialog-overlay" @click="closeAllDialogs">
			<view class="dialog-content" @click.stop>
				<text class="dialog-title">修改金额</text>
				<view class="form-item">
					<text>选择账户</text>
					<view class="picker-container">
						<text class="picker-label">{{accounts[selectedAccountIndex]?.name || '请选择账户'}}</text>
						<picker :range="accounts" :range-key="'name'" :value="selectedAccountIndex" @change="handleAccountChange">
							<view class="picker-button">选择</view>
						</picker>
					</view>
				</view>
				<view class="form-item">
					<text>操作类型</text>
					<view class="picker-container">
						<text class="picker-label">{{amountOperationType === 0 ? '存入' : '取出'}}</text>
						<picker :range="['存入', '取出']" :value="amountOperationType" @change="handleOperationTypeChange">
							<view class="picker-button">选择</view>
						</picker>
					</view>
				</view>
				<view class="form-item">
					<text>金额</text>
					<input v-model.number="amountChange" type="number" placeholder="请输入金额" />
				</view>
				<view class="dialog-actions">
					<button @click="closeAllDialogs">取消</button>
					<button type="primary" @click="confirmEditAmount">确认</button>
				</view>
			</view>
		</view>

		<!-- 编辑历史记录对话框 -->
		<view v-if="showEditHistoryDialog" class="dialog-overlay" @click="closeAllDialogs">
			<view class="dialog-content" @click.stop>
				<text class="dialog-title">编辑历史记录</text>
				<view class="form-item">
					<text>日期</text>
					<input v-model="editHistoryItem.date" placeholder="请输入日期 (YYYY-MM-DD)" />
				</view>
				<view class="form-item">
					<text>金额</text>
					<input v-model.number="editHistoryItem.amount" type="number" placeholder="请输入金额" />
				</view>
				<view class="dialog-actions">
					<button @click="closeAllDialogs">取消</button>
					<button type="primary" @click="confirmEditHistory">确认</button>
				</view>
			</view>
		</view>

		<!-- 导出按钮 -->
		<view class="export-section">
			<button type="default" @click="showCsvData">导出所有记录</button>
		</view>

		<!-- CSV数据展示对话框 -->
		<view v-if="showCsvDialog" class="dialog-overlay" @click="closeCsvDialog">
			<view class="dialog-content dialog-content-large" @click.stop>
				<text class="dialog-title">导出数据</text>
				<view class="csv-content">
					<textarea v-model="csvData" readonly class="csv-textarea"></textarea>
				</view>
				<view class="dialog-actions">
					<button @click="closeCsvDialog">关闭</button>
					<button type="primary" @click="copyCsvData">复制数据</button>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				// 账户列表
				accounts: [],
				// 对话框显示状态
				showAddDialog: false,
				showEditDialog: false,
				showAmountDialog: false,
				showEditHistoryDialog: false,
				showCsvDialog: false,
				// 新账户信息
				newAccount: { name: '', amount: 0 },
				// 编辑账户信息
				editAccount: { name: '', amount: 0 },
				editAccountIndex: -1,
				// 修改金额相关
				selectedAccountIndex: 0,
				amountOperationType: 0, // 0: 存入, 1: 取出
				amountChange: 0,
				// 编辑历史记录相关
				editHistoryItem: { date: '', amount: 0 },
				editHistoryIndex: -1,
				// 时间范围选择
				timeRanges: ['最近7天', '最近1个月', '最近3个月', '全部'],
				selectedTimeRangeIndex: 1, // 默认最近1个月
				// 存款历史记录
				moneyHistory: [],
				// CSV数据
				csvData: ''
			}
		},
		computed: {
			// 计算总存款金额
			totalAmount() {
				return this.accounts.reduce((total, account) => total + parseFloat(account.amount), 0);
			},
			// 过滤后的历史记录
			filteredMoneyHistory() {
				// 按日期降序排序（从晚到早，最新的在最上面）
				const sortedHistory = [...this.moneyHistory].sort((a, b) => {
					return new Date(b.date) - new Date(a.date);
				});
				
				// 根据选择的时间范围过滤
				const now = new Date();
				let startDate;
				
				switch (this.selectedTimeRangeIndex) {
					case 0: // 最近7天
						startDate = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
						break;
					case 1: // 最近1个月
						startDate = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);
						break;
					case 2: // 最近3个月
						startDate = new Date(now.getTime() - 90 * 24 * 60 * 60 * 1000);
						break;
					default: // 全部
						return sortedHistory;
				}
				
				return sortedHistory.filter(item => {
					return new Date(item.date) >= startDate;
				});
			}
		},
		onLoad() {
			// 从本地存储加载数据
			this.loadData();
			// 页面加载时绘制图表
			this.$nextTick(() => {
				this.drawChart();
			});
		},
		methods: {
			// 从本地存储加载数据
			loadData() {
				try {
					const savedAccounts = uni.getStorageSync('accounts');
					const savedMoneyHistory = uni.getStorageSync('moneyHistory');
					
					if (savedAccounts && savedAccounts.length > 0) {
						this.accounts = savedAccounts;
					} else {
						// 设置默认数据
						this.accounts = [
							{ name: '储蓄账户', amount: 5000 },
							{ name: '支付宝', amount: 2000 },
							{ name: '微信钱包', amount: 1000 }
						];
						this.saveData();
					}
					
					if (savedMoneyHistory && savedMoneyHistory.length > 0) {
						this.moneyHistory = savedMoneyHistory;
					} else {
						// 设置默认数据
						this.moneyHistory = [
							{ date: '2026-04-01', amount: 8000 },
							{ date: '2026-04-02', amount: 8500 },
							{ date: '2026-04-03', amount: 9000 },
							{ date: '2026-04-04', amount: 8800 },
							{ date: '2026-04-05', amount: 9500 },
							{ date: '2026-04-06', amount: 9800 },
							{ date: '2026-04-07', amount: 10000 },
							{ date: '2026-04-08', amount: 10500 },
							{ date: '2026-04-09', amount: 11000 },
							{ date: '2026-04-10', amount: 11500 },
							{ date: '2026-04-11', amount: 12000 },
							{ date: '2026-04-12', amount: 12500 }
						];
						this.saveData();
					}
				} catch (error) {
					console.error('加载数据失败:', error);
					// 设置默认数据
					this.accounts = [
						{ name: '储蓄账户', amount: 5000 },
						{ name: '支付宝', amount: 2000 },
						{ name: '微信钱包', amount: 1000 }
					];
					this.moneyHistory = [
						{ date: '2026-04-01', amount: 8000 },
						{ date: '2026-04-02', amount: 8500 },
						{ date: '2026-04-03', amount: 9000 },
						{ date: '2026-04-04', amount: 8800 },
						{ date: '2026-04-05', amount: 9500 },
						{ date: '2026-04-06', amount: 9800 },
						{ date: '2026-04-07', amount: 10000 },
						{ date: '2026-04-08', amount: 10500 },
						{ date: '2026-04-09', amount: 11000 },
						{ date: '2026-04-10', amount: 11500 },
						{ date: '2026-04-11', amount: 12000 },
						{ date: '2026-04-12', amount: 12500 }
					];
					this.saveData();
				}
			},
			// 保存数据到本地存储
			saveData() {
				try {
					uni.setStorageSync('accounts', this.accounts);
					uni.setStorageSync('moneyHistory', this.moneyHistory);
				} catch (error) {
					console.error('保存数据失败:', error);
				}
			},
			// 显示添加账户对话框
			showAddAccountDialog() {
				this.newAccount = { name: '', amount: 0 };
				this.showAddDialog = true;
			},
			// 显示编辑账户对话框
			showEditAccountDialog(index) {
				this.editAccountIndex = index;
				this.editAccount = { ...this.accounts[index] };
				this.showEditDialog = true;
			},
			// 显示修改金额对话框
			showEditAmountDialog() {
				if (this.accounts.length > 0) {
					this.selectedAccountIndex = 0;
					this.amountOperationType = 0;
					this.amountChange = 0;
					this.showAmountDialog = true;
				} else {
					uni.showToast({
						title: '请先添加账户',
						icon: 'none'
					});
				}
			},
			// 关闭所有对话框
			closeAllDialogs() {
				this.showAddDialog = false;
				this.showEditDialog = false;
				this.showAmountDialog = false;
				this.showEditHistoryDialog = false;
			},
			// 编辑历史记录
			editHistory(index) {
				this.editHistoryIndex = index;
				this.editHistoryItem = { ...this.moneyHistory[index] };
				this.showEditHistoryDialog = true;
			},
			// 确认编辑历史记录
			confirmEditHistory() {
				if (this.editHistoryItem.date && this.editHistoryItem.amount >= 0 && this.editHistoryIndex >= 0) {
					this.moneyHistory[this.editHistoryIndex] = {
						date: this.editHistoryItem.date,
						amount: parseFloat(this.editHistoryItem.amount)
					};
					this.drawChart();
					this.saveData();
					this.closeAllDialogs();
				} else {
					uni.showToast({
						title: '请填写正确的历史记录信息',
						icon: 'none'
					});
				}
			},
			// 删除历史记录
			deleteHistory(index) {
				// 获取当前显示的历史记录
				const historyItem = this.filteredMoneyHistory[index];
				if (!historyItem) return;
				
				uni.showModal({
					title: '确认删除',
					content: '确定要删除这条历史记录吗？',
					confirmText: '确定',
					cancelText: '取消',
					success: (res) => {
						if (res.confirm) {
							// 找到该记录在原始数组中的索引
							const originalIndex = this.moneyHistory.findIndex(item => 
								item.date === historyItem.date && item.amount === historyItem.amount
							);
							if (originalIndex !== -1) {
								this.moneyHistory.splice(originalIndex, 1);
								this.drawChart();
								this.saveData();
							}
						}
					}
				});
			},
			// 处理账户选择变化
			handleAccountChange(e) {
				this.selectedAccountIndex = e.detail.value;
			},
			// 处理操作类型变化
			handleOperationTypeChange(e) {
				this.amountOperationType = e.detail.value;
			},
			// 确认添加账户
			confirmAddAccount() {
				if (this.newAccount.name && this.newAccount.amount >= 0) {
					this.accounts.push({
						name: this.newAccount.name,
						amount: parseFloat(this.newAccount.amount)
					});
					this.updateMoneyHistory();
					this.drawChart();
					this.saveData();
					this.closeAllDialogs();
				} else {
					uni.showToast({
						title: '请填写正确的账户信息',
						icon: 'none'
					});
				}
			},
			// 确认编辑账户
			confirmEditAccount() {
				if (this.editAccount.name && this.editAccount.amount >= 0 && this.editAccountIndex >= 0) {
					this.accounts[this.editAccountIndex] = {
						name: this.editAccount.name,
						amount: parseFloat(this.editAccount.amount)
					};
					this.updateMoneyHistory();
					this.drawChart();
					this.saveData();
					this.closeAllDialogs();
				} else {
					uni.showToast({
						title: '请填写正确的账户信息',
						icon: 'none'
					});
				}
			},
			// 确认删除账户
			confirmDeleteAccount(index) {
				uni.showModal({
					title: '确认删除',
					content: '确定要删除这个账户吗？',
					confirmText: '确定',
					cancelText: '取消',
					success: (res) => {
						if (res.confirm) {
							this.accounts.splice(index, 1);
							this.updateMoneyHistory();
							this.drawChart();
							this.saveData();
						}
					}
				});
			},
			// 确认修改金额
			confirmEditAmount() {
				if (this.selectedAccountIndex >= 0 && this.amountChange > 0) {
					const account = this.accounts[this.selectedAccountIndex];
					if (this.amountOperationType === 1 && account.amount < this.amountChange) {
						uni.showToast({
							title: '账户余额不足',
							icon: 'none'
						});
						return;
					}
					
					if (this.amountOperationType === 0) {
						account.amount += parseFloat(this.amountChange);
					} else {
						account.amount -= parseFloat(this.amountChange);
					}
					
					this.updateMoneyHistory();
					this.drawChart();
					this.saveData();
					this.closeAllDialogs();
				} else {
					uni.showToast({
						title: '请输入正确的金额',
						icon: 'none'
					});
				}
			},
			// 更新存款历史记录
			updateMoneyHistory() {
				const today = new Date().toISOString().split('T')[0];
				this.moneyHistory.push({
					date: today,
					amount: this.totalAmount
				});
				// 只保留最近10条记录
				if (this.moneyHistory.length > 10) {
					this.moneyHistory.shift();
				}
			},
			// 处理时间范围变化
			handleTimeRangeChange(e) {
				this.selectedTimeRangeIndex = e.detail.value;
				this.drawChart();
			},
			// 绘制存款变化图表
		drawChart() {
			let history = this.filteredMoneyHistory;
			if (history.length === 0) return;
			
			// 按日期升序排序用于图表显示（从早到晚）
			history = [...history].sort((a, b) => {
				return new Date(a.date) - new Date(b.date);
			});
			
			// 处理相同日期的记录，合并为一个点（取最后一条记录的值）
			const uniqueHistory = [];
			const dateMap = new Map();
			
			history.forEach(item => {
				dateMap.set(item.date, item);
			});
			
			dateMap.forEach((value, key) => {
				uniqueHistory.push(value);
			});
			
			// 重新按日期排序
			uniqueHistory.sort((a, b) => {
				return new Date(a.date) - new Date(b.date);
			});
			
			const ctx = uni.createCanvasContext('moneyChart', this);
			
			// 获取画布实际尺寸（动态获取）
			uni.createSelectorQuery()
				.select('.chart')
				.boundingClientRect()
				.exec((res) => {
					if (res && res[0]) {
						const canvasWidth = res[0].width;
						const canvasHeight = res[0].height;
						
						// 动态计算边距
						const padding = Math.max(40, canvasWidth * 0.1); // 左侧边距
						const bottomPadding = Math.max(60, canvasHeight * 0.2); // 底部边距
						const topPadding = Math.max(20, canvasHeight * 0.1); // 顶部边距
						
						// 清空画布
						ctx.clearRect(0, 0, canvasWidth, canvasHeight);
						
						// 计算数据范围
						const amounts = uniqueHistory.map(item => item.amount);
						const minAmount = Math.min(...amounts);
						const maxAmount = Math.max(...amounts);
						const amountRange = maxAmount - minAmount || 1;
						
						// 绘制坐标轴
						ctx.beginPath();
						ctx.moveTo(padding, topPadding);
						ctx.lineTo(padding, canvasHeight - bottomPadding);
						ctx.lineTo(canvasWidth - padding, canvasHeight - bottomPadding);
						ctx.strokeStyle = '#ccc';
						ctx.stroke();
						
						// 绘制y轴金额标签
						const ySteps = 4; // y轴刻度数量
						const yStepHeight = (canvasHeight - topPadding - bottomPadding) / ySteps;
						
						for (let i = 0; i <= ySteps; i++) {
							const y = canvasHeight - bottomPadding - i * yStepHeight;
							const amount = minAmount + (i / ySteps) * amountRange;
							
							// 绘制刻度线
							ctx.beginPath();
							ctx.moveTo(padding - 5, y);
							ctx.lineTo(padding, y);
							ctx.strokeStyle = '#ccc';
							ctx.stroke();
							
							// 绘制金额标签
							ctx.fillStyle = '#666';
							ctx.font = `${Math.max(8, Math.floor(canvasHeight * 0.025))}px Arial`; // 动态字体大小
							ctx.textAlign = 'right';
							ctx.fillText('¥' + amount.toFixed(0), padding - 10, y + 3);
						}
						
						// 绘制数据点和连线
						if (uniqueHistory.length > 1) {
							const stepX = (canvasWidth - 2 * padding) / (uniqueHistory.length - 1);
							
							ctx.beginPath();
							uniqueHistory.forEach((item, index) => {
								const x = padding + index * stepX;
								const y = canvasHeight - bottomPadding - ((item.amount - minAmount) / amountRange) * (canvasHeight - topPadding - bottomPadding);
								
								if (index === 0) {
									ctx.moveTo(x, y);
								} else {
									ctx.lineTo(x, y);
								}
							});
							ctx.strokeStyle = '#007AFF';
							ctx.lineWidth = 2;
							ctx.stroke();
							
							// 绘制数据点
							uniqueHistory.forEach((item, index) => {
								const x = padding + index * stepX;
								const y = canvasHeight - bottomPadding - ((item.amount - minAmount) / amountRange) * (canvasHeight - topPadding - bottomPadding);
								
								ctx.beginPath();
								ctx.arc(x, y, 4, 0, 2 * Math.PI);
								ctx.fillStyle = '#007AFF';
								ctx.fill();
							});
							
							// 绘制日期标签（避免重叠）
							const maxLabels = Math.min(5, uniqueHistory.length); // 最大标签数量
							const labelStep = Math.max(1, Math.ceil(uniqueHistory.length / maxLabels));
							
							uniqueHistory.forEach((item, index) => {
								// 只显示每隔labelStep个的标签
								if (index % labelStep === 0 || index === uniqueHistory.length - 1) {
									const x = padding + index * stepX;
									ctx.fillStyle = '#666';
									ctx.font = `${Math.max(8, Math.floor(canvasHeight * 0.025))}px Arial`; // 动态字体大小
									ctx.textAlign = 'center';
									ctx.fillText(item.date.substring(5), x, canvasHeight - bottomPadding + 20); // 调整标签位置
								}
							});
						}
						
						ctx.draw();
					}
				});
			},
			// 显示CSV数据
			showCsvData() {
				// 准备导出数据
				let csvContent = "数据类型,日期,金额\n";
				
				// 添加历史记录
				this.moneyHistory.forEach(item => {
					csvContent += `历史记录,${item.date},${item.amount.toFixed(2)}\n`;
				});
				
				// 添加账户信息
				csvContent += "\n账户名称,金额\n";
				this.accounts.forEach(account => {
					csvContent += `${account.name},${account.amount.toFixed(2)}\n`;
				});
				
				// 保存CSV数据
				this.csvData = csvContent;
				// 显示对话框
				this.showCsvDialog = true;
			},
			// 关闭CSV对话框
			closeCsvDialog() {
				this.showCsvDialog = false;
			},
			// 复制CSV数据
			copyCsvData() {
				uni.setClipboardData({
					data: this.csvData,
					success: () => {
						uni.showToast({
							title: '复制成功',
							icon: 'success'
						});
						// 2秒后关闭对话框
						setTimeout(() => {
							this.showCsvDialog = false;
						}, 1000);
					},
					fail: () => {
						uni.showToast({
							title: '复制失败',
							icon: 'none'
						});
					}
				});
			}
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		padding: 20rpx;
		background-color: #f5f5f5;
		min-height: 100vh;
	}

	/* 总存款样式 */
	.total-amount {
		background-color: #fff;
		border-radius: 10rpx;
		padding: 30rpx;
		text-align: center;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
	}

	.total-label {
		font-size: 32rpx;
		color: #666;
		display: block;
		margin-bottom: 10rpx;
	}

	.total-value {
		font-size: 48rpx;
		font-weight: bold;
		color: #007AFF;
	}

	/* 操作入口样式 */
	.actions-section {
		display: flex;
		gap: 20rpx;
		margin-bottom: 20rpx;
	}

	.actions-section button {
		flex: 1;
	}

	/* 账户列表样式 */
	.accounts-section {
		background-color: #fff;
		border-radius: 10rpx;
		padding: 20rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
	}

	.section-title {
		font-size: 32rpx;
		font-weight: bold;
		color: #333;
		margin-bottom: 20rpx;
	}

	.accounts-list {
		display: flex;
		flex-direction: column;
		gap: 15rpx;
	}

	.account-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 15rpx;
		border-bottom: 1rpx solid #eee;
	}

	.account-info {
		display: flex;
		flex-direction: column;
	}

	.account-name {
		font-size: 28rpx;
		color: #333;
		margin-bottom: 5rpx;
	}

	.account-amount {
		font-size: 32rpx;
		font-weight: bold;
		color: #007AFF;
	}

	.account-actions {
		display: flex;
		gap: 10rpx;
	}

	.empty-account {
		text-align: center;
		padding: 40rpx;
		color: #999;
		font-size: 28rpx;
	}

	/* 图表样式 */
	.chart-section {
		background-color: #fff;
		border-radius: 10rpx;
		padding: 20rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
	}

	.chart-container {
		align-items: center;
		justify-content: center;
		display: flex;
	}

	.chart {
			width: 100%;
			height: 500rpx; /* 进一步增加画布高度 */
		}

		/* 图表头部样式 */
		.chart-header {
			display: flex;
			justify-content: space-between;
			align-items: center;
			margin-bottom: 20rpx;
		}

		.time-range-selector {
		}

		.time-range-button {
			padding: 10rpx 20rpx;
			background-color: #f0f0f0;
			border-radius: 20rpx;
			font-size: 26rpx;
			color: #333;
		}

	/* 历史记录样式 */
	.history-section {
		background-color: #fff;
		border-radius: 10rpx;
		padding: 20rpx;
		margin-bottom: 20rpx;
		box-shadow: 0 2rpx 5rpx rgba(0, 0, 0, 0.1);
	}

	.history-list {
		display: flex;
		flex-direction: column;
		gap: 10rpx;
	}

	.history-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 10rpx 0;
		border-bottom: 1rpx solid #f0f0f0;
	}

	.history-info {
		display: flex;
		flex-direction: column;
	}

	.history-date {
		font-size: 26rpx;
		color: #666;
		margin-bottom: 5rpx;
	}

	.history-amount {
		font-size: 26rpx;
		color: #007AFF;
		font-weight: bold;
	}

	.history-actions {
		display: flex;
		gap: 10rpx;
	}

	/* 对话框样式 */
	.dialog-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0, 0, 0, 0.5);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 999;
	}

	.dialog-content {
		background-color: #fff;
		border-radius: 10rpx;
		padding: 30rpx;
		width: 80%;
		max-width: 500rpx;
	}

	.dialog-title {
		font-size: 32rpx;
		font-weight: bold;
		color: #333;
		text-align: center;
		margin-bottom: 30rpx;
	}

	.form-item {
		margin-bottom: 20rpx;
	}

	.form-item text {
		display: block;
		font-size: 28rpx;
		color: #666;
		margin-bottom: 10rpx;
	}

	.form-item input {
		border: 1rpx solid #ddd;
		border-radius: 5rpx;
		padding: 15rpx;
		font-size: 28rpx;
	}

	.picker-container {
		display: flex;
		justify-content: space-between;
		align-items: center;
		border: 1rpx solid #ddd;
		border-radius: 5rpx;
		padding: 15rpx;
		background-color: #f9f9f9;
	}

	.picker-label {
		font-size: 28rpx;
		color: #333;
	}

	.picker-button {
		color: #007AFF;
		font-size: 28rpx;
	}

	.dialog-actions {
		display: flex;
		gap: 20rpx;
		margin-top: 30rpx;
	}

	.dialog-actions button {
			flex: 1;
		}

		/* 导出按钮样式 */
		.export-section {
			margin-top: 30rpx;
			margin-bottom: 30rpx;
		}

		.export-section button {
			width: 100%;
			padding: 20rpx;
			font-size: 28rpx;
		}

		/* CSV对话框样式 */
		.dialog-content-large {
			width: 90%;
			max-height: 80vh;
		}

		.csv-content {
			margin: 20rpx 0;
		}

		.csv-textarea {
			width: 100%;
			height: 400rpx;
			padding: 15rpx;
			border: 1rpx solid #ddd;
			border-radius: 5rpx;
			font-size: 24rpx;
			background-color: #f9f9f9;
		}

		.csv-textarea:read-only {
			background-color: #f0f0f0;
		}
</style>