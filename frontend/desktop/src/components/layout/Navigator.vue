/**
* Tencent is pleased to support the open source community by making 蓝鲸智云PaaS平台社区版 (BlueKing PaaS Community
* Edition) available.
* Copyright (C) 2017-2020 THL A29 Limited, a Tencent company. All rights reserved.
* Licensed under the MIT License (the "License"); you may not use this file except in compliance with the License.
* You may obtain a copy of the License at
* http://opensource.org/licenses/MIT
* Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
* an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
* specific language governing permissions and limitations under the License.
*/
<template>
    <header>
        <!-- 轻应用打开页面，logo不支持单击和右键跳转到首页 -->
        <span v-if="view_mode === 'appmaker'" class="nav-logo">
            <img :src="logo" class="logo" />
            <span class="header-title">{{ $t('标准运维') }}</span>
        </span>
        <router-link v-else :to="{ name: 'home' }" class="nav-logo" @click.native="onLogoClick">
            <img :src="logo" class="logo" />
            <span class="header-title">{{ $t('标准运维') }}</span>
        </router-link>
        <ul class="nav-left" v-if="!appmakerDataLoading">
            <li
                v-for="(item, index) in routerList"
                :key="index"
                :class="['nav-item', { 'active': isNavActived(item) }]">
                <router-link :to="getNavPath(item)" :event="''" @click.native="onGoToPath(item)">
                    {{ item.name }}
                </router-link>
                <div
                    v-if="item.children"
                    class="sub-nav">
                    <template v-for="(sub, subIndex) in item.children">
                        <router-link
                            v-if="isSubRouteShow(sub)"
                            :key="subIndex"
                            :to="{ name: item.routerName }"
                            :class="['sub-nav-item', { 'active': isNavActived(sub) }]"
                            :event="''"
                            @click.native="onGoToPath(sub)">
                            {{ sub.name }}
                        </router-link>
                    </template>
                </div>
            </li>
        </ul>
        <!-- 右侧项目选择和其他信息 -->
        <ul class="nav-right">
            <li v-if="!isProjectHidden" class="project-select">
                <ProjectSelector
                    :show="!isProjectHidden"
                    :read-only="isProjectReadOnly"
                    @reloadHome="reloadHome">
                </ProjectSelector>
            </li>
            <li class="right-icon help-doc">
                <a
                    class="common-icon-help"
                    :href="bkDocUrl"
                    target="_blank"
                    v-bk-tooltips="{
                        content: $t('帮助文档'),
                        placement: 'bottom-end',
                        theme: 'light',
                        zIndex: 1001
                    }">
                </a>
            </li>
            <li class="right-icon version-log">
                <i
                    class="bk-icon  icon-info-circle-shape"
                    v-bk-tooltips="{
                        content: $t('版本日志'),
                        placement: 'bottom-end',
                        theme: 'light',
                        zIndex: 1001
                    }"
                    @click="onOpenVersion">
                </i>
            </li>
            <li class="right-icon user-avatar">
                <span
                    class="common-icon-dark-circle-avatar"
                    v-bk-tooltips="{
                        content: username,
                        placement: 'bottom-end',
                        theme: 'light',
                        zIndex: 1001
                    }">
                </span>
            </li>
        </ul>
        <!-- 日志组件 -->
        <version-log
            ref="versionLog"
            :log-list="logList"
            :log-detail="logDetail"
            :loading="logListLoading || logDetailLoading"
            @active-change="handleVersionChange">
        </version-log>
    </header>
</template>
<script>
    import i18n from '@/config/i18n/index.js'
    import { mapState, mapActions } from 'vuex'
    import tools from '@/utils/tools.js'
    import { errorHandler } from '@/utils/errorHandler.js'
    import ProjectSelector from './ProjectSelector.vue'
    import VersionLog from './VersionLog.vue'

    const ROUTE_LIST = [
        {
            routerName: 'process',
            name: i18n.t('项目流程'),
            path: '/template/',
            params: ['project_id']
        },
        {
            routerName: 'taskList',
            name: i18n.t('任务管理'),
            path: '/taskflow/',
            params: ['project_id']
        },
        {
            routerName: 'appMakerList',
            name: i18n.t('轻应用'),
            path: '/appmaker/',
            params: ['project_id']
        },
        {
            routerName: 'commonProcessList',
            name: i18n.t('公共流程'),
            path: '/common'
        },
        {
            routerName: 'functionHome',
            path: '/function/',
            name: i18n.t('职能化')
        },
        {
            routerName: 'auditHome',
            path: '/audit/',
            name: i18n.t('审计中心')
        },
        {
            routerName: 'projectHome',
            path: '/project/',
            name: i18n.t('项目管理')
        },
        {
            path: '/admin',
            name: i18n.t('管理员入口'),
            children: [
                {
                    routerName: 'adminSearch',
                    path: '/admin/manage/',
                    name: i18n.t('后台管理')
                },
                {
                    routerName: 'statisticsTemplate',
                    path: '/admin/statistics/',
                    name: i18n.t('运营数据')
                },
                {
                    routerName: 'atomDev',
                    path: '/admin/atomdev',
                    name: i18n.t('插件开发')
                }
            ]
        }
    ]
    const APPMAKER_ROUTER_LIST = [
        {
            routerName: 'appmakerTaskCreate',
            path: 'appmakerTaskCreate',
            params: ['app_id', 'project_id'],
            name: i18n.t('新建任务')
        },
        {
            routerName: 'appmakerTaskHome',
            path: 'appmakerTaskHome',
            params: ['project_id'],
            name: i18n.t('任务记录')
        }
    ]
    export default {
        inject: ['reload'],
        name: 'Navigator',
        components: {
            ProjectSelector,
            VersionLog
        },
        props: {
            appmakerDataLoading: Boolean
        },
        data () {
            return {
                logo: require('../../assets/images/logo/logo_icon.svg'),
                bkDocUrl: window.BK_DOC_URL,
                logList: [],
                logDetail: '',
                logListLoading: false,
                logDetailLoading: false
            }
        },
        computed: {
            ...mapState({
                site_url: state => state.site_url,
                view_mode: state => state.view_mode,
                username: state => state.username,
                app_id: state => state.app_id,
                notFoundPage: state => state.notFoundPage,
                hasAdminPerm: state => state.hasAdminPerm,
                hasStatisticsPerm: state => state.hasStatisticsPerm,
                permissionMeta: state => state.permissionMeta
            }),
            ...mapState('appmaker', {
                appmakerTemplateId: state => state.appmakerTemplateId
            }),
            ...mapState('project', {
                projectList: state => state.userProjectList,
                project_id: state => state.project_id,
                authResource: state => state.authResource
            }),
            // 隐藏右侧项目信息
            isProjectHidden () {
                if (this.view_mode !== 'appmaker' && this.projectList.length === 0) {
                    return true
                }
                const paths = ['/home', '/common/', '/admin/', '/project/', '/function/home', '/audit/home']
                return paths.some(path => this.$route.path.startsWith(path))
            },
            // 项目名称只读
            isProjectReadOnly () {
                if (this.view_mode === 'appmaker') {
                    return true
                }
                const paths = ['/audit/', '/function/']
                return paths.some(path => this.$route.path.startsWith(path))
            },
            // 页面展示的导航项
            routerList () {
                if (this.view_mode === 'appmaker') {
                    return APPMAKER_ROUTER_LIST
                }
                return ROUTE_LIST
            }
        },
        mounted () {
            this.initNavgator()
        },
        methods: {
            ...mapActions('project', [
                'loadUserProjectList'
            ]),
            ...mapActions([
                'getVersionList',
                'getVersionDetail'
            ]),
            onLogoClick (e) {
                e.preventDefault()
                if (this.view_mode !== 'app') {
                    return false
                }
                if (this.$route.name === 'home') {
                    return this.reload()
                }
                this.$router.push({ name: 'home' })
            },
            async initNavgator () {
                if (this.view_mode !== 'appmaker') {
                    await this.loadUserProjectList({ limit: 0 })
                }
            },
            // 获取导航的链接，供右键点击跳转
            getNavPath (route) {
                const params = {}
                const query = {}
                route.params && route.params.forEach(m => {
                    params[m] = this[m] || undefined
                })
                route.query && route.query.forEach(m => {
                    query[m] = this[m] || undefined
                })
                if (route.routerName === 'appmakerTaskCreate') {
                    params['step'] = 'selectnode'
                    query['template_id'] = this.appmakerTemplateId
                }
                return {
                    name: route.routerName,
                    params,
                    query
                }
            },
            isSubRouteShow (route) {
                if (!this.hasAdminPerm && route.routerName === 'adminSearch') {
                    return false
                }
                if (!this.hasStatisticsPerm && route.routerName === 'statisticsTemplate') {
                    return false
                }
                return true
            },
            /**
             * 左键点击导航跳转
             * @param {Object} route 路由信息
             */
            onGoToPath (route) {
                const config = this.getNavPath(route)
                if (route.children && route.children.length > 0) {
                    return
                }
                if (this.$route.name === route.routerName && tools.isDataEqual(this.$route.query, config.query)) {
                    return this.reload()
                }
                /** 404 页面时，导航统一跳转到首页 */
                if (this.notFoundPage && this.view_mode === 'app') {
                    return this.$router.push({ name: 'home' })
                }
                if (config.name) {
                    this.$router.push(config)
                }
            },
            /* 打开版本日志 */
            async onOpenVersion () {
                this.$refs.versionLog.show()
                try {
                    this.logListLoding = true
                    const res = await this.getVersionList()
                    this.logList = res.data
                } catch (error) {
                    errorHandler(error, this)
                } finally {
                    this.logListLoding = false
                }
            },
            async loadLogDetail (version) {
                try {
                    this.logDetailLoding = true
                    const res = await this.getVersionDetail({ version })
                    this.logDetail = res.data
                } catch (error) {
                    errorHandler(error, this)
                } finally {
                    this.logDetailLoding = false
                }
            },
            isNavActived (route) {
                // 轻应用打开页面导航选中态
                if (this.view_mode === 'appmaker') {
                    // 任务记录、任务执行两个模块都激活任务记录导航项
                    if (route.routerName === 'appmakerTaskHome') {
                        return ['appmakerTaskHome', 'appmakerTaskExecute'].includes(this.$route.name)
                    }
                    return this.$route.name === route.path
                }
                return this.$route.path.indexOf(route.path) === 0
            },
            handleVersionChange (data) {
                const version = data[0]
                this.loadLogDetail(version)
            },
            reloadHome () {
                this.reload()
            }
        }
    }
</script>
<style lang="scss" scoped>
@import '@/scss/config.scss';
header {
    min-width: 1334px;
    padding: 0 25px;
    height: 50px;
    font-size: 14px;
    background: #182131;
    .nav-logo {
            float: left;
            height: 100%;
            line-height: 50px;
            .logo,.header-title{
                display: inline-block;
                vertical-align: middle;
            }
            .logo {
                width: 28px;
                height: 28px;
            }
            .header-title{
                margin-left: 6px;
                font-size: 18px;
                color: #979ba5;
            }
        }
    .nav-left {
        float: left;
        .nav-item {
            position: relative;
            float: left;
            &:first-child {
                margin-left: 141px;
                @media screen and (max-width: 1420px){
                    margin-left: 38px;
                }
            }
            &:hover {
                color: $whiteDefault;
                .sub-nav {
                    display: block;
                }
            }
            &.active > a{
                color: $whiteDefault;
            }
            > a {
                display: inline-block;
                padding: 0 17px;
                min-width: 90px;
                height: 50px;
                line-height: 50px;
                color: #979ba5;;
                text-align: center;
                border-radius: 2px;
                cursor: pointer;
                transition: all .5s linear;
                &:hover {
                    color: $whiteDefault;
                }
            }
            /*二级导航*/
            .sub-nav {
                display: none;
                position: absolute;
                top: 50px;
                left: 0;
                min-width: 100%;
                background: $whiteDefault;
                border: 1px solid #c4c6cc;
                border-radius: 2px;
                box-shadow: 0 2px 6px rgba(0, 0, 0, 0.16);
                z-index: 1001;
                .sub-nav-item {
                    width: 100%;
                    display: block;
                    padding: 0 10px;
                    height: 35px;
                    line-height: 35px;
                    color: #63656E;
                    white-space: nowrap;
                    text-align: center;
                    &.selected {
                        color: #313238;
                        background: #f0f1f5;
                    }
                    &:hover {
                        color: #313238;
                        background: #f0f1f5;
                    }
                    &.active {
                        color: #313238;
                        background: #f0f1f5;
                    }
                }
            }
        }
    }
    .nav-right {
        float: right;
        .project-select {
            float: left;
        }
        .right-icon {
            float: left;
            height: 50px;
            & > [class^='common-icon'] {
                margin-top: 17px;
                font-size: 16px;
                display: inline-block;
                color: #63656e;
                cursor: pointer;
                &:hover {
                    color: #616d7d;
                }
            }
        }
        .help-doc {
            margin-left: 18px;
        }
        .version-log {
            margin-left: 10px;
            & > .bk-icon{
                margin-top: 16px;
                font-size: 18px;
                display: inline-block;
                color: #63656e;
                cursor: pointer;
                &:hover {
                    color: #616d7d;
                }
            }
        }
        .user-avatar {
            margin-left: 10px;
            .common-icon-dark-circle-avatar {
                cursor: text;
            }
        }
        /deep/ .bk-select.is-disabled {
            background: none;
        }
    }
}
</style>
