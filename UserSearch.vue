<template lang="pug">
.search-wrapper
  .chip-wrapper(v-if="!single")
    v-chip(
      x-small
      close
      @click:close="remove(item)"
      v-for="item in selectedUsers"
      :key="item.id"
    )
      v-img(:src="userAvatar(item)" width="20")
      div {{ item.name }}
  v-autocomplete.ma-1(
    v-model="selectedUsers"
    :items="entries"
    :loading="isLoading"
    :search-input.sync="search"
    :disabled="disabled"
    hide-selected
    :label="label"
    :clearable="displayInSearch"
    :error-messages="errorMessage"
    item-value="name"
    :filter="customFilter"
    :multiple="!single"
    :menu-props="{ maxWidth: '200px', width: '200px' }"
    cache-items
    :dense="dense"
    :hide-details="hideDetails"
    :rules="rules"
    :no-data-text="isLoading ? 'ロード中' : '該当者が見つかりません'"
    placeholder="検索"
    return-object
    @click:clear="clearSelected()"
    @input="submitSelections()"
  )
    //- empty to prevent selections from appearing in search bar
    template(v-slot:selection="data")
      v-chip(v-if="single && displayInSearch")
        v-img(:src="userAvatar(data.item)" width="30")
        span {{ data.item.name }}
    template(v-slot:item="data")
      span.mr-1: v-img(:src="userAvatar(data.item)" width="30")
      span {{ data.item.name }}

    template(v-slot:append-item)
      observer(@intersect="queryApi(search)")
</template>
<script lang="ts">
import cloneDeep from 'lodash/cloneDeep'
import debounce from 'lodash/debounce'
import {
  defineComponent,
  computed,
  ref,
  watch,
  useContext,
  PropType,
  onMounted,
} from '@nuxtjs/composition-api'
import { User } from '~/utilities/user'

import Observer from '~/components/Elements/Observer.vue'

type FilterType = {
  availability: []
}

export default defineComponent({
  name: 'UserSearch',

  components: {
    Observer,
  },

  props: {
    currentlySelectedUsers: {
      type: Array as PropType<User[]>,
      default: () => [],
    },

    single: {
      type: Boolean,
      default: false,
    },

    displayInSearch: {
      type: Boolean,
      default: false,
    },

    disabled: {
      type: Boolean,
      default: false,
    },

    label: {
      type: String,
      default: 'スタッフ',
    },

    filter: {
      type: Object as PropType<FilterType>,
      default: () => ({}),
    },

    dense: {
      type: Boolean,
      default: false,
    },

    hideDetails: {
      type: Boolean,
      default: false,
    },

    errorMessage: {
      type: String,
      default: '',
    },

    rules: {
      type: Array as PropType<String[]>,
      default: () => [],
    },
  },

  emit: ['submit-selections'],

  setup(props, { emit }) {
    const search = ref<string>('')
    const isLoading = ref<boolean>(false)
    const selected = ref<User[]>([])
    const entries = ref<User[]>([])
    const currentPage = ref<number>(0)
    const lastPage = ref<number>(1)

    const selectedUsers = computed({
      get() {
        return selected.value
      },
      set(value: User[]) {
        selected.value = value
      },
    })

    onMounted(() => {
      if (props.currentlySelectedUsers) {
        selectedUsers.value = cloneDeep(props.currentlySelectedUsers)
      }
    })

    watch(
      search,
      debounce((value, oldValue) => {
        /**
         *  Reset the currentPage and lastPage
         *   whenever searching is done, since the
         *   hits and page numbers will change with
         *   each API call.
         */
        if (value !== oldValue) {
          currentPage.value = 0
          lastPage.value = 1
        }

        queryApi(value)
      }, 1000)
    )

    const context = useContext()
    const queryApi = async searchValue => {
      if (currentPage.value === lastPage.value) {
        return
      }

      currentPage.value += 1
      isLoading.value = true

      const searchParams = {
        keywords: searchValue,
        ...props.filter,
      }

      await context.$axios
        .$get(`/users?page=${currentPage.value}`, { params: searchParams })
        .then(res => {
          lastPage.value = res.lastPage
          entries.value = res.items
        })
        .catch(() => {
          context.app.$toast.error(
            'エラーがありました。管理者とお問い合わせください'
          )
        })
        .finally(() => (isLoading.value = false))
    }

    const customFilter = (item, queryText) => {
      const staffNumber = item.staffNumber
      const userName = item.name

      return staffNumber.includes(queryText) || userName.includes(queryText)
    }

    const remove = item => {
      selectedUsers.value = selectedUsers.value.filter(s => s.id !== item.id)
      submitSelections()
    }

    const clearSelected = () => {
      selectedUsers.value = []
    }

    const submitSelections = () => {
      emit('submit-selections', selectedUsers.value)
    }

    return {
      search,
      isLoading,
      selectedUsers,
      entries,
      currentPage,
      lastPage,
      queryApi,
      customFilter,
      remove,
      clearSelected,
      submitSelections,
    }
  },
})
</script>
<style lang="stylus" scoped>
.search-wrapper
  background-color white
  margin 0px
  padding 5px

.chip-wrapper
  max-width 250px
</style>
