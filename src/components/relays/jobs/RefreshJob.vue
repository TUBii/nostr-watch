<template>
  <div class="hidden lg:inline-block">
    <div 
      v-if="store.jobs.getActiveSlug?.includes('relays/check/') && !isSingle"
      class="text-white/30">
      checking {{ store.jobs.getActiveSlug.replace('relays/check/', '') }}
    </div>
    <div
        v-if="(!store.jobs.isActive || store.jobs.getActiveSlug === this.slug) && !this.isSingle"
        class="text-inherit">
      <span class="text-inherit">
        <span v-if="!store.jobs.isJobActive(this.slug)" class="hidden lg:inline mr-2">Checked {{ sinceLast }} ago</span>
        <span v-if="store.jobs.isJobActive(this.slug)" class="italic text-inherit ml-2 inline-block">
          <svg class="-mt-1.5 animate-spin mr-1 h-4 w-5 text-white inline-block" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
          {{ this.store.jobs.getProcessed(this.slug).length }}/{{ this.relays.length }} Relays Checked
        </span>
      </span>
      <span class="text-inherit hidden lg:inline mr-1" v-if="!store.jobs.isJobActive(this.slug)">-</span>
      <span class="text-inherit mr-2" v-if="store.prefs.refresh && !store.jobs.isJobActive(this.slug)"> 
        next check in: {{ untilNext }}
      </span>
      <button 
        v-if="!store.jobs.isJobActive(this.slug)"
        class=" text-xs -mt-1.5 my-1 py-1 px-3 rounded border-b-3 border-slate-700 bg-slate-500  font-bold text-white hover:border-slate-500 hover:bg-slate-400" 
        :disabled='store.jobs.isJobActive(this.slug)' 
        @click="checkNow()">  
          check{{ relay ? ` ${relay}` : "" }} Now
      </button>
    </div>
    <span
      v-if="store.jobs.getActiveSlug === this.slug && this.isSingle"
        class="text-inherit ml-2">
        <svg class="animate-spin mr-2 h-4 w-5 text-white inline" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
        checking {{ relayFromUrl }}
    </span>
  </div>
</template>

<style scoped>

</style>

<script>
import { defineComponent, toRefs } from 'vue'

import { setupStore } from '@/store'
import RelaysLib from '@/shared/relays-lib.js'
import SharedComputed from '@/shared/computed.js'

import { Inspector } from 'nostr-relay-inspector'

// import { relays } from '../../../../relays.yaml'

const localMethods = {
  async CheckRelaysJob(force, single){
    // console.log('invalidate?', !((!this.isExpired(this.slug, this.getRefreshInterval) && !force) && !this.isSingle), this.windowActive)
    if( (!this.isExpired(this.slug, this.getRefreshInterval) && !force) && !this.isSingle ) 
      return

    if(!this.windowActive)
      return

    // console.log('queue job', single, this.slug)
    
    this.queueJob(
      this.slug, 
      async () => await this.checkJob(single), 
      true
    )
  },

  pruneResult: function(result){
    let resultPruned

    if(result) {
      resultPruned = {
        url: result.url,
        aggregate: result.aggregate,
        check: {
          connect: result.check.connect,
          read: result.check.read,
          write: result.check.write,
          latency: result.check.latency,
          averageLatency: result.check.averageLatency
        },
      } 

      if(Object.keys(result.info).length) //should be null, but is an empty object. Need to fix in nostr-relay-inspector
        resultPruned.info = result.info

      if(result.latency) 
        resultPruned.latency = result.latency

      if(result?.pubkeyValid)
        resultPruned.pubkeyValid = result.pubkeyValid

      if(result?.pubkeyError)
        resultPruned.pubkeyError = result.pubkeyError

      const uptime = this.getUptimePercentage(result.url)
      if(uptime)
        resultPruned.uptime = uptime
    }

    return resultPruned
  },

  checkJob: async function(single){
    if(single) {
      this.checkSingle(single, this.slug)
    } 
    else {
      this.relays = this.store.relays.getAll
      this.store.jobs.applyUncommitted(this.slug)
      const relays = this.relays.filter( relay => !this.store.jobs.isProcessed(this.slug, relay) )
      let relayChunks = this.chunk(100, relays)
      for(let c=0;c<relayChunks.length;c++){
        let promises = [],
            resultsChunk = {}
        const chunk = relayChunks[c]
        for(let index = 0; index < chunk.length; index++) {
          await new Promise( resolve => setTimeout(resolve, 100))
          const promise = new Promise( resolve => {
            const relay = chunk[index] 
            this.check(relay)
              .then((result) => {
                this.store.jobs.addProcessed(this.slug, result.url)
                this.store.jobs.addUncommitted(this.slug, result.url)
                resultsChunk[result.url] = this.pruneResult(result)
                resolve()
              })
              .catch( (err) => { 
                console.error(err)
                resolve()
              })
          })
          promises.push(promise)
          this.store.jobs.commitUncommitted(this.slug)
        }
        await Promise.all(promises)
        this.store.results.mergeDeep(resultsChunk)
      }
    } 
    this.completeAll(single)
  },

  checkSingle: async function(relay, slug){
    await this.check(relay)
        .then((result) =>{
          result.aggregate = this.getAggregate(result)
          this.store.results.mergeDeep({ 
            [result.url]: this.pruneResult(result)
          })
          this.completeRelay(result)
        })
        .catch( (err) => console.error(`there was an error: ${err}`) )
      this.store.jobs.completeJob(slug)
  },

  completeRelay: function(result){
    this.store.jobs.addProcessed(this.slug, result.url)
    return this.pruneResult(result)
  },

  completeAll: function(){
    this.store.jobs.completeJob(this.slug)
  },

  check: async function(relay){
    // console.log('checking', relay)
    if(this.stop)
      return
    return new Promise( (resolve) => {
      const opts = {
          checkRead: true, 
          checkWrite: true,   
          checkLatency: true,
          getInfo: this.store.prefs.checkNip11 || this.isSingle,
          getIdentities: false,
          run: true,
          connectTimeout: this.inspectTimeout,
          readTimeout: this.inspectTimeout,
          writeTimeout: this.inspectTimeout,
        }
      
      // if(this.isSingle)
      opts.checkAverageLatency = true
      
      if(this.store.user.testEvent)
        opts.testEvent = this.store.user.testEvent

      const $inspector = new Inspector(relay, opts)

      $inspector
        .on('open', () => {})
        .on('complete', (instance) => {
          // console.log('aggr', instance.result.url, this.getAggregate(instance.result), instance.result.check.connect, instance.result.check.read, instance.result.check.write)
          instance.result.aggregate = this.getAggregate(instance.result)
          instance.result.log = instance.log
          this.closeRelay(instance.relay)
          resolve(instance.result)
        })
        .on('error', () => {
          resolve()
        })
      
      this.inspectors.push($inspector)
    })
  },

  setAverageLatency: function(){
    const latencies = new Array()
    this.relays.forEach( relay => {
      latencies.push(this.store.results.get(relay)?.latency?.final)
    })
    this.averageLatency =  this.average(latencies)
  },

  average(arr){
    let sum = 0,
        total = arr.length;
    for (let i = 0;i<total;i++) 
      sum += arr[i];
    return Math.floor(parseFloat(sum/total));
  },

  setRefreshInterval: function(){
    clearInterval(this.interval)
    this.interval = setInterval(() => {
      if( this.store.jobs.isIdle )
        this.lazyChecks()

      if( (!this.store.prefs.refresh || !this.store.prefs.clientSideProcessing) && !this.isSingle )
        return
      
      this.untilNext = this.timeUntilRefresh()
      this.sinceLast = this.timeSinceRefresh()

      if(!this.store.jobs.isJobActive(this.slug) && !this.isSingle)
        this.CheckRelaysJob()
        
    }, 1000)
  },
  setLatencyInterval: function(){
    this.setLatencyInterval = setInterval( () => {

    })
  },

  checkNow(){
    this.CheckRelaysJob(true)
  },

  async lazyChecks(){
    if(this.lazyLast && (Date.now()-this.lazyLast)<this.lazyInterval )
      return

    // relays with uptime in the last ~10hr
    const relays = Object.keys(this.store.results.all).filter( async (relay) => {
      const result = this.store.results.get(relay)
      return 'offline' === result?.aggregate && result?.uptime > 0
    })
    
    relays.forEach( async (relay) => {
      const slug = `relays/check/${relay}`,
            expired = (Date.now()-this.store.jobs.getLastUpdate(slug))>this.lazyInterval 
      if(!expired)
        return
      const result = this.store.results.get(relay)
      if('offline' === result?.aggregate && result?.uptime > 90)
        this.queueJob(
          slug, 
          async () => await this.checkSingle(result.url, slug), 
          true
        )
    })

    this.lazyLast = Date.now()
  },

  handleVisibility(){
    if(document.visibilityState === 'hidden')
      this.windowActive = false 
    else 
      this.windowActive = true

    if(this.windowActive) 
      this.store.layout.setActiveTab(this.$tabId)
  },


  timeUntilRefresh(){
    return this.timeSince(Date.now()-(this.store.jobs.getLastUpdate(this.slug)+this.store.prefs.duration-Date.now())) 
  },
  timeSinceRefresh(){
    return this.timeSince(this.store.jobs.getLastUpdate(this.slug)) || Date.now()
  },
}

export default defineComponent({

  name: 'CheckRelaysJob',

  components: {},

  setup(props){
    const {resultsProp: results} = toRefs(props)
    return { 
      store : setupStore(),
      results: results
    }
  },

  data() {
    return {
      relay: "",
      relays: this.store.relays.getAll,
      refresh: {},
      untilNext: null,
      lastUpdate: null,
      sinceLast: null,
      interval: null,
      windowActive: true,
      averageLatency: 200,
      pageOpen: 0,
      slug: 'relays/check',
      latencies: [],
      inspectors: [],
      stop: false,
      inspectTimeout: 15*1000,
      retry: [],
      retries: 1,
      lazyLast: null,
      lazyInterval: 5*60*1000
      // history: null
    }
  },

  created(){
    clearInterval(this.interval)
    document.addEventListener('visibilitychange', this.handleVisibility, false)
  },

  unmounted(){
    clearInterval(this.interval)
    this.inspectors.forEach( $inspector => $inspector.close())
    this.stop = true
  },

  beforeMount(){
    this.lastUpdate = this.store.jobs.getLastUpdate(this.slug)
    this.untilNext = this.timeUntilRefresh()
    this.sinceLast = this.timeSinceRefresh()

    // for(let ri=0;ri-this.relays.length;ri++){
    //   const relay = this.relays[ri],
    //         cache = this.getCache(relay)
    //   this.store.results.get(relay) = cache
    // }
  },

  mounted(){
    if( this.isSingle ){
      this.slug = `relays/check/${this.relayFromUrl}`
      this.CheckRelaysJob(true, this.relayFromUrl)
      // this.runLatencyCheck()
    } else {
      this.CheckRelaysJob()
    }
    if(this.store.prefs.clientSideProcessing && !this.isSingle)
      this.setRefreshInterval()
  },

  updated(){},

  computed: Object.assign(SharedComputed, {
    relayFromSlug: function(){
      return slug => {
        const segments = slug.split('/')
        if(segments.length === 3)
          return segments[2]
      }
    },
    getDynamicTimeout: function(){
      const calculated = this.averageLatency*this.relays.length
      return calculated > 10000 ? calculated : 10000
    },
    getRefreshInterval: function(){
      const relay = this.relayFromUrl
      if( !relay )
        return this.store.prefs.duration
      if( this.store.results.get(relay)?.check?.connect && this.store.results.get(relay)?.check?.read && this.store.results.get(relay)?.check?.write && typeof this.store.results.get(relay)?.latency?.final !== 'undefined' )
        return this.store.results.get(relay).latency.final * 5 
      // if(this.store.results.get(relay)?.check?.connect && this.store.results.get(relay)?.check?.read && this.store.results.get(relay)?.check?.write)
        // return 30*1000
      return this.store.prefs.duration
    }
  }),

  methods: Object.assign(localMethods, RelaysLib),

  props: {
    resultsProp: {
      type: Object,
      default(){
        return {}
      }
    },
  },

})
</script>

<style scoped>
 #refresh { font-size: 12pt; color:#666; margin-bottom:15px }
 #refresh button { cursor: pointer; border-radius: 3px; border: 1px solid #a0a0a0; color:#333 }
 #refresh button:hover {color:#000;}
 #refresh button[disabled] {color:#999 !important; border-color:#e0e0e0}
</style>

