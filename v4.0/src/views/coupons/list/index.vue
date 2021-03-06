<template>
  <div class="p-ads">
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
          <Button
            type="primary"
            @click="showSendForm">
            发放
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
          label="类型"
          prop="type">
          <Row>
            <Col span="20">
              <Select
                v-model="cForm.formValidate.type"
                style="width: 320px;">
                <Option
                  v-for="item in $consts.COUPON_TYPES"
                  :key="item.value"
                  :value="item.value">
                  {{ item.label }}
                </Option>
              </Select>
            </Col>
          </Row>
        </Form-item>
        <Form-item
          v-show="cForm.formValidate.type === 'DESIGNATED_PRODUCT'"
          label="指定商品"
          prop="productIds">
          <Row>
            <Col span="20">
              <CProductSelect
                v-if="cForm.modal"
                :value="cForm.formValidate.productIds"
                multiple
                @change="value => { cForm.formValidate.productIds = value }"
              />
            </Col>
          </Row>
        </Form-item>
        <Form-item
          v-show="cForm.formValidate.type === 'DESIGNATED_CATEGORY'"
          label="指定分类"
          prop="categoryIds">
          <CCategories
            alias="products"
            multiple
            select-parent
            v-model="cForm.formValidate.categoryIds"
            @on-change="value => { cForm.formValidate.categoryIds = value }"
            style="width: 320px;"
          />
        </Form-item>
        <Form-item
          label="抵扣金额"
          prop="value">
          <Row>
            <Col span="20">
              <InputNumber
                :min="0"
                :max="100000"
                v-model="cForm.formValidate.value" />
              元
            </Col>
          </Row>
        </Form-item>
        <Form-item
          v-show="cForm.formValidate.type !== 'REDUCTION'"
          label="最低消费"
          prop="value">
          <Row>
            <Col span="20">
              <InputNumber
                :min="0"
                :max="100000"
                v-model="cForm.formValidate.minPrice" />
              元
            </Col>
          </Row>
        </Form-item>
        <Form-item
          label="有效期"
          prop="period">
          <InputNumber
            :min="0"
            :max="100000"
            v-model="cForm.formValidate.period" />
          天
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
    <Modal
      width="600"
      v-model="cSendForm.modal"
      title="发放优惠券">
      <Form
        ref="sendFormValidate"
        :model="cSendForm.formValidate"
        :rules="cSendForm.ruleValidate"
        :label-width="80">
        <Form-item
          label="发放对象"
          prop="wxUserIds">
          <Row>
            <Col span="22">
              <Transfer
                :data="wxUsersList.items | toTransfer"
                :target-keys="cSendForm.cTransfer.targetKeys"
                :render-format="item => item.label"
                :list-style="{ height: '350px' }"
                filterable
                @on-change="newTargetKeys => { cSendForm.cTransfer.targetKeys = newTargetKeys }">
              </Transfer>
            </Col>
          </Row>
        </Form-item>
      </Form>
      <div slot="footer">
        <Button
          type="text"
          size="large"
          @click="cSendForm.modal = false">
          取消
        </Button>
        <Button
          type="primary"
          size="large"
          @click="send">
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

const module = 'coupons'
const initForm = {
  type: 'FULL_REDUCTION',
  value: 0,
  minPrice: 0,
  period: 30
}
const initSendForm = {
  wxUserIds: []
}

export default {
  mixins: [
    routeParamsMixin,
    listMixin,
    formMixin
  ],
  data () {
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
            key: 'name'
          },
          {
            type: 'type',
            title: '类型',
            width: 90,
            render: (h, params) => h('span', null, this.$helpers.getItem(this.$consts.COUPON_TYPES, 'value', params.row.type)['label'])
          },
          {
            title: '价值',
            key: 'value',
            width: 100,
            render: (h, params) => h('span', null, `${params.row.value} 元`)
          },
          {
            title: '最低消费',
            key: 'minPrice',
            width: 100,
            render: (h, params) => h('span', null, `${params.row.minPrice} 元`)
          },
          {
            title: '有效期',
            width: 100,
            render: (h, params) => h('span', null, `${params.row.period} 天`)
          },
          {
            title: '操作',
            key: 'action',
            width: 240,
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
              }, '删除'),
              h('CDropdown', {
                attrs: {
                  title: '排序',
                  options: this.$consts.ORDER_ACTIONS
                },
                on: {
                  click: async value => {
                    this.handleSort(params.row.id, value)
                  }
                }
              })
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
          ],
          type: [
            {
              required: true,
              message: '类型不能为空'
            }
          ]
        }
      },
      wxUsersList: {},
      cSendForm: {
        modal: false,
        formValidate: this.$helpers.deepCopy(initSendForm),
        ruleValidate: {},
        cTransfer: {
          targetKeys: []
        }
      }
    }
  },
  computed: mapState({
    list: state => state[module].list
  }),
  filters: {
    toTransfer (items = []) {
      return items.map(item => ({
        key: item.id,
        label: item.nickName
      }))
    }
  },
  watch: {
    'cForm.modal': {
      handler (newVal) {
        !newVal && this.resetFields(initForm)
      }
    },
    'cSendForm.modal': {
      handler () {
        this.cSendForm.cTransfer.targetKeys = []
      }
    }
  },
  async beforeRouteUpdate (to, from, next) {
    await this.getList()
    this.wxUsersList = await this.getWxUsersList()
    next()
  },
  async created () {
    await this.getList()
    this.wxUsersList = await this.getWxUsersList()
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
    getWxUsersList () {
      return this.$store.dispatch('wxUsers/getList', {
        query: {
          offset: 0,
          limit: 10000
        }
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
    async handleChangeStatus (id, value) {
      await this.$store.dispatch(`${module}/put`, {
        id,
        body: { status: value }
      })

      this.$Message.success('修改状态成功')
      this.getList()
    },
    async handleSort (id, value) {
      await this.$store.dispatch(`${module}/postAction`, {
        query: { where: { ...this.listSearchWhere, alias: this.alias } },
        body: { type: value, id }
      })

      this.getList()
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
    showSendForm () {
      if (!this.listSelectedItems.length) {
        this.$Message.error('请选择优惠券')
        return
      }

      this.cSendForm.modal = true
    },
    async send () {
      const { targetKeys } = this.cSendForm.cTransfer

      if (!targetKeys.length) {
        this.$Message.error('请选择发放对象')
        return
      }

      await this.$store.dispatch(`${module}/postAction`, {
        body: {
          type: 'SEND',
          wxUserIds: targetKeys,
          couponIds: this.listSelectedItems.map(item => item.id)
        }
      })

      this.$Message.success('发放成功')
      this.getList()

      this.cSendForm.modal = false
    }
  }
}
</script>

<style
  lang="scss"
  src="./styles/index.scss">
</style>
