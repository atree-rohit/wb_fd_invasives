<style scoped>

    .beat-data {
        display: flex;
        flex-direction: column;
        margin: 10px;
        padding: 10px;
        border: .2rem solid #080;
        border-radius: var(--border-radius);
    }
    .beat-details {
        border: 0.2rem solid #880;
        border-radius: var(--border-radius);
    }

    .lists-area {
        display: flex;
        justify-content: space-around;
        border: 0.2rem solid #800;
        border-radius: var(--border-radius);
    }

    .lists-area table{
        margin: 10px;
        width: 100%;
        border-collapse: collapse;
        border-radius: var(--border-radius);
        border: 0.2rem solid #008;
    }
    .lists-area table thead th{
        background-color: #008;
        color: #fff;
    }

    .lists-area table tbody tr td{
        border: 1px solid red;
    }
</style>

<template>
    <div>
        <input type="file" @change="onFileSelected"/>
        {{ groupedData.length }}
        <div class="lists-area">
            <table class="list-shrub" id="shrubs_table">
                <thead>
                    <tr>
                        <th>No</th>
                        <th>Local Name</th>
                        <th>Scientific Name</th>
                        <th>Individuals</th>
                    </tr>
                    <tr>
                        <th colspan="4">
                            <button @click="copyTable('shrubs_table')">Copy</button>
                        </th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(shrub, k) in all_shrubs" :key="k">
                        <td>{{ k + 1 }}</td>
                        <td>{{ shrub.name }}</td>
                        <td>{{ shrub.scientific_name }}</td>
                        <td>{{ shrub.individuals }}</td>
                    </tr>
                </tbody>
            </table>
            <table class="list-shrub" id="herbs_table">
                <thead>
                    <tr>
                        <th>No</th>
                        <th>Local Name</th>
                        <th>Scientific Name</th>
                        <th>Individuals</th>
                    </tr>
                    <tr>
                        <th colspan="4">
                            <button @click="copyTable('herbs_table')">Copy</button>
                        </th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(herb, k) in all_herbs" :key="k">
                        <td>{{ k + 1 }}</td>
                        <td>{{ herb.name }}</td>
                        <td>{{ herb.scientific_name }}</td>
                        <td>{{ herb.cover_percent }}</td>
                    </tr>
                </tbody>

            </table>
        </div>
        <div
            class="beat-data"
            v-for="(block, k) in groupedData"
            :key="k" 
            @click="selectedBlock = (selectedBlock == block[0]) ? '' : block[0]"
            v-if="false"
        >   
            <div class="beat-details">
                <div
                    v-for="(value, key) in blockDetails(block)"
                    :key="key"
                >
                    {{ key }}  - {{ value }}

                </div>
            </div>
            <div class="lists-area" v-if="selectedBlock == block[0]">
                <table class="list-shrub">
                    <thead>
                        <tr>
                            <th>No</th>
                            <th>Local Name</th>
                            <th>Individuals</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(shrub, k) in shrubList(block)" :key="k">
                            <td>{{ k + 1 }}</td>
                            <td>{{ shrub.name }}</td>
                            <td>{{ shrub.individuals }}</td>
                        </tr>
                    </tbody>

                </table>
                <table class="list-herb">
                    <thead>
                        <tr>
                            <th>No</th>
                            <th>Local Name</th>
                            <th>Cover Percent</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(herb, k) in herbList(block)" :key="k">
                            <td>{{ k + 1 }}</td>
                            <td>{{ herb.name }}</td>
                            <td>{{ herb.cover }}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</template>

<script>
import * as XLSX from 'xlsx'
import * as d3 from 'd3'

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
            nestedData: null,
            groupedData: [],
            selectedBlock: "",
        }
    },
    mounted(){
        // console.clear()
        console.log(this.all_shrubs)
    },
    watch:{
        xlsxData: {
            handler: function (val, oldVal) {
                if(Object.keys(val).length === 4){
                    this.mergeData()
                    this.groupedData = d3.groups(this.nestedData, d => d["data-site_details-block"])
                }
            },
            deep: true
        }
    },
    computed:{
        all_shrubs(){
            let op = []
            const all_data = d3.groups(this.xlsxData[2], d => d["data-subplot_group-shrubs-shrub_local_name"])
            all_data.forEach((row) => {
                op.push({
                    name: row[0],
                    scientific_name: this.unique_array(row[1].map((r) => r["data-subplot_group-shrubs-shrub_scientific_name"])).join(", "),
                    individuals: row[1].map((r) => r["data-subplot_group-shrubs-shrub_individuals"]).reduce((a, b) => a + b, 0),
                })
            })
            op.sort((a, b) => b.individuals - a.individuals)
            return op
        },
        all_herbs(){
            let op = []
            const all_data = d3.groups(this.xlsxData[3], d=> d["data-subplot_group-herbs-herb_local_name"])
            all_data.forEach((row) => {
                op.push({
                    name: row[0],
                    scientific_name: this.unique_array(row[1].map((r) => r["data-subplot_group-herbs-herb_scientific_name"])).join(", "),
                    cover_percent: this.getAverage(row[1].map((r) => r["data-subplot_group-herbs-herb_percentage_cover"])),
                })

            })
            op.sort((a, b) => b.cover_percent - a.cover_percent)
            return op
        }
    },
    methods: {
        copyTable(table_id) {
            // Get the table element
            const table = document.getElementById(table_id);
            
            // Create a range object and select the table contents
            const range = document.createRange();
            range.selectNode(table);
            
            // Add the range to the clipboard
            const selection = window.getSelection();
            selection.removeAllRanges();
            selection.addRange(range);
            document.execCommand("copy");
            
            // Deselect the table contents
            selection.removeAllRanges();
        },
        getAverage(arr){
            let total = 0
            let count = 0
            let average = 0
            arr.forEach((a) => {
                if(typeof(a) == "number"){
                    total += a
                    count++
                    average = total/count
                }
            })
            return average
        },
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
        unique_array(arr){
            return [...new Set(arr)].filter((el) => [undefined, "", " "].indexOf(el) < 0)
        },
        table_sort_by_name(table_data){
            return table_data.sort((a, b) => {
                if(a.name < b.name) { return -1; }
                if(a.name > b.name) { return 1; }
                return 0;
            })
        },
        blockDetails(block){
            console.log(block)
            return {
                beat: this.unique_array(block[1].map((b) => b["data-site_details-beat"])),
                block: block[0],
                compartment: this.unique_array(block[1].map((b) => b["data-site_details-compartment"])),
                subplots: block[1].map((b) => b["subplots"]).flat().length,
                observers: this.unique_array(block[1].map((b) => b["data-site_details-observer_name"])),
                points: this.unique_array(block[1].map((b) => b["data-site_details-location"])).map((l) => {
                    let coords = l.split(",")
                    return {
                        lat: coords[0],
                        lon: coords[1]
                    }
                }),
                shrubs: this.shrubList(block).length,
                herbs: this.herbList(block).length,
            }
        },
        shrubList(block){
            let fields = [ "data-subplot_group-shrubs-shrub_local_name", "data-subplot_group-shrubs-shrub_scientific_name", "data-subplot_group-shrubs-shrub_individuals"]
            let table_data = []
            const shrubs = block[1].map((b) => b.subplots.map((s) => s.shrubs.map((obj) => {
                const {KEY, PARENT_KEY, ...rest} = obj
                return rest
            })).flat()).flat()
            let grouped_data = d3.groups(shrubs, d => d[fields[0]])
            grouped_data.forEach((d) => {
                table_data.push({
                    name: d[0],
                    individuals: d[1].reduce((a, b) => a + b[fields[2]], 0)
                })
            })

            return this.table_sort_by_name(table_data)
        },
        herbList(block){
            let fields = [ "data-subplot_group-herbs-herb_local_name", "data-subplot_group-herbs-herb_scientific_name", "data-subplot_group-herbs-herb_percentage_cover" ]
            let table_data = []
            const herbs = block[1].map((b) => b.subplots.map((s) => s.herbs.map((obj) => {
                const {KEY, PARENT_KEY, ...rest} = obj
                return rest
            })).flat()).flat()
            let grouped_data = d3.groups(herbs, d => d[fields[0]])
            grouped_data.forEach((d) => {
                table_data.push({
                    name: d[0],
                    cover: d[1].reduce((a, b) => a + b[fields[2]], 0) / d[1].length
                })
            })

            return this.table_sort_by_name(table_data)
        },
    },
}
</script>
