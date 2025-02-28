<template>
  <page-view>
    <a-row>
      <a-col :span="24">
        <a-card :bodyStyle="{ padding: '16px' }" :bordered="false">
          <div class="table-page-search-wrapper">
            <a-form layout="inline">
              <a-row :gutter="48">
                <a-col :md="6" :sm="24">
                  <a-form-item label="关键词：">
                    <a-input v-model="list.params.keyword" @keyup.enter="handleQuery()" />
                  </a-form-item>
                </a-col>
                <a-col :md="6" :sm="24">
                  <a-form-item label="状态：">
                    <a-select v-model="list.params.type" placeholder="请选择状态" @change="handleQuery()">
                      <a-select-option v-for="type in Object.keys(list.journalType)" :key="type" :value="type">
                        {{ list.journalType[type].text }}
                      </a-select-option>
                    </a-select>
                  </a-form-item>
                </a-col>
                <a-col :md="6" :sm="24">
                  <span class="table-page-search-submitButtons">
                    <a-space>
                      <a-button type="primary" @click="handleQuery()">查询</a-button>
                      <a-button @click="handleResetParam()">重置</a-button>
                    </a-space>
                  </span>
                </a-col>
              </a-row>
            </a-form>
          </div>
          <div class="table-operator">
            <a-button icon="plus" type="primary" @click="handleOpenPublishModal">写日志</a-button>
          </div>
          <a-divider />
          <div class="mt-4">
            <a-empty v-if="!list.loading && list.data.length === 0" />
            <a-list v-else :dataSource="list.data" :loading="list.loading" :pagination="false" itemLayout="vertical">
              <template #renderItem="item, index">
                <a-list-item :key="index">
                  <template #actions>
                    <a-button class="!p-0" type="link">
                      <a-icon type="like-o" />
                      {{ item.likes }}
                    </a-button>
                    <a-button class="!p-0" type="link" @click="handleOpenJournalCommentsDrawer(item)">
                      <a-icon type="message" />
                      {{ item.commentCount }}
                    </a-button>
                    <a-button v-if="item.type === 'INTIMATE'" class="!p-0" disabled type="link">
                      <a-icon type="lock" />
                    </a-button>
                    <a-button v-else class="!p-0" type="link">
                      <a-icon type="unlock" />
                    </a-button>
                  </template>
                  <template #extra>
                    <a-button class="!p-0" type="link" @click="handleOpenEditModal(item)">编辑</a-button>
                    <a-divider type="vertical" />
                    <a-popconfirm
                      cancelText="取消"
                      okText="确定"
                      title="你确定要删除这条日志？"
                      @confirm="handleDelete(item.id)"
                    >
                      <a-button class="!p-0" type="link">删除</a-button>
                    </a-popconfirm>
                  </template>

                  <a-list-item-meta>
                    <template #description>
                      <div class="journal-list-content" v-html="item.content"></div>
                    </template>
                    <template #title>
                      <span>{{ item.createTime | moment }}</span>
                    </template>
                    <template #avatar>
                      <a-avatar :src="user.avatar" size="large" />
                    </template>
                  </a-list-item-meta>
                </a-list-item>
              </template>
              <div class="page-wrapper">
                <a-pagination
                  :current="pagination.page"
                  :defaultPageSize="pagination.size"
                  :pageSizeOptions="['10', '20', '50', '100']"
                  :total="pagination.total"
                  class="pagination"
                  showLessItems
                  showSizeChanger
                  @change="handlePageChange"
                  @showSizeChange="handlePageSizeChange"
                />
              </div>
            </a-list>
          </div>
        </a-card>
      </a-col>
    </a-row>

    <div style="position: fixed; bottom: 30px; right: 30px">
      <a-button
        icon="setting"
        shape="circle"
        size="large"
        type="primary"
        @click="optionModal.visible = true"
      ></a-button>
    </div>

    <a-modal v-model="optionModal.visible" :afterClose="() => (optionModal.visible = false)" title="页面设置">
      <template #footer>
        <a-button key="submit" type="primary" @click="handleSaveOptions()">保存</a-button>
      </template>
      <a-form layout="vertical">
        <a-form-item help="* 需要主题进行适配" label="页面标题：">
          <a-input v-model="optionModal.options.journals_title" />
        </a-form-item>
        <a-form-item label="每页显示条数：">
          <a-input-number v-model="optionModal.options.journals_page_size" style="width: 100%" />
        </a-form-item>
      </a-form>
    </a-modal>

    <!-- 编辑日志弹窗 -->
    <a-modal v-model="form.visible" :title="formTitle" :width="820">
      <template #footer>
        <ReactiveButton
          :errored="form.saveErrored"
          :loading="form.saving"
          erroredText="发布失败"
          loadedText="发布成功"
          text="发布"
          type="primary"
          @callback="handleSaveOrUpdateCallback"
          @click="handleSaveOrUpdate"
        ></ReactiveButton>
      </template>
      <a-form-model ref="journalForm" :model="form.model" :rules="form.rules" layout="vertical">
        <a-form-model-item prop="sourceContent">
          <div id="editor" style="height: 520px">
            <MarkdownEditor
              v-if="form.visible"
              :originalContent.sync="form.model.sourceContent"
              :subfield="false"
              :toolbars="simpleEditorToolbars"
              @change="onContentChange"
            />
          </div>
        </a-form-model-item>
        <a-form-model-item>
          <a-switch v-model="form.isPublic" checkedChildren="公开" defaultChecked unCheckedChildren="私密" />
        </a-form-model-item>
      </a-form-model>
    </a-modal>

    <TargetCommentListModal
      :target-id="list.selected.id"
      :title="`「${$options.filters.moment(list.selected.createTime)}」的评论`"
      :visible.sync="journalCommentDrawer.visible"
      target="journal"
      @close="onJournalCommentsDrawerClose"
    >
      <template #extraFooter>
        <a-button :disabled="selectPreviousButtonDisabled" @click="handleSelectPrevious"> 上一篇</a-button>
        <a-button :disabled="selectNextButtonDisabled" @click="handleSelectNext"> 下一篇</a-button>
      </template>
    </TargetCommentListModal>
  </page-view>
</template>

<script>
// components
import { PageView } from '@/layouts'
import TargetCommentListModal from '@/components/Comment/TargetCommentListModal'

// libs
import { mixin, mixinDevice } from '@/mixins/mixin.js'
import { mapActions, mapGetters } from 'vuex'
import { simpleEditorToolbars } from '@/core/constant'
import { deepClone } from '@/utils/util'
import apiClient from '@/utils/api-client'
import MarkdownEditor from '@/components/Editor/MarkdownEditor'

export default {
  mixins: [mixin, mixinDevice],
  components: { MarkdownEditor, PageView, TargetCommentListModal },
  data() {
    return {
      simpleEditorToolbars,
      list: {
        data: [],
        loading: false,
        total: 0,
        params: {
          page: 0,
          size: 10,
          keyword: undefined,
          type: undefined
        },
        hasPrevious: false,
        hasNext: false,
        selected: {},
        journalType: {
          PUBLIC: {
            text: '公开'
          },
          INTIMATE: {
            text: '私密'
          }
        }
      },

      form: {
        model: {},
        rules: {
          sourceContent: [{ required: true, message: '* 内容不能为空', trigger: [] }]
        },
        visible: false,
        saving: false,
        saveErrored: false,
        isPublic: true
      },
      journalCommentDrawer: {
        visible: false
      },
      optionModal: {
        visible: false,
        options: []
      }
    }
  },
  beforeMount() {
    this.handleListJournals()
    this.handleListOptions()
  },
  computed: {
    ...mapGetters(['user']),
    formTitle() {
      return this.form.model.id ? '编辑' : '发表'
    },
    pagination() {
      return {
        page: this.list.params.page + 1,
        size: this.list.params.size,
        total: this.list.total
      }
    },
    selectPreviousButtonDisabled() {
      const index = this.list.data.findIndex(journal => journal.id === this.list.selected.id)
      return index === 0 && !this.list.hasPrevious
    },
    selectNextButtonDisabled() {
      const index = this.list.data.findIndex(journal => journal.id === this.list.selected.id)
      return index === this.list.data.length - 1 && !this.list.hasNext
    }
  },
  methods: {
    ...mapActions(['refreshOptionsCache']),
    async handleListJournals() {
      try {
        this.list.loading = true

        const { data } = await apiClient.journal.list(this.list.params)
        if (data.content.length === 0 && this.list.params.page > 0) {
          this.list.params.page--
          await this.handleListJournals()
          return
        }

        this.list.data = data.content
        this.list.total = data.total
        this.list.hasPrevious = data.hasPrevious
        this.list.hasNext = data.hasNext
      } catch (e) {
        this.$log.error(e)
      } finally {
        this.list.loading = false
      }
    },
    handleListOptions() {
      apiClient.option.list().then(response => {
        this.optionModal.options = response.data
      })
    },
    handleQuery() {
      this.handlePageChange(1)
    },
    handleResetParam() {
      this.list.params.keyword = undefined
      this.list.params.type = undefined
      this.handlePageChange(1)
    },
    handleOpenPublishModal() {
      this.form.visible = true
      this.form.model = {
        sourceContent: '',
        content: ''
      }
    },
    handleOpenEditModal(item) {
      this.form.model = deepClone(item)
      this.form.isPublic = item.type !== 'INTIMATE'
      this.form.visible = true
    },
    handleDelete(id) {
      apiClient.journal.delete(id).finally(() => {
        this.handleListJournals()
      })
    },
    handleOpenJournalCommentsDrawer(journal) {
      this.list.selected = journal
      this.journalCommentDrawer.visible = true
    },

    onContentChange({ originalContent, renderContent }) {
      this.form.model.sourceContent = originalContent
      this.form.model.content = renderContent
    },

    handleSaveOrUpdate() {
      const _this = this
      _this.$refs.journalForm.validate(valid => {
        if (valid) {
          _this.form.model.type = _this.form.isPublic ? 'PUBLIC' : 'INTIMATE'
          _this.form.model.keepRaw = true
          _this.form.saving = true
          if (_this.form.model.id) {
            apiClient.journal
              .update(_this.form.model.id, _this.form.model)
              .catch(() => {
                _this.form.saveErrored = true
              })
              .finally(() => {
                setTimeout(() => {
                  _this.form.saving = false
                }, 400)
              })
          } else {
            apiClient.journal
              .create(_this.form.model)
              .catch(() => {
                _this.form.saveErrored = true
              })
              .finally(() => {
                setTimeout(() => {
                  _this.form.saving = false
                }, 400)
              })
          }
        }
      })
    },

    handleSaveOrUpdateCallback() {
      if (this.form.saveErrored) {
        this.form.saveErrored = false
      } else {
        this.form.isPublic = true
        this.form.visible = false
        this.handleListJournals()
      }
    },

    /**
     * Handle page change
     */
    handlePageChange(page = 1) {
      this.list.params.page = page - 1
      this.handleListJournals()
    },

    /**
     * Handle page size change
     */
    handlePageSizeChange(current, size) {
      this.$log.debug(`Current: ${current}, PageSize: ${size}`)
      this.list.params.page = 0
      this.list.params.size = size
      this.handleListJournals()
    },

    onJournalCommentsDrawerClose() {
      this.form.model = {}
      this.journalCommentDrawer.visible = false
      this.handleListJournals()
    },

    handleSaveOptions() {
      apiClient.option
        .save(this.optionModal.options)
        .then(() => {
          this.$message.success('保存成功！')
          this.optionModal.visible = false
        })
        .finally(() => {
          this.handleListOptions()
          this.refreshOptionsCache()
        })
    },

    /**
     * Select previous journal
     */
    async handleSelectPrevious() {
      const index = this.list.data.findIndex(journal => journal.id === this.list.selected.id)
      if (index > 0) {
        this.list.selected = this.list.data[index - 1]
        return
      }
      if (index === 0 && this.list.hasPrevious) {
        this.list.params.page--
        await this.handleListJournals()
        this.list.selected = this.list.data[this.list.data.length - 1]
      }
    },

    /**
     * Select next journal
     */
    async handleSelectNext() {
      const index = this.list.data.findIndex(journal => journal.id === this.list.selected.id)
      if (index < this.list.data.length - 1) {
        this.list.selected = this.list.data[index + 1]
        return
      }
      if (index === this.list.data.length - 1 && this.list.hasNext) {
        this.list.params.page++
        await this.handleListJournals()
        this.list.selected = this.list.data[0]
      }
    }
  }
}
</script>
