<style scoped>
    .beat-data {
        display: flex;
        flex-direction: column;
        margin: 10px;
        padding: 10px;
        border: 1px solid #ccc;
    }
</style>

<template>
    <div>
        <input type="file" @change="onFileSelected" />
        <div
            class="beat-data"
            v-for="(beat, k) in nestedData"
            :key="k" 
        >
            <div class="beat-details">{{ beatDetails(beat)}}</div>
            <pre class="beat-details">{{ shrubList(beat)}}</pre>
            <pre class="beat-details">{{ herbList(beat)}}</pre>
        </div>
    </div>
</template>

<script>
import * as XLSX from 'xlsx';

export default {
    name: 'MyComponent',
    data() {
        return {
            xlsxData: {
                0: [],
                1: [],
                2: [],
                3: [],
            },
            headers: [
                "data-subplot_group-subplot_data-subplot_no",
                "data-subplot_group-subplot_data-aspect",
                "data-subplot_group-subplot_data-slope",
                "data-subplot_group-subplot_data-canopy_cover",
                "data-subplot_group-subplot_data-fodder_collection",
                "data-subplot_group-subplot_data-fuelwood_collection",
                "data-subplot_group-subplot_data-leaflitter_collection",
                "data-subplot_group-subplot_data-timber_extraction",
                "data-subplot_group-shrubs",
                "data-subplot_group-herbs",
                "PARENT_KEY",
                "KEY"
            ],
            mergedData:[],
            nestedData: null,
        }
    },
    created(){
        console.clear()
    },
    watch:{
        xlsxData: {
            handler: function (val, oldVal) {
                if(Object.keys(val).length === 4){
                    this.mergeData()
                }
            },
            deep: true
        }
    },
    methods: {
        importXLSX(file) {
            for(let i = 0; i < 4; i++) {
                const reader = new FileReader()
                reader.onload = (event) => {
                    const data = event.target.result
                    const workbook = XLSX.read(data, { type: 'binary' })
                    const worksheet = workbook.Sheets[workbook.SheetNames[i]]
                    let xlsxData
                    if(i == 1){
                        xlsxData = XLSX.utils.sheet_to_json(worksheet, { header: this.headers })
                    } else {
                        xlsxData = XLSX.utils.sheet_to_json(worksheet, { header: i })
                    }
                    this.xlsxData[i] = xlsxData
                }
                reader.readAsBinaryString(file)                
            }
        },
        onFileSelected(event) {
            const file = event.target.files[0]
            if (file) {
                this.importXLSX(file)
            }
        },
        mergeData(){
            const nestedData = this.xlsxData[0].map((row1) => {
                const rows2 = this.xlsxData[1].filter((row2) => row2.PARENT_KEY === row1.KEY);
                const nestedRows2 = rows2.map((row2) => {
                    const rows3 = this.xlsxData[2].filter((row3) => row3.PARENT_KEY === row2.KEY);
                    const rows4 = this.xlsxData[3].filter((row4) => row4.PARENT_KEY === row2.KEY);
                    return { ...row2, shrubs: rows3, herbs: rows4 };
                });
                return { ...row1, subplots: nestedRows2 };
            });

            // Store the nested data in a Vue data property
            this.nestedData = nestedData;
        },
        beatDetails(beat){
            // return {}
            let { subplots, ...op } = beat
            return {
                beat: op["data-site_details-beat"],
                block: op["data-site_details-block"],
                compartment: op["data-site_details-compartment"],
            }
        },
        shrubList(beat){
            let { subplots, ...op } = beat
            let {KEY, PARENT_KEY, ...shrubs} = subplots.map((s) => s.shrubs)

            return shrubs
        },
        herbList(beat){
            let { subplots, ...op } = beat
            let {KEY, PARENT_KEY, ...herbs} = subplots.map((s) => s.herbs)

            return herbs
        },
    },
}
</script>
