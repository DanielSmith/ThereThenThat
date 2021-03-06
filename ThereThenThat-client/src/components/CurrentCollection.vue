<template>
  <v-app light
    @dragover.native="dragOver"
    @drop.native="doDrop"
    @dragstart.native="dragOver"
    @dragend.native="dragEnd"
    @paste.native="onPaste($event)">

    <TTTHeader :context="this.headerTitle"></TTTHeader>
    <v-container fluid>
      <v-layout row wrap>
        <v-flex xs12>
          <v-card v-if="this.numItems === 0">
            <v-container fluid grid-list-lg>
              <v-layout row wrap>
                <v-flex xs7>

                    <p>Drag media items here...</p>
                </v-flex>
              </v-layout>
            </v-container>
          </v-card>
          <v-card v-for="curItem in this.curCollectionList.renderLinks" :key="curItem.data.clientId">
            <v-container fluid grid-list-lg>
              <v-layout row>
                <v-flex xs7 class="mediaBox">

                  <component :itemPath="getCurMedia(curItem.data)" :allData="curItem"  v-bind:is="curItem.componentType">
                  </component>

                </v-flex>
                <v-flex xs5>
                  <v-btn color="indigo" dark @click="toggleEdit(curItem.data.clientId)"><v-icon dark left>mode_edit</v-icon></v-btn>
                <!-- </v-flex> -->

                <v-form v-if="showEditTags[curItem.data.clientId]" ref="form">
                  <v-layout pl-5 row wrap>
                    <v-flex xs5>
                      <v-text-field
                      label="Enter new tags"
                      v-model="allTagEdits[curItem.data.clientId]"
                      ></v-text-field>
                    </v-flex>
                    <v-flex xs4>
                      <v-btn @click="submitTags(curItem.data.clientId)">Add Tags</v-btn>
                    </v-flex>
                  </v-layout>
                </v-form>
                <v-btn  v-for="curTag in allTags[curItem.data.clientId]" key="curKey++"
                  @click="chooseTag(curItem.data.clientId, curTag)"
                  >
                  <span class="showEditTag" v-if="showEditTags[curItem.data.clientId]">X </span>
                  <strong> {{ curTag }} </strong> 
                </v-btn>
                </v-flex>
              </v-layout>
            </v-container>
          </v-card>
        </v-flex>
      </v-layout>
    </v-container>
  </v-app>
</template>

<script>
import TTTHeader from '@/components/TTTHeader';
import SearchAndCreate from '@/components/SearchAndCreate';
import axios from 'axios';

import audioComponent from './Audio';
import videoComponent from './Video';
import imageComponent from './Image';
import textComponent from './Text';
import mimeUtils from '../../../common/mimeUtils';
import { CollectionMixin }  from '../mixins/CollectionMixin';

export default {
  name: 'CurrentCollection',
  mixins: [CollectionMixin],

  components: {
    TTTHeader,
    SearchAndCreate,
    audioComponent,
    videoComponent,
    imageComponent,
    textComponent
  },
  
  data() {
    return {
      currentView: 'videoComponent',

      curKey: 1,
      itemList: [],

      allTags: {},
      allTagEdits: {},
      showEditTags: {},

      imageList: [],
      numItems: 0,

      headerTitle: 'fetching collection...'
    }
  },

  mounted: function() { 
    this.getSingleCollection();
  },

  methods: {

    // call server for JSON data
    getSingleCollection() {
      const path = `${this.$config.SERVER}${this.$config.SERVER_PORT}${this.$route.path}`;
      this.numItems = 0;

      fetch(path)
        .then(response => response.json())
        .then(response => {

          this.headerTitle = response.title;
          this.curCollectionList = response;
          this.curCollectionList.renderLinks = [];
          this.curCollectionList.links.map(cur => {
            let newObj = {};
            this.numItems++;

            newObj.data = cur;
            newObj.componentType = mimeUtils.getItemType(cur.fileName)
            newObj.tags = cur.tags || [];
            newObj.id = cur.clientId;
            newObj.data.sourceType = "remote";

            this.$set(this.showEditTags, newObj.id, false);
            this.$set(this.allTagEdits, newObj.id, '');
            this.$set(this.allTags, newObj.id, newObj.tags);
            this.curCollectionList.renderLinks.push(newObj);
          });
        })        
        .catch(err => {
          console.log(err);
        });
    },

    onPaste(event) {
      // event.preventDefault();

      let index = 0;
      let items = (event.clipboardData || event.originalEvent.clipboardData).items;
      const clientId = this.newUUID();

      // do a check here to see if it is an image...
      let imageItem = items[0];
      let imageFile = imageItem.getAsFile();
      const container = this.curCollectionList._id;

      if (imageItem.kind === 'file') {
        let reader = new FileReader();

        let URLObj = window.URL || window.webkitURL;
        let source = URLObj.createObjectURL(imageFile);
        this.numItems++;

        reader.onload = (event) => {
          let myImage = new Image();
          myImage.src = event.target.result;
          const clientId = this.newUUID();
          let newObj = {};
          newObj.componentType = mimeUtils.getItemType('.png');
          newObj.data = myImage;
          newObj.data.clientId = clientId;
          // // look into how to get rid of this?
          newObj.id = clientId;
          newObj.data.sourceType = "local";
          newObj.tags = [];
          newObj.data.path = reader.result;

          this.$set(this.showEditTags, newObj.id, false);
          this.$set(this.allTagEdits, newObj.id, '');
          this.$set(this.allTags, newObj.id, newObj.tags);

          this.curCollectionList.renderLinks.push(newObj);
          this.doUpload(imageFile, clientId);
          
        };
        reader.readAsDataURL(imageFile);
      }
    },


    doDroppedLink: function(event) {
      let src = '';

      if ([...event.dataTransfer.types].includes('text/plain')) {
        event.dataTransfer.types.map(curType => {
          if (curType === 'text/plain') {
            let link = event.dataTransfer.getData('Text');

            const container = this.curCollectionList._id;
            const address = this.curCollectionList.address;
            const clientId = this.newUUID();

            let mynow = Date.now();

            axios.post(`${this.$config.SERVER}${this.$config.SERVER_PORT}/api/addlink`, { 
              link, container, clientId })
              .then(response => {
                  this.numItems++;
                  let newLink = {
                    // may need to add clientId here...
                    _id: response.data._id,
                    url: response.data.url,
                    title: response.data.title,
                    description: response.data.description,
                    clientId: clientId
                  }

                  let newObj = {};
                  newObj.id = clientId;
                  newObj.tags = [];
                  newObj.componentType = "textComponent";
                  newObj.data = newLink;

                  this.$set(this.showEditTags, newObj.id, false);
                  this.$set(this.allTagEdits, newObj.id, '');
                  this.$set(this.allTags, newObj.id, newObj.tags);

                  this.curCollectionList.renderLinks.push(newObj);
              })
              .catch((err) => {
                  console.log(err);
              })

              return;
            }
          })
        }

        if (src !== '') {
          let curImage = new Image();
          let canvas = document.createElement('canvas');
          let ctx = canvas.getContext('2d');
          const clientId = this.newUUID();
          this.numItems++;

          curImage.onload = () => {
            canvas.width = curImage.width;
            canvas.height = curImage.height;
            ctx.drawImage(curImage, 0, 0);

            fetch(canvas.toDataURL('image/png'))
              .then(res => res.blob())
              .then(blob => {
                this.doUpload(blob, clientId);
              })
          }

        let newObj = {};
        newObj.id = clientId;
        newObj.tags = [];
        newObj.data = curImage;

        newObj.componentType = "image";
        this.$set(this.showEditTags, newObj.id, false);
        this.$set(this.allTagEdits, newObj.id, '');
        this.$set(this.allTags, newObj.id, newObj.tags);


        curImage.setAttribute('crossOrigin', 'anonymous');
        curImage.src = src;
        this.curCollectionList.renderLinks.push(newObj);
      }
    },

    doDroppedFiles: function(event) {
      let theFiles = Array.from(event.dataTransfer.files);

      theFiles.map(curFile => {
        // get file type
        let curFileData = mimeUtils.getData(curFile);
        this.numItems++;

        // am sure this should be reworked.. hacked it from being
        // image-only to image, audio, or video
        let reader = new FileReader();
        reader.onload = (inner) => {
          
          let droppedItem = this.getNewElementForType(curFileData.type);
          const clientId = this.newUUID();
          let newObj = {};
          newObj.componentType = mimeUtils.getItemType(curFileData.ext);
          newObj.data = curFile;
          newObj.data.clientId = clientId;

          // look into how to get rid of this?
          newObj.id = clientId;
          newObj.data.sourceType = "local";
          newObj.tags = [];
          newObj.data.path = reader.result;

          this.$set(this.showEditTags, newObj.id, false);
          this.$set(this.allTagEdits, newObj.id, '');
          this.$set(this.allTags, newObj.id, newObj.tags);

          this.curCollectionList.renderLinks.push(newObj);
          this.doUpload(curFile, clientId, curFileData.ext);
        }

        reader.readAsDataURL(curFile);
      })
    },


    doUpload(uploadFile, clientId, extension = "png") {
      const uploadData = new FormData();
      uploadData.append('thefile', uploadFile);
      uploadData.append('title', 'media upload');
      uploadData.append('extension', extension);
      uploadData.append('description', 'media description');
      uploadData.append('container', this.curCollectionList._id);
      uploadData.append('clientId', clientId);

      const config = {
        headers: { 'content-type': 'multipart/form-data' }
      }

      axios.post(`${this.$config.SERVER}${this.$config.SERVER_PORT}/api/fileupload`, uploadData, config)
        .then(function (response) {
          console.log(response);
        })
        .catch(function (error) {
          console.log(error);
        });

      this.numItems++;
    },


    doDrop: function(event) {
      event.preventDefault();

      if ([...event.dataTransfer.types].includes('text/plain')) {
        this.doDroppedLink(event)
      } else {
        this.doDroppedFiles(event);
      }
    },



    toggleEdit(id) {
      console.log('toggle for ' + id);
      this.showEditTags[id] = !this.showEditTags[id];
    },

    submitTags(id) {
      
      // rewrite this.. the empty and null cases can cause problems
      // am also being careful not to send an empty tag
      let tAry = (this.allTagEdits[id]).split(/ +/);
      tAry = tAry.filter(val => val !== '');

      if (tAry.length === 0) { tAry = ['']; }
      if (this.allTags[id] === null) { this.allTags[id] = []; }

      const newTagsAr = [...this.allTags[id], ...tAry].sort();
      let newTags = [...new Set(newTagsAr)];
      newTags = newTags.filter(val => val !== '');
      this.$set(this.allTags, id, newTags);

      this.showEditTags[id] = false;
      this.allTagEdits[id] = '';
      this.syncTags(id);
    },




    syncTags(id) {

      const tags = this.allTags[id];

      let apiPath = `${this.$config.SERVER}${this.$config.SERVER_PORT}/api/synctags`,              
        dbArgs = { clientId: id, tagquery: tags };

      const config = { headers: { 'Content-Type': 'application/json' } };

      axios.post(apiPath, dbArgs, config)
        .then(response => {
          console.log(response);
        })
        .catch(err => {
          console.log(err);
        });
    }
  }
}
</script>

<style lang="stylus">
@import '../stylus/main'

img, audio, video {
  max-width: 60%;
}

.showEditTag {
  color: red;
  width: 50px;
  text-align: left;
}

.mediaBox {
  /* border: solid 1px red; */
}
</style>
