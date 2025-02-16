<template>
    <div>
        <el-card width="30%" v-if="dockerStatus != 'Running'" class="mask-prompt">
            <span style="font-size: 14px">{{ $t('container.serviceUnavailable') }}</span>
            <el-button type="primary" link style="font-size: 14px; margin-bottom: 5px" @click="goSetting">
                【 {{ $t('container.setting') }} 】
            </el-button>
            <span style="font-size: 14px">{{ $t('container.startIn') }}</span>
        </el-card>

        <LayoutContent v-loading="loading" :title="$t('container.volume')" :class="{ mask: dockerStatus != 'Running' }">
            <template #toolbar>
                <el-row>
                    <el-col :span="16">
                        <el-button type="primary" @click="onCreate()">
                            {{ $t('container.createVolume') }}
                        </el-button>
                        <el-button type="primary" plain :disabled="selects.length === 0" @click="batchDelete(null)">
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
                    :data="data"
                    @search="search"
                >
                    <el-table-column type="selection" fix />
                    <el-table-column :label="$t('commons.table.name')" min-width="80" prop="name" fix>
                        <template #default="{ row }">
                            <Tooltip @click="onInspect(row.name)" :text="row.name" />
                        </template>
                    </el-table-column>
                    <el-table-column
                        :label="$t('container.mountpoint')"
                        show-overflow-tooltip
                        min-width="120"
                        prop="mountpoint"
                    />
                    <el-table-column
                        :label="$t('container.driver')"
                        show-overflow-tooltip
                        min-width="80"
                        prop="driver"
                    />
                    <el-table-column
                        prop="createdAt"
                        min-width="90"
                        :label="$t('commons.table.date')"
                        :formatter="dateFormat"
                    />
                    <fu-table-operations :buttons="buttons" :label="$t('commons.table.operate')" fix />
                </ComplexTable>
            </template>
        </LayoutContent>

        <CodemirrorDialog ref="codemirror" />
        <CreateDialog @search="search" ref="dialogCreateRef" />
    </div>
</template>

<script lang="ts" setup>
import LayoutContent from '@/layout/layout-content.vue';
import Tooltip from '@/components/tooltip/index.vue';
import ComplexTable from '@/components/complex-table/index.vue';
import TableSetting from '@/components/table-setting/index.vue';
import CreateDialog from '@/views/container/volume/create/index.vue';
import CodemirrorDialog from '@/components/codemirror-dialog/codemirror.vue';
import { reactive, onMounted, ref } from 'vue';
import { dateFormat } from '@/utils/util';
import { deleteVolume, searchVolume, inspect, loadDockerStatus } from '@/api/modules/container';
import { Container } from '@/api/interface/container';
import i18n from '@/lang';
import { useDeleteData } from '@/hooks/use-delete-data';
import router from '@/routers';

const loading = ref();
const detailInfo = ref();
const codemirror = ref();

const data = ref();
const selects = ref<any>([]);
const paginationConfig = reactive({
    currentPage: 1,
    pageSize: 10,
    total: 0,
});
const searchName = ref();

const dockerStatus = ref();
const loadStatus = async () => {
    const res = await loadDockerStatus();
    dockerStatus.value = res.data;
    if (dockerStatus.value === 'Running') {
        search();
    }
};
const goSetting = async () => {
    router.push({ name: 'ContainerSetting' });
};

const dialogCreateRef = ref<DialogExpose>();

interface DialogExpose {
    acceptParams: () => void;
}
const onCreate = async () => {
    dialogCreateRef.value!.acceptParams();
};

const search = async () => {
    const params = {
        info: searchName.value,
        page: paginationConfig.currentPage,
        pageSize: paginationConfig.pageSize,
    };
    loading.value = true;
    await searchVolume(params)
        .then((res) => {
            loading.value = false;
            data.value = res.data.items || [];
            paginationConfig.total = res.data.total;
        })
        .catch(() => {
            loading.value = false;
        });
};

const onInspect = async (id: string) => {
    const res = await inspect({ id: id, type: 'volume' });
    detailInfo.value = JSON.stringify(JSON.parse(res.data), null, 2);
    let param = {
        header: i18n.global.t('commons.button.view'),
        detailInfo: detailInfo.value,
    };
    codemirror.value!.acceptParams(param);
};

const batchDelete = async (row: Container.VolumeInfo | null) => {
    let names: Array<string> = [];
    if (row === null) {
        selects.value.forEach((item: Container.VolumeInfo) => {
            names.push(item.name);
        });
    } else {
        names.push(row.name);
    }
    await useDeleteData(deleteVolume, { names: names }, 'commons.msg.delete');
    search();
};

const buttons = [
    {
        label: i18n.global.t('commons.button.delete'),
        click: (row: Container.VolumeInfo) => {
            batchDelete(row);
        },
    },
];

onMounted(() => {
    loadStatus();
});
</script>
