<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">{{params.title}}</div>
    <div slot="body">
      <dialog-selected-tips :count="params.data.length" action="部署" name="节点" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 2)" />
      <a-divider orientation="left">指定系统用户</a-divider>
      <a-form
        :form="form.fc">
        <a-form-item label="项目" v-bind="formItemLayout">
          <a-input v-decorator="decorators.proj" placeholder="字母开头，数字和字母大小写组合，长度为2-128个字符，不含'.','_','@'" />
        </a-form-item>
        <a-form-item v-bind="formItemLayout">
          <template slot="label">
            <span>
              系统用户
              <a-tooltip placement="topLeft">
                <template slot="title">
                  <div>需要使用系统管理员用户，如需创建，请参照一下命令
                  <br />climc user-create lbagent --password XXX --system-account
                  <br />climc project-add-user system lbagent admin</div>
                </template>
                <a-icon type="info-circle" />
              </a-tooltip>
            </span>
          </template>
          <base-select
            v-decorator="decorators.user"
            resource="users"
            version="v1"
            :params="userParams"
            :mapper="userMapper"
            :label-format="labelFormat"
            :select-props="{ placeholder: '需要选择有管理员权限的用户' }"
            style="width: 320px" />
        </a-form-item>
        <a-form-item label="密码" v-bind="formItemLayout">
          <a-input v-decorator="decorators.pass" type="password" placeholder="请输入系统用户的密码" />
        </a-form-item>
        <a-divider orientation="left">指定目标主机</a-divider>
        <a-form-item v-bind="formItemLayout">
          <template slot="label">
            <span>
              部署类型
              <a-tooltip placement="topLeft">
                <template slot="title">
                  <div>
                    支持3种部署类型：
                    <br />1.OneCloud平台下，与LB集群同一个可用区下的云主机
                    <br />2.OneCloud平台下，与LB集群同一个可用区下的宿主机
                    <br />3.非OneCloud平台下的其他主机，需要满足以下条件：
                    <br />AnsibleServer能够访问目标机器
                    <br />目标机器存在cloudroot用户
                    <br />目标机器允许管理员pubkey免密登录
                    <br />目标机器的cloudroot用户支持sudo免密
                  </div>
                </template>
                <a-icon type="info-circle" />
              </a-tooltip>
            </span>
          </template>
          <a-radio-group v-decorator="decorators.hostName" @change="hostChange">
            <a-radio-button v-for="(item, index) in nameServers" :value="item.value" :key="index">{{item.label}}</a-radio-button>
          </a-radio-group>
          <a-form-item v-if="this.hostName === ''">
            <a-input v-decorator="decorators.ip" placeholder="请输入外部机器的IP" />
          </a-form-item>
          <a-form-item v-if="this.hostName === 'server'">
            <base-select
              v-decorator="decorators.server"
              resource="servers"
              :params="{}"
              :label-format="labelFormat"
              :select-props="{ placeholder: '请选择云主机' }"
              style="width: 320px" />
          </a-form-item>
          <a-form-item v-if="this.hostName === 'host'">
            <base-select
              v-decorator="decorators.host"
              resource="hosts"
              :params="{}"
              :label-format="labelFormat"
              :select-props="{ placeholder: '请选择宿主机' }"
              style="width: 320px" />
          </a-form-item>
        </a-form-item>
        <a-form-item v-bind="formItemLayout">
          <template slot="label">
            <span>
              部署方式
              <a-tooltip placement="topLeft">
                <template slot="title">
                  <div>
                    支持2种部署方式：
                    <br />1.yum部署，需要输入Yum源地址，一般为：https://控制节点IP/yumrepo
                    <br />2.copy部署，直接从安装目录拷贝.rpm文件到目标机器
                  </div>
                </template>
                <a-icon type="info-circle" />
              </a-tooltip>
            </span>
          </template>
          <a-radio-group v-decorator="decorators.deploy_method" @change="deployMethodChange">
            <a-radio-button value="yum">Yum</a-radio-button>
            <a-radio-button value="copy">Copy</a-radio-button>
          </a-radio-group>
        </a-form-item>
        <a-form-item label="Yum源地址" v-bind="formItemLayout" v-if="deployMethod === 'yum'">
          <a-input v-decorator="decorators.repo_base_url" placeholder="请输入Yum源地址，一般为：https://控制节点IP/yumrepo" />
        </a-form-item>
        <a-form-item v-bind="tailFormItemLayout" v-if="deployMethod === 'yum'">
          <a-checkbox v-decorator="decorators.repo_sslverify">Yum源TLS校验</a-checkbox>
        </a-form-item>
      </a-form>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import { mapGetters } from 'vuex'
import Ansible from '../controls/ansible'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'AgentDeployDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      isRunning: false,
      form: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        proj: [
          'proj',
          {
            validateTrigger: ['blur'],
            initialValue: 'system',
            rules: [
              { required: true, message: '请输入项目' },
            ],
          },
        ],
        user: [
          'user',
          {
            validateTrigger: ['blur'],
            rules: [
              { required: true, message: '请选择系统管理员用户' },
            ],
          },
        ],
        pass: [
          'pass',
          {
            validateTrigger: ['blur'],
            rules: [
              { required: true, message: '请输入系统用户的密码' },
            ],
          },
        ],
        hostName: [
          'hostName',
        ],
        ip: [
          'ip',
          {
            validateTrigger: ['blur'],
            rules: [
              { required: true, message: '请输入外部机器的IP' },
            ],
          },
        ],
        host: [
          'host',
          {
            validateFirst: true,
            rules: [
              { required: true, message: '请选择宿主机' },
            ],
          },
        ],
        server: [
          'server',
          {
            validateFirst: true,
            validateTrigger: ['blur'],
            rules: [
              { required: true, message: '请选择云主机' },
              { validator: this.serverOldCheck },
            ],
          },
        ],
        deploy_method: [
          'deploy_method',
        ],
        repo_base_url: [
          'repo_base_url',
          {
            validateFirst: true,
            validateTrigger: ['blur'],
            initialValue: `${location.protocol}//${location.host}/yumrepo`,
            rules: [
              { required: true, message: '请输入Yum源地址' },
              { validator: this.$validate('url') },
            ],
          },
        ],
        repo_sslverify: [
          'repo_sslverify',
          {
            valuePropName: 'checked',
          },
        ],
      },
      formItemLayout: {
        wrapperCol: {
          span: 20,
        },
        labelCol: {
          span: 4,
        },
      },
      tailFormItemLayout: {
        wrapperCol: {
          sm: {
            span: 20,
            offset: 4,
          },
        },
      },
      userParams: {
        system: true,
      },
      nameServers: [
        { label: '云主机', value: 'server' },
        { label: '宿主机', value: 'host' },
        { label: '外部机器', value: '' },
      ],
      hostName: '',
      deploymentHost: '',
      deployMethod: '',
      ansiblePlaybookId: '',
    }
  },
  computed: {
    ...mapGetters(['isAdminMode', 'scope', 'userInfo']),
  },
  created () {
    this.backfill()
  },
  methods: {
    userMapper (data) {
      data = data.filter(item => item.is_system_account)
      return data
    },
    labelFormat (item) {
      if (item.ips) {
        return `${item.name}(${item.ips})`
      }
      return item.name
    },
    hostChange (e) {
      this.hostName = e.target.value
    },
    deployMethodChange (e) {
      this.deployMethod = e.target.value
    },
    // 更改云主机时与旧的云主机校验
    serverOldCheck (rule, value, callback) {
      if (value && this.deploymentHost) {
        const { hostName } = this.deploymentHost
        const formHost = this.form.fc.getFieldValue('hostName')
        if (formHost !== this.deploymentHost[hostName]) {
          return callback(new Error('更换目标机器，需要提前先将节点从旧机器下线'))
        }
      }
      return callback()
    },
    backfill () {
      const { deployment } = this.params.data[0] || {}
      if (deployment && deployment.ansible_playbook) {
        this.ansiblePlaybookId = deployment.ansible_playbook
        new Ansible(deployment.ansible_playbook)
          .get()
          .then(({ data }) => {
            if (data && data.playbook && data.playbook.inventory && data.playbook.inventory.hosts && data.playbook.inventory.hosts.length > 0) {
              const { vars } = data.playbook.inventory.hosts[0] || {}
              if (vars) {
                this.form.fc.setFieldsValue({
                  user: vars.user,
                  pass: vars.pass,
                  deploy_method: vars.repo_base_url ? 'yum' : 'copy',
                })
                this.form.fc.getFieldDecorator('repo_base_url', {
                  preserve: true,
                  initialValue: vars.repo_base_url,
                })
                this.deployMethod = vars.repo_base_url ? 'yum' : 'copy'
                this.$nextTick(() => {
                  this.form.fc.setFieldsValue({ 'repo_sslverify': !!parseInt(vars.repo_sslverify) })
                })
              }
            }
            // 是否为部署中
            this.isRunning = data.status === 'running'
          })

        /* 处理云主机start */
        let arrHost = deployment.host.split(':') || []
        if (arrHost && arrHost.length === 2) {
          const obj = {
            hostName: arrHost[0],
          }
          obj[arrHost[0]] = arrHost[1]
          this.$nextTick(() => {
            this.form.fc.setFieldsValue(obj)
          })
          this.deploymentHost = {
            hostName: arrHost[0],
          }
          this.deploymentHost[arrHost[0]] = arrHost[1]
          this.hostName = arrHost[0]
        } else {
          // 外部主机
          this.$nextTick(() => {
            this.form.fc.setFieldsValue({
              hostName: '',
              ip: arrHost[0],
            })
          })
          this.deploymentHost = {
            hostName: '',
            ip: arrHost[0],
          }
          this.hostName = ''
        }
        /* 处理云主机end */
      }
    },
    doCreate (data) {
      const { id } = this.params.data[0]
      return new this.$Manager('loadbalanceragents').performAction({
        id,
        action: 'deploy',
        data,
      })
    },
    async handleConfirm () {
      this.loading = true
      try {
        let values = await this.form.fc.validateFields()
        values['repo_sslverify'] = values['repo_sslverify'] ? 1 : 0
        const { hostName, ip, host, proj, pass, server, user } = values
        const params = {
          host: {
            name: hostName ? `${hostName}:${host}` : ip,
            vars: {
              host: host,
              pass,
              proj,
              repo_base_url: values.repo_base_url,
              repo_sslverify: values.repo_sslverify,
              server,
              user,
            },
          },
          deploy_method: values.deploy_method,
        }
        await this.doCreate(params)
        this.loading = false
        this.cancelDialog()
        this.params.list.refresh()
      } catch (error) {
        this.loading = false
      }
    },
  },
}
</script>