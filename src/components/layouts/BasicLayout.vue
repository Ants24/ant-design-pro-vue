<template>
  <global-layout>
    <contextmenu :itemList="menuItemList" :visible.sync="menuVisible" @select="onMenuSelect"/>
    <div class="page-header-index-wide">
      <a-tabs
        @contextmenu.native="e => onContextmenu(e)"
        v-if="multiPage"
        :active-key="activePage"
        style="margin-top: -20px; margin-bottom: 8px; "
        :hide-add="true"
        type="editable-card"
        @change="changePage"
        @edit="editPage">
        <a-tab-pane :id="page.fullPath" :key="page.fullPath" v-for="page in pageList">
          <span slot="tab" :pagekey="page.fullPath">{{ page.meta.title }}</span>
        </a-tab-pane>
      </a-tabs>
    </div>
    <keep-alive :include="groupList" v-if="multiPage">
      <router-view/>
    </keep-alive>
    <keep-alive v-else-if="keepAlive">
      <router-view/>
    </keep-alive>
    <router-view v-else/>
  </global-layout>
</template>

<script>
import RouteView from '@/components/layouts/RouteView'
import GlobalLayout from '@/components/page/GlobalLayout'
import Contextmenu from '@/components/menu/Contextmenu'
import { mixin, mixinDevice } from '@/utils/mixin.js'

export default {
  name: 'BasicLayout',
  mixins: [mixin, mixinDevice],
  components: {
    RouteView,
    GlobalLayout,
    Contextmenu
  },

  data() {
    return {
      pageList: [],
      linkList: [],
      groupList: [],
      activePage: '',
      menuVisible: false,
      menuItemList: [
        { key: '1', icon: 'arrow-left', text: '关闭左侧' },
        { key: '2', icon: 'arrow-right', text: '关闭右侧' },
        { key: '3', icon: 'close', text: '关闭其它' }
      ]
    }
  },
  computed: {
    keepAlive () {
      return this.$route.meta.keepAlive
    },
    multiPage() {
      return this.$store.state.app.multiPage
    }
  },
  created() {
    this.pageList.push(this.$route)
    this.linkList.push(this.$route.fullPath)
    this.groupList.push(this.$route.meta.groupId === undefined ? this.$route.name : this.$route.meta.groupId)
    this.activePage = this.$route.fullPath
  },
  watch: {
    $route: function(newRoute) {

      if (!this.multiPage) {
        this.groupList = [newRoute.meta.groupId === undefined ? newRoute.name : newRoute.meta.groupId]
        this.linkList = [newRoute.fullPath]
        this.pageList = [newRoute]
      } else if (this.linkList.indexOf(newRoute.fullPath) < 0) {
        if (newRoute.meta.groupId === undefined) {
          this.groupList.push(newRoute.name)
          this.linkList.push(newRoute.fullPath)
          this.pageList.push(newRoute)
        } else {
          if (this.groupList.indexOf(newRoute.meta.groupId) < 0) {
            this.groupList.push(newRoute.meta.groupId)
            this.linkList.push(newRoute.fullPath)
            this.pageList.push(newRoute)
          } else {
            const index = this.groupList.indexOf(newRoute.meta.groupId)
            this.linkList.splice(index, 1, newRoute.fullPath)
            this.pageList.splice(index, 1, newRoute)
          }
        }
      }
      this.activePage = newRoute.fullPath
    },
    activePage: function(key) {
      this.$router.push(key)
    },
    multiPage: function(newVal) {
      if (!newVal) {
        this.linkList = [this.$route.fullPath]
        this.pageList = [this.$route]
        this.groupList = [this.$route.meta.groupId === undefined ? this.$route.name : this.$route.meta.groupId]
      }
    }
  },
  methods: {
    changePage(key) {
      this.activePage = key
    },
    editPage(key, action) {
      this[action](key)
    },
    remove(key) {
      if (this.pageList.length === 1) {
        this.$message.warning('这是最后一页，不能再关闭了啦')
        return
      }

      const filter = this.pageList.filter(item => item.fullPath === key)

      this.groupList = this.groupList.filter(item => item !== (filter[0].meta.groupId === undefined ? filter[0].name : filter[0].meta.groupId))
      this.pageList = this.pageList.filter(item => item.fullPath !== key)
      let index = this.linkList.indexOf(key)
      this.linkList = this.linkList.filter(item => item !== key)
      index = index >= this.linkList.length ? this.linkList.length - 1 : index
      this.activePage = this.linkList[index]

    },
    onContextmenu(e) {
      const pagekey = this.getPageKey(e.target)
      if (pagekey !== null) {
        e.preventDefault()
        this.menuVisible = true
      }
    },
    /**
     * 由于ant-design-vue组件库的TabPane组件暂不支持自定义监听器，无法直接获取到右键target所在标签页的 pagekey 。故增加此方法用于
     * 查询右键target所在标签页的标识 pagekey ，以用于自定义右键菜单的事件处理。
     * 注：TabPane组件支持自定义监听器后可去除该方法并重构 ‘自定义右键菜单的事件处理’
     * @param target 查询开始目标
     * @param count 查询层级深度 （查找层级最多不超过3层，超过3层深度直接返回 null）
     * @returns {String}
     */
    getPageKey(target, depth) {
      depth = depth || 0
      if (depth > 2) {
        return null
      }
      let pageKey = target.getAttribute('pagekey')
      pageKey =
        pageKey || (target.previousElementSibling ? target.previousElementSibling.getAttribute('pagekey') : null)
      return pageKey || (target.firstElementChild ? this.getPageKey(target.firstElementChild, ++depth) : null)
    },
    onMenuSelect(key, target) {
      const pageKey = this.getPageKey(target)
      switch (key) {
        case '1':
          this.closeLeft(pageKey)
          break
        case '2':
          this.closeRight(pageKey)
          break
        case '3':
          this.closeOthers(pageKey)
          break
        default:
          break
      }
    },
    closeOthers(pageKey) {
      const index = this.linkList.indexOf(pageKey)
      this.linkList = this.linkList.slice(index, index + 1)
      this.pageList = this.pageList.slice(index, index + 1)
      this.activePage = this.linkList[0]
    },
    closeLeft(pageKey) {
      const index = this.linkList.indexOf(pageKey)
      this.linkList = this.linkList.slice(index)
      this.pageList = this.pageList.slice(index)
      this.groupList = this.groupList.slice(index)
      if (this.linkList.indexOf(this.activePage) < 0) {
        this.activePage = this.linkList[0]
      }
    },
    closeRight(pageKey) {
      const index = this.linkList.indexOf(pageKey)
      this.linkList = this.linkList.slice(0, index + 1)
      this.pageList = this.pageList.slice(0, index + 1)
      this.groupList = this.groupList.slice(0, index + 1)
      if (this.linkList.indexOf(this.activePage < 0)) {
        this.activePage = this.linkList[this.linkList.length - 1]
      }
    }
  }
}
</script>

<style lang="less">

	/*
 * The following styles are auto-applied to elements with
 * transition="page-transition" when their visibility is toggled
 * by Vue.js.
 *
 * You can easily play with the page transition by editing
 * these styles.
 */

	.page-transition-enter {
		opacity: 0;
	}

	.page-transition-leave-active {
		opacity: 0;
	}

	.page-transition-enter .page-transition-container,
	.page-transition-leave-active .page-transition-container {
		-webkit-transform: scale(1.1);
		transform: scale(1.1);
	}
</style>
