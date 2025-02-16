<template>
    <el-drawer :close-on-click-modal="false" v-model="open" size="50%" :before-close="handleClose">
        <template #header>
            <Header :header="$t('website.group')" :back="handleClose"></Header>
        </template>

        <ComplexTable :data="data" @search="search()">
            <template #toolbar>
                <el-button type="primary" @click="openCreate">{{ $t('website.createGroup') }}</el-button>
            </template>
            <el-table-column :label="$t('commons.table.name')" prop="name">
                <template #default="{ row }">
                    <span v-if="!row.edit">
                        {{ row.name }}
                        <span v-if="row.default">({{ $t('website.default') }})</span>
                    </span>
                    <el-form ref="groupForm" :model="row" :rules="rules">
                        <el-form-item prop="name" v-if="row.edit">
                            <el-input v-model="row.name"></el-input>
                        </el-form-item>
                    </el-form>
                </template>
            </el-table-column>
            <el-table-column :label="$t('commons.table.operate')">
                <template #default="{ row, $index }">
                    <div>
                        <el-button link v-if="row.edit" type="primary" @click="saveGroup(groupForm, row, false)">
                            {{ $t('commons.button.save') }}
                        </el-button>
                        <el-button link v-if="!row.edit" type="primary" @click="editGroup($index)">
                            {{ $t('commons.button.edit') }}
                        </el-button>
                        <el-button
                            link
                            v-if="!row.edit"
                            :disabled="row.default"
                            type="primary"
                            @click="deleteGroup($index)"
                        >
                            {{ $t('commons.button.delete') }}
                        </el-button>
                        <el-button link v-if="row.edit" type="primary" @click="cancelEdit($index)">
                            {{ $t('commons.button.cancel') }}
                        </el-button>
                        <el-button
                            link
                            v-if="!row.edit && !row.default"
                            type="primary"
                            @click="saveGroup(groupForm, row, true)"
                        >
                            {{ $t('website.setDefault') }}
                        </el-button>
                    </div>
                </template>
            </el-table-column>
        </ComplexTable>
    </el-drawer>
</template>
<script lang="ts" setup>
import { ref } from 'vue';
import i18n from '@/lang';
import ComplexTable from '@/components/complex-table/index.vue';
import { ListGroups, CreateGroup, DeleteGroup, UpdateGroup } from '@/api/modules/website';
import Header from '@/components/drawer-header/index.vue';
import { MsgSuccess } from '@/utils/message';
import { Rules } from '@/global/form-rules';
import { FormInstance } from 'element-plus';

interface groupData {
    id: number;
    name: string;
    edit: boolean;
    default: boolean;
}

let rules = ref<any>({
    name: [Rules.name],
});
const groupForm = ref<FormInstance>();

let open = ref(false);
let data = ref<groupData[]>([]);
const handleClose = () => {
    open.value = false;
    data.value = [];
};

const search = () => {
    data.value = [];
    ListGroups().then((res) => {
        for (const d of res.data) {
            const g = {
                id: d.id,
                name: d.name,
                default: d.default,
                edit: false,
            };
            data.value.push(g);
        }
    });
};

const saveGroup = async (formEl: FormInstance, create: groupData, isDefault: boolean) => {
    if (!formEl) return;
    await formEl.validate((valid) => {
        if (!valid) {
            return;
        }
        const group = {
            name: create.name,
            id: create.id,
            default: create.default,
        };
        if (isDefault) {
            group.default = isDefault;
        }
        if (create.id == 0) {
            CreateGroup(group).then(() => {
                MsgSuccess(i18n.global.t('commons.msg.createSuccess'));
                search();
            });
        } else {
            UpdateGroup(group).then(() => {
                MsgSuccess(i18n.global.t('commons.msg.updateSuccess'));
                search();
            });
        }
    });
};

const acceptParams = async () => {
    open.value = true;
    search();
};

const openCreate = () => {
    for (const d of data.value) {
        if (d.name == '') {
            return;
        }
        if (d.edit) {
            d.edit = false;
        }
    }
    const g = {
        id: 0,
        name: '',
        default: false,
        edit: true,
    };
    data.value.push(g);
};

const deleteGroup = (index: number) => {
    const group = data.value[index];

    if (group.id > 0) {
        DeleteGroup({ id: group.id }).then(() => {
            data.value.splice(index, 1);
            MsgSuccess(i18n.global.t('commons.msg.deleteSuccess'));
        });
    } else {
        data.value.splice(index, 1);
    }
};

const editGroup = (index: number) => {
    for (const i in data.value) {
        const d = data.value[i];
        if (d.name == '') {
            data.value.splice(Number(i), 1);
        }
        if (d.edit) {
            d.edit = false;
        }
    }
    data.value[index].edit = true;
};

const cancelEdit = (index: number) => {
    if (data.value[index].id == 0) {
        data.value.splice(index, 1);
    } else {
        data.value[index].edit = false;
        search();
    }
};

defineExpose({ acceptParams });
</script>
