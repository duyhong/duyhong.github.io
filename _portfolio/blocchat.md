---
layout: post
title: BlocChat
thumbnail-path: "img/blocchat.png"
short-description: BlocChat is a chat aplication that sends and receives messages in real time.

---

{:.center}
![]({{ site.baseurl }}/img/blocchat.png)

## Explanation

Nowadays, chat application becomes popular because it fully satisfies real needs in human societies such as chatting, sharing information, team work, communication, and so on.

## Problem

These needs requires the chat application to have features such as listing available chat rooms, creating chat rooms, listing messages, setting username to display in chat rooms, and sending messages.

## Solution

In order to store and sync chatrooms their messages between users in realtime, I use Firebase Realtime Database, a cloud-hosted NoSQL database.

{:.center}
![]({{ site.baseurl }}/img/firebase.PNG)

I created an Angular factory, named `Room`, in which I defined all Room-related API queries, to query a list of Rooms from Firebase.

{% highlight javascript %}
(function() {
    function Room($firebaseArray) {
        var Room = {};
        var ref = firebase.database().ref().child("rooms");
        var rooms = $firebaseArray(ref);
        
        Room.all = rooms;
        
        Room.add = function(roomName) {
            //Use the firebase method $add here
            rooms.$add({"name": roomName});
        };
        
        Room.getRoomName = function(roomId) {
            return $firebaseArray(ref.orderByChild($id).equalTo(roomId));
        } 
            
        return Room;
    }
    
    angular
        .module('blocChat')
        .factory('Room',['$firebaseArray', Room]);
 })();
{% endhighlight %}

In order to display my queried rooms in the view, I created a controller, `HomeCtrl`, and associated it with the home template. Moreover, I injected the `Room` factory into `HomeCtrl` so that the `Room` factory's queried list of rooms can be assigned to the `HomeCtrl`'s `this.rooms` property.

{% highlight javascript %}
(function() {
    function HomeCtrl(Room, $uibModal, Message) {
        this.rooms = Room.all;
        this.open = function() {
            $uibModal.open({
              animation: true,
              templateUrl: '/templates/addRoom.html',
              controller: 'ModalCtrl as modal',
              backdrop: 'static'
            });
        };
        
        this.getMessages = function (roomId, roomName) {
            this.messages = Message.getByRoomId(roomName);
            this.activeRoom = roomName;
        };
        
        this.submitMsg = function(typedMsg, activeRoom) {
            this.typedMsg = typedMsg;
            
            Message.send(typedMsg, activeRoom);  
            this.typedMsg = null;
        };
    }

    //main controller
    angular
        .module('blocChat')
        .controller('HomeCtrl', [ 'Room', '$uibModal', 'Message', HomeCtrl]);
    
})();
{% endhighlight %}

By this way, the rooms can be displayed in the template using ng-repeat.

{% highlight javascript %}
<div ng-repeat = "room in home.rooms track by $index" ng-click = "home.getMessages(room.$id, room.name)">
            {{room.name}}
</div>
{% endhighlight %}

In order to create a new chat room, in the home template, I created a button and used `ngClick` to call `open` method in `HomeCtrl`.

{% highlight javascript %}
<button class="new-room-button" ng-click="home.open()">New room</button>
{% endhighlight %}

Inside the `open` method, I defined a method for toggling a modal on the frontend by using UI Bootstrap's `$uibModal` service.

{% highlight javascript %}
this.open = function() {
            $uibModal.open({
              animation: true,
              templateUrl: '/templates/addRoom.html',
              controller: 'ModalCtrl as modal',
              backdrop: 'static'
            });
        };
{% endhighlight %}

Next, I created `addRoom.html` template and a separate controller, `ModalCtrl`, for the modal.

{:.center}
![]({{ site.baseurl }}/img/addRoom.PNG)

In`addRoom.html` template, I used `ngClick` to call `ModalCtrl`'s **ok** method.

{% highlight javascript %}
<button type="button" ng-click="modal.ok()">Create room</button>
{% endhighlight %}

`modal` is the alias of `ModalCtrl`:

{% highlight javascript %}
(function() {
    angular
        .module('blocChat')
        .controller('ModalCtrl', ['Room', '$uibModalInstance', '$cookies', function(Room, $uibModalInstance, $cookies) {
            ...
            
            this.ok = function() {
                Room.add(this.roomName);
                alert(this.roomName + ' is created');
                $uibModalInstance.close(this.roomName);
            };
            
            ...
        
            }
        }]);
})();
{% endhighlight %}

In order to see a list of messages in each chat room, I associated messages with a room so that only an active room's messages show when I've selected it by clicking on the name of the room in the sidebar.

{:.center}
![]({{ site.baseurl }}/img/messages.png)

Each sent message should be an object with four properties:

{% highlight javascript %}
{
    username: "<USERNAME HERE>",
    content: "<CONTENT OF THE MESSAGE HERE>",
    sentAt: "<TIME MESSAGE WAS SENT HERE>",
    roomId: "<ROOM UID HERE>"
}
{% endhighlight %}

The `roomId` property references the room where the message was sent. This `roomId` is generated every time a message object saves to Firebase.

A `Message` factory in Angular was created to define all Message-related API queries:

{% highlight javascript %}
(function() {
  function Message($firebaseArray, $cookies, $filter) {
    var Message = {};
    var ref = firebase.database().ref().child("messages");
    var messages = $firebaseArray(ref);
      
    Message.getByRoomId = function(roomName) {
        
        // Filter the messages by their room ID.
        return $firebaseArray(ref.orderByChild('roomID').equalTo(roomName));
    };
      
    Message.send = function(newMessage, activeRoom) {
        
        var username = $cookies.get('blocChatCurrentUser');
        var date = new Date();
        var time = $filter('date')(new Date(), 'hh:mm:ss a');
        
        Message.typedMsg = newMessage;
        
        messages.$add({"username": username, "roomID": activeRoom, "content": newMessage, "sentAt": time});
    };
      
    return Message;
  }

  angular
    .module('blocChat')
    .factory('Message', ['$firebaseArray', '$cookies', '$filter', Message]);
})();
{% endhighlight %}

The `getByRoomId` method is to filter the messages by their room ID.

Messages will be associated with a username in a chat room. In `Message` factory, the `send` method takes a message object as an argument and submits it to your Firebase server.

In order to efficiently store and display a **username** in chat rooms, I used cookies. Each user is required to enter a username when he or she visits the chat application for the first time. This can be handled by Angular's `.run()` method whose code runs when the app instance is created:

{% highlight javascript %}
(function() {
    function BlocChatCookies($cookies, $uibModal) {

        var currentUser = $cookies.get('blocChatCurrentUser');

        if (!currentUser || currentUser === '') {
          // Do something to allow users to set their username
            $uibModal.open({
              animation: true,
              templateUrl: '/templates/setUsername.html',
              controller: 'ModalCtrl as modal',
              backdrop: 'static'
            });
        }
    }
    
    angular
        .module('blocChat')
        .run(['$cookies', '$uibModal', BlocChatCookies]);
})();
{% endhighlight %}

If a username is not present, another UI Bootstrap modal is triggered to require the user to enter one. 

{:.center}
![]({{ site.baseurl }}/img/username.PNG)

## Results

Now we have a nice chat app updating messages in real time.

{:.center}
![]({{ site.baseurl }}/img/blocchat.png)

## Conclusion

This chat application satisfies human needs in a modern society such as chatting, sharing information, and communication.