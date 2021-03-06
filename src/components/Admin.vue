<template>
  <div class="component-container expand-panel">
    <h3>Solution Admin</h3>
    <button class="btn btn-secondary btn-sm expand-button" v-on:click="fullscreen()">
      <Octicon :icon="screenNormal" v-if="isScreenExpanded"></Octicon>
      <Octicon :icon="screenFull" v-else-if="!isScreenExpanded"></Octicon>
    </button>

    <div v-if="!isLoggedIn">
      <button class="btn btn-warning btn-sm admin-login-button" v-if="!isLogin" v-on:click="login()">
        <Octicon :icon="person"></Octicon>
      </button>
      <div class="form-group component-inner-container" v-else-if="isLogin">
        <textarea class="form-control" v-model="token" placeholder="Paste token here..."></textarea>
        <button type="submit" class="btn btn-info btn-sm jwt-submit" v-on:click="verifyLogin()">Login</button>
      </div> 
    </div>
    <div class="top-div" v-if="isLoggedIn">
      <tabs
        ref="foo"
        :tabs="tabs"
        :currentTab="currentTab"
        :wrapper-class="'default-tabs'"
        :tab-class="'default-tabs__item'"
        :tab-active-class="'default-tabs__item_active'"
        :line-class="'default-tabs__active-line'"
        @onClick="tabClick" />
     
      <!-- GET SOLUTION -->
      <div class="component-inner-container" v-if="currentTab=='get-solution'">
      </div>

      <!-- PUT ORG -->
      <div class="component-inner-container" v-if="currentTab=='put-org'">
        <div class="form-row">
          <div class="col">
            <input type="text" v-model="admin.putorgorgname" class="form-control" placeholder="Organization Name">
          </div>
          <div class="col">
            <button type="button" class="btn btn-success" v-on:click="putOrgs()">Commit</button>
          </div>
        </div>
      </div>

      <!-- POST ORG ADMIN -->
      <div class="component-inner-container" v-if="currentTab=='post-org-admin'">
        <div class="form-row">
          <div class="col">
            <select class="form-control" v-model="admin.postorgadminorgid">
              <option value="" selected disabled>Select Organization</option>
              <option v-for="org in orgs">
                {{ org.name }}
              </option>
            </select>
          </div>
          <div class="col">
            <input type="text" v-model="admin.postorgadminadmin" class="form-control" placeholder="Admin Email-Id">
          </div>
          <div class="col">
            <button type="button" class="btn btn-success" v-on:click="postOrgAdmin()">Commit</button>
          </div>
        </div>
      </div>
      
      <!-- DELETE ORG ADMIN -->
      <div class="component-inner-container" v-if="currentTab=='delete-org-admin'">
        <div class="form-row">
          <div class="col">
            <select class="form-control" v-model="admin.delorgadminorgid">
              <option value="" selected disabled>Select Organization</option>
              <option v-for="org in orgs">
                {{ org.name }}
              </option>
            </select>
          </div>
          <div class="col">
            <input type="text" v-model="admin.delorgadminadmin" class="form-control" placeholder="Admin Email-Id">
          </div>
          <div class="col">
            <button type="button" class="btn btn-success" v-on:click="deleteOrgAdmin()">Commit</button>
          </div>
        </div>
      </div>
    </div>

    <div class="component-shell-container" v-if="isLoggedIn">
      <tree-view :data="response" />
    </div>
  </div>
</template>

<script>
import config from '@/secrets/config.json'
import Api from '@/apis/AdminApi'
import Tabs from 'vue-tabs-with-active-line'
import VueJwtDecode from 'vue-jwt-decode'
import Octicon, { screenFull, screenNormal, person } from 'octicons-vue'
import { serverBus } from '@/main'

const TABS = [
  { title: 'View Solution', value: 'get-solution'},
  { title: 'Add Hospital', value: 'put-org'},
  { title: 'Add Hospital Admin', value: 'post-org-admin'},
  { title: 'Delete Hospital Admin', value: 'delete-org-admin'},
]

export default {
  name: 'Admin',
  components: {
    Octicon,
    Tabs
  },
  data: () => ({
    screenFull, screenNormal, person,
    response: {},
    orgs: [],
    docs: [],
    tabs: TABS,
    currentTab: 'get-solution',
    admin: {},
    isScreenExpanded: false,
    isLoggedIn: false,
    isLogin: false,
    token: '',
    solutionId: '',
    jwt: {}
  }),
  created () {
    serverBus.$on('triggerGetOrgs', (orgId) => {
      this.searchAllUsersForOrgId(orgId)
    })
  },
  mounted () {
  },
  methods: {
    tabClick (newTab) {
      this.currentTab = newTab
      this.response = {}
      this.admin = {}

      if (this.currentTab == 'get-solution')
        this.getSolutionById()
    },

    setRequestProcessing () {
      this.response = {
        status: "Request processing..."
      }
    },

    fullscreen () {
      if (this.isScreenExpanded) {
        document.getElementsByClassName('expand-panel')[0].style.height = '50vh'
        document.getElementsByClassName('expand-panel')[0].style.width = '50vw'
      } else {
        document.getElementsByClassName('expand-panel')[0].style.height = '100vh'
        document.getElementsByClassName('expand-panel')[0].style.width = '100vw'
      }
      this.isScreenExpanded = !this.isScreenExpanded

      setTimeout(() => { 
        this.currentTab = 'delete-org-admin'
        setTimeout(() => {
          this.currentTab = 'get-solution'
        }, 1)
      }, 501)
    },

    async getSolutionById () {
      var solId = this.solutionId
      
      if (solId) {
        this.setRequestProcessing()

        const apiResponse = await Api.getSolutionById(solId)
        this.response = apiResponse.data
      }
    },

    async putOrgs () {
      var orgName = this.admin.putorgorgname
      var solId = this.solutionId
      if (orgName && solId) {
        this.setRequestProcessing()
        
        const apiResponse = await Api.putOrgs({
          name: orgName,
          solutionId: solId
        })
        this.response = apiResponse.data
        this.searchAllOrgs()
      }
    },

    async searchAllOrgs () {
      var solId = this.solutionId
      if (solId) {
        const apiResponse = await Api.searchAllOrgs({
          solutionId: solId
        })
        this.orgs = apiResponse.data.response
        serverBus.$emit('allOrgs', this.orgs)
        this.searchAllUsers()
      }
    },

    searchAllUsers () {
      for (var org of this.orgs) {
        this.searchAllUsersForOrgId(org.id)
      }
      serverBus.$emit('allOrgs', this.orgs)
    },

    async searchAllUsersForOrgId (orgId) {
      var solId = this.solutionId
      if (solId) {
        const apiResponse = await Api.searchAllUsersForOrgId({
          solutionId: solId,
          organizationId: orgId,
          count: false
        })

        var orgIndex = null
        for (var index in this.orgs) {
          if (this.orgs[index].id == orgId) {
            orgIndex = index
            break
          }
        }

        if (apiResponse.data.response) {
          this.$set(this.orgs[orgIndex], 'users', apiResponse.data.response)
        } else {
          this.$set(this.orgs[orgIndex], 'users', [])
        }
      }
    },

    async postOrgAdmin () {
      var solId = this.solutionId
      var admin = this.admin.postorgadminadmin
      var orgName = this.admin.postorgadminorgid
      
      var orgId = false
      // Get orgId from orgName
      for (var org of this.orgs) {
        if (org.name == orgName) {
          orgId = org.id
          break
        }
      }

      if (solId && admin && orgId) {
        this.setRequestProcessing()
        
        const apiResponse = await Api.postOrgAdmin({
          solutionId: solId,
          organizationId: orgId,
          adminEmailId: admin
        })
        this.response = apiResponse.data
        this.searchAllUsersForOrgId(orgId)
      }
    },
    
    async deleteOrgAdmin () {
      var solId = this.solutionId
      var admin = this.admin.delorgadminadmin
      var orgName = this.admin.delorgadminorgid
      
      var orgId = false
      // Get orgId from orgName
      for (var org of this.orgs) {
        if (org.name == orgName) {
          orgId = org.id
          break
        }
      }

      var adminId = false
      for (var org of this.orgs) {
        if (org.name == orgName) {
          for (var user of org.users) {
            if (user.userId == admin) {
              adminId = user.uid
              break
            }
          }
        }
      }

      if (solId && adminId && orgId) {
        this.setRequestProcessing()
        
        const apiResponse = await Api.deleteOrgAdmin({
          solutionId: solId,
          organizationId: orgId,
          adminId: adminId
        })
        this.response = apiResponse.data
        this.searchAllUsersForOrgId(orgId)
      } else {
        this.response = {
          "status": "Provided admin doesn't exist in Org"
        }
      }
    },

    async getAllRoles () {
      var solId = this.solutionId
      
      if (solId) {
        const apiResponse = await Api.getSolutionRoles({
          solutionId: solId
        })
        serverBus.$emit('allRoles', apiResponse.data.response)
      }
    },

    login () {
      this.isLogin = !this.isLogin
      if (this.isLogin) {
        window.open(config.iss + '/onboarding/v1/logins', 'loginwindow', 'height=435,width=962')
      }
    },

    verifyLogin () {
      if (!this.token) {
        this.token = ''
        return this.loginFail()
      }

      return this.loginSuccess()
    },

    loginSuccess () {
      sessionStorage[`admin-token`] = this.token
      this.jwt = VueJwtDecode.decode(this.token)
      this.token = ''
      this.solutionId = this.jwt.sid
      console.log("login success")
      this.isLoggedIn = !this.isLoggedIn
      this.getSolutionById()
      this.searchAllOrgs()
      this.getAllRoles()
    },

    loginFail () {
      console.log("login fail")
    }
  }
}

</script>                                                                  
                                                                            
<!-- Add "scoped" attribute to limit CSS to this component only -->          
<style scoped>                           

.component-container {
  position: absolute;
  left: 0;
  top: 0;
}

.component-shell-container {
  height: calc(100% - 158px);
}

.expand-button {
  position: absolute;
  top: 10px;
  left: 10px;
}

.jwt-submit {
  margin-top: 10px;
}

.admin-login-button {
  position: relative;
  top: 50px;
}

textarea {
  height: calc(50vh - 160px) !important;
  background-color: rgb(73, 73, 73) !important;
  color: white !important;
}

</style>
