<template>
	<div class="st-stable" :class="[config.layout=='expand'?'st-expand-stable':'st-fixed-stable']">
		<x-title></x-title>
		<x-tip></x-tip>
		<x-toolbar></x-toolbar>
		<x-search ref="search"></x-search>
		<x-table ref="table"></x-table>
		<x-pagination></x-pagination>
		<x-chart v-if="config.chart"></x-chart>
	</div>
</template>
<script>
	import XTitle from 'js/section/Title.vue';
	import XTip from 'js/section/Tip.vue';
	import XToolbar from 'js/section/Toolbar.vue';
	import XSearch from 'js/section/Search.vue';
	import XTable from 'js/section/Table.vue';
	import XPagination from 'js/section/Pagination.vue';
	import XChart from 'js/section/Chart.vue';
	import {hashCode} from "./util.js";
	import defaultLang from '../lang/zh.js';
	let stableCount = 0;
	export default {
		props: ['config'],
		provide() {
			let conf = Object.assign({
				//app的标题，显示在顶部。并且用做导出excel的默认文件名
				title: '',
				rowNumberVisible: false,
				selectMode: 'none',
				pageMode: 'normal', //normal、waterfall
				pageIndex: 'id',
				listeners: {},
				ignoreEmptySearchParam: true,
				params: {
					count: 20
				},
				parallelCount: 6,
				downloadTimeout: 10000,
				downloadAllFromJustOnePage: false,
				labelVisible: true,
				layout: 'fixed',
				searchResetable: false
			}, window&&window.STable && window.STable.default||{}, this.config);

			//国际化
			if(!conf.locale) {
				conf.locale = defaultLang;
			}else if(typeof conf.locale == 'string') {
				if(window.STable && Window.STable.lang[conf.locale]) {
					conf.locale = Window.STable.lang[conf.locale];
					conf.locale = conf.locale.default||conf.locale;
				} else {
					conf.locale = defaultLang;
				}
			}

			let methods = conf.actionMethods||conf.requestMethod||'GET';
			if(typeof methods=='string'){
				methods = {read: methods};
			}
			let actionMethods = {
				create: 'POST',
				read: 'GET',
				update: 'POST',
				destroy: 'POST'
			};
			conf.actionMethods = Object.assign({}, actionMethods, methods);

			if(conf.groupBy){
				if(!Array.isArray(conf.groupBy))
					conf.groupBy = [conf.groupBy];
			} else {
				conf.groupBy = [];
			}
			if(conf.sublistAt){
				if(!Array.isArray(conf.sublistAt))
					conf.sublistAt = [conf.sublistAt];
			} else {
				conf.sublistAt = [];
			}

			let additionalColumnConfig = conf.additionalColumnConfig||false;
			let columns = conf.columns.map((item,idx)=>{
				if(typeof item == 'string') {
					item = {
						text: item,
						dataIndex: item
					};
				}
				if(additionalColumnConfig) {
					if(Array.isArray(additionalColumnConfig)) {
						Object.assign(item, additionalColumnConfig[idx]);
					} else if(item.dataIndex && additionalColumnConfig[item.dataIndex]) {
						Object.assign(item, additionalColumnConfig[item.dataIndex]);
					}
				}
				if (item.header) {
					item.text = item.header;
				}
				//防止有options字段，又没有配置可选值
				if (item.options && Object.keys(item.options).length<=0){
					item.options = false;
				}
				if (item.buttons) {
					if(!item.type)
						item.type = 'button';
					item.buttons.forEach(btn=>{
						if(btn.icon) {
							if(!btn.icon.startsWith('el-icon-')){
								btn.icon = 'el-icon-_fa fa fa-'+btn.icon;
							}
						}
					});
					if(item.type=='button' && typeof item.width=='undefined') {
						item.width = item.buttons.length*100;
					}
				}
				if (!item.dataIndex) {
					item.dataIndex = "stable_column_"+idx;
				}
				if (item.render) {
					item.type = 'render';
				}
				if (!item.type) {
					item.type = 'text';
				}
				if(typeof item.width=='undefined' && typeof item.flex=='undefined') {
					item.flex = 1;
				}

				if(typeof item.width!='undefined') {
					if(typeof item.width=='string') {}
				}

				let _type = typeof item.width;
				if(_type != 'undefined') {
					if(_type=='string') {
						//可能是百分比，否则全转化为整数
						if(!/^[\d\.]+%$/.test(item.width)) {
							item.width = parseInt(item.width, 10);
						}
					}
				} else if(typeof item.flex != 'undefined') {
					item.flex = parseFloat(item.flex);
				} else {
					item.flex = 1;
				}
				item = Object.assign({
					visible: true,
					locked: false,
					cellWrap: true,
					_st_idx: idx,
					_st_ori_idx: idx
				},item);
				if(!item.text)
					item.text = '-';
				if(item.fx)
					item.fx = item.fx.toLowerCase();
				
				return item;
			});
			
			if(stableCount==0 && window.location.search.includes('stable=on')) {
				let searchParams = new URLSearchParams(window.location.search);
				let sp = {};
				for(let key of searchParams.keys()) {
					let val = searchParams.getAll(key);
					if(val.length>1)
						sp[key] = val;
					else
						sp[key] = val[0];
				}
				delete sp.stable;

				//兼容之前版使用的postParam 和 postData
				Object.assign(conf.params, conf.postParam, conf.postData, sp);
			}

			if(conf.addUrl) {
				conf.addConf = conf.addConf || conf.editConfig||conf.editConf||conf.metaEditConf;
			}
			if(conf.editUrl) {
				conf.updateUrl = conf.editUrl;
			}
			if(conf.updateUrl) {
				conf.editConf = conf.editConfig||conf.editConf||conf.metaEditConf;
			}
			if(conf.deleteUrl || conf.updateUrl) {
				columns.push({
					dataIndex:'_wd_aux_op',
					type: 'button',
					text: conf.locale.operation,
					_width: 0,
					visible: true,
					locked: false,
					cellWrap: true,
					_st_idx: columns.length,
					_st_ori_idx: columns.length,
					dynamicParallelCount: false,
					buttons: []
				});
			}
			let selectMode = conf.selectMode.trim().toLowerCase();
			if(['radio', 'single'].includes(selectMode)){
				conf.selectMode = 'single';
			} else if(['checkbox', 'multi', 'multiple'].includes(selectMode)){
				conf.selectMode = 'multiple';
			} else {
				conf.selectMode = 'none';
			}

			if(typeof conf.page != 'undefined') {
				conf.params.page = conf.page;
			}
			if(conf.sort_key)
				conf.sortKey = conf.sort_key;
			if(conf.sort_direction)
				conf.sortDirection = conf.sort_direction;
			if(!conf.sortDirection)
				conf.sortDirection = 'asc';

			conf._key = hashCode(JSON.stringify(columns));

			let localColumnSet;
			try{
				localColumnSet = window.localStorage.getItem(conf._key);
				if(localColumnSet)
					localColumnSet = JSON.parse(localColumnSet);
			}catch(e){
				localColumnSet = '';
				alert(conf.locale.columnStorageError);
			}
			if(localColumnSet) {
				let _columns = [];
				for(let col of localColumnSet) {
					for(let colConf of columns){
						if(colConf._st_ori_idx == col._st_ori_idx){
							Object.assign(colConf, {
								locked: col.locked,
								visible: col.visible,
								_st_idx: col._st_idx
							});
							_columns.push(colConf);
							break;
						}
					}
				}
				columns = _columns;
			}

			conf.store = new Vue({
				data: {
					columns,
					page: conf.params.page || 1,
					page_count: 1,
					total: 0,
					hasNextPage: true,
					hasPrePage: false,
					loadAction: '',
					radioVal: '',
					checkboxVal: [],
					sortKey: conf.sortKey||'',
					sortDirection: conf.sortDirection
				},
				methods: {
					saveColumnsState(){
						let colState = this.columns.map(col=>{
							return {
								text: col.text,
								visible: col.visible,
								locked: col.locked,
								_st_idx: col._st_idx,
								_st_ori_idx: col._st_ori_idx
							};
						});

						try{
							window.localStorage.setItem(conf._key, JSON.stringify(colState));
						}catch(e){
							console.log(e);
							//
						}
					},
					resetColumnsState(){
						try{
							window.localStorage.removeItem(conf._key);
							location.reload();
						}catch(e){
							console.log(e);
							//
						}
					}
				}
			});
			conf.store.searchParams = {};
			conf.store.$on('cellclick', evt=>{
				if(conf.listeners.cellclick){
					conf.listeners.cellclick.call(this, evt.record, evt.col, evt.evt);
				}
			});
			conf.store.$on('rowclick', evt=>{
				if(conf.listeners.rowclick) {
					conf.listeners.rowclick.call(this, evt.record, evt.evt);
				}
			});
			conf.store.$on('refresh', records=>{
				if(conf.listeners.refresh) {
					conf.listeners.refresh.call(this, records);
				}
			});
			delete conf.columns;
			delete conf.params.page;

			if(conf.listeners.beforemounted) {
				conf.listeners.beforemounted.call(this, conf);
			}
			if(window.STable && window.STable.processConfig){
				conf = window.STable.processConfig(conf);
			}

			return conf;
		},
		components: {
			XTitle,
			XTip,
			XToolbar,
			XSearch,
			XTable,
			XPagination,
			XChart
		},
		mounted(){
			if(this.config.listeners && this.config.listeners.ready){
				this.config.listeners.ready.call(this);
			}
			stableCount++;
		},
		methods: {
			refresh(pno) {
				return this.$refs.table.refresh(pno);
			},
			getSearchParam(){
				return this.$refs.search.getParams();
			},
			getSelectRows(){
				return this.$refs.table.getSelectRows();
			},
			getSelectedRows(){
				return this.getSelectRows();
			},
			layout(){
				this.$refs.table.layout();
			},
			setRecords(list){
				this.$refs.table.setRecords(list);
			}
		}
	}
</script>
<style>
	@import url(../css/index.css);
</style>