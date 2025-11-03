<template>
	<div class="system-role-dialog-container">
		<el-dialog width="30%" v-model="state.dialog.isShowDialog" :close-on-click-modal="false" draggable>
			<template #header>
				<div class="flex items-center justify-between">
					<div>{{ state.dialog.title }}</div>
					<div class="flex mr-16">
						<el-checkbox :label="$t('common.expand')" @change="handleExpand" />
						<el-checkbox :label="$t('common.selectAll')" @change="handleSelectAll" />
					</div>
				</div>
			</template>
			<el-scrollbar class="h-[400px] sm:h-[600px]">
				<el-tree
					v-loading="loading"
					ref="menuTree"
					:data="state.treeData"
					:check-strictly="true"
					:props="state.defaultProps"
					:checkedKeys="checkedKeysArr"
					class="filter-tree"
					node-key="id"
					highlight-current
					show-checkbox
					@check-change="handleCheckChange"
				/>
			</el-scrollbar>
			<template #footer>
				<span class="dialog-footer">
					<el-button @click="state.dialog.isShowDialog = false">取 消</el-button>
					<el-button type="primary" @click="onSubmit">{{ state.dialog.submitTxt }}</el-button>
				</span>
			</template>
		</el-dialog>
	</div>
</template>

<script setup lang="ts" name="role-permession">
import { fetchRoleTree, permissionUpd } from '/@/api/admin/role';
import { pageList } from '/@/api/admin/menu';
import { useMessage } from '/@/hooks/message';
import { Ref } from 'vue';
import { useI18n } from 'vue-i18n';
// import other from '/@/utils/other';
import { CheckboxValueType } from 'element-plus';

const { t } = useI18n();

const menuTree = ref();
const isInit = ref(true); // 标志位，控制是否为初始化阶段

const loading = ref(false);

const state = reactive({
	checkedKeys: [] as any[],
	treeData: [] as any[],
	defaultProps: {
		label: 'name',
		value: 'id',
	},
	roleId: '',
	dialog: {
		isShowDialog: false,
		title: '分配权限',
		submitTxt: '更新',
	},
});

const checkedKeysArr = ref<any[]>([]);

const checkedKeys: Ref<any[]> = ref([]);

// 获取所有子节点 ID
const getChildIds = (node: any) => {
	let ids: any = [];
	if (node.children) {
		node.children.forEach((child: any) => {
			ids.push(child.id);
			if (child.children) {
				ids = ids.concat(getChildIds(child));
			}
		});
	}
	return ids;
};

// 打开弹窗
const openDialog = (row: any) => {
	isInit.value = true;
	state.checkedKeys = [];
	state.treeData = [];
	checkedKeys.value = [];
	state.roleId = row.roleId;
	loading.value = true;
	fetchRoleTree(row.roleId)
		.then((res) => {
			checkedKeys.value = res.data;
			return pageList();
		})
		.then((r) => {
			// 原本
			// state.treeData = r.data;
			// state.checkedKeys = other.resolveAllEunuchNodeId(state.treeData, checkedKeys.value, []);
			// 重构
			state.treeData = r.data;
			nextTick(() => {
				menuTree.value.setCheckedKeys(checkedKeys.value, false); // 取消级联
				setTimeout(() => {
					isInit.value = false; // 初始化完成后，设置为 false
				}, 2000);
			});
		})
		.finally(() => {
			loading.value = false;
		});
	state.dialog.isShowDialog = true;
};

// 处理点击选中逻辑
const handleCheckChange = (node: any, checked: any) => {
	if (isInit.value) return; // 如果是初始化阶段，不执行逻辑
	let currentCheckedKeys = menuTree.value.getCheckedKeys(); // 获取当前选中的节点

	if (checked) {
		// **点击的是父节点**
		if (node.children && node.children.length) {
			const childIds = getChildIds(node);
			currentCheckedKeys = [...new Set([...currentCheckedKeys, node.id, ...childIds])];
		}
	} else {
		// **取消选中**
		if (node.children && node.children.length) {
			const childIds = getChildIds(node);
			currentCheckedKeys = currentCheckedKeys.filter((id: any) => id !== node.id && !childIds.includes(id));
		}
	}

	// 设置新的选中状态
	menuTree.value.setCheckedKeys(currentCheckedKeys);
};

const handleExpand = (check: CheckboxValueType) => {
	const treeList = state.treeData;
	for (let i = 0; i < treeList.length; i++) {
		//@ts-ignore
		menuTree.value.store.nodesMap[treeList[i].id].expanded = check;
	}
};

const handleSelectAll = (check: CheckboxValueType) => {
	if (check) {
		menuTree.value?.setCheckedKeys(state.treeData.map((item) => item.id));
	} else {
		menuTree.value?.setCheckedKeys([]);
	}
};

// 提交授权数据
const onSubmit = () => {
	// 初始角色选择节点必须包含 【分配权限】 菜单
	if (state.roleId === '1') {
		if (
			!menuTree.value
				.getCheckedNodes()
				.map((item: { name: string }) => {
					return item.name;
				})
				.includes('分配权限')
		) {
			useMessage().error(t('sysrole.mustCheckOneTip'));
			return;
		}
	}

	const menuIds = menuTree.value.getCheckedKeys().join(',').concat(',').concat(menuTree.value.getHalfCheckedKeys().join(','));
	loading.value = true;
	permissionUpd(state.roleId, menuIds)
		.then(() => {
			state.dialog.isShowDialog = false;
			useMessage().success(t('common.editSuccessText'));
		})
		.finally(() => {
			loading.value = false;
		});
};

// 暴露变量
defineExpose({
	openDialog,
});
</script>
