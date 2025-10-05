# Source Code

## BO SDK
```
(function(global, $){

  function ECoreChat(opts){
    this.tokenEndpoint = opts.tokenEndpoint;
    this.baseUrl = opts.baseUrl;
    this.wsUrl = opts.wsUrl;
    this.sttUrl = opts.sttUrl;
    this.ttsUrl = opts.ttsUrl;
    this.meetingId = opts.meetingId;
    this.socket = null;
    this.idToken = null;
    this.userId = null;
    this.audioContext = null;
  }

  /** --- Core Connection --- **/
  ECoreChat.prototype.connect = function(cb){
    var self = this;
    $.get(this.tokenEndpoint, function(res){
      self.idToken = res.id_token || res.access_token;
      self.userId = res.sub;

      self.socket = io(self.wsUrl, { transports: ['websocket'] });

      self.socket.on('connect', function(){
        self.socket.send('40' + JSON.stringify({
          token: 'Bearer ' + self.idToken,
          userId: self.userId
        }));
        if(cb) cb();
      });
    });
  };

  /** --- Generic API Helper --- **/
  ECoreChat.prototype.apiCall = function(method, path, data, cb){
    var self = this;
    $.ajax({
      url: this.baseUrl + path,
      method: method,
      headers: { "Authorization": "Bearer " + self.idToken },
      contentType: "application/json",
      data: data ? JSON.stringify(data) : null,
      success: function(resp){ if(cb) cb(null, resp); },
      error: function(xhr){ if(cb) cb(xhr.responseJSON || xhr.responseText); }
    });
  };

  /** ------------------------------------------------------------------
   * PRE-MEETING FEATURES
   * ------------------------------------------------------------------ **/

  // Create a draft meeting
  ECoreChat.prototype.createDraft = function(draftData, cb){
    this.apiCall("POST", "/meeting-drafts", draftData, cb);
  };

  // Get AI-generated insights
  ECoreChat.prototype.getInsights = function(cb){
    this.apiCall("GET", "/meetings/" + this.meetingId + "/get-insights", null, cb);
  };

  // Upload supporting docs
  ECoreChat.prototype.uploadFile = function(file, cb){
    var self = this;
    var form = new FormData();
    form.append("file", file);

    $.ajax({
      url: this.baseUrl + "/files/upload/azure",
      method: "POST",
      headers: { "Authorization": "Bearer " + self.idToken },
      data: form,
      processData: false,
      contentType: false,
      success: function(resp){ if(cb) cb(null, resp); },
      error: function(xhr){ if(cb) cb(xhr.responseJSON || xhr.responseText); }
    });
  };

  /** ------------------------------------------------------------------
   * IN-MEETING FEATURES
   * ------------------------------------------------------------------ **/

  // Start the meeting
  ECoreChat.prototype.startMeeting = function(cb){
    this.apiCall("PATCH", "/meetings/" + this.meetingId, { status: "started" }, cb);
  };

  // Send chat message (text)
  ECoreChat.prototype.sendMessage = function(text, metadata){
    if(this.socket){
      this.socket.emit("chat_message", {
        meetingId: this.meetingId,
        userId: this.userId,
        text: text,
        metadata: metadata || {}
      });
    }
  };

  // Listen for chat messages
  ECoreChat.prototype.listenForMessages = function(cb){
    if(this.socket){
      this.socket.on("chat_message", function(msg){
        cb(msg);
      });
    }
  };

  // Create an action item
  ECoreChat.prototype.createActionItem = function(item, cb){
    this.apiCall("POST", "/action-items", item, cb);
  };

  // Voice capture → STT → send transcript as message
  ECoreChat.prototype.startVoiceCapture = function(onTranscript){
    var self = this;
    navigator.mediaDevices.getUserMedia({ audio: true })
      .then(function(stream){
        self.audioContext = new AudioContext();
        var source = self.audioContext.createMediaStreamSource(stream);
        var processor = self.audioContext.createScriptProcessor(4096, 1, 1);

        source.connect(processor);
        processor.connect(self.audioContext.destination);

        processor.onaudioprocess = function(e){
          var inputData = e.inputBuffer.getChannelData(0);
          var pcm16 = self._floatTo16BitPCM(inputData);

          $.ajax({
            url: self.sttUrl,
            method: "POST",
            headers: { "Authorization": "Bearer " + self.idToken },
            processData: false,
            contentType: "application/octet-stream",
            data: pcm16,
            success: function(resp){
              if(resp.transcript){
                onTranscript(resp.transcript);
                self.sendMessage(resp.transcript, { source: "voice" });
              }
            }
          });
        };
      });
  };

  // Speak bot reply
  ECoreChat.prototype.speakText = function(text){
    var self = this;
    $.ajax({
      url: self.ttsUrl,
      method: "POST",
      headers: { "Authorization": "Bearer " + self.idToken },
      contentType: "application/json",
      data: JSON.stringify({ text: text }),
      xhrFields: { responseType: "arraybuffer" },
      success: function(audioBuffer){ self._playAudio(audioBuffer); }
    });
  };

  /** ------------------------------------------------------------------
   * POST-MEETING FEATURES
   * ------------------------------------------------------------------ **/

  // Get meeting summary
  ECoreChat.prototype.getSummary = function(cb){
    this.apiCall("GET", "/meetings/" + this.meetingId, null, cb);
  };

  // Get action items
  ECoreChat.prototype.getActionItems = function(cb){
    this.apiCall("GET", "/action-items/meeting/" + this.meetingId, null, cb);
  };

  // Update action items (bulk)
  ECoreChat.prototype.updateActionItemStatus = function(statusUpdates, cb){
    this.apiCall("PUT", "/action-items/status", statusUpdates, cb);
  };

  // Download a file
  ECoreChat.prototype.downloadFile = function(fileReq, cb){
    this.apiCall("POST", "/files/download/azure", fileReq, cb);
  };

  /** ------------------------------------------------------------------
   * Helpers
   * ------------------------------------------------------------------ **/

  ECoreChat.prototype._floatTo16BitPCM = function(float32Array){
    var buffer = new ArrayBuffer(float32Array.length * 2);
    var view = new DataView(buffer);
    for (var i = 0; i < float32Array.length; i++){
      var s = Math.max(-1, Math.min(1, float32Array[i]));
      view.setInt16(i*2, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
    }
    return buffer;
  };

  ECoreChat.prototype._playAudio = function(arrayBuffer){
    var self = this;
    if(!self.audioContext){
      self.audioContext = new (window.AudioContext || window.webkitAudioContext)();
    }
    self.audioContext.decodeAudioData(arrayBuffer, function(buffer){
      var source = self.audioContext.createBufferSource();
      source.buffer = buffer;
      source.connect(self.audioContext.destination);
      source.start(0);
    });
  };

  global.ECoreChat = ECoreChat;

})(window, jQuery);
```

## Embedded in E-Cabinet
```
<script>
$(function(){
  var chat = new ECoreChat({
    tokenEndpoint: "/api/chat/token",
    baseUrl: "https://ecore.dev/api/v1",
    wsUrl: "https://ecore.dev/ws",
    sttUrl: "https://ecore.dev/stt",
    ttsUrl: "https://ecore.dev/tts",
    meetingId: "68d10f80929b82f38cf08b22"
  });

  // Connect & start
  chat.connect(function(){
    console.log("Connected to eCore");

    /** PRE-MEETING **/
    chat.createDraft({ title: "Weekly Sync" }, function(err,res){
      console.log("Draft created:", res);
    });

    chat.uploadFile($("#docUpload")[0].files[0], function(err,res){
      console.log("Doc uploaded:", res);
    });

    chat.getInsights(function(err,res){
      console.log("Agenda insights:", res);
    });

    /** IN-MEETING **/
    chat.startMeeting(function(){ console.log("Meeting started"); });

    chat.listenForMessages(function(msg){
      console.log("Bot:", msg.text);
      // Voice playback
      chat.speakText(msg.text);
    });

    $("#sendBtn").click(function(){
      chat.sendMessage($("#msgInput").val());
    });

    $("#voiceBtn").click(function(){
      chat.startVoiceCapture(function(transcript){
        console.log("You said:", transcript);
      });
    });

    chat.createActionItem({ text: "Follow up with client" }, function(err,res){
      console.log("Action item created:", res);
    });

    /** POST-MEETING **/
    chat.getSummary(function(err,res){ console.log("Summary:", res); });

    chat.getActionItems(function(err,res){ console.log("Action items:", res); });

    chat.downloadFile({ fileId: "doc123" }, function(err,res){
      console.log("Download link:", res);
    });
  });
});
</script>
```