# HttpPubSub

## prerequisite 
need two tools for generating rfc style txt file
* install mmark from https://github.com/mmarkdown/mmark
* install xml2rfc from https://xml2rfc.tools.ietf.org/

## compile
  % PATH/mmark pubsub.md > pubsub.xml
  % xml2rfc --text pubsub.xml
  % ls -hl pubsub.txt