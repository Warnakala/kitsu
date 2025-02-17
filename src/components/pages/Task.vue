<template>
  <div class="page fixed-page">
  <div style="overflow-y: auto;">
    <div class="page-header">
      <div
        class="flexrow header-title"
        v-if="currentTask"
      >
        <router-link
          :to="entityPage"
          class="flexrow-item has-text-centered back-link"
        >
          <chevron-left-icon />
        </router-link>
        <div class="title flexrow-item">
          <router-link :to="taskEntityPath">
            {{ currentTask ? title : 'Loading...'}}
          </router-link>
        </div>
        <task-type-name
          class="flexrow-item task-type"
          :task-type="currentTaskType"
          :production-id="currentProduction.id"
          v-if="currentTaskType"
        />
        <subscribe-button
          class="flexrow-item action-button"
          :subscribed="isAssigned || isSubscribed"
          @click="toggleSubscribe"
          v-if="!isAssigned"
        />
      </div>

      <div
        class="flexrow task-information"
        v-if="currentTask"
      >
        <span class="flexrow-item">{{ $t('tasks.current_status') }}</span>
        <validation-tag
          class="is-medium flexrow-item"
          :task="currentTask"
          :is-static="true"
          v-if="currentTask"
        />
        <span class="flexrow-item">{{ $t('tasks.fields.assignees') }}:</span>
        <span
          class="flexrow-item avatar-wrapper"
          :key="personId"
          v-for="personId in currentTask.assignees"
        >
          <people-avatar
            class="flexrow-item"
            :key="personId"
            :person="personMap.get(personId)"
            :size="30"
            :font-size="16"
          />
       </span>
      </div>
    </div>

    <div class="task-columns" ref="task-columns">
      <div class="task-column comments-column">
        <div v-if="currentTask">
          <div>
            <add-comment
              ref="add-comment"
              :is-loading="loading.addComment"
              :is-error="errors.addComment"
              :user="user"
              :team="currentTeam"
              :task="currentTask"
              :task-status="taskStatusForCurrentUser"
              :attached-file-name="attachedFileName"
              @add-comment="addComment"
              @add-preview="onAddPreviewClicked"
              @duplicate-comment="onDuplicateComment"
              @file-drop="selectFile"
              v-if="isCommentingAllowed"
            />
            <div
              class="comments"
              v-if="currentTaskComments && currentTaskComments.length > 0"
            >
              <comment
                :comment="comment"
                :task="currentTask"
                :highlighted="false"
                :key="comment.id"
                :current-user="user"
                :editable="(
                  comment.person && user.id === comment.person.id ||
                  isCurrentUserAdmin
                )"
                :is-first="index === 0"
                :is-last="index === pinnedCount"
                :is-change="isStatusChange(index)"
                @ack-comment="ackComment"
                @pin-comment="onPinComment"
                @edit-comment="onEditComment"
                @delete-comment="onDeleteComment"
                @checklist-updated="saveComment"
                v-for="(comment, index) in currentTaskComments"
              />
            </div>
            <div class="no-comment" v-else>
              <em>
                {{ $t('tasks.no_comment')}}
              </em>
            </div>
          </div>
        </div>

        <div class="has-text-centered" v-else>
          <spinner />
        </div>
      </div>

      <div class="task-column preview-column">
        <div class="preview-column-content">
          <div class="flexrow preview-header">
            <h2 class="subtitle flexrow-item">
              {{ $t('tasks.preview') }}
            </h2>
            <div
              class="set-main-preview flexrow-item flexrow pull-right"
              v-if="isPreviewButtonVisible"
            >
              <button
                :class="{
                  button: true,
                  'flexrow-item': true,
                  'is-loading': loading.setPreview
                }"
                @click="setPreview"
                v-if="isPreviews && isCurrentUserManager"
              >
                <image-icon class="icon" />
                <span class="text">
                  {{ $t('tasks.set_preview') }}
                </span>
              </button>
              <span class="error flexrow-item" v-if="errors.setPreview">
                {{ $t('tasks.set_preview_error') }}
              </span>
            </div>
            <div
              class="set-main-preview flexrow-item pull-right"
              v-if="currentTask && currentTask.entity && currentTask.entity.preview_file_id === currentPreviewId">
              <em>{{ $t('tasks.set_preview_done') }}</em>
            </div>
          </div>

          <div
            class="preview-list mt2"
            v-if="isPreviews"
          >
            <preview-row
              :key="preview.id"
              :preview="preview"
              :preview-path="previewPath(preview.id)"
              :taskId="currentTask ? currentTask.id : ''"
              :selected="preview.id === currentPreviewId"
              v-for="preview in currentTaskPreviews"
            />
          </div>
          <div v-else>
            <em>
              {{ $t('tasks.no_preview')}}
            </em>
          </div>

          <div class="preview-area">
            <div
              v-if="isPreviews"
            >
              <preview-player
                :previews="currentPreview.previews"
                :task-type-map="taskTypeMap"
                :entity-preview-files="taskEntityPreviews"
                :read-only="isCurrentUserArtist"
                :last-preview-files="lastFivePreviews"
                :task="currentTask"
                @annotation-changed="onAnnotationChanged"
                @add-extra-preview="onAddExtraPreviewClicked"
                @remove-extra-preview="onRemoveExtraPreviewClicked"
                @change-current-preview="changeCurrentPreview"
                ref="preview-player"
                v-if="currentPreview"
              />
            </div>
          </div>
        </div>
      </div>
    </div>

    <add-preview-modal
      ref="add-preview-modal"
      :active="modals.addPreview"
      :is-loading="loading.addPreview"
      :is-error="errors.addPreview"
      :form-data="addPreviewFormData"
      @cancel="modals.addPreview = false"
      @fileselected="selectFile"
      @confirm="closeAddPreviewModal"
    />

    <add-preview-modal
      ref="add-extra-preview-modal"
      :active="modals.addExtraPreview"
      :is-loading="loading.addExtraPreview"
      :is-error="errors.addExtraPreview"
      :form-data="addExtraPreviewFormData"
      @cancel="hideExtraPreviewModal"
      @fileselected="selectFile"
      @confirm="createExtraPreview"
    />

    <edit-comment-modal
      :active="modals.editComment"
      :is-loading="loading.editComment"
      :is-error="errors.editComment"
      :comment-to-edit="commentToEdit"
      :team="currentTeam"
      @confirm="confirmEditTaskComment"
      @cancel="onCancelEditComment"
    />

    <delete-modal
      :active="modals.deleteComment"
      :is-loading="loading.deleteComment"
      :is-error="errors.deleteComment"
      :text="$t('tasks.delete_comment')"
      :error-text="$t('tasks.delete_comment_error')"
      @confirm="confirmDeleteTaskComment"
      @cancel="onCancelDeleteComment"
    />

    <delete-modal
      :active="modals.deleteExtraPreview"
      :is-loading="loading.deleteExtraPreview"
      :is-error="errors.deleteExtraPreview"
      :text="$t('tasks.delete_preview')"
      :error-text="$t('tasks.delete_preview_error')"
      @cancel="hideRemoveExtraPreviewModal"
      @confirm="confirmDeleteTaskPreview"
    />
  </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import {
  ChevronLeftIcon,
  ImageIcon
} from 'vue-feather-icons'

import AddComment from '../widgets/AddComment'
import AddPreviewModal from '../modals/AddPreviewModal'
import Comment from '../widgets/Comment'
import DeleteModal from '../modals/DeleteModal'
import EditCommentModal from '../modals/EditCommentModal'
import PeopleAvatar from '../widgets/PeopleAvatar'
import PreviewRow from '../widgets/PreviewRow'
import Spinner from '../widgets/Spinner'
import SubscribeButton from '../widgets/SubscribeButton'
import TaskTypeName from '../widgets/TaskTypeName'
import ValidationTag from '../widgets/ValidationTag'
import PreviewPlayer from '../previews/PreviewPlayer'

export default {
  name: 'task',
  components: {
    AddComment,
    AddPreviewModal,
    Comment,
    ChevronLeftIcon,
    DeleteModal,
    EditCommentModal,
    ImageIcon,
    PeopleAvatar,
    PreviewRow,
    PreviewPlayer,
    Spinner,
    SubscribeButton,
    TaskTypeName,
    ValidationTag
  },

  data () {
    return {
      attachedFileName: '',
      entityPage: this.getEntityPage(),
      selectedTab: 'validation',
      taskLoading: {
        isLoading: true,
        isError: false
      },
      modals: {
        addPreview: false,
        addExtraPreview: false,
        deleteExtraPreview: false,
        deleteTask: false,
        deleteComment: false,
        editComment: false
      },
      loading: {
        addComment: false,
        addPreview: false,
        addExtraPreview: false,
        setPreview: false,
        deleteTask: false,
        deleteComment: false,
        editComment: false
      },
      errors: {
        addComment: false,
        addPreview: false,
        addExtraPreview: false,
        setPreview: false,
        deleteTask: false,
        deleteComment: false,
        editComment: false
      },
      currentTask: null,
      currentTaskComments: [],
      currentTaskPreviews: [],
      commentToEdit: null,
      addPreviewFormData: null,
      addExtraPreviewFormData: null,
      isSubscribed: false
    }
  },

  created () {
    this.clearSelectedTasks()
    this.loadTaskData()
  },

  mounted () {
    this.reset()
    this.$nextTick(() => {
      if (this.$refs['task-columns']) {
        this.$refs['task-columns'].scrollTop = 100
        window.scrollTo(0, 0)
      }
    })
  },

  computed: {
    ...mapGetters([
      'currentEpisode',
      'currentProduction',
      'displayedShots',
      'displayedAssets',
      'getTaskComments',
      'getTaskPreviews',
      'getTaskComment',
      'isCurrentUserAdmin',
      'isCurrentUserArtist',
      'isCurrentUserManager',
      'isSingleEpisode',
      'isTVShow',
      'personMap',
      'productionMap',
      'route',
      'taskEntityPreviews',
      'taskStatus',
      'taskStatusForCurrentUser',
      'taskMap',
      'taskTypeMap',
      'user'
    ]),

    isPreviewButtonVisible () {
      return (
        this.currentTask &&
        this.currentTask.entity &&
        this.currentTask.entity.preview_file_id !== this.currentPreviewId &&
        ['png', 'mp4'].includes(this.extension)
      )
    },

    extension () {
      return this.currentPreview ? this.currentPreview.extension : ''
    },

    currentPreviewId () {
      return this.currentPreview ? this.currentPreview.id : ''
    },

    currentPreview () {
      if (this.isPreviews) {
        let currentPreview = this.currentTaskPreviews[0]
        const previewId = this.route.params.preview_id
        if (previewId) {
          currentPreview = this.currentTaskPreviews.find((preview) => {
            return preview.id === previewId
          })
        }
        return currentPreview
      } else {
        return null
      }
    },

    isCommentingAllowed () {
      return this.isCurrentUserManager || this.currentTask.assignees.find(
        (personId) => personId === this.user.id
      )
    },

    taskTypeBorder () {
      let border = 'transparent'
      if (this.currentTask) border = this.currentTask.task_type_color
      return {
        'border-left': `4px solid ${border}`
      }
    },

    deleteTaskPath () {
      return this.taskPath(this.currentTask, 'task-delete')
    },

    isPreviews () {
      return this.currentTaskPreviews && this.currentTaskPreviews.length > 0
    },

    taskEntityPath () {
      if (this.currentTask) {
        const type = this.currentTask.entity_type_name
        let entityId = ''
        if (this.currentTask.entity) {
          entityId = this.currentTask.entity.id
        } else {
          entityId = this.currentTask.entity_id
        }

        const route = {
          name: type === 'Shot' ? 'shot' : 'asset',
          params: {
            production_id: this.currentTask.project_id
          }
        }

        if (type === 'Shot') {
          route.params.shot_id = entityId
        } else {
          route.params.asset_id = entityId
        }

        if (this.$route.params.episode_id) {
          route.name = `episode-${route.name}`
          route.params.episode_id = this.$route.params.episode_id
        }

        return route
      } else {
        return {
          name: 'open-productions'
        }
      }
    },

    lastFivePreviews () {
      if (this.currentTaskPreviews) {
        return this.currentTaskPreviews.slice(0, 5)
      } else {
        return []
      }
    },

    entityList () {
      const entity = this.displayedShots.find((entity) => {
        return entity.id === this.currentTask.entity_id
      })
      if (entity) {
        return this.displayedShots
      } else {
        return this.displayedAssets
      }
    },

    previousEntity () {
      if (this.currentTask) {
        const taskTypeId = this.currentTask.task_type_id
        const entityIndex = this.entityList.findIndex((entity) => {
          return entity.id === this.currentTask.entity_id
        })
        let firstTraversal = false

        let previousEntityIndex = entityIndex - 1
        if (previousEntityIndex < 0) {
          previousEntityIndex = this.entityList.length - 1
        }

        let taskId = null
        while (!taskId) {
          if (this.entityList[previousEntityIndex]) {
            const entity = this.entityList[previousEntityIndex]
            taskId = entity.tasks.find((ctaskId) => {
              const task = this.taskMap.get(taskId)
              if (task) {
                return task.task_type_id === taskTypeId
              } else {
                return false
              }
            })
          } else {
            taskId = this.currentTask.id
          }

          if (!taskId) {
            previousEntityIndex--
            if (previousEntityIndex < 0) {
              previousEntityIndex = this.entityList.length
              if (firstTraversal) {
                return null
              }
              firstTraversal = true
            }
          }
        }

        return this.taskPath({ id: taskId })
      } else {
        return {
          name: 'open-productions'
        }
      }
    },

    nextEntity () {
      if (this.currentTask) {
        const taskTypeId = this.currentTask.task_type_id
        let firstTraversal = false
        const entityIndex = this.entityList.findIndex((entity) => {
          return entity.id === this.currentTask.entity_id
        })

        let nextEntityIndex = entityIndex + 1
        if (nextEntityIndex >= this.entityList.length) {
          nextEntityIndex = 0
        }

        let taskId = null
        while (!taskId) {
          if (this.entityList[nextEntityIndex]) {
            const entity = this.entityList[nextEntityIndex]
            taskId = entity.tasks.find((ctaskId) => {
              const task = this.taskMap.get(taskId)
              if (task) {
                return task.task_type_id === taskTypeId
              } else {
                return false
              }
            })
          } else {
            taskId = this.currentTask.id
          }

          if (!taskId) {
            nextEntityIndex++
            if (nextEntityIndex >= this.entityList.length) {
              nextEntityIndex = 0

              if (firstTraversal) {
                return null
              }
              firstTraversal = true
            }
          }
        }

        return this.taskPath({ id: taskId })
      } else {
        return {
          name: 'open-productions'
        }
      }
    },

    previewPlayer () {
      return this.$refs['preview-player']
    },

    title () {
      if (this.currentTask) {
        const type = this.currentTask.entity_type_name
        let entityName =
          this.currentTask.full_entity_name || this.currentTask.entity_name
        if (this.isTVShow && type === 'Shot') {
          entityName = entityName
            .split('/')
            .splice(1)
            .join('/')
        }
        return `${entityName}`
      } else {
        return 'Loading...'
      }
    },

    windowTitle () {
      if (this.currentTask) {
        const taskTypeName = this.currentTask.task_type_name
        return `${this.title} / ${taskTypeName}`
      } else {
        return 'Loading...'
      }
    },

    deleteText () {
      if (this.currentTask) {
        const taskType = this.taskTypeMap.get(this.currentTask.task_type_id)
        return this.$t('main.delete_text', {
          name: `${this.currentTask.entity_name} / ${taskType.name}`
        })
      } else {
        return ''
      }
    },

    isAssigned () {
      if (this.currentTask) {
        return this.currentTask.assignees.some((assigneeId) => {
          return assigneeId === this.user.id
        })
      } else {
        return false
      }
    },

    currentTaskType () {
      if (this.currentTask) {
        return this.taskTypeMap.get(this.currentTask.task_type_id)
      } else {
        return null
      }
    },

    currentTeam () {
      return this.currentProduction.team.map(id => this.personMap.get(id))
    },

    pinnedCount () {
      return this.currentTaskComments.filter(c => c.pinned).length
    }
  },

  methods: {
    ...mapActions([
      'ackComment',
      'addCommentPreview',
      'addCommentExtraPreview',
      'commentTask',
      'commentTaskWithPreview',
      'changeCommentPreview',
      'clearSelectedTasks',
      'deleteTask',
      'deleteTaskPreview',
      'deleteTaskComment',
      'editTaskComment',
      'loadEpisodes',
      'loadTask',
      'loadShots',
      'loadAssets',
      'loadPreviewFileFormData',
      'loadTaskComments',
      'loadTaskSubscribed',
      'refreshPreview',
      'pinComment',
      'subscribeToTask',
      'setCurrentEpisode',
      'unsubscribeFromTask',
      'updatePreviewAnnotation'
    ]),

    getEntityPage () {
      if (this.currentTask) {
        const route = {
          name: this.$route.params.type,
          params: { production_id: this.currentTask.project_id }
        }

        if (route.name === 'asset') {
          route.params.asset_id = this.currentTask.entity_id
        } else {
          route.params.shot_id = this.currentTask.entity_id
        }

        if (this.isTVShow) {
          route.name = `episode-${route.name}`
          route.params.episode_id =
            this.currentEpisode ? this.currentEpisode.id : this.$route.params.episode_id
        }
        route.query = { search: '' }
        return route
      } else {
        return {
          name: 'open-productions'
        }
      }
    },

    loadTaskData () {
      const task = this.getCurrentTask()
      if (!task) {
        this.taskLoading = { isLoading: true, isError: false }
        this.loadTask({ taskId: this.route.params.task_id })
          .then(task => {
            let loadingFunction = (callback) => {
              this.loadAssets()
                .then(callback)
            }

            if (task.entity_type_name === 'Shot') {
              loadingFunction = (callback) => {
                this.loadEpisodes()
                  .then(() => {
                    if (this.isTVShow) {
                      this.setCurrentEpisode(task.episode.id)
                    }
                    this.loadShots(callback)
                  })
                  .catch(callback)
              }
            }
            loadingFunction(() => {
              this.currentTask = task
              this.loadTaskComments({
                taskId: task.id,
                entityId: task.entity_id
              })
                .then(() => this.loadTaskSubscribed({ taskId: task.id }))
                .then((subscribed) => {
                  this.isSubscribed = subscribed
                  this.reset()
                  this.entityPage = this.getEntityPage()
                  this.taskLoading = { isLoading: false, isError: false }
                }).catch((err) => {
                  console.error(err)
                  this.taskLoading = { isLoading: false, isError: true }
                })
            })
          })
      } else {
        const taskId = this.route.params.task_id
        this.currentTask = task
        this.loadTaskComments({ taskId, entityId: task.entity_id })
          .then(() => this.loadTaskSubscribed({ taskId }))
          .then(subscribed => {
            this.isSubscribed = subscribed
            this.currentTaskComments = this.getCurrentTaskComments()
            this.currentTaskPreviews = this.getCurrentTaskPreviews()
            this.entityPage = this.getEntityPage()
          })
          .catch(err => {
            console.error(err)
            this.taskLoading.isError = true
          })
      }
    },

    getCurrentTask () {
      return this.taskMap.get(this.route.params.task_id)
    },

    getCurrentComment () {
      if (this.route.params.comment_id) {
        return this.getTaskComment(
          this.route.params.task_id,
          this.route.params.comment_id
        )
      }
    },

    getCurrentTaskComments () {
      return this.getTaskComments(this.route.params.task_id)
    },

    getCurrentTaskPreviews () {
      return this.getTaskPreviews(this.route.params.task_id)
    },

    addComment (comment, attachment, checklist, taskStatusId) {
      const params = {
        taskId: this.currentTask.id,
        taskStatusId: taskStatusId,
        comment: comment,
        checklist,
        attachment
      }
      let action = 'commentTask'
      if (this.attachedFileName) action = 'commentTaskWithPreview'
      this.loading.addComment = true
      this.errors.addComment = false
      this.$store.dispatch(action, params)
        .then(() => {
          this.currentTaskPreviews = this.getCurrentTaskPreviews()
          this.resetPreview()
          this.$refs['add-preview-modal'].reset()
          this.reset()
          this.attachedFileName = ''
          this.loading.addComment = false
        })
        .catch((err) => {
          console.error(err)
          this.errors.addComment = true
          this.loading.addComment = false
        })
    },

    reset () {
      this.currentTaskComments = this.getCurrentTaskComments()
      this.currentTaskPreviews = this.getCurrentTaskPreviews()
      this.currentTask = this.getCurrentTask()
    },

    selectFile (forms) {
      this.loadPreviewFileFormData(forms)
      this.attachedFileName = forms
        .map((form) => form.get('file').name)
        .join(', ')
    },

    isHighlighted (comment) {
      return comment.preview && comment.preview.id === this.currentPreviewId
    },

    createExtraPreview () {
      const previews = this.currentTaskPreviews
      const preview = previews.length > 0 ? previews[0] : null
      this.errors.addExtraPreview = false
      this.loading.addExtraPreview = true
      const comment = this.getCurrentTaskComments().find((comment) => {
        return comment.previews.findIndex((p) => p.id === preview.id) >= 0
      })
      this.addCommentExtraPreview({
        taskId: this.currentTask.id,
        previewId: this.currentPreview.id,
        commentId: comment.id
      })
        .then(() => {
          this.loading.addExtraPreview = false
          this.modals.addExtraPreview = false
          this.$refs['add-extra-preview-modal'].reset()
          setTimeout(() => {
            this.previewPlayer.displayLast()
          }, 0)
        })
        .catch((err) => {
          console.error(err)
          this.errors.addExtraPreview = true
          this.loading.addExtraPreview = false
        })
    },

    resetPreview (changeRoute = true) {
      const previews = this.currentTaskPreviews
      const preview = previews.length > 0 ? previews[0] : null
      this.currentTaskComments = this.getCurrentTaskComments()
      this.currentTaskPreviews = this.getCurrentTaskPreviews()
      if (preview && changeRoute) {
        this.$router.push(this.previewPath(preview.id))
      }
    },

    setPreview () {
      this.loading.setPreview = true
      this.errors.setPreview = false
      this.$store.dispatch('setPreview', {
        taskId: this.currentTask.id,
        entityId: this.currentTask.entity.id,
        previewId: this.currentPreviewId
      })
        .then(() => {
          this.loading.setPreview = false
        })
        .catch((err) => {
          console.error(err)
          this.errors.setPreview = true
        })
    },

    confirmEditTaskComment (comment) {
      this.loading.editComment = true
      this.errors.editComment = false
      this.editTaskComment({
        taskId: this.currentTask.id,
        comment
      })
        .then(() => {
          this.loading.editComment = false
          this.modals.editComment = false
        })
        .catch(err => {
          console.error(err)
          this.errors.editComment = true
        })
    },

    saveComment (comment, checklist) {
      this.editTaskComment({
        taskId: this.currentTask.id,
        comment,
        checklist
      })
    },

    confirmDeleteTaskComment () {
      this.loading.deleteComment = true
      this.errors.deleteComment = false
      const commentId = this.commentToEdit.id

      this.deleteTaskComment({
        taskId: this.currentTask.id,
        commentId
      })
        .then(() => {
          this.loading.deleteComment = false
          this.reset()
          if (this.isPreviews) this.resetPreview()
          this.modals.deleteComment = false
        })
        .catch(err => {
          console.error(err)
          this.loading.deleteComment = false
          this.errors.deleteComment = true
        })
    },

    confirmDeleteTaskPreview () {
      this.loading.deleteExtraPreview = true
      this.errors.deleteExtraPreview = false
      const previewId = this.currentPreviewId
      const comment = this.getCurrentTaskComments().find((comment) => {
        return comment.previews.findIndex((p) => p.id === previewId) >= 0
      })

      this.previewPlayer.displayFirst()
      this.deleteTaskPreview({
        taskId: this.currentTask.id,
        commentId: comment.id,
        previewId: this.currentExtraPreviewId
      })
        .then(() => {
          this.loading.deleteExtraPreview = false
          this.resetPreview()
          this.hideRemoveExtraPreviewModal()
        })
        .catch((err) => {
          console.error(err)
          this.loading.deleteExtraPreview = false
          this.errors.deleteExtraPreview = true
        })
    },

    onPreviewAdded (eventData) {
      const taskId = eventData.task_id
      const commentId = eventData.comment_id
      const previewId = eventData.preview_file_id
      const revision = eventData.revision
      const extension = eventData.extension
      const comment = this.$store.getters.getTaskComment(taskId, commentId)

      if (
        this.currentTask &&
        comment &&
        (
          comment.previews.length === 0 ||
          comment.previews[0].id !== previewId
        ) &&
        taskId === this.currentTask.id
      ) {
        this.$store.commit('ADD_PREVIEW_END', {
          preview: {
            id: previewId,
            revision,
            extension: extension
          },
          taskId,
          commentId,
          comment
        })
        this.resetPreview()
      }
    },

    toggleSubscribe () {
      if (this.currentTask && !this.isAssigned) {
        if (this.isSubscribed) {
          this.unsubscribeFromTask(this.currentTask.id)
          this.isSubscribed = false
        } else {
          this.subscribeToTask(this.currentTask.id)
          this.isSubscribed = true
        }
      }
    },

    taskPath (task, section = 'task') {
      if (!task) {
        task = this.currentTask
      } else {
        task.project_id = this.currentTask.project_id
        task.episode_id = this.currentTask.episode_id
      }

      let route = { name: 'open-productions' }
      if (task) {
        route = {
          name: section,
          params: {
            production_id: task.project_id,
            task_id: task.id
          }
        }

        if (this.isTVShow && this.currentEpisode) {
          route.name = `episode-${section}`
          route.params.episode_id = task.episode_id || this.currentEpisode.id
        }
      }
      return route
    },

    previewPath (previewId) {
      const route = this.taskPath(this.currentTask, 'task-preview')

      if (route.params) {
        route.params.preview_id = previewId
      }
      return route
    },

    onAnnotationChanged ({ preview, additions, deletions, updates }) {
      const taskId = this.currentTask.id
      this.updatePreviewAnnotation({
        taskId, preview, additions, deletions, updates
      })
    },

    onAddExtraPreviewClicked () {
      this.modals.addExtraPreview = true
    },

    onRemoveExtraPreviewClicked (preview) {
      this.showRemoveExtraPreviewModal(preview)
    },

    hideExtraPreviewModal () {
      this.modals.addExtraPreview = false
    },

    showRemoveExtraPreviewModal (preview) {
      this.currentExtraPreviewId = preview.id
      this.modals.deleteExtraPreview = true
    },

    hideRemoveExtraPreviewModal () {
      this.modals.deleteExtraPreview = false
    },

    onAddPreviewClicked () {
      this.modals.addPreview = true
    },

    closeAddPreviewModal () {
      this.modals.addPreview = false
    },

    onDuplicateComment (comment) {
      this.$refs['add-comment'].setValue(comment)
    },

    onPinComment (comment) {
      this.pinComment(comment)
    },

    onEditComment (comment) {
      this.commentToEdit = comment
      this.modals.editComment = true
    },

    onDeleteComment (comment) {
      this.commentToEdit = comment
      this.modals.deleteComment = true
    },

    onCancelEditComment (comment) {
      this.modals.editComment = false
    },

    onCancelDeleteComment (comment) {
      this.modals.deleteComment = false
    },

    changeCurrentPreview (preview) {
      this.$router.push(this.previewPath(preview.id))
    },

    onRemoteAcknowledge (eventData, type) {
      if (this.currentTask) {
        const comment = this.currentTaskComments.find(
          c => c.id === eventData.comment_id
        )
        const user = this.personMap.get(eventData.person_id)
        if (comment && user) {
          if (this.user.id === user.id) {
            if (
              (
                type === 'ack' &&
                !comment.acknowledgements.includes(user.id)
              ) ||
              (
                type === 'unack' &&
                comment.acknowledgements.includes(user.id)
              )
            ) {
              this.$store.commit('ACK_COMMENT', { comment, user })
            }
          } else {
            this.$store.commit('ACK_COMMENT', { comment, user })
          }
        }
      }
    },

    isStatusChange (index) {
      const comments = this.currentTaskComments
      const comment = comments[index]
      return (
        index === comments.length - 1 ||
        comment.task_status_id !== comments[index + 1].task_status_id
      )
    }
  },

  watch: {
    $route () {
      if (this.$route.params.task_id !== this.currentTask.id) {
        this.loadTaskData()
      }
    }
  },

  socket: {
    events: {
      'preview-file:add-file' (eventData) {
        this.onPreviewAdded(eventData)
      },

      'comment:acknowledge' (eventData) {
        this.onRemoteAcknowledge(eventData, 'ack')
      },

      'comment:unacknowledge' (eventData) {
        this.onRemoteAcknowledge(eventData, 'unack')
      }
    }
  },

  metaInfo () {
    let title = 'Loading task... - Kitsu'
    if (this.currentTask) {
      const taskTypeName =
        this.taskTypeMap.get(this.currentTask.task_type_id).name
      title = `${this.title} / ${taskTypeName} - Kitsu`
    }
    return { title }
  }
}
</script>

<style lang="scss" scoped>
.dark .page {
  background: $dark-grey-light;
  padding-bottom: 1em;
}

.dark .page-header,
.dark .add-comment,
.dark .comment,
.dark .no-comment,
.dark .preview-column-content,
.dark .column {
  background: #46494F;
  border-color: $dark-grey;
  box-shadow: 0px 0px 6px #333;
}

h2.subtitle {
  margin: 0;
  padding: 0;
}

.page {
  background: #F9F9F9;
  margin-top: 60px;
  padding: 0;
}

.page-header {
  padding: 1em;
  margin-top: 1em;
  background: white;
  box-shadow: 0px 0px 6px #E0E0E0;
  margin: 2em 1em 0 1em;
}

.navigation-buttons {
  font-size: 1em;
  margin-top: 0.5em;
  margin-bottom: 0.5em;
}

.navigation-buttons .arrow {
  font-size: 1.2em;
  font-weight: bold;
}

.navigation-buttons a {
  color: $grey;
}

.task-information {
  margin-top: 1em;
}

.selected {
  border: 0;
}

.source {
  color: #AAA;
  font-size: 0.8em;
}

video {
  width: 100%;
}

.validation-buttons button {
  width: 100%;
  margin-bottom: 0.3em;
  border-width: 2px;
  font-weight: bold;
}

.preview-row {
  margin-bottom: 0.5em;
}

.add-comment {
  margin-bottom: 1em;
  padding: 1em;
  box-shadow: 0px 0px 6px #E0E0E0;
}

.no-comment {
  background: white;
  box-shadow: 0px 0px 6px #E0E0E0;
  padding: 1em;
  border-radius: 5px;
}

.comment {
  box-shadow: 0px 0px 6px #E0E0E0;
  margin-top: 0.3em;
}

.task-columns {
  display: flex;
  flex-direction: row;
}

.task-column {
  width: 50%;
  padding: 1em;
}

.preview-column-content {
  background: white;
  box-shadow: 0px 0px 6px #E0E0E0;
  padding: 1em;
  border-radius: 5px;
  overflow-x: hidden;
}

.preview-list {
  display: flex;
  flex-wrap: wrap;
}

.page-header .tag {
  border-radius: 0;
  font-weight: bold;
  color: $grey-strong;
}

.assignees {
  display: flex;
}

.assignees span {
  margin-right: 0.2em;
}

.avatar-wrapper {
  margin-right: 0.5em;
}

.entity-thumbnail {
  width: 50px;
  margin-right: 0.3em;
}

.title {
  margin: 0;
  flex: 1;
}

.pull-right {
  margin-left: auto;
}

.title a {
  color: inherit;
}

.set-main-preview {
  height: 30px;
}

.back-link {
  padding-top: 6px;
}

@media screen and (max-width: 768px) {
  .action-button {
    display: none;
  }

  .title {
    font-size: 1.3em;
    line-height: 1.5em;
  }

  .header-title {
    flex-direction: column;
    align-items: flex-start;
  }

  .header-title .flexrow-item {
    margin-bottom: 0.5em;
  }

  .task-columns {
    flex-direction: column-reverse;
  }

  .task-column {
    width: 100%;
    overflow-y: initial;
  }
}
</style>
