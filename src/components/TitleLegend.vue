<template>
    <div class="trading-vue-legend title-legend" :style="calc_style">
        <div v-if="(grid_id === 0 && showTitleChartLegend)" class="trading-vue-ohlcv"
            :style="{ 'max-width': common.width + 'px' }">
            <template v-if="common?.showLegendPropsData && common.showLegendPropsData.length">
                <b v-for="(n, i) in common.showLegendPropsData" :key="i">{{ n.k }} : {{ n.v }}&nbsp;</b><br />
            </template>
            <template v-if="show_CustomProps"><span v-for="(n, i) in legendTxtConfig" :key="i"
                    :style="n.style">{{ n.name }}&nbsp;</span></template>
            <span class="t-vue-title" v-if="!show_CustomProps" :style="{ color: common.colors.title }">
                {{ common.title_txt }}
            </span>
            <span class="t-vue-exchange">
                {{ common.exchange_txt }}
            </span>
            <span v-if="show_values && !show_CustomProps">
                O<span class="t-vue-lspan">{{ ohlcv[0] }}</span>
                H<span class="t-vue-lspan">{{ ohlcv[1] }}</span>
                L<span class="t-vue-lspan">{{ ohlcv[2] }}</span>
                C<span class="t-vue-lspan">{{ ohlcv[3] }}</span>
                V<span class="t-vue-lspan">{{ ohlcv[4] }}</span>
            </span>
            <span v-if="!show_values" class="t-vue-lspan" :style="{ color: common.colors.text }">
                {{ (common.meta.last || [])[4] }}
            </span>
        </div>

    </div>
</template>
<script>

import ButtonGroup from './ButtonGroup.vue'
import Spinner from './Spinner.vue'
import LegendButton from "./LegendButton.vue";
import Icons from '../stuff/icons.json'
const settingPng = Icons['gear.png']
export default {
    name: 'TitleChartLegend',
    components: { LegendButton, ButtonGroup, Spinner },
    props: [
        'common', 'values', 'decimalPlace', 'grid_id', 'meta_props', 'legendDecimal', 'showTitleChartLegend',
    ],
    computed: {
        show_CustomProps() {
            return this.common?.show_CustomProps || false;
        },
        show_Settings() {
            return this.common?.showSettingsMain || false;
        },
        settingIcon() {
            return settingPng
        },
        legendTxtConfig() {
            return this.common?.legendTxtConfig;
        },
        ohlcv() {
            if (!this.$props.values || !this.$props.values.ohlcv) {
                return Array(6).fill('n/a')
            }
            // const prec = this.layout.prec
            const prec = this.decimalPlace
            // const prec = 3
            // TODO: main the main legend more customizable
            let id = this.main_type + '_0'
            let meta = this.$props.meta_props[id] || {}
            if (meta.legend) {
                return (meta.legend() || []).map(x => x.value)
            }

            if (this.$props.legendDecimal) {
                return [
                    this.$props.values.ohlcv[1].toFixed(this.$props.values.ohlcv[1] < 1 ? 3 : 2),
                    this.$props.values.ohlcv[2].toFixed(this.$props.values.ohlcv[2] < 1 ? 3 : 2),
                    this.$props.values.ohlcv[3].toFixed(this.$props.values.ohlcv[3] < 1 ? 3 : 2),
                    this.$props.values.ohlcv[4].toFixed(this.$props.values.ohlcv[4] < 1 ? 3 : 2),
                    this.$props.values.ohlcv[5] ?
                        Number(this.$props.values.ohlcv[5].toFixed(0)).toLocaleString('en-AU') :
                        'n/a'
                ]
            } else {
                return [
                    this.$props.values.ohlcv[1].toFixed(prec),
                    this.$props.values.ohlcv[2].toFixed(prec),
                    this.$props.values.ohlcv[3].toFixed(prec),
                    this.$props.values.ohlcv[4].toFixed(prec),
                    this.$props.values.ohlcv[5] ?
                        Number(this.$props.values.ohlcv[5].toFixed(0)).toLocaleString('en-AU') :
                        'n/a'
                ]
            }
        },
        // TODO: add support for { grid: { id : N }}
        indicators() {
            const values = this.$props.values
            const f = this.format
            var types = {}

            return this.json_data.filter(
                x => x.settings.legend !== false && !x.main
            ).map(x => {
                if (!(x.type in types)) types[x.type] = 0
                const id = x.type + `_${types[x.type]++}`
                return {
                    v: 'display' in x.settings ? x.settings.display : true,
                    name: x.name || id,
                    index: (this.off_data || this.json_data).indexOf(x),
                    id: id,
                    values: values ? f(id, values) : this.n_a(1),
                    unk: !(id in (this.$props.meta_props || {})),
                    loading: x.loading,
                    showLegendButtons: 'legendButtons' in x.settings ? x.settings.legendButtons : true
                }
            })
        },
        calc_style() {
            let top = this.layout.height > 150 ? 3 : 1
            let grids = this.$props.common.layout.grids
            let w = grids[0] ? grids[0].width : undefined
            return {
                top: `${this.layout.offset + top}px`,
                width: `${w - 20}px`
            }
        },
        layout() {
            const id = this.$props.grid_id
            return this.$props.common.layout.grids[id]
        },
        json_data() {
            return this.$props.common.data
        },
        off_data() {
            return this.$props.common.offchart
        },
        main_type() {
            let f = this.common.data.find(x => x.main)
            return f ? f.type : undefined
        },
        show_values() {
            return this.common.cursor.mode !== 'explore'
        }
    },
    methods: {
        format(id, values) {
            let meta = this.$props.meta_props[id] || {}
            // Matches Overlay.data_colors with the data values
            // (see Spline.vue)
            if (!values[id]) return this.n_a(1)

            // Custom formatter
            if (meta.legend) return meta.legend(values[id])

            return values[id].slice(1).map((x, i) => {
                const cs = meta.data_colors ? meta.data_colors() : []
                if (typeof x == 'number') {
                    // Show 8 digits for small values
                    x = x.toFixed(Math.abs(x) > 0.001 ? 4 : 8)
                }
                return {
                    value: x,
                    color: cs ? cs[i % cs.length] : undefined
                }
            })
        },
        n_a(len) {
            return Array(len).fill({ value: 'n/a' })
        },
        button_click(event) {
            this.$emit('legend-button-click', event)
        }
    }
}
</script>
<style>
.trading-vue-legend {
    position: relative;
    z-index: 1;
    font-size: 1.25em;
    margin-left: 10px;
    pointer-events: none;
    text-align: left;
    user-select: none;
    font-weight: 300;
}

@media (min-resolution: 2x) {
    .trading-vue-legend {
        font-weight: 400;
    }
}

.trading-vue-ohlcv {
    pointer-events: none;
    margin-bottom: 0.5em;
}

.t-vue-lspan {
    font-variant-numeric: tabular-nums;
    font-size: 0.95em;
    color: #999999;
    /* TODO: move => params */
    margin-left: 0.1em;
    margin-right: 0.2em;
}

.t-vue-title {
    font-size: 1.45em;
}

.t-vue-exchange{
    font-size: 0.9em;
    font-weight: 600;
    margin-right: 0.25em;
}

.t-vue-ind {
    display: flex;
    margin-left: 0.2em;
    margin-bottom: 0.5em;
    font-size: 1.0em;
    margin-top: 0.3em;
}

.t-vue-ivalue {
    margin-left: 0.5em;
}

.t-vue-unknown {
    color: #999999;
    /* TODO: move => params */
}

.tvjs-appear-enter-active,
.tvjs-appear-leave-active {
    transition: all .25s ease;
}

.tvjs-appear-enter,
.tvjs-appear-leave-to {
    opacity: 0;
}
</style>
