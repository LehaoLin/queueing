<template>
<p>Coming rate</p>
<input v-model="coming_rate">
<button @click="start_()">Start/Stop</button>
<div style="display: flex; flex: 2 2 0%; flex-flow: row wrap; justify-content: space-around;">
    <div>
        <b>Traditional BitTorrent</b>
        <div>[Nodes num]: {{bt_nodes_num}}</div>
        <div>[Total]: {{user_downloading_bt_num + user_finish_bt_num}}</div>
        <div>[Downloading]: {{user_downloading_bt_num}}</div>
        <div>[Finshed]: {{user_finish_bt_num}}</div>
        <div>[Seconds]: {{parseInt(current_frame * 0.1)}}s</div>
        <div>[Downloaded]: {{Math.round(bt_downloaded)}} MB</div>
        <div>[To be downloading]: {{Math.round(downloading_size_bt)}} MB</div>
        <div>[Bandwidth per user]: {{Math.round(current_bandwidth_bt_per_user)}} MB/s</div>
    </div>

    <div>
        <b>Pricing BT</b>
        <div>[Nodes num]: {{pricing_nodes_num}}</div>
        <div>[Total]: {{user_downloading_paid_num + user_downloading_unpaid_num + user_finish_paid_num + user_finish_unpaid_num}}</div>
        <div>[Downloading]: Paid:{{user_downloading_paid_num}} Unpaid:{{user_downloading_unpaid_num}}</div>
        <div>[Finshed]: Paid:{{user_finish_paid_num}} Unpaid:{{user_finish_unpaid_num}}</div>
        <div>Seconds: {{parseInt(current_frame * 0.1)}}s</div>
        <div>[Downloaded]: {{Math.round(paid_downloaded + unpaid_downloaded)}} MB</div>
        <div>[To be downloading]: Paid:{{Math.round(downloading_size_paid)}} MB  Unpaid:{{Math.round(downloading_size_unpaid)}} MB</div>
        <div>[Bandwidth per user]: Paid:{{Math.round(current_bandwidth_paid_per_user)}} MB/s  Unpaid:{{current_bandwidth_unpaid_per_user}} MB/s</div>
    </div>

    <div>
        <b>IPFS</b>
        <div>[IPFS nodes num]: {{this.ipfs_nodes_num}}</div>
        <div>[Total]: {{user_downloading_ipfs_num + user_finish_ipfs_num}}</div>
        <div>[Downloading]: {{user_downloading_ipfs_num}}</div>
        <div>[Finshed]: {{user_finish_ipfs_num}}</div>
        <div>[Seconds]: {{parseInt(current_frame * 0.1)}}s</div>
        <div>[Downloaded]: {{Math.round(ipfs_downloaded)}} MB</div>
        <div>[To be downloading]: {{Math.round(downloading_size_ipfs)}} MB</div>
        <div>[Bandwidth per user]: {{Math.round(current_bandwidth_ipfs_per_user)}} MB/s</div>
    </div>

    <div>
        <b>QueueBT</b>
        <div>[Nodes num]: {{this.queue_nodes_num}} [Restricted num]: {{this.restricted_num}}</div>
        <div>[Total]: {{user_downloading_queue_num + user_finish_queue_num}}</div>
        <div>[Downloading]: {{queue_user_serving_buffer.length}} [Queue]: {{queue_user_waiting_buffer.length}}</div>
        <div>[Finshed]: {{user_finish_queue_num}}</div>
        <div>[Seconds]: {{parseInt(current_frame * 0.1)}}s</div>
        <div>[Downloaded]: {{Math.round(queue_downloaded)}} MB</div>
        <div>[To be downloading]: {{Math.round(downloading_size_queue)}} MB</div>
        <div>[Bandwidth per user]: {{Math.round(current_bandwidth_queue_per_user)}} MB/s</div>
    </div>
</div>



    <!-- <button @click="start_()">okok</button> -->
<div id="main" style="width:100%;height:70vh;"></div>


</template>

<script>
import * as echarts from 'echarts'

function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

function sum(arr) {
    return arr.reduce(function(prev, curr, idx, arr){
        return prev + curr;
    });
}


export default {
    name: 'Queue',
  components: {
    // Model, List
  },
  data(){
    return {
        coming_rate: 1,
        user_num: 0,
        user_downloading_bt_num: 0,
        user_downloading_ipfs_num: 0,
        user_downloading_paid_num: 0,
        user_downloading_unpaid_num: 0,
        user_downloading_queue_num: 0,

        user_finish_bt_num: 0, 
        user_finish_ipfs_num: 0, 
        user_finish_paid_num: 0,
        user_finish_unpaid_num: 0,
        user_finish_queue_num: 0,

        next_user: {},
        
        user_list: [],
        bt_user_buffer: [],
        ipfs_user_buffer: [],
        paid_user_buffer: [],
        unpaid_user_buffer: [],
        queue_user_serving_buffer: [],
        queue_user_waiting_buffer: [],
        
        bt_nodes_num: 10,
        ipfs_nodes_num: 1000000,
        pricing_nodes_num: 10,
        queue_nodes_num: 10,

        restricted_num: 15,

        bandwidth: 100,
        current_bandwidth_bt_per_user: 0,
        current_bandwidth_paid_per_user: 0,
        current_bandwidth_unpaid_per_user: 0.5,
        current_bandwidth_ipfs_per_user: 0,
        current_bandwidth_queue_per_user: 0,

        current_frame: 0,

        data_buffer_bt: [],
        data_finished_list: [],
        data_buffer_ipfs: [],
        data_buffer_paid: [],
        data_buffer_unpaid: [],
        data_buffer_serving_queue: [],
        data_buffer_waiting_queue: [],


        start: 1,

        bt_downloaded: 0,
        ipfs_downloaded: 0,
        paid_downloaded: 0,
        unpaid_downloaded: 0,
        queue_downloaded: 0,

        // unit frame
        dht_interval: 0.01,
        per_section_table: 0.001,

    }
  },
  computed: {
      downloading_size_bt(){
          if (this.bt_user_buffer.length == 0){
              return 0
          }
          var temp = this.bt_user_buffer.map(x => x.size)
          return sum(temp)
      },
      downloading_size_ipfs(){
          if (this.ipfs_user_buffer.length == 0){
              return 0
          }
          var temp = this.ipfs_user_buffer.map(x => x.size)
          return sum(temp)
      },
      downloading_size_paid(){
          if (this.paid_user_buffer.length == 0){
              return 0
          }
          var temp = this.paid_user_buffer.map(x => x.size)
          return sum(temp)
      },
      downloading_size_unpaid(){
          if (this.unpaid_user_buffer.length == 0){
              return 0
          }
          var temp = this.unpaid_user_buffer.map(x => x.size)
          return sum(temp)
      },
      downloading_size_queue(){
          if (this.queue_user_serving_buffer.length == 0){
              return 0
          }
          var temp1 = this.queue_user_serving_buffer.map(x => x.size)
        //   var temp2 = this.queue_user_waiting_buffer.map(x => x.size)
        //   return sum([...temp1, ...temp2])
          return sum([...temp1])
      },
    
  },
  mounted(){
    // ;(async () => {
    //     // await this.user_list_genenrate()
    //     for (let i =0; i < 6000; i++){
    //         // setTimeout(this.tick, 1000)
    //         // if ((this.data_buffer[this.data_buffer.length - 1] == 0) && (this.current_frame != 0)){
    //         //     break
    //         // }
    //         if (this.start != 1){
    //             break
    //         }
    //         await this.tick()
    //         await sleep(100)
    //     }
    // })()
  },
  methods: {
    start_(){
        this.start = this.start * -1
            ;(async () => {
        // await this.user_list_genenrate()
        for (let i =0; i < 6000; i++){
            // setTimeout(this.tick, 1000)
            // if ((this.data_buffer[this.data_buffer.length - 1] == 0) && (this.current_frame != 0)){
            //     break
            // }
            if (this.start != 1){
                break
            }
            await this.tick()
            await sleep(100)
        }
    })()

    },
    file_location(){
        return Math.random() * 1000000
    },
    user_list_genenrate(){
        var exp_gen_result = []
        for (let i=0; i < this.user_num; i++){
            exp_gen_result.push(this.exp_gen(0.1) * 0.3)
        }
        exp_gen_result.sort(function(a, b){return a - b})
        var max = exp_gen_result[exp_gen_result.length-1]
        var min = exp_gen_result[0]
        var new_list = []
        for (let i of exp_gen_result){
            new_list.push(Math.round((i - min)/(max - min) * 600))
        }
        // console.log(new_list)

        var exp_gen_result = []
        for (let i=0; i < this.user_num; i++){
            exp_gen_result.push(this.exp_gen(0.1))
        }
        var max = Math.max(...exp_gen_result)
        var min = Math.min(...exp_gen_result)
        var new_list_ = []
        for (let i of exp_gen_result){
            new_list_.push((i-min)/(max - min) * 1024 * 10)
        }
        // console.log(new_list_)

        var uu = 0
        while (uu < new_list.length){
            var temp = {}
            temp.uid = uu
            temp.frame = new_list[uu]
            temp.size = new_list_[uu]
            this.user_list.push(temp)
            uu += 1
        }
        console.log(this.user_list)
    },
    
    user_generate(){
        var next_user_frame = Math.round(this.exp_gen(0.1) * 0.1) * (1 / this.coming_rate)

        var exp_gen_result = []
        for (let i=0; i < 10; i++){
            exp_gen_result.push(this.exp_gen(0.1))
        }
        var max = Math.max(...exp_gen_result)
        var min = Math.min(...exp_gen_result)
        var new_list_ = []
        for (let i of exp_gen_result){
            new_list_.push((i-min)/(max - min) * 1024 * 10)
        }
        var next_user_size = new_list_[(Math.random() * new_list_.length) | 0]
        
        this.next_user.frame = next_user_frame
        this.next_user.size = next_user_size
        console.log(next_user_frame, next_user_size)
        
    },
    exp_gen(lambda) {
        if ('number' !== typeof lambda) {
            throw new TypeError('nextExponential: lambda must be number.');
        }
        if (lambda <= 0) {
            throw new TypeError('nextExponential: ' +
                                'lambda must be greater than 0.');
        }
        return - Math.log(1 - Math.random()) / lambda;
    },
    bt_in(){
        if (this.next_user.size){
            if (this.next_user.frame < 0){
                console.log(111)
                // copyObj2 = JSON.parse(JSON.stringify(srcObj));
                this.bt_user_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                this.ipfs_user_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                this.user_downloading_bt_num += 1
                this.user_downloading_ipfs_num += 1
                var random = Math.random()
                if (random > 0.5){
                    this.paid_user_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                    this.user_downloading_paid_num += 1
                } else {
                    this.unpaid_user_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                    this.user_downloading_unpaid_num += 1
                }
                this.user_downloading_queue_num += 1
                this.queue_user_waiting_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                if (this.queue_user_serving_buffer.length < this.restricted_num){
                    this.queue_user_serving_buffer.push(JSON.parse(JSON.stringify(this.next_user)))
                    this.queue_user_waiting_buffer.pop()
                }

                this.user_generate()

                this.current_bandwidth_bt_per_user = this.bt_nodes_num / this.user_downloading_bt_num * this.bandwidth
                this.current_bandwidth_ipfs_per_user = this.ipfs_nodes_num / this.user_downloading_ipfs_num * this.bandwidth
                this.current_bandwidth_paid_per_user = (this.pricing_nodes_num * this.bandwidth - this.current_bandwidth_unpaid_per_user * this.user_downloading_unpaid_num) / this.user_downloading_paid_num
                this.current_bandwidth_queue_per_user = this.queue_nodes_num / this.queue_user_serving_buffer.length * this.bandwidth

                if (this.current_bandwidth_bt_per_user > this.bandwidth){
                    this.current_bandwidth_bt_per_user = this.bandwidth
                }   
                if (this.current_bandwidth_ipfs_per_user > this.bandwidth){
                    this.current_bandwidth_ipfs_per_user = this.bandwidth
                }   
                if (this.current_bandwidth_paid_per_user > this.bandwidth){
                    this.current_bandwidth_paid_per_user = this.bandwidth
                }
                if (this.current_bandwidth_queue_per_user > this.bandwidth){
                    this.current_bandwidth_queue_per_user = this.bandwidth
                }
            }
            this.next_user.frame -= 1
        }else {
            console.log(222)
            this.user_generate()
            console.log(this.next_user)
        }

        // this.user_generate()


        // this.user_downloading_bt_num += 1
        // this.current_bandwidth_bt_per_user = this.bt_nodes_num / this.user_downloading_bt_num * this.bandwidth
        // if (this.current_bandwidth_bt_per_user > this.bandwidth){
        //     this.current_bandwidth_bt_per_user = this.bandwidth
        // }
        // this.bt_user_buffer.push(user)
    },
    bt_out(){
        this.user_downloading_bt_num -= 1
        this.user_finish_bt_num += 1
        this.current_bandwidth_bt_per_user = this.bt_nodes_num / this.user_downloading_bt_num * this.bandwidth
        if (this.current_bandwidth_bt_per_user > this.bandwidth){
            this.current_bandwidth_bt_per_user = this.bandwidth
        }
    },
    bt_download(){
        var uu = 0
        while (uu < this.bt_user_buffer.length){
            var remain = this.bt_user_buffer[uu].size - this.current_bandwidth_bt_per_user * 0.1
            this.bt_downloaded += this.current_bandwidth_bt_per_user * 0.1
            // console.log('remain', remain)
            this.bt_user_buffer[uu].size = remain
            if (remain < 0){
                // console.log(this.bt_user_buffer)
                // var pre = this.bt_user_buffer.slice(0,uu)
                // var late = this.bt_user_buffer.slice(uu+1)
                // this.bt_user_buffer = [...pre, ...late]
                console.log(remain)
                this.bt_user_buffer[uu] = {size: 0}
                this.bt_out()
            }
            uu += 1
        }
        var temp_list = []
        for (let i of this.bt_user_buffer){
            if (i.size != 0){
                temp_list.push(i)
            }
        }
        this.bt_user_buffer = temp_list

    },
    ipfs_out(){
        this.user_downloading_ipfs_num -= 1
        this.user_finish_ipfs_num += 1
        this.current_bandwidth_ipfs_per_user = this.ipfs_nodes_num / this.user_downloading_bt_num * this.bandwidth
        if (this.current_bandwidth_bt_per_user > this.bandwidth){
            this.current_bandwidth_bt_per_user = this.bandwidth
        }
    },
    ipfs_download(){
        var uu = 0
        while (uu < this.ipfs_user_buffer.length){
            var remain = this.ipfs_user_buffer[uu].size - this.current_bandwidth_ipfs_per_user * 0.1
            this.ipfs_downloaded += this.current_bandwidth_ipfs_per_user * 0.1
            this.ipfs_user_buffer[uu].size = remain
            if (remain < 0){
                this.ipfs_user_buffer[uu] = {size: 0}
                this.ipfs_out()
            }
            uu += 1
        }
        var temp_list = []
        for (let i of this.ipfs_user_buffer){
            if (i.size != 0){
                temp_list.push(i)
            }
        }
        this.ipfs_user_buffer = temp_list
    },
    pricing_out(pricing_flag){
        if (pricing_flag == 1){
            this.user_downloading_paid_num -= 1
            this.user_finish_paid_num += 1
            this.current_bandwidth_paid_per_user = (this.pricing_nodes_num * this.bandwidth - this.current_bandwidth_unpaid_per_user * this.user_downloading_unpaid_num) / this.user_downloading_paid_num
            if (this.current_bandwidth_paid_per_user > this.bandwidth){
                this.current_bandwidth_paid_per_user = this.bandwidth
            }
        }
        if (pricing_flag == 0){
            this.user_downloading_unpaid_num -= 1
            this.user_finish_unpaid_num += 1
        }

    },
    paid_download(){
        var uu = 0
        while (uu < this.paid_user_buffer.length){
            var remain = this.paid_user_buffer[uu].size - this.current_bandwidth_paid_per_user * 0.1
            this.paid_downloaded += this.current_bandwidth_paid_per_user * 0.1
            this.paid_user_buffer[uu].size = remain
            if (remain < 0){
                this.paid_user_buffer[uu] = {size: 0}
                this.pricing_out(1)
            }
            uu += 1
        }
        var temp_list = []
        for (let i of this.paid_user_buffer){
            if (i.size != 0){
                temp_list.push(i)
            }
        }
        this.paid_user_buffer = temp_list
    },
    unpaid_download(){
        var uu = 0
        while (uu < this.unpaid_user_buffer.length){
            var remain = this.unpaid_user_buffer[uu].size - this.current_bandwidth_unpaid_per_user * 0.1
            this.unpaid_downloaded += this.current_bandwidth_unpaid_per_user * 0.1
            this.unpaid_user_buffer[uu].size = remain
            if (remain < 0){
                this.unpaid_user_buffer[uu] = {size: 0}
                this.pricing_out(0)
            }
            uu += 1
        }
        var temp_list = []
        for (let i of this.unpaid_user_buffer){
            if (i.size != 0){
                temp_list.push(i)
            }
        }
        this.unpaid_user_buffer = temp_list
    },
    queue_out(){
        this.user_downloading_queue_num -= 1
        this.user_finish_queue_num += 1
        this.current_bandwidth_queue_per_user = this.queue_nodes_num / this.queue_user_serving_buffer.length * this.bandwidth
        if (this.current_bandwidth_queue_per_user > this.bandwidth){
            this.current_bandwidth_queue_per_user = this.bandwidth
        }
    },
    queue_download(){
        var uu = 0
        while (uu < this.queue_user_serving_buffer.length){
            var remain = this.queue_user_serving_buffer[uu].size - this.current_bandwidth_queue_per_user * 0.1
            this.queue_downloaded += this.current_bandwidth_queue_per_user * 0.1
            this.queue_user_serving_buffer[uu].size = remain
            if (remain < 0){
                this.queue_user_serving_buffer[uu] = {size: 0}
                this.queue_out()
            }
            uu += 1
        }
        var temp_list = []
        for (let i of this.queue_user_serving_buffer){
            if (i.size != 0){
                temp_list.push(i)
            }
        }
        this.queue_user_serving_buffer = temp_list
        if ((this.queue_user_serving_buffer.length < this.restricted_num) & (this.queue_user_waiting_buffer.length > 0)){
            this.queue_user_serving_buffer.push(JSON.parse(JSON.stringify(this.queue_user_waiting_buffer.pop())))
        }
    },
    tick(){

        this.bt_in()
        // this.user_generate()
        console.log('frame',this.next_user.frame)
        
        // for (let i of this.user_list){
        //     if (i.frame == this.current_frame){
        //         this.bt_in(i)
        //     }
        // }
        // if (this.bt_user_buffer.length != 0){
            this.bt_download()
            this.ipfs_download()
            this.paid_download()
            this.unpaid_download()
            this.queue_download()
        // }
        // console.log('current',this.bt_user_buffer)
        // this.data_buffer.push(this.user_finish_bt_num)
        this.data_buffer_bt.push(this.bt_user_buffer.length)
        this.data_finished_list.push(this.user_finish_bt_num)

        this.data_buffer_ipfs.push(this.ipfs_user_buffer.length)
        this.data_buffer_paid.push(this.paid_user_buffer.length)
        this.data_buffer_unpaid.push(this.unpaid_user_buffer.length)
        this.data_buffer_serving_queue.push(this.queue_user_serving_buffer.length)
        this.data_buffer_waiting_queue.push(this.queue_user_waiting_buffer.length)

        this.draw()
        this.current_frame += 1
    },
    ipfs_out(){
        this.user_downloading_ipfs_num -= 1
        this.user_finish_ipfs_num += 1
        this.current_bandwidth_ipfs_per_user = this.ipfs_nodes_num / this.user_downloading_ipfs_num * this.bandwidth
        if (this.current_bandwidth_ipfs_per_user > this.bandwidth){
            this.current_bandwidth_ipfs_per_user = this.bandwidth
        }
    },
    draw(){
        var chartDom = document.getElementById('main');
        var myChart = echarts.init(chartDom);
        var option;
        let gridWidth = '35%';
        let gridHeight = '35%';
        let gridLeft = 80;
        let gridRight = 50;
        let gridTop = 50;
        let gridBottom = 80;

        option = {
            //   visualMap: [
            //         {
            //         show: false,
            //         type: 'continuous',
            //         seriesIndex: 0,
            //         min: 0,
            //         max: 400
            //         },
            //         {
            //         show: false,
            //         type: 'continuous',
            //         seriesIndex: 1,
            //         dimension: 0,
            //         min: 0,
            //         // max: .length - 1
            //         }
            //     ],
            title:  [
                    {
                    // left: 'center',
                    text: 'Jobs in the BT system'
                    },
                    {
                    // top: '45%',
                    left: 'right',
                    text: 'Jobs in the pricing BT system'
                    },
                    {
                    top: '45%',
                    // left: 'center',
                    // left: 'right',
                    text: 'Jobs in the IPFS system'
                    },
                    {
                    top: '45%',
                    // left: 'center',
                    left: 'right',
                    text: 'Jobs in the QueueBT system'
                    }
                ],
            tooltip: {
                trigger: 'axis',
            },
            xAxis: [
                {
                    type: 'category',
                    gridIndex: 0
                },
                {
                    type: 'category',
                     gridIndex: 1
                },
                {
                    type: 'category',
                     gridIndex: 2
                },
                {
                    type: 'category',
                    gridIndex: 3
                },
            ],
            yAxis: [{
                    type: 'value',
                    gridIndex: 0
                },
                {
                    type: 'value',
                    gridIndex: 1
                },
                {
                    type: 'value',
                    gridIndex: 2
                },
                                {
                    type: 'value',
                    gridIndex: 3
                }
            ],
              grid: [
           {
        id: 'xAxisLeft-yAxisTop',
        left: gridLeft,
        top: gridTop,
        width: gridWidth,
        height: gridHeight
      },
      {
        id: 'xAxisLeft-yAxisBottom',
        left: gridLeft,
        bottom: gridBottom,
        width: gridWidth,
        height: gridHeight
      },
      {
        id: 'xAxisRight-yAxisTop',
        right: gridRight,
        top: gridTop,
        width: gridWidth,
        height: gridHeight
      },
      {
        id: 'xAxisRight-yAxisBottom',
        right: gridRight,
        bottom: gridBottom,
        width: gridWidth,
        height: gridHeight
      }
                ],
            series: [
                {
                    name: 'buffer',
                    data: this.data_buffer_bt,
                    type: 'line',
                    xAxisIndex: 0,
                    yAxisIndex: 0
                    // smooth: true
                },
                {
                    name: 'buffer',
                    data: this.data_buffer_ipfs,
                    type: 'line',
                    xAxisIndex: 1,
                    yAxisIndex: 1
                    // smooth: true
                },
                {
                    name: 'Paid',
                    data: this.data_buffer_paid,
                    type: 'line',
                    xAxisIndex: 2,
                    yAxisIndex: 2
                    // smooth: true
                },
                {
                    name: 'Unpaid',
                    data: this.data_buffer_unpaid,
                    type: 'line',
                    xAxisIndex: 2,
                    yAxisIndex: 2,
                    color: 'red',
                    // smooth: true
                },
                {
                    name: 'Serving',
                    data: this.data_buffer_serving_queue,
                    type: 'line',
                    xAxisIndex: 3,
                    yAxisIndex: 3
                    // smooth: true
                },
                {
                    name: 'Waiting',
                    data: this.data_buffer_waiting_queue,
                    type: 'line',
                    xAxisIndex: 3,
                    yAxisIndex: 3
                    // smooth: true
                },
                // {
                //     name: 'finished',
                //     data: this.data_finished_list,
                //     type: 'line',
                //     // smooth: true
                // }
            ]
        };

        option && myChart.setOption(option);
    },


  },

  
}
</script>

<style scoped>

</style>