<script>

//import Greet from "./components/Greet.vue";
import {Store} from '@tauri-apps/plugin-store';
import {BaseDirectory, open, readDir, readFile, readTextFile} from '@tauri-apps/plugin-fs';
import {upload} from '@tauri-apps/plugin-upload';
import { fetch } from '@tauri-apps/plugin-http';
import { resolve, resourceDir, basename } from '@tauri-apps/api/path';

const store = new Store('store.bin');

export default {
  components: {},
  data() {
    return {
      page: "home",
      interval: false,
      intervalToggle: 0,

      folder: "",
      folderPrefixTeacher: "",
      folderPrefixPupil: "",
      apiUrl: "https://www.schule-intern.de/rest.php/vplan/updater",
      apiKey: "",
      power: false
    };
  },
  created() {
    this.load();

  },
  methods: {

    async load() {

      this.folder = await store.get('folder');
      this.power = await store.get('power');
      this.folderPrefixTeacher = await store.get('folderPrefixTeacher');
      this.folderPrefixPupil = await store.get('folderPrefixPupil');
      this.apiUrl = await store.get('apiUrl');
      this.apiKey = await store.get('apiKey');

      this.watchFolder();

    },
    async handlerFolder() {


      const file = await open({
        multiple: false,
        directory: true,
      });
      //console.log(file)
      if (file) {
        this.folder = file;
        await store.set('folder', file);
        await store.save();

      }


    },
    getFolderNames (name) {
      return [ this.folderPrefixTeacher+'heute', this.folderPrefixTeacher+'morgen', this.folderPrefixPupil+'heute', this.folderPrefixPupil+'morgen' ];
    },
    async watchFolder() {

      //console.log('watch-0', this.power)

      if (this.power) {
        if (this.folder && this.apiUrl && this.apiKey) {


          //console.log('watch-1', this.folder)
          //console.log('watch-2', this.apiUrl)
          //console.log('watch-3', this.apiKey)


          this.interval = setInterval(this.handlerInterval, 5000);

          //console.log(this.interval)


          /*
          await watch(
              this.folder.value,
              (event) => {
                console.log('app.log event', event);
              },
              {
                baseDir: BaseDirectory.AppLog,
                recursive: true,
              }
          );
          */


        } else {
          this.handlerPage('settings');
        }
      }

    },

    async sendData(filepath) {
      if (!filepath) {
        filepath = await resolve(await resourceDir(), '../../../index.php');
      }

      console.log('sendData:', filepath)
      this.intervalToggle++;

      const res = await upload( // get on server with php://input
          this.apiUrl,
          filepath,
          ({progress, total}) => {
            console.log('upload:', `Uploaded ${progress} of ${total} bytes`)
          }, // a callback that will be called with the upload progress
          {
            'Content-Type': 'text/plain',
            "auth-app": this.apiKey,
            'auth-session': "false"
          } // optional headers to send with the request
      );

      console.log('upload:', res)

      if (res) {
        if (this.intervalToggle > 0) {
          this.intervalToggle--;
        }
      }




      var formdata = new FormData();
      // formdata.append("html", data);
      formdata.append("file", 1111);

      fetch(this.apiUrl, {
        method: 'POST',
        body: JSON.stringify({file: 2222}), // get on server with php://input
        // body: formdata, // get on server with $_POST
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
          "auth-app": this.apiKey,
          'auth-session': false
        }
      })
      .then(async (response) => {

        console.log('json body:', response)
        if (response.status == 200) {
          const json = await response.json();
          console.log('json body:', json);
        } else {
          console.log('json body:', 'Error', response)
        }
        this.intervalToggle--;

      })




      // Upload file

      const fileData = new File([await readTextFile(filepath)], await basename(filepath));
      var fileformdata = new FormData();
      fileformdata.append("file", fileData);

      fetch(this.apiUrl, {
        method: 'POST',
        body: fileformdata, // get on server with $_FILES
        headers: {
          // 'Content-Type': 'multipart/form-data', // lol: https://stackoverflow.com/a/39281156
          "auth-app": this.apiKey,
          'auth-session': false
        }
      })
      .then(async (response) => {

        console.log('post file:', response)
        if (response.status == 200) {
            const json = await response.json();
            console.log('post file:', json);
        } else {
          console.log('post file:', 'Error', response)
        }
        this.intervalToggle--;

      })


    },

    handlerInterval() {

      //console.log('intrval');


      readDir(this.folder, {baseDir: BaseDirectory.Home}).then((folders) => {


        folders.forEach((folder) => {
          if (folder.isDirectory) {

            if (this.getFolderNames().includes(folder.name)) {

              readDir(this.folder + '/' + folder.name, {baseDir: BaseDirectory.Home}).then((files) => {

                files.forEach(async (file) => {
                  if (file.isFile) {

                    //console.log(this.folder+'/'+folder.name+'/'+file.name)
                    /*
                        const contents = await open(this.folder + '/' + folder.name + '/' + file.name, {baseDir: BaseDirectory.Home});
                        const buf = new Uint8Array(50000);
                        const numberOfBytesRead = await contents.read(buf); // 11 bytes
                        const text = new TextDecoder().decode(buf);  // "hello world"
                        await contents.close();
                    */


                    this.sendData(this.folder + '/' + folder.name + '/' + file.name);

                  }
                })
              });


            }
          }
        })
      });
      //this.intervalToggle = false;


    },

    handlerSave() {

      store.set('folderPrefixTeacher', this.folderPrefixTeacher);
      store.set('folderPrefixPupil', this.folderPrefixPupil);
      store.set('apiUrl', this.apiUrl);
      store.set('apiKey', this.apiKey);
      store.save();

      this.handlerPage();
    },

    handlerPower() {
      this.power = !this.power;
      store.set('power', this.power);
      store.save();

      if (this.power) {
        this.load();
      } else {
        clearInterval(this.interval);

      }

    },

    handlerPage(page) {
      if (!page) {
        page = 'home';
      }
      this.page = page;
    }
  },
};


</script>

<template>
  <div class="container">
    <div class="header">
      <button @click="handlerPage('home')">Home</button>
      <button @click="handlerPage('settings')">Settings</button>
    </div>
    <div class="container-home" v-if="page == 'home'">
      <div class="button" @click="handlerPower">Button {{ this.power }}</div>
      {{ intervalToggle }}

      <h1 @click="sendData()">Hello :) (press me to sendData())</h1>
    </div>

    <div class="container-settings" v-if="page == 'settings'">
      <ul>
        <li>
          <label>Folder</label>
          <input v-model="folder" readonly>
          <button @click="handlerFolder">Folder</button>
        </li>
        <li>
          <label>folderPrefixTeacher:</label>
          <input v-model="folderPrefixTeacher">
        </li>
        <li>
          <label>folderPrefixPupil:</label>
          <input v-model="folderPrefixPupil">
        </li>
        <li>
          <label>apiUrl:</label>
          <input v-model="apiUrl">
        </li>
        <li>
          <label>apiKey:</label>
          <input v-model="apiKey">
        </li>
        <li>
          <button @click="handlerSave">Speichern</button>
        </li>
      </ul>
    </div>

  </div>
</template>

<style scoped>

</style>
