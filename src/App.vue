<script>
import { fetch, ResponseType } from '@tauri-apps/api/http';
import { readTextFile } from '@tauri-apps/api/fs';
import { writeBinaryFile } from '@tauri-apps/api/fs';
import { ask, message, open } from '@tauri-apps/api/dialog';
import { Body } from "@tauri-apps/api/http"

export default {
  data() {
    return {
      apiKey: "",
      voices: [],
      topics: [],
      messages: [],
      userInfo: null,
      selectedVoice: null,
      outputPath: null,
      stability: 80,
      clarity: 80,

      
      generationState: false,
      generationIntervalId: null,
      generationIndex: 0,
    };
  },
  methods: {
    async apply() {
      try {
        const response = await fetch("https://api.elevenlabs.io/v1/voices", {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            "xi-api-key": this.apiKey,
          },
          timeout: 30,
        });
        if(response.data.detail?.status == "invalid_api_key"){
          this.messages.push("You have entered an invalid API key");
        }else{
          this.voices = response.data.voices;
          await this.getUserInfo();
          this.messages.push("Please select a voice type above!");
        }
      } catch (err) {
        this.messages.push(err);
      }
    },
    selectionChanged(event) {
      const json = JSON.parse(event.target.value);
      this.selectedVoice = json;
      this.messages.push("Selected "+json.name+" as the voice type");
    },
    playPreviewAudio() {
      const select = document.getElementById("voice_selection");
      const selectedOption = select.options[select.selectedIndex];
      const json = JSON.parse(selectedOption.value);
      const audio = new Audio(json.preview_url);
      audio.play();
      this.messages.push("Playing preview audio for "+json.name);
    },
    async importTopics(){
      const filePath = await open({
        multiple: false,
        filters: [{
          name: "Topics Text File",
          extensions: ['txt']
        }]
      });
      const topics = await readTextFile(filePath);
      topics.split('\n').forEach(topic => {
        this.topics.push(topic);
      })
      this.messages.push("Loaded "+filePath);
    },

    async startGenerating() {
      this.generationIntervalId = setInterval(async () => {
        if(!this.generationState) {
          clearInterval(this.generationIntervalId);
          return;
        }

        await this.executeGeneration();
      }, 250);
    },
    stopGenerating() {
      clearInterval(this.generationIntervalId);
      this.generationState = false;
      this.generateIndex = 0;
    },
    async executeGeneration() {
      if(this.topics.length <= 0) {
        this.stopGenerating();
        this.messages.push("Generation complete!");
        return;
      }

      const currentTopic = this.topics[this.generationIndex];
      await this.postTTS(currentTopic);
      this.messages.push("Generating topic: "+currentTopic);

      this.topics.splice(this.generationIndex, 1);
      if(this.generationIndex >= this.topics.length) {
        this.stopGenerating();
        this.messages.push("Generation complete!");
      }
    },
    async generate(skipWarn) {
      const currCharCount = this.userInfo.subscription.character_count;
      const charLimit = this.userInfo.subscription.character_limit;
      if(currCharCount > charLimit && this.generationState == false && skipWarn == false){
        const askWarningOverLimit = await ask('Your character limit has been exceeded, are you sure you want to proceed?\n\nThis could increase the price of your next ElevenLabs invoice!', "WARNING!");
        if(!askWarningOverLimit){return}
      }
      if(this.outputPath == null){
        await message('Please select the folder where the files should be saved to!', "Select a folder");
        const selectedFolder = await open({
          directory: true,
          multiple: false,
        });
        if(!selectedFolder) {
          this.messages.push("You have not select a valid path.");
        }else{
          this.outputPath = selectedFolder;
          this.messages.push("Output path set to: "+selectedFolder);
          this.generate(true);
        }
      }else{
        if(this.generationState){
          this.generationState = false;
          this.stopGenerating();
        }else{
          this.generationState = true;
          await this.startGenerating();
        }
      }
    },

    async postTTS(topic){
      const requestBody = Body.json({
        "text": this.removeParentheses(topic),
        "voice_settings": {
          "stability": this.stability/100,
          "similarity_boost": this.clarity/100
        }
      });
      const headers = {
        'xi-api-key': this.apiKey,
        'Content-Type': 'application/json',
        'Accept': 'audio/mpeg'
      };
      fetch("https://api.elevenlabs.io/v1/text-to-speech/"+this.selectedVoice.voice_id, {
        method: 'POST',
        headers: headers,
        body: requestBody,
        responseType: ResponseType.Binary
      }).then(async response => {
        if (response.ok) {
          await this.saveAudio(response, topic);
        } else {
          this.messages.push("There was a unknown error in postTTS");
          console.log(JSON.stringify(response));
          this.stopGenerating();
        }
      })
    },

    async saveAudio(response, topic) {
      const audioBinary = new Uint8Array(await response.data);
      await writeBinaryFile({
        path: this.outputPath+"\\"+this.parseString(topic)+".mp3",
        contents: audioBinary,
        options: { dir: this.outputPath }
      })
    },

    async getUserInfo() {
      try {
        const response = await fetch("https://api.elevenlabs.io/v1/user", {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            "xi-api-key": this.apiKey,
          },
          timeout: 30,
        });
        this.userInfo = response.data;
      } catch (err) {
        this.messages.push("There was a unknown error in getUserInfo: "+err);
      }
    },

    parseString(str) {
      const regex = /[\[\]{}()*+?.,\\^$|#\s]/g;
      str = this.removeParentheses(str);
      str = str.replace(regex, '_');
      return str;
    },

    removeParentheses(str){
      return str.replace(/\s*\([^)]*\)\s*/g, "");
    }
  },
  computed: {
    defaultSelectValue() {
      return this.voices.length == 0 ? "Enter your API key above then click Apply" : "Select a voice type";
    },
    isVoicesEmpty() {
      return this.voices.length === 0;
    },
    enableGenerateBuitton() {
      return this.topics.length === 0;
    },
    generatingButton() {
      if(!this.generationState) {
        return "Start Generation";
      } else {
        return "Stop Generation";
      }
    },
    infoLabel() {
      if(this.topics.length === 0){
        return "";
      }else{
        const secs = this.topics.length*0.25;
        const mins = Math.ceil(secs / 60);
        const remainingSecs = Math.floor(secs % 60);
        const currCharCount = this.userInfo.subscription.character_count;
        const charLimit = this.userInfo.subscription.character_limit;
        const currSub = this.userInfo.subscription.tier;
        const costPer1kChar = currSub === "creator" ? 0.30 : currSub === "pro" ? 0.24 : currSub === "business" ? 0.18 : undefined;
        const firstLine = this.topics.length+" topics left to generate, ~ "+mins+" minutes and "+remainingSecs+" seconds";
        var secondLine = "";
        const topicsCharCount = this.topics.reduce((acc, string) => acc + string.length, 0);
        if(currCharCount+topicsCharCount > charLimit){
          var estimatedCost = ((topicsCharCount/1000)*costPer1kChar).toFixed(2);
          secondLine = "Characters: "+currCharCount+" (+"+topicsCharCount+") / "+charLimit+" : "+"Extra cost: ~ $"+estimatedCost
        }else{
          secondLine = "Characters: "+currCharCount+" (+"+topicsCharCount+") / "+charLimit
        }
        return firstLine+"\n"+secondLine;
      }
    }
  }
}
</script>

<template>
  <div class="container">
    <div class="top">
      <div class="user_info_section">
        <input type="password" v-model="apiKey" placeholder="Enter your API key here">
        <button @click="apply">Apply</button>
      </div>
      <div class="voices_section">
        <select :disabled="isVoicesEmpty" id="voice_selection" placeholder="Select a voice type" @change="selectionChanged">
          <option value="" disabled selected hidden>{{ defaultSelectValue }}</option>
          <option v-for="voice in voices" :value="JSON.stringify(voice)">{{ voice.name }}</option>
        </select>
        <button @click="playPreviewAudio" :disabled="isVoicesEmpty">Preview</button>
      </div>
      <div class="voice_settings">
        <label for="stability_slider">Stability: {{ stability }}%</label>
        <input type="range" id="stability_slider" name="stability_slider" min="0" max="100" v-model="stability">
        <label for="stability_slider">Clarity: {{ clarity }}%</label>
        <input type="range" id="stability_slider" name="stability_slider" min="0" max="100" v-model="clarity">
      </div>
    </div>
    <div class="middle">
      <div class="message-box">
        <div v-for="(message, index) in messages.slice().reverse()" :key="index">
          <span class="message" :style="{ 'background-color': index % 2 === 0 ? 'hsla(0, 0%, 30%, 0.9)' : 'hsla(0, 0%, 20%, 0.9)' }" :class="{ 'first-message': index === 0, 'last-message': index === messages.length - 1 }">{{ message }}</span>
        </div>
      </div>
    </div>
    <div class="bottom">
      <div class="action_section">
        <button :disabled="isVoicesEmpty" @click="importTopics">Import Topics</button>
        <label class="info-label" title="[Current (+Generated) / Limit]">{{ infoLabel }}</label>
        <button :disabled="enableGenerateBuitton" @click="generate(false)">{{ generatingButton }}</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  margin: 0;
  padding: 0;
}
.user_info_section {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.user_info_section input {
  width: 80%;
}
.voices_section {
  padding-top: 0.5rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.voices_section select {
  width: 80%;
  height: 2.5rem;
  color: #ffffff;
  background-color: #0f0f0f98;
  border-style: none;
  border-radius: 0.5rem;
  padding-left: 1rem;
  padding-right: 1rem;
}
.action_section {
  position: fixed;
  display: flex;
  justify-content: space-between;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 0.5rem;
}
.action_section label {
  font-size: smaller;
}
.middle {
  padding: 0.5rem;
}
.message-box {
  text-align: left;
  border-radius: 0.5rem;
  width: 100%;
  height: calc(100vh - 323px);
  overflow-y: auto;
  box-sizing: border-box;
  background-color: #0f0f0f98;
  box-shadow: inset 0 0 10px #000;
  padding: 1rem;
}
.message-box::-webkit-scrollbar {
  width: 0.5rem;
  height: 0.5rem;
}
.message-box::-webkit-scrollbar-track {
  background: transparent;
}
.message-box::-webkit-scrollbar-thumb {
  background-color: hsla(0, 0%, 40%, 0.9);
  border-radius: 0.25rem;
}
.message {
  display: block;
  background-color: hsla(0, 0%, 20%, 0.9);
  padding-left: 0.5rem;
  padding-right: 0.5rem;
  box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.2);
}
.first-message {
  border-top-left-radius: 0.5rem;
  border-top-right-radius: 0.5rem;
}
.last-message {
  border-bottom-left-radius: 0.5rem;
  border-bottom-right-radius: 0.5rem;
}
button:disabled {
  color: gray;
  cursor: default;
}
.info-label {
  white-space: pre-line;
}

.voice_settings {
  display: flex;
  flex-direction: column;
  align-items: left;
  width: 100%;
  text-align: left;
  padding-top: 1rem;
}
.voice_settings input {
  box-shadow: none;
}
</style>