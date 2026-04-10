<template>
    <div :class="!plugin ? 'flex gap-1 h-full' : 'flex flex-col gap-2'">
        <button
            class="flex bg-black text-white rounded justify-center border border-black hover:bg-gray-900 font-bold"
            :class="
                !plugin
                    ? 'items-center ml-1 gap-0.5 sm:gap-1 px-1 sm:px-2 md:px-4 py-1 sm:py-2 text-sm max-w-60'
                    : 'w-full px-4 py-2 my-2'
            "
            :style="!plugin ? { lineHeight: '0.9rem' } : {}"
            :aria-label="$t('HACK.toc.importChart')"
            tabindex="0"
            @click="uploadHighchartsConfig"
        >
            <svg
                class="flex-shrink-0"
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                stroke="#FFFFFF"
                viewBox="0 0 24 24"
            >
                <path
                    d="M12.5 17L12.5 4M12.5 4L18 8.78947M12.5 4L7 8.78947"
                    stroke-width="2"
                    stroke-linecap="round"
                ></path>
                <path d="M6 21H19" stroke-width="2" stroke-linecap="round"></path>
            </svg>
            {{ $t('HACK.toc.importChart') }}
        </button>
        <input
            ref="highchartsInput"
            type="file"
            accept=".json"
            class="hidden"
            tabindex="-1"
            :aria-label="$t('HACK.toc.importChart')"
            @change="handleConfigFileUpload"
        />
        <button
            class="flex bg-white justify-center rounded border border-black hover:bg-gray-100 font-bold"
            :class="
                btnDisabled && 'disabled hover:bg-gray-400',
                !plugin ? 'items-center sm:px-2 text-sm leading-none max-w-60' : 'w-full px-4 py-2'
            "
            :style="!plugin && { paddingInline: compactHeader ? '0.25rem' : '2rem' }"
            :aria-label="$t('HACK.toc.exportConfig')"
            :disabled="btnDisabled"
            :tabIndex="btnDisabled ? -1 : 0"
            @click="exportHighchartsConfig"
        >
            <svg
                class="flex-shrink-0"
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                stroke="#000000"
                viewBox="0 0 24 24"
            >
                <path d="M6 21H18M12 3V17M12 17L17 12M12 17L7 12" stroke-width="2" stroke-linecap="round"></path>
            </svg>
            {{ !compactHeader ? $t('HACK.toc.exportConfig') : '' }}
        </button>

        <a
            class="absolute right-2 bottom-2"
            href="https://www.youtube.com/watch?v=zzk0VQ0dVMU&feature=youtu.be"
            target="_blank"
            v-if="chartStore.chartConfig && chartStore.chartConfig.title?.text?.en.toLowerCase() === 'breakfast'"
        >
            <svg width="24" height="24" viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg" fill="#000000">
                <g id="SVGRepo_iconCarrier">
                    <path
                        fill="#F3F3F3"
                        d="M67.283 96.847c-31.675 10.868-26.284-8.737-53.132-23.014-26.776-14.24-12.708-42.414 18.464-60.401 37.712-21.76 74.573-17.987 66.18 17.987-8.393 35.972-4.946 55.519-31.512 65.428z"
                    ></path>
                    <circle fill="#F0C419" cx="50" cy="47" r="19"></circle>
                    <path
                        stroke="#ffffff"
                        stroke-width="4"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-miterlimit="10"
                        d="M79 7.031c6.138 0 13.807 6.252 15.077 14.969M53 35.378c3.757 1.081 6.71 3.965 8.025 7.622"
                        fill="none"
                    ></path>
                </g>
            </svg>
        </a>
    </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useChartStore } from '@/stores/chartStore';
import { useDataStore } from '@/stores/dataStore';

import { saveAs } from 'file-saver';
import type { LangId } from '@/definitions';
import { buildContextMenuLabels } from '@/utils/contextMenu';

import JSZip from 'jszip';

defineProps({
    plugin: {
        type: Boolean
    },
    compactHeader: {
        type: Boolean
    }
});

const { t } = useI18n();
const chartStore = useChartStore();
const dataStore = useDataStore();

const highchartsInput = ref<HTMLInputElement | null>(null);

const uploaded = computed(() => dataStore.uploaded);
const btnDisabled = computed(() => {
    const chartKeys = Object.keys(chartStore.chartConfig);
    const onlyContextMenu = chartKeys.length === 1 && chartKeys[0] === 'lang';
    return (!uploaded.value && (chartKeys.length === 0 || onlyContextMenu)) || dataStore.datatableView === false;
});

// upload an existing highcharts configured JSON
const uploadHighchartsConfig = () => {
    (highchartsInput.value as HTMLInputElement).click();
};

// handle uploaded highcharts config file
const handleConfigFileUpload = (event: Event) => {
    const configFile = Array.from((event.target as HTMLInputElement).files as ArrayLike<File>)[0];
    if (!configFile) {
        return;
    }
    // TODO: add file validation

    const reader = new FileReader();
    reader.onload = (e) => {
        const res = JSON.parse(e.target?.result as string);
        chartStore.setChartConfig(res);
        // extract data from the config file
        dataStore.extractGridData(chartStore.chartConfig);
    };
    reader.readAsText(configFile);

    // clear input
    (event.target as HTMLInputElement).value = '';
    dataStore.setDatatableView(true);
};

const localizeConfig = (chartConfig: any, lang: string): any => {
    if (Array.isArray(chartConfig)) {
        return chartConfig.map((item) => localizeConfig(item, lang));
    }

    if (chartConfig && typeof chartConfig === 'object') {
        // if it's a translation object
        if ('en' in chartConfig || 'fr' in chartConfig) {
            return chartConfig[lang] || '';
        }

        // otherwise recurse
        const result = {};
        for (const key in chartConfig) {
            result[key] = localizeConfig(chartConfig[key], lang);
        }
        return result;
    }

    return chartConfig;
};

const exportHighchartsConfig = async () => {
    // if bilingual charts is enabled, export both lang configs, otherwise just the uploaded/page file lang config
    const langsToExport = chartStore.isBilingual ? ['en', 'fr'] : [chartStore.activeLang];
    const files = langsToExport.map((lang) => {
        // stringify + create blob for exported config
        const localized = localizeConfig(chartStore.chartConfig, lang);

        // override the context menu translations for this language
        localized.lang = buildContextMenuLabels(t, lang as LangId);

        const filename = `${localized.title?.text?.toLowerCase() || 'chart'}-${lang}.json`;
        const json = JSON.stringify(localized, null, 2);

        return { filename, json };
    });

    if (files.length === 1) {
        // single file (no zip)
        const blob = new Blob([files[0].json], { type: 'application/json' });
        saveAs(blob, files[0].filename);
    } else {
        const zip = new JSZip();
        files.forEach(({ filename, json }) => zip.file(filename, json));
        const zipName = `${chartStore.chartConfig.title?.text[dataStore.uploadedFileLang].toLowerCase()}.zip`;

        const content = await zip.generateAsync({ type: 'blob' });
        saveAs(content, zipName);
    }
};
</script>
