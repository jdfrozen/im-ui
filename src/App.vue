<template>
  <div id="app">
    <c-dialog
      ref="loginDialog"
      title='请输入你的昵称'
      confirmBtn="开始聊天"
      @confirm="login"
    >
      <input class="nickname" v-model="nickname" type="text" placeholder="请输入你的昵称">
    </c-dialog>

    <c-dialog
      ref="createGroupDialog"
      title='请输入群名称'
      confirmBtn="确认"
      @confirm="createGroup"
    >
      <input class="nickname" v-model="groupName" type="text" placeholder="请输入群名称">
    </c-dialog>

    <div class="web-im dis-flex">
      <div class="left">
        <div class="aside content">
          <div class="header">
            <div class="tabbar dis-flex">
              <label :class="{active:switchType==1, unread: usersUnRead}" for="" @click="switchType=1">联系人</label>
              <label :class="{active:switchType==2, unread: groupsUnRead}" for="" @click="switchType=2">群聊</label>
            </div>
          </div>
          <div class="body user-list">
            <div v-if="switchType==2" class="user" @click="triggerGroup(item)" v-for="item in currentGroups">
              {{item.name}}
              <span class="tips-num" v-if="item.unread">{{item.unread}}</span>
              <span v-if="!checkUserIsGroup(item)" @click.stop="addGroup(item)" class="add-group">+</span>
            </div>
            <div v-if="switchType==1 && item.uid!=uid" class="user" @click="triggerPersonal(item)" :class="{offline: !item.status}" v-for="item in currentUserList">
              {{item.nickname}}
              <span class="tips-num" v-if="item.unread">{{item.unread}}</span>
            </div>
          </div>
          <div class="footer">
            <div class="func dis-flex">
              <label @click="$refs.createGroupDialog.show()">新建群</label>
            </div>
          </div>
        </div>
      </div>
      <div class="right content">
        <div class="header im-title">{{title}}</div>
        <div class="body im-record" id="im-record">
          <div class="ul">
            <div class="li" :class="{user: item.uid == uid}" v-for="item in currentMessage">
              <template v-if="item.type===1">
                <p class="join-tips">{{item.msg}}</p>
              </template>
              <template v-else>
                <p class="message-date">
                  <span class="m-nickname">{{item.nickname}}</span> {{item.date}}</p>
                <p class="message-box">{{item.msg}}</p>
              </template>
            </div>
          </div>
        </div>
        <div class="footer im-input dis-flex">
          <input type="text" v-model="msg" placeholder="请输入内容">
          <button @click="send">发送</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import moment from 'moment'
import axios from 'axios'


export default {
  name: 'App',
  components: {
    'c-dialog': Vue.extend(require('@/components/dialog/index.vue').default)
  },
  data(){
    return {
      title: '请选择群或者人员进行聊天',
      switchType: 2,
      uid: '',
      nickname: '',
      socket: '',
      msg: '',
      messageList: [],
      users: [],
      groups: [],
      groupId: '',
      bridge: [],
      groupName: ''
    }
  },
  mounted() {
    let vm = this;
    let user = localStorage.getItem('WEB_IM_USER');
    user = user && JSON.parse(user) || {};
    vm.uid = user.uid;
    vm.nickname = user.nickname;

    if(!vm.uid){
      vm.$refs.loginDialog.show()
    } else {
      vm.conWebSocket();
    }

    document.onkeydown = function (event) {
        var e = event || window.event;
        if (e && e.keyCode == 13) { //回车键的键值为13
            vm.send()
        }
    }
    window.onbeforeunload = function (e) {
      vm.socket.send(JSON.stringify({
        uid: vm.uid,
        type: 2,
        nickname: vm.nickname,
        bridge: []
      }));
    }
  },
  computed: {
      // var jsonStr ='[{"type":"01","msg":"fenfenfff","uid":1,"status":1,"bridge":{"length":true}}]';
      // vm.messageList =  JSON.parse(jsonStr);
    currentMessage() {
      let vm = this;
      let data = vm.messageList.filter(item=>{
        if(item.type === 1) {
          return item;
        } else if(this.groupId) {
          return item.groupId === this.groupId
        } else if(item.bridge.length){
            return true;
          //return item.bridge.sort().join(',') == vm.bridge.sort().join(',')
        }
      })
      data.map(item=>{
        item.status = 0
        return item;
      })
      return data;
    },
      //实时更新过滤群列表信息
    currentGroups() {
        let vm = this;
        vm.groups =  this.groups;
        return  vm.groups
    },
    groupsUnRead(){
      return this.messageList.some(item=>{
        return item.groupId && item.status === 1
      })
    },
    usersUnRead(){
      return this.messageList.some(item=>{
        return item.bridge.length && item.status === 1
      })
    },
      //实时更新过滤用户列表
    currentUserList() {
      let vm = this;
      vm.users =  this.users;
      return  vm.users
    }
  },
  methods: {
    addGroup(item){
      this.socket.send(JSON.stringify({
        uid: this.uid,
        type: 20,
        nickname: this.nickname,
        groupId: item.id,
        groupName: item.name,
        bridge: []
      }));
      this.$message({type: 'success', message: `成功加入${item.name}群`})
    },
    checkUserIsGroup (item) {
      return item.users.some(item=>{
        return item.uid === this.uid
      })
    },
    createGroup(){
      this.groupName = this.groupName.trim();
      if(!this.groupName){
        this.$message({type: 'error', message: '请输入群名称'})
        return;
      }
      let userGroupVo = {"name":this.groupName,"adminId":this.uid,"nick":this.nickname};
      axios.post(`http://localhost:8082/group/createGroup`,userGroupVo)
        .then(res=>{
          console.log('新建群成功',userGroupVo);
        }).catch(function (res) {
        alert(res)
      });
    },
    triggerGroup(item) {
      let issome = item.users.some(item=>{
        return item.uid === this.uid
      })
      if(!issome){
        this.$message({type: 'error', message: `您还不是${item.name}群成员`})
        return
      }
      this.bridge = [];
      this.groupId = item.id;
      this.title = `和${item.name}群成员聊天`;
    },
    triggerPersonal(item) {
      if(this.uid === item.uid){
        return;
      }
      this.groupId = '';
      this.bridge = [this.uid, item.uid];
      this.title = `和${item.nickname}聊天`;
    },
      //发送聊天消息
    send(){
      this.msg = this.msg.trim();
      if(!this.msg){
        return
      }
      if(!this.bridge.length && !this.groupId){
        this.$message({type: 'error', message: '请选择发送人或者群'})
        return;
      }
      this.sendHttpMessage(this.uid,this.groupId,this.bridge,this.msg)
    },
      //调用ws进行发送消息
    sendWsMessage(str){
      this.socket.send(str);
    },
      //调用http发送聊天消息
    sendHttpMessage(userId,receiverId,type, msg){
      let msgVo = {"userId":userId,"receiverId":receiverId,"msg":msg,"type":type};
      axios.post(`http://localhost:8082/im/sendMsg`,msgVo)
        .then(res=>{
          console.log('消息发送成功',msgVo);
        }).catch(function (res) {
        alert(res)
      })
      this.msg='';
    },
      //根据昵称获取用户ID
    getUserId(nickname){
      let userVo = {"name":nickname};
      axios.post(`http://localhost:8082/user/getByName`,userVo)
        .then(function (res) {
          console.log('消息发送成功',userVo);
          let dataBody = res.data.dataBody;
          localStorage.setItem('WEB_IM_USER', JSON.stringify({
            uid: dataBody.id,
            nickname: nickname
          }))
            return dataBody.id;
        }).catch(function (res) {
        alert(res)
      })
    },
    getGroupByUser(nickname){
      let vm = this;
      let userVo = {"name":nickname};
      axios.post(`http://localhost:8082/group/getGroupByUser`,userVo)
          .then(function (res) {
              vm.groups = res.data.dataBody;
          }).catch(function (res) {
          alert(res)
      })
      },
    getRelations(nickname){
      let vm = this;
      let userVo = {"name":nickname};
      axios.post(`http://localhost:8082/user/getRelations`,userVo)
          .then(function (res) {
              vm.users = res.data.dataBody;
          }).catch(function (res) {
          alert(res)
      })
    },
    conWebSocket(){
      let vm = this;
      if(window.WebSocket){
        vm.socket = new WebSocket('ws://localhost:8899/ws');
        let socket = vm.socket;
        socket.onopen = function(e){
          console.log("连接服务器成功");
          vm.$message({type: 'success', message: '连接服务器成功'})
          //获取用户ID
          vm.getUserId(vm.nickname);
          //初始化群
          vm.getGroupByUser(vm.nickname);
          //初始化用户列表
          vm.getRelations(vm.nickname);
          //WS登录
          vm.sendWsMessage('login:'+vm.uid);
        }
        socket.onclose = function(e){
          console.log("服务器关闭");
        }
        socket.onerror = function(){
          console.log("连接出错");
        }
        // 接收服务器的消息
        socket.onmessage = function(e){
          let message = JSON.parse(e.data);
          vm.messageList.push(message);
          vm.$nextTick(function(){
            var div = document.getElementById('im-record');
            div.scrollTop = div.scrollHeight;
          })
        }
      }
    },
    login(){
      this.nickname = this.this.nickname.trim();
      if(!this.nickname){
        this.$message({type: 'error', message: '请输入您的昵称'})
        return;
      }
      this.$refs.loginDialog.hide()
      this.conWebSocket();
    }
  }
}
</script>

<style lang='stylus'>
  @import './app.styl';
</style>
