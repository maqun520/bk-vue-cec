<!--
 * Tencent is pleased to support the open source community by making
 * 科技内在设计（T-inside） available.
 *
 * Copyright (C) 2021 THL A29 Limited, a Tencent company.  All rights reserved.
 *
 * 科技内在设计（T-inside） is licensed under the MIT License.
 *
 * License for 科技内在设计（T-inside）:
 *
 *
 * Terms of the MIT License:
 * ---------------------------------------------------
 * Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
 * documentation files (the "Software"), to deal in the Software without restriction, including without limitation
 * the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
 * to permit persons to whom the Software is furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in all copies or substantial portions of
 * the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT
 * THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
 * CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
 * IN THE SOFTWARE.
-->

<template>
    <ul class="bk-tree" :class="[{ 'bk-has-border-tree': isBorder }, extCls]">
        <li v-for="(item, index) in shownData"
            @drop="drop(item, $event)"
            @dragover="dragover($event)"
            :key="item[nodeKey] ? item[nodeKey] : item.name"
            :class="{
                'leaf': isLeaf(item),
                'tree-first-node': !parent && index === 0,
                'tree-only-node': !parent && shownData.length === 1,
                'tree-second-node': !parent && index === 1,
                'single': !multiple
            }">
            <div class="tree-drag-node"
                :draggable="draggable" @dragstart="drag(item, $event)">
                <span
                    @click="expandNode(item)"
                    v-if="!item.parent || item.children && item.children.length || item.async"
                    :class="['bk-icon', 'tree-expanded-icon', item.expanded ? 'icon-down-shape' : 'icon-right-shape']">
                </span>
                <label :class="[item.halfcheck ? 'bk-form-half-checked' : 'bk-form-checkbox','bk-checkbox-small', 'mr5']" v-if="multiple">
                    <input type="checkbox"
                        v-if="multiple"
                        :disabled="item.disabled"
                        v-model="item.checked"
                        @change.stop="changeCheckStatus(item, $event)">
                </label>
                <div class="tree-node" @click="triggerExpand(item)">
                    <span class="node-icon" v-if="isShowIcon">
                        <i class="bk-icon" :class="setNodeIcon(item)"></i>
                    </span>
                    <div class="bk-spin-loading bk-spin-loading-mini bk-spin-loading-primary loading" v-if="item.loading && item.expanded">
                        <div class="rotate rotate1"></div>
                        <div class="rotate rotate2"></div>
                        <div class="rotate rotate3"></div>
                        <div class="rotate rotate4"></div>
                        <div class="rotate rotate5"></div>
                        <div class="rotate rotate6"></div>
                        <div class="rotate rotate7"></div>
                        <div class="rotate rotate8"></div>
                    </div>
                    <Render :node="item" :tpl="tpl" />
                </div>
            </div>
            <collapse-transition>
                <bk-tree v-if="!isLeaf(item)"
                    @dropTreeChecked="nodeCheckStatusChange"
                    @async-load-nodes="asyncLoadNodes"
                    @on-expanded="onExpanded"
                    @on-click="onClick"
                    @on-check="onCheck"
                    @on-broadcast-check="onBroadcastCheck"
                    @on-drag-node="onDragNode"
                    :drag-after-expanded="dragAfterExpanded"
                    :draggable="draggable"
                    :drag-sort="dragSort"
                    v-show="item.expanded"
                    :tpl="tpl"
                    :data="item.children"
                    :halfcheck="halfcheck"
                    :parent="item"
                    :is-delete-root="isDeleteRoot"
                    :multiple="multiple"
                    :show-icon="showIcon">
                </bk-tree>
            </collapse-transition>
        </li>
    </ul>
</template>

<script>
    import Render from './render'
    import CollapseTransition from '@/utils/transition'
    import locale from 'bk-magic-vue/lib/locale'

    export default {
        name: 'bk-tree',
        components: { Render, CollapseTransition },
        mixins: [locale.mixin],
        props: {
            data: {
                type: Array,
                default: () => []
            },
            parent: {
                type: Object,
                default: () => null
            },
            multiple: {
                type: Boolean,
                default: false
            },
            nodeKey: {
                type: String,
                default: 'id'
            },
            draggable: {
                type: Boolean,
                default: false
            },
            hasBorder: {
                type: Boolean,
                default: false
            },
            dragAfterExpanded: {
                type: Boolean,
                default: true
            },
            isDeleteRoot: {
                type: Boolean,
                default: false
            },
            emptyText: {
                type: String,
                default: ''
            },
            dragSort: {
                type: Boolean,
                default: false
            },
            openedIcon: {
                type: String,
                default: 'icon-folder-open'
            },
            closedIcon: {
                type: String,
                default: 'icon-folder'
            },
            nodeIcon: {
                type: String,
                default: 'icon-file'
            },
            showIcon: {
                type: Boolean,
                default: true
            },
            tpl: Function,
            // 外部设置的 class name
            extCls: {
                type: String,
                default: ''
            }
        },
        data () {
            return {
                halfcheck: true,
                isBorder: this.hasBorder,
                bkTreeDrag: {},
                isDragSort: this.dragSort,
                isShowIcon: this.showIcon
            }
        },
        computed: {
            shownData () {
                return this.data.filter(item => item.hasOwnProperty('visible') ? item.visible : true)
            }
        },
        watch: {
            data () {
                this.initTreeData()
            },
            hasBorder (value) {
                this.isBorder = !!value
            },
            dragSort (value) {
                this.isDragSort = !!value
            },
            showIcon (value) {
                this.isShowIcon = !!value
            }
        },
        mounted () {
            /**
            * @event monitor 子节点 selected event
            */
            this.$on('childChecked', (node, checked) => {
                if (node.children && node.children.length) {
                    for (const child of node.children) {
                        if (!child.disabled) {
                            this.$set(child, 'checked', checked)
                        }
                        this.$emit('on-broadcast-check', child, checked)
                    }
                }
            })

            /**
            * @event monitor 父节点 selected event
            */
            this.$on('parentChecked', (node, checked) => {
                if (!node.parent) {
                    const allChildNodeChecked = node.children.every(node => node.checked)
                    const someChildNodeChecked = (node.children.some(node => node.halfcheck) || node.children.some(node => node.checked)) && !allChildNodeChecked
                    if (this.halfcheck) {
                        if (!node.disabled) {
                            this.$set(node, 'checked', allChildNodeChecked)
                            this.$set(node, 'halfcheck', someChildNodeChecked)
                        }
                    }
                    return false
                }

                if (!node.parent.children.filter(child => child[this.nodeKey] === node[this.nodeKey]).length && !node.disabled) {
                    this.$set(node, 'checked', checked)
                }

                const allBortherNodeChecked = node.parent.children.every(node => node.checked)
                const someBortherNodeChecked = (node.parent.children.some(node => node.halfcheck) || node.parent.children.some(node => node.checked)) && !allBortherNodeChecked
                if (this.halfcheck) {
                    if (allBortherNodeChecked) {
                        this.$set(node.parent, 'halfcheck', false)
                        this.$set(node.parent, 'checked', true)
                    } else {
                        if (someBortherNodeChecked) {
                            this.$set(node.parent, 'halfcheck', true)
                            this.$set(node.parent, 'checked', false)
                        } else {
                            this.$set(node.parent, 'halfcheck', false)
                            this.$set(node.parent, 'checked', false)
                        }
                    }
                    if (!checked && someBortherNodeChecked) {
                        this.$set(node.parent, 'halfcheck', true)
                    }
                    this.$emit('parentChecked', node.parent, checked)
                } else {
                    if (checked && allBortherNodeChecked) this.$emit('parentChecked', node.parent, checked)
                    if (!checked) this.$emit('parentChecked', node.parent, checked)
                }
            })

            /**
            * @event monitor 节点 selected event on-broadcast-check
            */
            this.$on('on-broadcast-check', (node, checked) => {
                // 根节点下无子节点需异步加载数据时 不进行check事件的传递
                if (!node.parent && !node.children) {
                    return
                }
                this.$emit('parentChecked', node, checked)
                this.$emit('childChecked', node, checked)
                this.$emit('dropTreeChecked', node, checked)
            })

            /**
            * @event monitor 节点过滤时 可见/不可见 visible event
            */
            this.$on('toggleshow', (node, isShow) => {
                this.$set(node, 'visible', isShow)
                if (isShow && node.parent) {
                    this.$emit('toggleshow', node.parent, isShow)
                }
            })
            this.$on('cancelSelected', (root) => {
                for (const child of root.$children) {
                    for (const node of child.data) {
                        child.$set(node, 'selected', false)
                    }
                    if (child.$children) child.$emit('cancelSelected', child)
                }
            })
            this.initTreeData()
        },
        destroyed () {
            this.$delete(window, 'bkTreeDrag')
        },
        methods: {
            /**
             * 拖拽时生成随机guid将节点暂存window['bkTreeDrag'][guid]上
             */
            gid () {
                return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
                    const r = Math.random() * 16 | 0
                    const v = c === 'x' ? r : (r & 0x3 | 0x8)
                    return v.toString(16)
                })
            },

            /**
             * 设置拖拽节点
             *
             * @param {String} id 节点挂载id
             * @param {Object} node 需要设置的节点
             */
            setDragNode (id, node) {
                window['bkTreeDrag'] = {}
                window['bkTreeDrag'][id] = node
            },

            /**
             * 获取拖拽节点
             *
             * @param {String} id 节点挂载id
             */
            getDragNode (id) {
                return window['bkTreeDrag'][id]
            },

            /**
             * 节点是否可拖拽到某个节点下的标识
             *
             * @param {Object} root 根节点
             * @param {Object} node 当前节点
             */
            hasInGenerations (root, node) {
                if (root.hasOwnProperty('children') && root.children) {
                    for (const rn of root.children) {
                        if (rn === node) return true
                        if (rn.children) return this.hasInGenerations(rn, node)
                    }
                    return false
                }
            },

            /**
             * 节点是否可拖拽排序标识
             *
             * @param {Object} target 目标节点
             * @param {Object} node 当前节点
             */
            isDragSortHolder (target, node) {
                if (target.hasOwnProperty('children') && target.children) {
                    for (const rn of target.children) {
                        if (rn[this.nodeKey] === node[this.nodeKey]) return false
                        if (rn.children) return this.isDragSortHolder(rn, node)
                    }
                    return true
                }
            },

            /**
             * 设置节点icon(父节点和子节点的传参不同)
             *
             * @param {Object} node 当前节点
             */
            setNodeIcon (node) {
                if (node.children && node.children.length) {
                    if (node.expanded) {
                        return node.openedIcon || this.openedIcon
                    } else {
                        return node.closedIcon || this.closedIcon
                    }
                } else {
                    return node.icon || this.nodeIcon
                }
            },

            /**
            * 节点拖拽
            *
            * @param {Object} node 当前拖拽节点
            * @param {Object} ev   $event
            */
            drop (node, ev) {
                ev.preventDefault()
                ev.stopPropagation()
                const gid = ev.dataTransfer.getData('gid')
                const drag = this.getDragNode(gid)
                // console.warn(drag)
                // if drag node's parent is enter node or root node
                if (drag.parent === node || drag.parent === null || drag === node) {
                    return false
                }
                // drag from parent node to child node
                if (this.hasInGenerations(drag, node)) {
                    return false
                }

                // 开启 dragSort 后，拖拽节点仅仅是位置交互，不会改变层级
                if (this.isDragSort) {
                    if ([true, false].includes(this.isDragSortHolder(node, drag))) {
                        if (!this.isDragSortHolder(node, drag)) {
                            return false
                        }
                    }
                    const dragNodeIndex = drag.parent.children.findIndex(dragItem => dragItem[this.nodeKey] === drag[this.nodeKey])

                    const targetNodeIndex = node.parent.children.findIndex(targetItem => targetItem[this.nodeKey] === node[this.nodeKey])

                    drag.parent.children.splice(dragNodeIndex, 1, node)

                    node.parent.children.splice(targetNodeIndex, 1, drag)

                    node.parent.children[targetNodeIndex].parent = drag.parent

                    if (this.multiple) {
                        this.$emit('on-broadcast-check', drag, drag.checked)
                    }

                    drag.parent.children[dragNodeIndex].parent = node.parent

                    if (this.multiple) {
                        this.$emit('on-broadcast-check', node, node.checked)
                    }
                } else {
                    const dragHost = drag.parent.children
                    if (node.children && node.children.indexOf(drag) === -1) {
                        node.children.push(drag)

                        if (node.children.length && this.isShowIcon) {
                            this.$set(node, 'openedIcon', this.openedIcon)
                            this.$set(node, 'closedIcon', this.closedIcon)
                        }

                        dragHost.splice(dragHost.indexOf(drag), 1)

                        if (this.isShowIcon) {
                            if (drag.parent.children && drag.parent.children.length) {
                                this.$set(drag.parent, 'openedIcon', this.openedIcon)
                                this.$set(drag.parent, 'closedIcon', this.closedIcon)
                            } else {
                                this.$set(drag.parent, 'icon', this.nodeIcon)
                            }
                        }

                        node.children[node.children.length - 1].parent = node
                    } else {
                        if (this.isShowIcon) {
                            this.$set(node, 'openedIcon', this.openedIcon)
                            this.$set(node, 'closedIcon', this.closedIcon)
                        }

                        this.$set(node, 'children', [drag])
                        dragHost.splice(dragHost.indexOf(drag), 1)
                    }
                    this.$set(node, 'expanded', this.dragAfterExpanded)
                }
                this.$emit('on-drag-node', { dragNode: drag, targetNode: node })
            },

            drag (node, ev) {
                const gid = this.gid()
                this.setDragNode(gid, node)
                ev.dataTransfer.setData('gid', gid)
            },

            /**
            * 节点拖拽 阻止默认事件和禁止冒泡
            *
            * @param {Object} ev $event
            */
            dragover (ev) {
                ev.preventDefault()
                ev.stopPropagation()
            },

            /*
            * 数据初始化
            */
            initTreeData () {
                for (const node of this.data) {
                    this.$set(node, 'parent', this.parent)
                    if (node.children && node.children.length) {
                        if (node.hasOwnProperty('icon')) {
                            this.$delete(node, 'icon')
                        }
                    } else {
                        if (node.hasOwnProperty('openedIcon')) {
                            this.$delete(node, 'openedIcon')
                        }
                        if (node.hasOwnProperty('closedIcon')) {
                            this.$delete(node, 'closedIcon')
                        }
                    }
                    if (this.multiple) {
                        if (node.hasOwnProperty('selected')) {
                            this.$delete(node, 'selected')
                        }
                        if (node.hasOwnProperty('checked') && node.checked) {
                            this.$emit('on-broadcast-check', node, true)
                        }
                    } else {
                        if (node.hasOwnProperty('checked')) {
                            this.$delete(node, 'checked')
                        }
                    }
                }
            },

            /**
            * 节点展开/收起
            *
            * @param {Object} node 当前节点
            */
            expandNode (node) {
                this.$set(node, 'expanded', !node.expanded)
                if (node.async && !node.children) {
                    this.$emit('async-load-nodes', node)
                }
                if (node.children && node.children.length) {
                    this.$emit('on-expanded', node, node.expanded)
                }
            },

            /**
            * 节点展开/收起 绑定事件
            *
            * @param {Object} node 当前节点
            */
            onExpanded (node) {
                if (node.children && node.children.length) {
                    this.$emit('on-expanded', node, node.expanded)
                }
            },

            /**
            * 节点展开/收起
            *
            * @param {Object} node 当前节点
            */
            triggerExpand (item) {
                if (!item.parent || (item.children && item.children.length) || item.async) {
                    this.expandNode(item)
                }
            },

            /**
            * 异步加载节点
            *
            * @param {Object} node 当前点击节点
            */
            asyncLoadNodes (node) {
                if (node.async && !node.children) {
                    this.$emit('async-load-nodes', node)
                }
            },

            /**
            * 判断是否子节点
            *
            * @param {Object} node 当前节点
            */
            isLeaf (node) {
                return !(node.children && node.children.length) && node.parent && !node.async
            },

            /**
            * 添加单个节点
            *
            * @param {Object} parent 父节点
            * @param {Object} newnode  新节点
            */
            addNode (parent, newNode) {
                let addnode = {}
                this.$set(parent, 'expanded', true)
                if (typeof newNode === 'undefined') {
                    throw new ReferenceError('newNode is required but undefined')
                }
                if (typeof newNode === 'object' && !newNode.hasOwnProperty('name')) {
                    throw new ReferenceError('the name property is missed')
                }
                if (typeof newNode === 'object' && !newNode.hasOwnProperty(this.nodeKey)) {
                    throw new ReferenceError('the nodeKey property is missed')
                }
                if (typeof newNode === 'object' && newNode.hasOwnProperty('name') && newNode.hasOwnProperty(this.nodeKey)) {
                    addnode = Object.assign({}, newNode)
                }
                if (this.isLeaf(parent)) {
                    this.$set(parent, 'children', [])
                    parent.children.push(addnode)
                } else {
                    parent.children.push(addnode)
                }
                this.$emit('addNode', { parentNode: parent, newNode: newNode })
            },

            /**
            * 添加多个节点
            *
            * @param {Object} parent 父节点
            * @param {Array} newChildren  子节点数组
            */
            addNodes (parent, newChildren) {
                for (const n of newChildren) {
                    this.addNode(parent, n)
                }
            },

            /**
            * 获取搜索时返回结果
            */
            getSearchResult () {
                const list = []
                for (let i = 0; i < this.data.length; i++) {
                    const queue = []
                    queue.push(this.data[i])
                    while (queue.length) {
                        const node = queue.shift()
                        if (node.children && node.children.length) {
                            const tempList = []
                            tempList.splice(0, 0, ...node.children)
                            queue.push(...tempList)
                        }
                        list.push(node)
                    }
                }
                return {
                    results: list.filter(item => item.hasOwnProperty('visible') && item.visible),
                    isEmpty: list.every(item => item.hasOwnProperty('visible') && !item.visible)
                }
            },

            /**
            * 节点点击事件
            *
            * @param {Object} node 当前点击节点
            */
            onClick (node) {
                this.$emit('on-click', node)
            },

            /**
            * 节点复选框 change 事件 不触发传递给子节点
            *
            * @param {Object} node 当前节点
            * @param {Boolean} checked 选中状态
            */
            onCheck (node, checked) {
                this.$emit('on-check', node, checked)
            },

            /**
            * 节点复选框 change 事件 触发传递给子节点
            *
            * @param {Object} node 当前节点
            * @param {Boolean} checked 选中状态
            */
            onBroadcastCheck (node, checked) {
                this.$emit('on-broadcast-check', node, checked)
            },

            /**
            * 复选框状态改变时告知父组件
            *
            * @param {Object} node 改变状态的节点
            * @param {Boolean} checked 选中/非选中
            */
            nodeCheckStatusChange (node, checked) {
                this.$emit('dropTreeChecked', node, checked)
            },

            /**
            * 拖拽结束事件
            *
            * @param {Object} event $event
            */
            onDragNode (event) {
                this.$emit('on-drag-node', event)
            },

            /**
            * 删除节点
            *
            * @param {Object} parent 父节点
            * @param {Object} node 当前节点
            */
            delNode (parent, node) {
                if (parent === null || typeof parent === 'undefined') {
                    // isDeleteRoot 为false时不可删除根节点
                    if (this.isDeleteRoot) {
                        this.data.splice(0, 1)
                    } else {
                        throw new ReferenceError('the root element can\'t deleted!')
                    }
                } else {
                    parent.children.splice(parent.children.indexOf(node), 1)
                }
                this.$emit('delNode', { parentNode: parent, delNode: node })
            },

            /**
            * 复选框change事件
            *
            * @param {Object} node 当前节点
            * @param {Object} event $event
            */
            changeCheckStatus (node, $event) {
                this.$emit('on-check', node, $event.target.checked)
                this.$emit('on-broadcast-check', node, $event.target.checked)
            },

            /**
            * 节点selected
            *
            * @param {Object} node 当前节点
            */
            nodeSelected (node) {
                const getRoot = (el) => {
                    if (el.$parent.$el.nodeName === 'UL') {
                        el = el.$parent
                        return getRoot(el)
                    }
                    return el
                }
                const root = getRoot(this)
                if (!this.multiple) {
                    for (const rn of root.data || []) {
                        this.$set(rn, 'selected', false)
                        this.$emit('cancelSelected', root)
                    }
                }
                // 当为多选时 必须通过选择复选框触发
                // if (this.multiple) this.$set(node, 'checked', !node.selected)
                this.$set(node, 'selected', !node.selected)
                this.$emit('on-click', node)
            },

            /**
            * 节点数据处理
            *
            * @param {Object} opt 参数设置
            * @param {Array}  data 根节点或子节点数组
            * @param {Array/String}  keyParton 自定义键值
            */
            nodeDataHandler (opt, data, keyParton) {
                data = data || this.data
                let res = []
                const keyValue = keyParton
                for (const node of data) {
                    for (const [key, value] of Object.entries(opt)) {
                        if (node[key] === value) {
                            if (!keyValue.length || !keyValue) {
                                const n = Object.assign({}, node)
                                delete n['parent']
                                if (!(n.children && n.children.length)) {
                                    res.push(n)
                                }
                            } else {
                                const n = {}
                                if (Object.prototype.toString.call(keyValue) === '[object Array]') {
                                    for (let i = 0; i < keyValue.length; i++) {
                                        if (node.hasOwnProperty(keyValue[i])) {
                                            n[keyValue[i]] = node[keyValue[i]]
                                        }
                                    }
                                }
                                if (Object.prototype.toString.call(keyValue) === '[object String]') {
                                    n[keyValue] = node[keyValue]
                                }
                                if (!(node.children && node.children.length)) {
                                    res.push(n)
                                }
                            }
                        }
                    }
                    if (node.children && node.children.length) {
                        res = res.concat(this.nodeDataHandler(opt, node.children, keyValue))
                    }
                }
                return res
            },

            /**
            * 获取所需节点
            *
            * @param {Array/String}  keyParton 自定义键值
            */
            getNode (keyParton) {
                if (!this.multiple) {
                    return this.nodeDataHandler({ selected: true }, this.data, keyParton)
                } else {
                    return this.nodeDataHandler({ checked: true }, this.data, keyParton)
                }
            },

            /**
            * 节点过滤
            *
            * @param {String/Function} filter 过滤器
            * @param {Object} data 所需过滤的数据
            */
            searchNode (filter, data) {
                data = data || this.data
                for (const node of data) {
                    const searched = filter ? (typeof filter === 'function' ? filter(node) : node['name'].indexOf(filter) > -1) : false
                    this.$set(node, 'searched', searched)
                    this.$set(node, 'visible', false)
                    this.$emit('toggleshow', node, filter ? searched : true)
                    if (node.children && node.children.length) {
                        if (searched) {
                            this.$set(node, 'expanded', true)
                        }
                        this.searchNode(filter, node.children)
                    }
                }
            }
        }
    }
</script>

<style>
    @import '../../ui/tree.css';
</style>
