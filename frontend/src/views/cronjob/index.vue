<template>
    <div>
        <RouterButton
            :buttons="[
                {
                    label: i18n.global.t('cronjob.cronTask'),
                    path: '/cronjobs',
                },
            ]"
        />
        <LayoutContent v-loading="loading" v-if="!isRecordShow" :title="$t('cronjob.cronTask')">
            <template #toolbar>
                <el-row>
                    <el-col :span="16">
                        <el-button type="primary" @click="onOpenDialog('create')">
                            {{ $t('commons.button.create') }}{{ $t('cronjob.cronTask') }}
                        </el-button>
                        <el-button type="primary" plain :disabled="selects.length === 0" @click="onBatchDelete(null)">
                            {{ $t('commons.button.delete') }}
                        </el-button>
                    </el-col>
                    <el-col :span="8">
                        <TableSetting @search="search()" />
                        <div class="search-button">
                            <el-input
                                v-model="searchName"
                                clearable
                                @clear="search()"
                                suffix-icon="Search"
                                @keyup.enter="search()"
                                @blur="search()"
                                :placeholder="$t('commons.button.search')"
                            ></el-input>
                        </div>
                    </el-col>
                </el-row>
            </template>
            <template #main>
                <ComplexTable
                    :pagination-config="paginationConfig"
                    v-model:selects="selects"
                    @search="search"
                    :data="data"
                >
                    <el-table-column type="selection" fix />
                    <el-table-column :label="$t('cronjob.taskName')" :min-width="120" prop="name">
                        <template #default="{ row }">
                            <Tooltip @click="loadDetail(row)" :text="row.name" />
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('commons.table.status')" :min-width="80" prop="status">
                        <template #default="{ row }">
                            <el-button
                                v-if="row.status === 'Enable'"
                                @click="onChangeStatus(row.id, 'disable')"
                                link
                                type="success"
                            >
                                {{ $t('commons.status.enabled') }}
                            </el-button>
                            <el-button v-else link type="danger" @click="onChangeStatus(row.id, 'enable')">
                                {{ $t('commons.status.disabled') }}
                            </el-button>
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('cronjob.cronSpec')" :min-width="120">
                        <template #default="{ row }">
                            <span v-if="row.specType.indexOf('N') === -1 || row.specType === 'perWeek'">
                                {{ $t('cronjob.' + row.specType) }}
                            </span>
                            <span v-else>{{ $t('cronjob.per') }}</span>
                            <span v-if="row.specType === 'perMonth'">
                                {{ row.day }}{{ $t('cronjob.day') }} {{ loadZero(row.hour) }} :
                                {{ loadZero(row.minute) }}
                            </span>
                            <span v-if="row.specType === 'perWeek'">
                                {{ loadWeek(row.week) }} {{ loadZero(row.hour) }} : {{ loadZero(row.minute) }}
                            </span>
                            <span v-if="row.specType === 'perDay'">
                                &#32;{{ loadZero(row.hour) }} : {{ loadZero(row.minute) }}
                            </span>
                            <span v-if="row.specType === 'perNDay'">
                                {{ row.day }} {{ $t('cronjob.day1') }}, {{ loadZero(row.hour) }} :
                                {{ loadZero(row.minute) }}
                            </span>
                            <span v-if="row.specType === 'perNHour'">
                                {{ row.hour }}{{ $t('cronjob.hour') }}, {{ loadZero(row.minute) }}
                            </span>
                            <span v-if="row.specType === 'perHour'">{{ loadZero(row.minute) }}</span>
                            <span v-if="row.specType === 'perNMinute'">{{ row.minute }}{{ $t('cronjob.minute') }}</span>
                            {{ $t('cronjob.handle') }}
                        </template>
                    </el-table-column>
                    <el-table-column :label="$t('cronjob.retainCopies')" :min-width="90" prop="retainCopies">
                        <template #default="{ row }">
                            {{ loadCopies(row) }}
                        </template>
                    </el-table-column>

                    <el-table-column :label="$t('cronjob.lastRecrodTime')" :min-width="120" prop="lastRecrodTime">
                        <template #default="{ row }">
                            {{ row.lastRecrodTime }}
                        </template>
                    </el-table-column>
                    <el-table-column :min-width="80" :label="$t('cronjob.target')" prop="targetDir">
                        <template #default="{ row }">
                            {{ row.targetDir }}
                        </template>
                    </el-table-column>
                    <fu-table-operations
                        width="200px"
                        :buttons="buttons"
                        :ellipsis="10"
                        :label="$t('commons.table.operate')"
                        fix
                    />
                </ComplexTable>
            </template>
        </LayoutContent>

        <OperatrDialog @search="search" ref="dialogRef" />
        <Records @search="search()" ref="dialogRecordRef" />
    </div>
</template>

<script lang="ts" setup>
import ComplexTable from '@/components/complex-table/index.vue';
import TableSetting from '@/components/table-setting/index.vue';
import Tooltip from '@/components/tooltip/index.vue';
import OperatrDialog from '@/views/cronjob/operate/index.vue';
import Records from '@/views/cronjob/record/index.vue';
import LayoutContent from '@/layout/layout-content.vue';
import { loadZero } from '@/utils/util';
import { onMounted, reactive, ref } from 'vue';
import RouterButton from '@/components/router-button/index.vue';
import { deleteCronjob, getCronjobPage, handleOnce, updateStatus } from '@/api/modules/cronjob';
import i18n from '@/lang';
import { Cronjob } from '@/api/interface/cronjob';
import { useDeleteData } from '@/hooks/use-delete-data';
import { ElMessageBox } from 'element-plus';
import { MsgSuccess } from '@/utils/message';

const loading = ref();
const selects = ref<any>([]);
const isRecordShow = ref();

const data = ref();
const paginationConfig = reactive({
    currentPage: 1,
    pageSize: 10,
    total: 0,
});
const searchName = ref();

const weekOptions = [
    { label: i18n.global.t('cronjob.monday'), value: 1 },
    { label: i18n.global.t('cronjob.tuesday'), value: 2 },
    { label: i18n.global.t('cronjob.wednesday'), value: 3 },
    { label: i18n.global.t('cronjob.thursday'), value: 4 },
    { label: i18n.global.t('cronjob.friday'), value: 5 },
    { label: i18n.global.t('cronjob.saturday'), value: 6 },
    { label: i18n.global.t('cronjob.sunday'), value: 7 },
];

const search = async () => {
    let params = {
        info: searchName.value,
        page: paginationConfig.currentPage,
        pageSize: paginationConfig.pageSize,
    };
    loading.value = true;
    await getCronjobPage(params)
        .then((res) => {
            loading.value = false;
            data.value = res.data.items || [];
            for (const item of data.value) {
                if (item.targetDir !== '-' && item.targetDir !== '') {
                    item.targetDir = i18n.global.t('setting.' + item.targetDir);
                }
            }
            paginationConfig.total = res.data.total;
        })
        .catch(() => {
            loading.value = false;
        });
};

const dialogRecordRef = ref();

const dialogRef = ref();
const onOpenDialog = async (
    title: string,
    rowData: Partial<Cronjob.CronjobInfo> = {
        specType: 'perMonth',
        type: 'shell',
        week: 1,
        day: 3,
        hour: 1,
        minute: 30,
        keepLocal: true,
        retainCopies: 7,
    },
) => {
    let params = {
        title,
        rowData: { ...rowData },
    };
    dialogRef.value!.acceptParams(params);
};

const onBatchDelete = async (row: Cronjob.CronjobInfo | null) => {
    let ids: Array<number> = [];
    if (row) {
        ids.push(row.id);
    } else {
        selects.value.forEach((item: Cronjob.CronjobInfo) => {
            ids.push(item.id);
        });
    }
    await useDeleteData(deleteCronjob, { ids: ids }, 'commons.msg.delete');
    search();
};

const onChangeStatus = async (id: number, status: string) => {
    ElMessageBox.confirm(i18n.global.t('cronjob.' + status + 'Msg'), i18n.global.t('cronjob.changeStatus'), {
        confirmButtonText: i18n.global.t('commons.button.confirm'),
        cancelButtonText: i18n.global.t('commons.button.cancel'),
    }).then(async () => {
        let itemStatus = status === 'enable' ? 'Enable' : 'Disable';
        await updateStatus({ id: id, status: itemStatus });
        MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
        search();
    });
};

const onHandle = async (row: Cronjob.CronjobInfo) => {
    loading.value = true;
    await handleOnce(row.id)
        .then(() => {
            loading.value = false;
            MsgSuccess(i18n.global.t('commons.msg.operationSuccess'));
            search();
        })
        .catch(() => {
            loading.value = false;
        });
};

const loadCopies = (item) => {
    if (item.type === 'shell' || item.type === 'curl') {
        return '-';
    } else {
        return item.retainCopies + '';
    }
};

const loadDetail = (row: any) => {
    isRecordShow.value = true;
    let params = {
        rowData: { ...row },
    };
    dialogRecordRef.value!.acceptParams(params);
};

const buttons = [
    {
        label: i18n.global.t('commons.button.handle'),
        click: (row: Cronjob.CronjobInfo) => {
            onHandle(row);
        },
    },
    {
        label: i18n.global.t('commons.button.edit'),
        click: (row: Cronjob.CronjobInfo) => {
            onOpenDialog('edit', row);
        },
    },
    {
        label: i18n.global.t('cronjob.record'),
        click: (row: Cronjob.CronjobInfo) => {
            loadDetail(row);
        },
    },
    {
        label: i18n.global.t('commons.button.delete'),
        click: (row: Cronjob.CronjobInfo) => {
            onBatchDelete(row);
        },
    },
];
function loadWeek(i: number) {
    for (const week of weekOptions) {
        if (week.value === i) {
            return week.label;
        }
    }
    return '';
}
onMounted(() => {
    search();
});
</script>
