<template>
	<div v-if="!hideTitle" class="st-title">
		<div class="st-title-text" v-text="title"></div>
		<div v-if="chart" @click="store.$emit('chartshow',$event)" class="st-title-tool st-title-">📊</div>
	</div>
</template>
<script>
	import Dialog from '../com/Dialog.js';

	export default {
		inject: {
			_key: '_key',
			chart: {
				default: false
			},
			title: {
				default: ''
			},
			hideTitle: {
				default: false
			},
			store: 'store',
			locale: 'locale'
		},
		methods: {
			showConfig() {
				let stable = this;
				Dialog.create({
					title: this.locale.columnSetting,
					width: 500,
					autoShow: true,
					bodyStyle: {padding: 0},
					data: {
						stableConfig: {
							hideTitle: true,
							columns: [{
								text: this.locale.columnName,
								dataIndex: 'text',
								cellWrap: true
							},{
								width: 60,
								text: this.locale.lock,
								dataIndex: 'locked',
								render(record, col, idx) {
									return `<label class="st-title-cog-label"><input type="checkbox" data-locked value="${idx}" ${record.locked?'checked':''} /></label>`;
								}
							},{
								width: 60,
								text: this.locale.visible,
								dataIndex: 'visible',
								render(record, col, idx) {
									return `<label class="st-title-cog-label"><input type="checkbox" data-visible value="${idx}" ${record.visible?'checked':''} /></label>`;
								}
							}],
							records: this.store.columns.map(col=>{
								return Object.assign({}, col);
							})
						}
					},
					html: '<x-stable :config="stableConfig"></x-stable>',
					buttons: [{
						text: this.locale.saveColumnSetting,
						type: 'success',
						click(){
							let lockedChecks = this.$el.querySelectorAll('[data-locked]');
							let visibleChecks = this.$el.querySelectorAll('[data-visible]');
							let columns = stable.store.columns;
							lockedChecks.forEach((c, idx)=>{
								columns[idx].locked = c.checked;
							});
							visibleChecks.forEach((c,idx)=>{
								columns[idx].visible = c.checked;
							});
							this.close();
							stable.store.columns = Array.from(columns);
							
							stable.store.saveColumnsState();
						}
					},{
						text: this.locale.clearColumnSetting,
						type: 'danger',
						click(){
							if(confirm(stable.locale.clearColumnSettingTips)) {
								stable.store.resetColumnsState();
							}
						}
					}, {
						text: this.locale.cancel,
						click(){
							this.close();
						}
					}]
				});
			}
		}
	}
</script>
<style>
	.st-title{
		color: #191919;
		height: 40px;
		background-color: #f8f8f8;
		border-bottom: 1px solid #d0d0d0;

		display: flex;
		justify-content: space-between;
		align-items: center;
	}
	.st-title-text{
		font-size: 16px;
		padding-left: 10px;
		font-weight: 500;
		flex: 1;
	}
	.st-title-tool{
		margin-right: 10px;
		cursor: pointer;
	}
	.st-title-cog-label{
		display: block;
	}
</style>