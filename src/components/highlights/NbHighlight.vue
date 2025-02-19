<template>
    <g
        class="nb-highlight"
        v-if="visible"
        :style="style"
        @click="onClick()"
        @mouseenter="onHover(true)"
        @mouseleave="onHover(false)"
        v-tooltip="{
            content: getTooltipContent(),
        }"
    >
        <rect
            v-for="(box, index) in bounds.boxes"
            :key="index"
            :x="box.left + bounds.offsetX"
            :y="box.top + bounds.offsetY"
            :height="box.height"
            :width="box.width">
            <animate
                v-if="showRecentActivityAnimation"
                attributeType="XML"
                attributeName="fill"
                values="#ffffff;#4a2270D9;#ffffff;#ffffff"
                dur="5.0s"
                repeatCount="indefinite"/>
            <animate
                v-if="showTypingActivityAnimation"
                attributeType="XML"
                attributeName="fill"
                values="#ffffff;#4a2270D9;#ffffff;#ffffff"
                dur="2.0s"
                repeatCount="indefinite"/>
        </rect>

    </g>
    <g
        class="nb-highlight"
        v-else
        :style="style"
        @click="$emit('select-thread',thread)"
        @mouseenter="onHover(true)"
        @mouseleave="onHover(false)">
        <rect
            v-for="(box, index) in bounds.boxes"
            :key="index"
            :x="bounds.offsetX"
            :y="box.top + bounds.offsetY"
            :height="box.height"
            width="10">
        </rect>
    </g>
</template>

<script>
import { getTextBoundingBoxes } from '../../utils/overlay-util.js'
import {RgbList} from '../../utils/highlight-util.js'
import axios from 'axios'

/**
 * Component for individual highlight overlay corresponding to each thread.
 * Each thread is represented by the head of thread {@link NbComment}.
 *
 * @vue-prop {?NbComment} thread - thread for this highlight,
 *   null if this is a draft
 * @vue-prop {NbComment} threadSelected - currently selected thread
 * @vue-prop {Array<NbComment>} threadsHovered=[] - currently hovered threads
 * @vue-prop {NbRange} range - text range for this higlight
 * @vue-prop {Boolean} showHighlights=true - true if highlights are overlayed
 *   on text, false if collapsed to the side
 *
 * @vue-computed {String} style - additional CSS for highlight color
 *   in case this is a draft, selected, or hovered
 * @vue-computed {Object} bounds - bounds of highlight
 * @vue-computed {Array<Rect>} bounds.boxes - bouding box rectangles of the
 *   text range, calculated by {@link getTextBoundingBoxes}
 * @vue-computed {Number} bounds.offsetX - x coordinate offset of highlight
 * @vue-computed {Number} bounds.offsetY - y coordinate offset of highlight
 * @vue-computed {Boolan} visible - true if this highlight should be overlayed
 *   on text (i.e. showHighlights is true or this thread is selected)
 *
 * @vue-event {NbComment} select-thread - Emit this thread when user clicks on
 *   this highlight
 * @vue-event {NbComment} hover-thread - Emit this thread when user starts
 *   hovering over this highlight
 * @vue-event {NbComment} unhover-thread - Emit this thread when user stops
 *   hovering over this highlight
 */
export default {
    name: 'nb-highlight',
    props: {
        thread: Object,
        user: Object,
        threadSelected: Object,
        threadsHovered: {
            type: Array,
            default: () => []
        },
        range: Object,
        showHighlights: {
            type: Boolean,
            default: true
        },
        heatmapMode: {
            type: String,
            default: "Default"
        },
        activeClass: {
            type: Object,
            default: () => {}
        },
        showSyncFeatures: {
            type: Boolean,
            default: false
        }, 
        currentConfigs: {
            type: Object,
            default: () => {}
        },
        hashtags: {
            type: Object,
            default: () => {}
        },
    },
    data () {
        return {
        recent: false
        }
    },
    mounted () {
        if (this.thread) {
            const totalTime = 60000
            let inView = true
            let rect = this.$el.getBoundingClientRect()
            let elTop = rect.top
            let elHeight = rect.height
            let viewHeight = window.innerHeight
            if ((elTop + elHeight) > viewHeight) { // past the user's location (true if before the user location)
                inView = false
            }
            let timeDiff = Date.now() - this.thread.getMostRecentTimeStamp()
            if (timeDiff < totalTime) {
                this.recent = true // show blue highlighting for any comments that were recently posted ~
                if (inView) {
                    this.$emit('new-recent-thread', this.thread) // emit to display notification if in view of current user
                } 
                setTimeout(() => {
                    this.recent = false // set back to false after the time diff is over
                }, totalTime-timeDiff) // we still have 60 seconds - time diff left to display this recent annotation
            }
        }
    },
    watch: {
        /**
        * When the currently selected thread changes, check if the highlight is
        * in the view. If not, scroll down/up the window to center the highlight.
        */
        threadSelected: function (val) {
            if (this.thread !== val) { return }
            let rect = this.$el.getBoundingClientRect()
            let elTop = rect.top
            let elHeight = rect.height
            let viewHeight = window.innerHeight
            if (elTop < 0 || (elTop + elHeight) > viewHeight) {
                let viewTop = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || 0
                let center = viewTop + elTop + (elHeight / 2) - (viewHeight / 2)
                window.scrollTo({ top: center, left: 0, behavior: 'smooth' })
            }
        }
    },
    computed: {
        /*
         The style param is responsible for the color of the text based on different criteria
        */
        style: function () {
            let hashtagIds = []
            if (this.hashtags) {
                hashtagIds = Object.values(this.hashtags).map(h => h.id).sort();
            }

            if (!this.thread) {
                return 'fill: rgb(231, 76, 60); fill-opacity: 0.3; cursor: pointer;'
            }
            if (this.thread === this.threadSelected) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.3; cursor: pointer;'
            }
            if (this.threadsHovered.includes(this.thread)) {
                return 'fill: rgb(1, 99, 255); fill-opacity: 0.12; cursor: pointer;'
            }
            if (this.thread.spotlight && this.thread.spotlight.type === 'EM' && this.currentConfigs.isEmphasize) {
                let color = this.thread.spotlight.color? this.thread.spotlight.color : 'lime'
                return `stroke: ${color}; fill: ${color}; fill-opacity: 0.3; stroke-opacity: 0.9; stroke-dasharray: 1,1; stroke-width: 2px; cursor: pointer;`
            }
            if (this.showTypingActivityAnimation) { // if typing, show a pink outline color
                // return 'stroke: rgb(255, 0, 255); stroke-width: 25'
                return
            }
            // if (this.showRecentActivityAnimation) { // if recently shown, show a cyan outline color
            //     // return 'stroke: rgb(0, 255, 255); stroke-width: 15'
            //     return
            // }
            // if (this.unseenNotificationThread) {
            //     return 'fill: rgb(80, 54, 255); opacity: 0.7;'
            //     // return 'stroke: rgb(80, 54, 255); stroke-width: 8; stroke-opacity: 0.2;'
            // }
            // if (this.replyRequestThread) {
            //     if (this.thread.isUnseen() && this.currentConfigs.isShowIndicatorForUnseenThread) {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.25;'
            //         return 'fill: rgb(255, 0, 255); opacity: 1.0;'
            //     } else {
            //         // return 'stroke: rgb(255, 0, 255); stroke-width: 8; stroke-opacity: 0.10;'
            //         return 'fill: rgb(255, 0, 255); opacity: 0.5;'
            //     }
            // }
            /*
             The reducer function gets 2 arguments previousValue(the paint accumulated so far) and hashtag(some valid hashtag),
             The reducer checks if the current thread has this hashtag, in case it does have it. it calculate new paint for the thread(based on the previous value and the color of the new hashtag)
            */
            const reducer = (hashtag, previousValue) => { 
                if (this.thread.hasHashtag(hashtag)){
                    let currentValue = RgbList[hashtagIds.indexOf(hashtag)]
                    if (previousValue == null){
                        return currentValue
                    }
                    else{
                        return {Red: currentValue['Red']+previousValue['Red']/2,
                        Green: currentValue['Green']+previousValue['Green'] / 2,
                        Blue: currentValue['Blue']+previousValue['Blue'] / 2 }
                    };
                }
                else{
                    return previousValue
                }
            } 
            /*
             This condition checks if the user selected to present the emoji heatmap
             In case the current thread has more then one hashtag, we use the reducer to evaluate the thread color.
            */
            if (this.heatmapMode === "Emoji"){
                if (this.thread.hashtags.length > 0) {
                    let prev = null
                    for (const hashtag of hashtagIds){
                        prev = reducer(hashtag, prev)
                    }
                    let resRGB = prev
                    return `fill: rgb(${resRGB['Red']}, ${resRGB['Green']}, ${resRGB['Blue']}); fill-opacity: 0.2; cursor: pointer;`
                }   
                else {
                    return 'fill: rgb(255, 255, 255); opacity: 0.1;'
                }
            }
            if(this.heatmapMode === "CE"){
                return 'fill: rgb(90, 240, 90); fill-opacity: 0.2;'
            }
            return null
        },
        isRecentThread: function () {
            return this.thread && this.recent && this.showSyncFeatures
        },
        isTypingThread: function () {
            return this.thread && this.thread.usersTyping && this.thread.usersTyping.length > 0 && this.showSyncFeatures
        },
        showRecentActivityAnimation: function () {
            return false
            if (this.thread && ( (this.thread === this.threadSelected) || this.threadsHovered.includes(this.thread))) { // if typing or hover, don't animate
                return false
            }
            return this.thread && this.recent && this.showSyncFeatures
        },
        showTypingActivityAnimation: function () {
            return false
            if (this.thread && ( (this.thread === this.threadSelected) || this.threadsHovered.includes(this.thread))) { // if typing or hover, don't animate
                return false
            }
            return this.thread && this.thread.usersTyping && this.thread.usersTyping.length > 0 && this.showSyncFeatures
        },
        replyRequestThread: function () {
            return this.showSyncFeatures && this.thread && this.thread.hasReplyRequests()
        },
        unseenNotificationThread: function () {
            return this.thread && this.thread.associatedNotification !== null && this.showSyncFeatures && this.thread.associatedNotification.unseen
        },
        bounds: function () {
            let bounds = {}
            if (this.thread) {
                bounds.boxes = getTextBoundingBoxes(this.thread.range.toRange())
            } else {
                bounds.boxes = getTextBoundingBoxes(this.range.toRange())
            }
            bounds.offsetX = window.pageXOffset || document.documentElement.scrollLeft || document.body.scrollLeft || 0
            bounds.offsetY = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || 0
            return bounds
        },
        visible: function () {
            return this.showHighlights || (this.thread === this.threadSelected)
        }
    },
    methods: {
        onHover: function (state) {
            this.$emit(state ? 'hover-thread' : 'unhover-thread', this.thread)
        },
        onClick: function () {
            if (!this.thread) {
                return this.$emit('select-thread', this.thread, 'NONE')
            }

            let type = 'HIGHLIGHT'
            
            if ((this.currentConfigs.isEmphasize && this.thread.spotlight && this.thread.spotlight.type === 'EM') || (this.currentConfigs.isInnotation && this.thread.spotlight && this.thread.spotlight.type === 'IN')) {
                type = this.thread.spotlight.type.toUpperCase()
            }

            const location = this.currentConfigs.isEmphasize && this.thread.spotlight && this.thread.spotlight.type === 'EM' ? 'SPOTLIGHT' : 'HIGHLIGHT'
            this.$emit('log-nb', 'CLICK', location, this.thread.spotlight ? this.thread.spotlight.type.toUpperCase() : 'NONE',  this.thread.isSync, this.thread.hasSync, this.thread.associatedNotification ? this.thread.associatedNotification.trigger : 'NONE', this.thread.id, this.thread.countAllReplies())

            const source = window.location.pathname === '/nb_viewer.html' ? window.location.href : window.location.origin + window.location.pathname
            const token = localStorage.getItem("nb.user");
            const config = { headers: { Authorization: 'Bearer ' + token }, params: { url: source } }
            axios.post(`/api/spotlights/log`, {
                spotlight_id: type === 'HIGHLIGHT' ? null : this.thread.spotlight.id,
                action: 'CLICK', 
                type: type, 
                annotation_id: this.thread.id, 
                class_id: this.activeClass.id,
                role: this.user.role.toUpperCase()
            }, config)

            this.logNbClick()

            if (this.currentConfigs.isEmphasize && this.thread.spotlight && (this.thread.spotlight.type === 'EM' || this.thread.spotlight.type === 'IN')) {
                this.$emit('select-thread', this.thread, 'SPOTLIGHT')
            } else {
                this.$emit('select-thread', this.thread, 'HIGHLIGHT')
            }
        },
        logNbClick: function () {
            if (this.unseenNotificationThread || this.isTypingThread || this.isRecentThread || this.showTypingActivityAnimation) {
                let trigger_type = ''
                if (this.isTypingThread || this.isRecentThread) {
                    trigger_type = 'USER_SAW_RECENT_ACTIVITY'
                } else if (this.unseenNotificationThread) {
                    trigger_type = this.thread.associatedNotification.trigger 
                } else {
                    trigger_type = 'REPLY_REQUESTED'
                }
                // console.log(trigger_type)
                const source = window.location.pathname === '/nb_viewer.html' ? window.location.href : window.location.origin + window.location.pathname
                const token = localStorage.getItem("nb.user");
                const config = { headers: { Authorization: 'Bearer ' + token }, params: { url: source } }
                axios.post(`/api/spotlights/log`, {
                    spotlight_id: null,
                    action: 'CLICK', 
                    type: 'NOTIFICATION_HIGHLIGHT', 
                    annotation_id: this.thread.id, 
                    class_id: this.activeClass.id,
                    role: this.user.role.toUpperCase(),
                    trigger: trigger_type
                }, config)
            }
        },
        getTooltipContent: function () {
            if (!this.thread || !this.showSyncFeatures) {
                return ""
            }
            let content = ""
            if (this.isRecentThread || this.isTypingThread) {
                content = "<span>recent comment:</span>"
            } else if (this.unseenNotificationThread) {
                content = "<span>" + 
                this.thread.associatedNotification.readableType + " notification:</span>"
            } else if (this.replyRequestThread) {
                content = "<span>reply requested comment:</span>"
            } else {
                return "" // no associated notification, return empty string
            }
            content += "<br>"

            let relevantComment = 
                (this.unseenNotificationThread && this.thread.associatedNotification.specificAnnotation !== null) 
                ? this.thread.associatedNotification.specificAnnotation : this.thread 

            let text = relevantComment.text
            content += text.substring(0, 30)
            if (text.length > 30) {
                content += "..."
            }
            return content
        }
    }
}
</script>