<template>
  <Row type="flex" align="top" justify="center" style="background: #fff">
    <Col span="24" style="display: flex;flex-direction: row;justify-content: flex-start; align-content: center;">
      <Form inline
            style="width:100%;display: flex;flex-direction: row;justify-content: space-between;align-content: center;">
        <FormItem style="width: 250px;">
          <Input size="default" v-model="searchKey" @on-search="searchByKey" placeholder="请输入商品名称/品种代码" search enter-button="搜索"></Input>
        </FormItem>
        <FormItem>
          <Button size="default" icon="ios-add" @click="addShopping" type="primary">添加商品</Button>
        </FormItem>
      </Form>
    </Col>
    <Col span="24">
      <Table style="margin-top: 10px;" size="large" border :columns="tableTitle" :data="tableData" :loading="loading"></Table>
      <div class="page-page">
        <Page @on-change="changePageData" :total="pages.total" show-total show-elevator
              :page-size="pages.pageSize"
              :current.sync="pages.pageNum"/>
      </div>
    </Col>
    <ImageView ref="imageViewRef" :viewSrc="viewSrc" :list="imgList"></ImageView>
  </Row>
</template>

<script type="text/jsx">
  import Mixin from '@/common/js/app-mixin'
  import ImageView from '@/common/vue/imgeView'

  export default {
    name: 'new-pro',
    mixins: [Mixin],
    components: {
      ImageView
    },
    data () {
      return {
        type: '3',
        imgList: [],
        loading: false,
        searchKey: null,
        visible: false,
        viewSrc: null,
        tableTitle: [
          {
            title: '商品ID',
            align: 'center',
            tooltip: true,
            key: 'id'
          }, {
            title: '商品图片',
            align: 'center',
            tooltip: true,
            render: (h, params) => {
              return <div style="display:flex;flex-direction:column;align-items:center;cursor:pointer;">
                <img onClick={this.viewHandler.bind(this, params)} src={params.row.cover} width="40" height="40" />
              </div>
            }
          }, {
            title: '商品名称',
            align: 'center',
            tooltip: true,
            key: 'name'
          }, {
            title: '颜色',
            align: 'center',
            tooltip: true,
            key: 'color'
          }, {
            title: '规格',
            align: 'center',
            tooltip: true,
            render: (h, params) => {
              let type = params.row.unit_type
              let str = ''
              if (type === '3') {
                str = '若干枝'
              } else {
                let s = this.$util.getNameByStatus(type, 'unit_type', this.systemMap)
                str = params.row.unit + s
              }
              return <div>
                <span>{str}</span>
              </div>
            }
          }, {
            title: '特性描述',
            align: 'center',
            tooltip: true,
            key: 'petal'
          }, {
            title: '品种描述',
            align: 'center',
            tooltip: true,
            key: 'des'
          }, {
            title: '品种代码',
            align: 'center',
            tooltip: true,
            key: 'brand'
          }, {
            title: '上下架',
            align: 'center',
            tooltip: true,
            key: 'statusStr'
          }, {
            title: '操作',
            align: 'center',
            tooltip: true,
            render: (h, params) => {
              let str = params.row.status === '1' ? <i-button size="small" icon="ios-add" type="primary"
                                                              nativeOnClick={this.downHandle.bind(this, params.row)}></i-button>
                : <i-button size="small" icon="ios-remove" type="warning"
                            nativeOnClick={this.downHandle.bind(this, params.row)}></i-button>

              return <div style="display:flex;flex-direction:row;justify-content:space-around;align-items:center;">
                <i-button size="small" icon="ios-create-outline" type="primary"
                          nativeOnClick={this.editorItem.bind(this, params)}></i-button>
                {str}
                <i-button size="small" icon="ios-trash" type="error" nativeOnClick={this.delHandler.bind(this, params)}></i-button>
              </div>
            }
          }
        ],
        tableData: []
      }
    },
    created () {
      this.initPage()
    },
    methods: {
      initPage () {
        this.getList()
      },
      async getList () {
        let query = {
          page: this.pages.pageNum,
          name: this.searchKey,
          type: this.type
        }
        this.loading = true
        let {data: {data, total}} = await this.$http.getShopList(query)
        let that = this
        data.forEach(item => {
          item.statusStr = that.$util.getNameByStatus(item.status, 'status', that.systemMap)
        })
        this.loading = false
        this.tableData = data
        this.pages.total = total
      },
      changePageData () {
        this.updateList()
      },
      viewHandler (o) {
        if (o.row && o.row.cover) {
          this.$refs.imageViewRef.show()
          this.viewSrc = o.row.cover
          this.imgList = o.row.photo
        }
      },
      updateList () {
        this.$nextTick(() => {
          this.getList()
        })
      },
      searchByKey () {
        this.initPageData()
        this.updateList()
      },
      addShopping () {
        this.$router.push({
          name: 'newProDetail',
          params: {
            id: '-1'
          },
          query: {
            type: this.type,
            title: this.$route.meta.title
          }
        })
      },
      async downHandle (row) {
        let query = {
          id: row.id,
          status: 1
        }
        let str = row.status === '1' ? '是否确认上架?' : '是否确认下架?'
        this.$Modal.confirm({
          title: '提示',
          content: `<p>${str}</p>`,
          onOk: async () => {
            let {code} = await this.$http.downShop(query)
            if (code === 0) {
              this.$Message.success({
                content: row.status === '1' ? '上架成功！' : '下架成功！'
              })
              this.updateList()
            } else {
              this.$Message.error({
                content: '操作失败！'
              })
            }
          },
          onCancel: () => {
            this.$Message.info('已取消！')
          }
        })
      },
      async delHandler (params) {
        let query = {
          id: params.row.id
        }

        this.$Modal.confirm({
          title: '提示',
          content: '<p>是否确认该商品</p>',
          onOk: async () => {
            let {code} = await this.$http.delShop(query)
            if (code === 0) {
              this.$Message.success({
                content: '删除成功！'
              })
              this.getList()
            } else {
              this.$Message.error({
                content: '删除失败！'
              })
            }
          },
          onCancel: () => {
            this.$Message.info('已取消！')
          }
        })
      },
      // async downHandle (row) {
      //   let query = {
      //     id: row.id,
      //     status: row.status
      //   }
      //   let {code} = await this.$http.downShop(query)
      //   if (code === 0) {
      //     this.$Message.success({
      //       content: row.status === '1' ? '上架成功！' : '下架成功！'
      //     })
      //     this.updateList()
      //   }
      // },
      // async delHandler (params) {
      //   let query = {
      //     id: params.row.id
      //   }
      //   let {code} = await this.$http.delShop(query)
      //   if (code === 0) {
      //     this.$Message.success({
      //       content: '删除成功！'
      //     })
      //     this.getList()
      //   }
      // },
      editorItem (params) {
        this.$router.push({
          name: 'newProDetail',
          params: {
            id: params.row.id
          },
          query: {
            type: this.type,
            title: this.$route.meta.title
          }
        })
      }
    }
  }
</script>

<style scoped>

</style>
