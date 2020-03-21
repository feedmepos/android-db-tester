<template>
  <q-page class="flex flex-center column">
    <div>
      <q-btn v-for="i in dbList" :key="i.name" @click="selectDb(i.db)">{{i.name}}</q-btn>
      <q-btn @click="compact">Compact</q-btn>
    </div>
    <div v-if="this.dbInfo ">
      <div v-for="v in dbs" :key="v[0]">
        {{ v[0] }}: {{v[1]}}
      </div>
      <div>
        <q-input type="number" v-model="completeCnt"></q-input>
        <q-btn @click="feed('COMPLETE', completeCnt)">Feed {{ completeCnt }} Complete Bill</q-btn>
        <q-btn @click="pumpRev('COMPLETE')">Pump Rev</q-btn>
        <q-input type="number" v-model="activeCnt"></q-input>
        <q-btn @click="feed('ACTIVE', activeCnt)">Feed {{activeCnt}} Active Bill</q-btn>
        <q-btn @click="pumpRev('ACTIVE')">Pump Rev</q-btn>
        <q-btn @click="benchmarkFind('ACTIVE', 10)">Benchmark Find Active Bill</q-btn>
        <div>Find Result: {{ findResult.length }}</div>
      </div>
      <div>
        Operation result:
        <div v-for="v in Object.entries(this.result)" :key="v[0]">
          {{ v[0] }}: {{v[1]}}
        </div>
      </div>
    </div>
    <div>
    </div>
  </q-page>
</template>

<script>
import PouchDB from 'pouchdb';
import Adapter from 'pouchdb-adapter-cordova-sqlite';
import PouchDbFind from 'pouchdb-find';
import pouchdbDebug from 'pouchdb-debug';
import sample from './sampleBill'

const defaultResult = () => ({
        name: 'nothing',
        cnt: 0,
        totalTime: 0,
        average: 0,
        max: 0,
        min: 99999999999999,
      })

export default {
  name: 'PageIndex',
  data() {
    return {
      dbList: [],
      currentDb: null,
      completeCnt: 1,
      activeCnt: 1,
      dbInfo: null,
      result: defaultResult(),
      findResult: [],
    }
  },
  computed: {
    dbKeys() {
      return this.dbList.map(v => v.name)
    },
    dbs() {
      return Object.entries(this.dbInfo)
    }
  },
  mounted() {
    PouchDB.plugin(Adapter);
    PouchDB.plugin(PouchDbFind);
    PouchDB.plugin(pouchdbDebug);
    PouchDB.debug.enable('pouchdb:find');
    console.log(this.dbList)
    console.log(this.$q.cordova)
    this.reset();
  },
  methods: {
    async selectDb(db) {
      this.currentDb = db;
      this.dbInfo = await db.info()
    },
    async feed(status, cnt) {
      const active = sample();
      active.status = status;
      const feed = (new Array((Number(cnt)))).fill(active);
      this.startOperation(`insert ${status}`, 1, () => this.currentDb.bulkDocs(feed))
    },
    async find(status) {
      await this.startOperation(`find ${status}`, 1, async () => {
        const result = await this.currentDb.find({
          selector: {
            type_bill: true,
            status: status
          }
        })
        this.findResult = result.docs
      })
    },
    async benchmarkFind(status, cnt) {
      await this.find(status);
      this.startOperation(`find ${status}`, cnt, async () => {
        const result = await this.currentDb.find({
          selector: {
            type_bill: true,
            status: status
          }
        })
      })
    },
    async pumpRev(status) {
      const result = await this.currentDb.find({
          selector: {
            type_bill: true,
            status: status
          }
        })
      this.startOperation(`pump doc ${status}`, 1, () => this.currentDb.bulkDocs(result.docs))
    },
    async compact() {
      this.dbList.forEach(db => {
        db.db.compact()
      });
    },
    async reset() {
      this.dbList.forEach(db => {
        db.db.destroy()
      });
      this.currentDb = null;
      this.dbList = [];
      this.dbList.push({
        name: 'default',
        db: new PouchDB('main.db'),
      })
      this.dbList.push({
        name: 'sqlite',
        db: new PouchDB('main.db', {
          adapter: 'cordova-sqlite',
          iosDatabaseLocation: 'Library',
          androidDatabaseImplementation: 2
        }),
      })
      this.dbList.forEach(({db}) => {
        db.createIndex({
          index: {
            fields: ['type_bill', 'status']
          }
        })
      })
    },
    async startOperation(name, cnt, fn) {
      console.log(`starting operation ${name}`)
      const result = defaultResult();
      const start = new Date()
      for(let i = 1; i <= cnt; ++i) {
        console.log(`running ${name}: ${i}`)
        const fnStart = new Date()
        await fn(i)
        const fnEnd = new Date()
        const time = fnEnd-fnStart;
        if(time > result.max) {
          result.max = time;
        }
        if(time < result.min) {
          result.min = time;
        }
      }
      const end = new Date()
      result.name = name;
      result.totalTime = end - start;
      result.average = result.totalTime/cnt;
      this.result = result;
      console.log(`finishing operation ${name}`)
    }
  }
}
</script>
