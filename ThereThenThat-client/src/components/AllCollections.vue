<template>
  <v-app light
    @dragover.native="dragOver"
    @drop.native="doDrop"
    @dragstart.native="dragOver"
    @dragend.native="dragEnd"  
  >

    <TTTHeader context="All Collections"></TTTHeader>
    <v-content>
      <SearchAndCreate></SearchAndCreate>

      <v-tabs light fixed>
        <v-tabs-bar light>
          <v-tabs-item
            key="NamedCollections"
            href="#NamedCollections"
            ripple>
            Collections
          </v-tabs-item>
          <v-tabs-item
            key="ItemsByDay"
            href="#ItemsByDay"
            ripple>
            Items By Day
          </v-tabs-item>
          <v-tabs-slider color="yellow"></v-tabs-slider>
        </v-tabs-bar>
        <v-tabs-items>
          <v-tabs-content
            key="NamedCollections"
            id="NamedCollections"
          >
            <v-card flat pb-5 v-for="item in mainList" :key="item._id">
              <h4><a :href="item.fullURL">{{ item.title }}</a></h4>
              <p>{{ item.description }}</p>
            </v-card>
          </v-tabs-content>

          <v-tabs-content
            key="ItemsByDay"
            id="ItemsByDay"
          >
            <v-card flat pb-5 v-for="item in dayList" :key="item._id">
              <!-- <h4><a :href="item.fullURL">{{ item.title }}</a></h4> -->
              <h4>{{ item.theDay }}</h4>
            </v-card>
          </v-tabs-content>
        </v-tabs-items>
      </v-tabs>

  </v-content>
  </v-app>
</template>

<script>
import Vue from 'vue'
import TTTHeader from '@/components/TTTHeader'
import SearchAndCreate from '@/components/SearchAndCreate'
import { mapMutations, mapActions, mapGetters, mapState } from 'vuex';
import axios from 'axios';

export default {
  name: 'AllCollections',

  components: {
    TTTHeader,
    SearchAndCreate
  },
  
  computed: {
  ...mapGetters({
      searchFromStore: 'getMySearchResult'
    })
  },

  watch: {
    searchFromStore() {

      const theResult = this.searchFromStore;
      console.log(theResult);
      this.mainList = []; // this.searchFromStore;

      theResult.map(cur => {
        let newObj = {};
        newObj = cur;
        newObj.fullURL = `${cur.address}`;
        this.mainList.push(newObj);
      });

      console.log(this.mainList);
    }
  },

  data() {
    return {
      tabs: ['tab-1', 'tab-2', 'tab-3'],
      mainList: [],
      // dayList: [],

      dayList: [
        { theDay: "12-25=2017" },
        { theDay: "12-15=2017" },
        { theDay: "12-5=2017" }
      ]
    }
  },

  mounted: function() {
    this.getTTTList();
    // this.getDays();


  },

  methods: {
    onPaste() {
      // this would be for pasting items to a shelf..
      // so that they could be referenced from within
      // an individual collection
    },

    getDays() {
      this.dayList = [];

      // // deliberately doing a POST, because we may want to narrow this to data ranges later
      axios.post(`${this.$config.SERVER}${this.$config.SERVER_PORT}/api/getDays`, {}, config)
        .then(function (response) {
          console.log(response);
        })
        .catch(function (error) {
          console.log(error);
        });
    },

    getTTTList() {
      // call server for JSON data
      const theAddress = `${this.$config.SERVER}${this.$config.SERVER_PORT}`;

      this.mainList = [];
      fetch(theAddress)
        .then(response => response.json())
        .then(response => {
          response.map(cur => {
            const newObj = cur;
            // might change..
            newObj.fullURL = `${cur.address}`;
            this.mainList.push(newObj);
          })
        })
        .catch(err => {
          console.log(err);
        });
    },


    // specific drag / drop / paste handlers for
    // Collection list - these would put items on
    // a shalf for use with a specific collection
    dragEnd: function(args) {
      args.preventDefault();
    },

    dragOver: function(args) {
      args.preventDefault();
    },

    doDrop(args) {
      args.preventDefault();
      const link = args.dataTransfer.getData("Text");

      // same idea for this as paste:
      // this would be for putting links on a shelf..
      // so that they could be referenced from within
      // an individual collection
    }
  }
}
</script>

<style lang="stylus">
@import '../stylus/main'
</style>