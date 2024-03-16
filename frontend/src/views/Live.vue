<template>
    <VContainer fluid align="center" color="surface">
        <VRow max-width="1200px">
            <!-- Row 1 -->

            <VCol cols="9">
                <!-- Row 1, Col 1 -->

                <figure class="highcharts-figure">
                    <div id="container"></div>
                </figure>

            </VCol>

            <VCol cols="3">
            <!-- Row 1, Col 2 -->
            
            <VCard class="mb-5" max-width="500" color="primaryContainer" subtitle="Temperature">
                    <VCardItem>
                        <span class="text-h3 text-onPrimaryContainer">{{ temperature }}</span>
                    </VCardItem>
                </VCard>

                <VCard class="mb-5" max-width="500" color="tertiaryContainer" subtitle="Heat Index (Feels like)">
                    <VCardItem>
                        <span class="text-h3 text-onTertiaryContainer">{{ heatindex }}</span>
                    </VCardItem>
                </VCard>

                <VCard class="mb-5" max-width="500" color="secondaryContainer" subtitle="Humidity">
                    <VCardItem>
                        <span class="text-h3 text-onSecondaryContainer">{{ humidity }}</span>
                    </VCardItem>
                </VCard>
            </VCol>
        </VRow>

        <VRow max-width="1200px" justify="start">
            <!-- Row 2 -->

            <VCol cols="9">
                <!-- Row 2, Col 1 -->
                <figure class="highcharts-figure">
                    <div id="container1"></div>
                </figure>
            </VCol>

        </VRow>

    </VContainer>
</template>

<script setup>
/** JAVASCRIPT HERE */

// IMPORTS
import { ref,reactive,watch ,onMounted,onBeforeUnmount,computed } from "vue";  
import { useRoute ,useRouter } from "vue-router";

import { useMqttStore } from "@/store/mqttStore";
import { storeToRefs } from "pinia";

import { useAppStore } from "@/store/appStore";

// Highcharts, Load the exporting module and Initialize exporting module. 
import Highcharts from 'highcharts'; 
import more from 'highcharts/highcharts-more'; 
import Exporting from 'highcharts/modules/exporting'; 
Exporting(Highcharts); 
more(Highcharts);
 
 
// VARIABLES
const router      = useRouter();
const route       = useRoute();  

const Mqtt        = useMqttStore();
const { payload, payloadTopic } = storeToRefs(Mqtt);

const AppStore = useAppStore();

const tempHiChart = ref(null); // Chart object
const humidHiChart = ref(null);

const points = ref(10); // Specify the quantity of points to be shown on the live graph simultaneously. 
const shift = ref(false); // Delete a point from the left side and append a new point to the right side of the graph.

// COMPUTED PROPERTIES 
const temperature = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.temperature.toFixed(2)} °C`; 
    } 
});

const humidity = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.humidity.toFixed(2)}%`; 
    } 
});

const heatindex = computed(()=>{ 
    if(!!payload.value){ 
        return `${payload.value.heatindex.toFixed(2)} °C`; 
    } 
});




// FUNCTIONS
onMounted(()=>{
    // THIS FUNCTION IS CALLED AFTER THIS COMPONENT HAS BEEN MOUNTED
    CreateCharts(); 
    
    Mqtt.connect(); // Connect to Broker located on the backend 
    
    setTimeout( ()=>{ 
        // Subscribe to each topic 
        Mqtt.subscribe("620156144");
        Mqtt.subscribe("620156144_pub"); 
    },3000);
});


onBeforeUnmount(()=>{
    // THIS FUNCTION IS CALLED RIGHT BEFORE THIS COMPONENT IS UNMOUNTED
    Mqtt.unsubcribeAll();
});

const CreateCharts = async () => { 
    // TEMPERATURE CHART 
    
    tempHiChart.value = Highcharts.chart('container', { 
        chart: { zoomType: 'x' }, 
        title: { text: 'Air Temperature and Heat Index Analysis', align: 'left' }, 
        
        yAxis: { 
            title: { text: 'Air Temperature & Heat Index' , style:{color:'#000000'}}, 
            labels: { format: '{value} °C' } 
        }, 
        
        xAxis: { 
            type: 'datetime', 
            title: { text: 'Time', style:{color:'#000000'} }, 
        }, 
        
        tooltip: { shared:true, }, 
        
        series: [ 
            { 
                name: 'Temperature', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[0] 
            }, 
            
            { 
                name: 'Heat Index', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[1] 
            } 
        ], 
    }); 

    // HUMIDITY CHART 
    
    humidHiChart.value = Highcharts.chart('container1', { 
        chart: { zoomType: 'x' }, 
        title: { text: 'Humidity Analysis (Live)', align: 'left' }, 
        
        yAxis: { 
            title: { text: 'Relative Humidity' , style:{color:'#000000'}}, 
            labels: { format: '{value} %' } 
        }, 
        
        xAxis: { 
            type: 'datetime', 
            title: { text: 'Time', style:{color:'#000000'} }, 
        }, 
        
        tooltip: { shared:true, }, 
        
        series: [ 
            { 
                name: 'Humidity', 
                type: 'spline', 
                data: [], 
                turboThreshold: 0, 
                color: Highcharts.getOptions().colors[0] 
            }, 
            
            // { 
            //     name: 'Heat Index', 
            //     type: 'spline', 
            //     data: [], 
            //     turboThreshold: 0, 
            //     color: Highcharts.getOptions().colors[1] 
            // } 
        ], 
    }); 
};

// WATCHERS 
watch(payload,(data)=> { 
    if(points.value > 0){ points.value -- ; } 
    else{ shift.value = true; } 
    
    tempHiChart.value.series[0].addPoint({y:parseFloat(data.temperature.toFixed(2)) ,x: data.timestamp * 1000 }, true, shift.value); 
    tempHiChart.value.series[1].addPoint({y:parseFloat(data.heatindex.toFixed(2)) ,x: data.timestamp * 1000 }, true, shift.value); 
    
    humidHiChart.value.series[0].addPoint({y:parseFloat(data.humidity.toFixed(2)), x: data.timestamp * 1000}, true, shift.value);
})

const updateLineCharts = async () => { 

    if (!!start.value && !!end.value) { 
        
        // Convert output from Textfield components to 10 digit timestamps 
        let startDate = new Date(start.value).getTime() / 1000; 
        let endDate = new Date(end.value).getTime() / 1000; 
        
        // Fetch data from backend 
        const data = await AppStore.getAllInRange(startDate,endDate); 
        
        // Create arrays for each plot 
        let temperature = []; 
        let heatindex = []; 
        let humidity = [];
        
        // Iterate through data variable and transform object to format recognized by highcharts 
        
        data.forEach(row => { 
            temperature.push({"x": row.timestamp * 1000, "y": parseFloat(row.temperature.toFixed(2)) }); 
            heatindex.push({ "x": row.timestamp * 1000,"y": parseFloat(row.heatindex.toFixed(2)) }); 
            humidity.push({ "x": row.timestamp * 1000,"y": parseFloat(row.humidity.toFixed(2)) }); 
        }); 
        
        // Add data to Temperature and Heat Index chart 
        tempChart.value.series[0].setData(temperature); 
        tempChart.value.series[1].setData(heatindex); 

        humidChart.value.series[0].setData(humidity);
    } 
}

</script>


<style scoped>
/** CSS STYLE HERE */

Figure { 
border: 2px solid black;

}

</style>
