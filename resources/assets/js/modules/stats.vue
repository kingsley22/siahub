<template>
<ul class="nav navbar-nav navbar-right">
    <li><a href="#" v-tooltip:bottom="'Total Sia network storage'">Storage: {{totalStorage()}}</a></li>
    <li><a href="#" v-tooltip:bottom="'Storage Utilization'"><span v-html="utilization()"></span></a></li>
    <li><a href="#" v-tooltip:bottom="'$/TB/month Avg/Min/Max'">Price: {{averagePrice()}}/{{minPrice()}}/{{maxPrice()}} $</a></li>
    <li><a href="#" v-tooltip:bottom="'CoinMarketCap Price'">{{ticker.price_btc}} BTC / {{ticker.price_usd}}$</a></li>
</ul>
</template>

<script>
import { updateHosts } from '../store/actions';
import { mapGetters, mapActions } from 'vuex'

export default {
    mounted() {
        this.refresh()
    },
    computed: mapGetters(['hosts']),
    methods: {
        updateHosts(hosts) {
          this.$store.dispatch('updateHosts', hosts);
        },
        refresh(){
            if(this.loading) return false;

            this.error = false;
            this.loading = true;
            axios.get('/api/sia_ticker')
                .then((response) => {
                    this.ticker = response.data;
                    this.loading = false;
                    this.loadHosts();
                })
                .catch((error) => {
                    console.log(error);
                    this.error = error.response.data;
                    this.loading = false;
                });
        },
        loadHosts(){
            if(this.hosts.length > 0) return;

            this.error = false;
            this.loading = true;
            axios.get('/api/hosts/active')
                .then((response) => {
                    //this.gridData = response.data;
                    this.updateHosts(response.data);
                    this.loading = false;
                })
                .catch((error) => {
                    console.log(error);
                    this.error = error.response.data;
                    this.loading = false;
                });
        },
        totalStorage: function(raw){
            if(!this.hosts) return 'loading';
            var result = this.hosts.reduce(function(a, b){
                        return a + parseInt(b.totalstorage);
                    }, 0);

            return (raw) ? result:humanFileSize(result, true);
        },
        totalRemainingStorage: function(raw){
            if(!this.hosts) return 'loading';
            var result = this.hosts.reduce(function(a, b){
                        return a + parseInt(b.remainingstorage);
                    }, 0);

            return (raw) ? result:humanFileSize(result, true);
        },
        averagePrice: function(raw){
            if(!this.hosts) return 'loading';
            var result = this.hosts.reduce(function(a, b){
                        return a + Math.round(b.storageprice/1e12*4320);
                    }, 0);

            return (result/_.filter(this.hosts, (o) => o.storageprice > 0).length*this.ticker.price_usd).toFixed(2);
        },
        minPrice: function(raw){
            if(!this.hosts) return 'loading';
            var result = Math.min.apply(null, _.filter(this.hosts, (o) => o.storageprice > 0).map(function(item) {
                return item.storageprice;
            }));

            return (result/1e12*4320*this.ticker.price_usd).toFixed(2);
        },
        maxPrice: function(raw){
            if(!this.hosts) return 'loading';
            var result = Math.max.apply(null, _.filter(this.hosts, (o) => o.storageprice > 0).map(function(item) {
                return item.storageprice;
            }));

            return (result/1e12*4320*this.ticker.price_usd).toFixed(2);
        },
        utilization: function(){
            if(!this.hosts) return 'loading';

            var utilization = this.totalStorage(true)-this.totalRemainingStorage(true);
            var percent = Math.round(utilization/this.totalStorage(true)*100);

            var color_num = Math.round(percent/100*255);
            var color = "rgba("+color_num+", "+ Math.max(0, 150 - color_num) +", 0, 0.7)";

            $('[data-toggle="tooltip"]').tooltip();

            return '<div class="progress" style="width:150px;">\
            <div class="progress-bar" style="background-color: '+color+';width:'+(percent-15)+'%;"></div>\
            <div class="progress-bar" style="background-color: '+color+';width:15%;">'+percent+'%</div>\
            <div class="progress-bar" style="color: #000;background-color: transparent;width:'+(100-15-percent)+'%">'+humanFileSize(utilization, true)+'</div>\
            </div>';
        }
    },
    data() {
        return {
            loading: false,
            error: false,
            ticker: {price_btc: 0, price_usd: 0}
        };
    }
}
</script>