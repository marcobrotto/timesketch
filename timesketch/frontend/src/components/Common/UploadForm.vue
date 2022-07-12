<!--
Copyright 2019 Google Inc. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <div>
    <form v-on:submit.prevent="submitForm">
      <div class="field">
        <div class="file has-name">
          <label class="file-label">
            <input id="datafile" class="file-input" type="file" name="resume" v-on:change="setFileName($event.target.files)" />
            <span class="file-cta">
              <span class="file-icon">
                <i class="fas fa-upload"></i>
              </span>
              <span class="file-label">
                Choose a file…
              </span>
            </span>
            <span class="file-name" v-if="fileName">
              <span v-if="!fileName">Please select a file</span>
              {{ fileName }}
            </span>
          </label>
        </div>
      </div>
      <div class="field">
        <div v-if="error">
          <article class="message is-danger mb-0">
            <div class="message-body">
              {{ error }}
            </div>
          </article>
        </div>
        <div v-if="!error && hMenu">
          <article class="message is-success mb-0">
            <div class="message-body">
              New headers names are correct
            </div>
          </article>
        </div>
        <div v-if="columnLength() || hMenu">
          <div v-if="!hSuggestedFlag">
            <div v-for="h in headers" :key="h.id">
              <input class="input" type="text" :id="h.id" v-model="h.val">
            </div>
          </div>
          <button @click="showHeaders" class="tag is-info" type="button">
            <span v-if="hSuggestedFlag">Suggest new headers</span>
            <span v-else>Hide headers</span>
          </button>          
        </div>
      </div>
      <div class="field" v-if="fileName">
        <label class="label">Name</label>
        <div class="control">
          <input v-model="form.name" class="input" type="text" required placeholder="Name your timeline" />
        </div>
      </div>
      <div class="error" v-if="!error">
        <div class="field" v-if="fileName">
          <label class="label">Name</label>
          <div class="control">
            <input v-model="form.name" class="input" type="text" required placeholder="Name your timeline" />
          </div>
        </div>

        <div class="field" v-if="fileName && percentCompleted === 0">
          <div class="control">
            <input class="button is-success" type="submit" value="Upload" />
          </div>
        </div>
      </div>
    </form>
    <br />
    <b-progress
      v-if="percentCompleted !== 0"
      :value="percentCompleted"
      show-value
      format="percent"
      type="is-info"
      size="is-medium"
    >
      <span v-if="percentCompleted === 100">Waiting for request to finish..</span>
    </b-progress>
  </div>
</template>

<script>
import ApiClient from '../../utils/RestApiClient'

export default {
  data() {
    return {
      headers: [], // headers in the CSV 
      hMissing : [], // headers missing in the CSV
      hMenu : false, // this var is used to show the menu for change the headers
      form: {
        name: '',
        file: '',
      },
      fileName: '', 
      error: '',
      percentCompleted: 0,
      hSuggestedFlag: true,
    }
  },
  methods: {
    clearFormData: function() {
      this.form.name = ''
      this.form.file = ''
      this.fileName = ''
    },
    submitForm: function() {
      if (this.error == 'Please select a file with a valid extension' ) {
        return;
      }
      alert("why")
      let formData = new FormData()
      formData.append('file', this.form.file)
      formData.append('name', this.form.name)
      formData.append('provider', 'WebUpload')
      formData.append('context', this.fileName)
      formData.append('total_file_size', this.form.file.size)
      formData.append('sketch_id', this.$store.state.sketch.id)
      let config = {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
        onUploadProgress: function(progressEvent) {
          this.percentCompleted = Math.round((progressEvent.loaded * 100) / progressEvent.total)
        }.bind(this),
      }
      ApiClient.uploadTimeline(formData, config)
        .then(response => {
          this.$store.dispatch('updateSketch', this.$store.state.sketch.id)
          this.$emit('toggleModal')
          this.clearFormData()
          this.percentCompleted = 0
       })
       .catch(e => {}) 
    },
    checkHeaders: function (headers){
      let hMissing = [] // list of columns that are missing in the CSV schema
      if( headers.indexOf("message") < 0 ){
          hMissing.push("message");
      }
      if( headers.indexOf("datetime") < 0 ){
          hMissing.push("datetime");
      }
      if( headers.indexOf("timestamp_desc") < 0 ){
          hMissing.push("timestamp_desc");
      }
      return hMissing;
    },
    setFileName: function(fileList) {
      let fileName = fileList[0].name
      let fileExtension = fileName.split('.')[1]
      this.form.file = fileList[0]
      this.form.name = fileName
        .split('.')
        .slice(0, -1)
        .join('.')
      this.fileName = fileName
      this.error = ''
      let allowedExtensions = ['csv', 'json', 'jsonl', 'plaso']
      if (!allowedExtensions.includes(fileExtension)) {
        this.error = 'Please select a file with a valid extension'
      }

      // need to verify with JSONL files

      if(fileExtension === "csv"){
        var reader = new FileReader()
        var file = document.getElementById("datafile").files[0]
        
        // read only 1000 B --> it is reasonable that the header of the CSV file ends before the 1000^ byte.
        // this is done to prevent JS reading a large CSV file (GBs) 
        var vueJS = this
        reader.readAsText(file.slice(0, 1000))
        reader.onloadend = function(e){
            if (e.target.readyState == FileReader.DONE){  // DONE == 2
              var data = e.target.result
              var headers = data.split("\n")[0] //<--- is there a better way to split columns?
                                              // or avoiding to upload the entire csv
              headers = headers.split(",") // all  headers of CSV uploaded
              for(let i in headers){
                vueJS.headers.push({
                    id : i,
                    val : headers[i]
                })
              }
              vueJS.hMissing = vueJS.checkHeaders(headers)
              if (vueJS.hMissing.length > 0)
                vueJS.error = 'Missing headers: ' + vueJS._data.hMissing.toString()
                vueJS.hMenu = true
            }
          }        
      }
    },
    showHeaders: function(){
      this.hSuggestedFlag = !(this.hSuggestedFlag);
    },
    columnLength: function(){
      let headers = this.headers.map(function(e) {
        return e.val;
      });
      this.hMissing = this.checkHeaders(headers);
      if(this.hMenu){
        this.error = 'Missing headers: ' + this._data.hMissing.toString()
      }
      if(this.hMissing.length === 0){
        this.error = '';
      }
    },
  },
}
</script>
