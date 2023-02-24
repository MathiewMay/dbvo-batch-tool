# DBVO Batch Tool
The DBVO Batch tool is a tool that automates the generation of audio file from a txt file, at the moment it works with https://beta.elevenlabs.io/

Simply type you elevenlabs api key, select a voice type and import the text file, the tool will go line by line and generate an audio file for each line in the text file.

## Dev environment
If you want to help with the development of DBVO Batch Tool you will need to setup a dev environment.

Install dev environment
```
git clone https://github.com/MathiewMay/dbvo-batch-tool
cd ./dbvo-batch-tool
npm install
```

Run dev environment
```
npm run tauri dev
```
