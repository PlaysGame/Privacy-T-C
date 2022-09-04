# How Disappearing messages are implemented in Android CometChat

### I am reading the disappearing time from 3 places. This will ensure that the time is never lost and is always in sync with the other user.
1) Conversation
2) In realtime, receibed message.
3) Local storage

### Please find the implementation below:

1) Whenever the user opens the message list screen, I am fetching the conversation object for that user. Then I am reading the tag in the conversation object, if there is a tag for disappearing message, I am saving it in the local storage.
   ```java
    private void getConversationTime(Integer time) {
        CometChat.getConversation(Id, CometChatConstants.CONVERSATION_TYPE_USER, new CometChat.CallbackListener<Conversation>() {
            @Override
            public void onSuccess(Conversation conversation) {
                mConversation = conversation;

                if (time != null) {
                    setConversationTime(conversation, time);
                } else {
                    List<String> tags = conversation.getTags();
                    if (tags != null) {
                        for (String tag: tags) {
                            if (tag.startsWith("disappear_time")) {
                                String[] split = tag.split(":");
                                int time = Integer.parseInt(split[1]);
                                saveDisappearingMsgTime(time);
                            }
                        }
                    }
                }
            }

            @Override
            public void onError(CometChatException e) {

            }
        });
    }
   ```

2) There are 4 times that I am keeping. All times sent or received are in hours. If the time value is 0, it means we need to disable the disappearing messages.

3) When the user selects a disappearing message time from the info screen, I am doing the below steps: 
    a) Sending a text message with metadata as:
      
     ```java
      metadata.put("disappear", true);
      metadata.put("disappear_time", time);
     ```
     
      The text message content is: 
      ```java
        if (time == 0) {
            message = "Disappearing messages is off";
        } else {
            message = "Messages will disappear after " + CometChatUserDetailScreenActivity.getDisTime(time);
        }
      ```
    b) Saving this time in local storage
    c) Updating the time in the conversation tag. The tags of a conversation is `List<String>`. So I am adding the time to the list in this format: `"disappear_time:1"`
    
    Below is the code for the same:
    ```java
      private void setConversationTime(Conversation conversation, int time) {
        if (conversation != null) {
            List<String> tags = conversation.getTags();
            if (tags == null) {
                tags = new ArrayList<>();
            }
            int deleteTagIndex = -1;
            for (String tag: tags) {
               if (tag.startsWith("disappear_time")) {
                   deleteTagIndex = tags.indexOf(tag);
               }
            }
            if (deleteTagIndex != -1) {
                tags.remove(deleteTagIndex);
            }
            String newTag = "disappear_time:" + time;
            tags.add(newTag);
            saveDisappearingMsgTime(time);
            CometChat.tagConversation(Id, CometChatConstants.CONVERSATION_TYPE_USER, tags, new CometChat.CallbackListener<Conversation>() {
                @Override
                public void onSuccess(Conversation conversation) {
                    mConversation = conversation;
                }

                @Override
                public void onError(CometChatException e) {

                }
            });
        } else {
            getConversationTime(time);
        }
    }
    ```
    
  4) When the user is on the message screen and the disappearing message time iis changed by the other user, I am intercepting the received messages on the screen and parsing the time from there:
     ```java
        if (message.getMetadata() != null && message.getMetadata().has("disappear")) {
            try {
                boolean disappear = message.getMetadata().getBoolean("disappear");
                if (disappear) {
                    saveDisappearingMsgTime(message.getMetadata().getInt("disappear_time"));
                }
            } catch (JSONException e) {
                e.printStackTrace();
            }
        }
     ```


Note: the time passed everywhere is in hours.

