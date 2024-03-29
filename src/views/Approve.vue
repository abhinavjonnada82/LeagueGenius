<template>
     <Menu />
   <img alt="logo" src="../assets/logo.png">
            <h1><b>Admin Dashboard</b></h1>
            <a-spin v-if="loading" size="large"></a-spin>
            <a-table bordered :data-source="dataSource" :columns="columns" :key="componentKey">
              <template #approve="{ record }">
                  <a-button @click="onApprove(record.key)">Approve</a-button>
              </template>
            </a-table>
</template>
<script>

import { defineComponent, ref, onMounted } from 'vue';
import { message } from 'ant-design-vue';
import firebase from "firebase/app"
import 'firebase/auth';
import Menu from '../components/Menu.vue';
import { baseAPI } from '../utils';

export default defineComponent({
  components: {
    Menu
  },

  setup() {
    let data = []
    const auth = firebase.auth();
    const approveStatus = ref(``);
    const dataSource = ref();
    const loading = ref(true);
    const columns = [
           {
            title: '',
            dataIndex: 'key',
            key: 'key',
          },
          {
            title: 'Team Name',
            dataIndex: 'tname',
            key: 'tname',
          },
          {
            title: 'Captain Name',
            dataIndex: 'captainName',
            key: 'captainName',
          },
          {
            title: '# of Players',
            dataIndex: 'numPlayers',
            key: 'numPlayers',
          },
          {
            title: 'Approve',
            dataIndex: 'approve',
            slots: {
                customRender: 'approve',
             },
          },
        ];

    onMounted(() =>  {
        getToken()
    })

    const getToken = async () => {
      const user = auth.currentUser;
      if (user) {
        const token = await user.getIdToken();
        const res = await getAllTeamInfo(token);
        appendDataTable(res.data)
      } else {
        auth.signOut().then(() => console.log('Signed out') )
        .catch(err => alert(err.message))
      }
  }

    const onDelete = key => {
      dataSource.value = dataSource.value.filter(item => item.key !== key);
    };
    
    const onApprove = key => {
        dataSource.value.filter(async item => {
            if (item.key === key )
                setTeamApproval(item)
                return
            })
    };

    const setTeamApproval = async (item) => {
      let token = ``
      let response = ``
      auth.onAuthStateChanged(async (user) => {
      if (user) {
        token = await user.getIdToken();
        const rosterPayload = {'userId': item.id, 'phone': item.phone}
        loading.value = true
        response = await initTeamApproval(token, rosterPayload);
        if (response.code == 200) {
          onDelete(item.key)
          loading.value = false
        }
      } else {
        console.log('error')
      }}) 
    }

    const getAllTeamInfo = async (token) => {
      const response = await fetch(`${baseAPI}teamData?api=getData&type=allTeams&PIN=${localStorage.getItem('PIN')}`, {
        method:'GET',
        headers:{
            Authorization:"Bearer "+token,
            "Content-Type": "application/json"
            }
      })
    
      if (response.status === 200) {
        return response.json()
        }
      else { 
            message.error({
                          content: `ERROR`,
                          duration: 2,
              }); 
          }
      }

    const initTeamApproval = async (token, rosterPayload) => {
        const response = await fetch(`${baseAPI}teamData?api=approveTeam`, {
        method:'POST',
        body: JSON.stringify(rosterPayload),
        headers:{
            Authorization:"Bearer "+token,
            "Content-Type": "application/json"
            }
      })
    
      if (response.status === 200) {
        approveStatus.value = true
             message.success({
              content: 'Approved!',
              duration: 2,
            }); 
        return response.json()
        }
      else { 
            message.error({
                          content: `ERROR`,
                          duration: 2,
              }); 
          }
    }

    const appendDataTable = (records) => {
      records && records.forEach((record, i) => {
           record.data && data.push({
            key: i+1,
            tname: record.data.teamName,
            captainName: record.data.capFullName,
            numPlayers: record.data.players.length,
            id: record.uid,
            phone: record.data.phone
    });
      });
      loading.value = false
      forceRerender()
        
    }

    dataSource.value = (data);

    const componentKey = ref(0);
    const forceRerender = () => {
          componentKey.value += 1;
    };

    return {
      columns,
      onDelete,
      componentKey,
      dataSource,
      loading,
      onApprove
    };
  },
});
</script>

<style>
img {
  width: 250px;
  height: 200px;
  display: block;
  margin-left: auto;
  margin-right: auto;
}
</style>