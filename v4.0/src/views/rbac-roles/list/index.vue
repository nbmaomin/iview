<template>
  <div>
    <CList
      :columns="cList.columns"
      :data="list.items"
      :total="list.total"
      :pageCurrent="listPageCurrent"
      @selection-change="handleListSelectionChange">
      <CListHeader>
        <CListOperations>
          <Button
            type="primary"
            @click="handleShowForm">
            新增
          </Button>
          <CBatchDel
            :selected-items="listSelectedItems"
            @ok="handleDelOk" />
        </CListOperations>
      </CListHeader>
    </CList>
    <Modal
      width="496"
      v-model="cForm.modal"
      :title="cForm.id ? '编辑' : '新增'">
      <Form
        ref="formValidate"
        :model="cForm.formValidate"
        :rules="cForm.ruleValidate"
        :label-width="80">
        <Form-item
          label="名称"
          prop="name">
          <Row>
            <Col span="20">
              <Input
                v-model="cForm.formValidate.name"
                placeholder="请输入名称" />
            </Col>
          </Row>
        </Form-item>
        <Form-item
          label="权限"
          prop="permissions">
          <Row>
            <Col span="20">
              <template v-if="rbacResourcesList.items && rbacResourcesList.items.length">
                <div
                  v-for="resource in rbacResourcesList.items"
                  :key="resource.id">
                  {{ resource.name }}：
                  <Checkbox
                    v-for="action in Object.keys($consts.REQUEST_METHODS)"
                    :key="action"
                    :value="cForm.formValidate.permissions[resource.code] && cForm.formValidate.permissions[resource.code].indexOf(action) !== -1"
                    @on-change="checked => { handleRbacActionsChange(resource.code, action)(checked) }">
                    {{ $consts.REQUEST_METHODS[action] }}
                  </Checkbox>
                </div>
              </template>
              <template v-else>
                暂无权限数据，请先
                <Button @click="$router.push('/rbac/rbac/resources')">添加资源</Button>
              </template>
            </Col>
          </Row>
        </Form-item>
        <Form-item
          label="描述"
          prop="description">
          <Row>
            <Col span="20">
              <Input
                v-model="cForm.formValidate.description"
                type="textarea"
                :rows="3"
                placeholder="请输入描述" />
            </Col>
          </Row>
        </Form-item>
      </Form>
      <div slot="footer">
        <Button
          type="text"
          size="large"
          @click="cForm.modal = false">
          取消
        </Button>
        <Button
          type="primary"
          size="large"
          @click="handleFormOk">
          确定
        </Button>
      </div>
    </Modal>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import routeParamsMixin from '@/mixins/route-params'
import listMixin from '@/mixins/list'
import formMixin from '@/mixins/form'

const module = 'rbacRoles'
const initForm = {
  permissions: {}
}

export default {
  mixins: [
    routeParamsMixin,
    listMixin,
    formMixin
  ],
  data () {
    const { LIST_COLUMN_WIDTHS } = this.$consts

    return {
      cList: {
        columns: [
          {
            type: 'selection',
            width: 60,
            align: 'center'
          },
          {
            title: '名称',
            key: 'name',
            width: LIST_COLUMN_WIDTHS.TITLE
          },
          {
            title: '权限',
            width: 300,
            render: (h, params) => {
              const { items } = this.rbacResourcesList
              const { permissions } = params.row
              const { REQUEST_METHODS } = this.$consts

              return h(
                'span',
                null,
                Object.keys(permissions || {}).map(resourceCode => {
                  const resource = this.$helpers.getItem(items, 'code', resourceCode)
                  return h(
                    'div',
                    null,
                    resource.name + '：' + permissions[resourceCode].map(action => REQUEST_METHODS[action]).join('、'))
                })
              )
            }
          },
          {
            title: '描述',
            render: (h, params) => h('span', null, params.row.description)
          },
          {
            title: '操作',
            key: 'action',
            width: 170,
            render: (h, params) => h('div', [
              h('Button', {
                on: {
                  click: () => {
                    this.handleShowForm(params.row)
                  }
                }
              }, '编辑'),
              h('CDel', {
                on: {
                  ok: () => {
                    this.handleDelOk(params.row.id)
                  }
                }
              }, '删除')
            ])
          }
        ]
      },
      cForm: {
        id: 0,
        modal: false,
        formValidate: this.$helpers.deepCopy(initForm),
        ruleValidate: {
          name: [
            {
              required: true,
              message: '名称不能为空'
            }
          ]
        }
      }
    }
  },
  computed: mapState({
    list: state => state[module].list,
    rbacResourcesList: state => state.rbacResources.list
  }),
  watch: {
    'cForm.modal': {
      handler (newVal) {
        !newVal && this.resetFields(initForm)
      }
    }
  },
  async beforeRouteUpdate (to, from, next) {
    await this.getRbacRolesList()
    await this.getList()
    next()
  },
  async created () {
    await this.getRbacRolesList()
    await this.getList()
  },
  methods: {
    getList () {
      return this.$store.dispatch(`${module}/getList`, {
        query: {
          offset: (this.listPageCurrent - 1) * this.$consts.PAGE_SIZE,
          limit: this.$consts.PAGE_SIZE,
          where: { alias: this.alias }
        }
      })
    },
    getRbacRolesList () {
      return this.$store.dispatch('rbacResources/getList', {
        query: { offset: 0, limit: -1 }
      })
    },
    handleShowForm (detail) {
      this.cForm.modal = true

      if (detail.id) {
        this.cForm.id = detail.id
        this.initFields(detail)
      } else {
        this.cForm.id = 0
      }
    },
    async handleDelOk (id) {
      await this.$store.dispatch(`${module}/del`, { id })
      this.$Message.success('删除成功！')

      const getListRes = await this.getList()
      !getListRes.items.length && this.goPrevPage()
    },
    handleFormOk () {
      this.$refs.formValidate.validate(async valid => {
        if (valid) {
          await this.$store.dispatch(this.cForm.id ? `${module}/put` : `${module}/post`, {
            id: this.cForm.id,
            body: {
              ...this.cForm.formValidate,
              alias: this.alias
            }
          })

          this.cForm.modal = false
          this.$Message.success((this.cForm.id ? '编辑' : '新增') + '成功！')
          !this.cForm.id && this.resetSearch()
          this.getList()
        }
      })
    },
    handleRbacActionsChange (resourceCode, action) {
      return checked => {
        const { permissions } = this.cForm.formValidate

        if (!permissions[resourceCode]) {
          permissions[resourceCode] = []
        }

        if (checked) {
          permissions[resourceCode].push(action)
        } else {
          permissions[resourceCode].splice(permissions[resourceCode].indexOf(action), 1)
        }
      }
    }
  }
}
</script>
