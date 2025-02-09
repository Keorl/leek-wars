<template lang="html">
	<div class="chat">
		<loader v-show="loading" />
		<div v-if="!loading && (!chat || !chat.messages.length)" v-autostopscroll class="messages">
			<div class="no-messages">{{ $t('main.no_messages_yet') }}</div>
		</div>
		<div v-if="chat && chat.messages.length" ref="messages" v-autostopscroll class="messages" @scroll="scroll">
			<template v-for="(message, m) in $store.state.chat[channel].messages">
				<div v-if="message.author.id === -1" :key="m" class="separator">
					{{ message.time | date }}
				</div>
				<div v-else-if="message.author.id === 0" :key="m" class="message">
					<img class="avatar" src="/image/favicon.png">
					<router-link :to="'/fight/' + message.texts[0].split('|')[1]">
						<div class="bubble br-notification">
							<div class="author">Leek Wars</div>
							{{ $t(message.texts[0].split('|')[0]) }}
						</div>
					</router-link>
				</div>
				<div v-else :key="m" class="message">
					<router-link :to="'/farmer/' + message.author.id" class="avatar-wrapper">
						<rich-tooltip-farmer :id="message.author.id" v-slot="{ on }">
							<avatar :farmer="message.author" :on="on" />
						</rich-tooltip-farmer>
					</router-link>
					<div class="bubble">
						<router-link :to="'/farmer/' + message.author.id" class="author">
							<rich-tooltip-farmer :id="message.author.id" v-slot="{ on }">
								<span :class="message.author.grade" v-on="on">{{ message.author.name }}</span>
							</rich-tooltip-farmer>
						</router-link>
						<div v-for="(text, i) in message.texts" :key="i" v-large-emojis v-chat-code-latex class="text" v-html="text"></div>
						<div class="right">
							<span :title="LeekWars.formatDateTime(message.time)" class="time">{{ LeekWars.formatTime(message.time) }}</span>
							<v-menu v-if="!privateMessages && !($store.state.farmer && message.author.id === $store.state.farmer.id)" offset-y>
								<template v-slot:activator="{ on }">
									<v-btn text small icon color="grey" v-on="on">
										<v-icon>mdi-dots-vertical</v-icon>
									</v-btn>
								</template>
								<v-list dense class="message-actions">
									<v-list-item v-ripple @click="report(message)">
										<v-icon>mdi-flag</v-icon>
										<span>Signaler</span>
									</v-list-item>
									<v-list-item v-if="$store.getters.moderator && !message.author.muted" v-ripple @click="mute(message.author)">
										<v-icon>mdi-volume-off</v-icon>
										<span>Mute</span>
									</v-list-item>
									<v-list-item v-if="$store.getters.moderator && message.author.muted" v-ripple @click="unmute(message.author)">
										<v-icon>mdi-volume-high</v-icon>
										<span>Unmute</span>
									</v-list-item>
								</v-list>
							</v-menu>
						</div>
					</div>
				</div>
			</template>
			<div v-show="unread" v-ripple class="chat-new-messages" @click="updateScroll(true)">{{ $t('main.unread_messages') }}</div>
		</div>
		<div v-if="$store.state.wsdisconnected" class="chat-disconnected">{{ $t('main.disconnected') }}</div>
		<chat-input @message="sendMessage" />

		<report-dialog v-if="reportFarmer" v-model="reportDialog" :target="reportFarmer" :reasons="reasons" :parameter="reportContent" class="report-dialog" />

		<popup v-model="muteDialog" :width="600">
			<v-icon slot="icon">mdi-volume-off</v-icon>
			<span slot="title">{{ $t('warning.mute') }}</span>
			<template v-if="muteFarmer">
				<i18n path="moderation.mute_popup">
					<b slot="farmer">{{ muteFarmer.name }}</b>
				</i18n>
			</template>
			<div slot="actions">
				<div v-ripple @click="muteDialog = false">{{ $t('main.cancel') }}</div>
				<div v-ripple class="mute red" @click="muteConfirm">{{ $t('warning.confirm_mute') }}</div>
			</div>
		</popup>
		<popup v-model="unmuteDialog" :width="600">
			<v-icon slot="icon">mdi-volume-high</v-icon>
			<span slot="title">{{ $t('warning.unmute') }}</span>
			<div v-if="muteFarmer">
				<i18n path="moderation.unmute_popup">
					<b slot="farmer">{{ muteFarmer.name }}</b>
				</i18n>
			</div>
			<div slot="actions">
				<div v-ripple @click="unmuteDialog = false">{{ $t('main.cancel') }}</div>
				<div v-ripple class="unmute red" @click="unmuteConfirm">{{ $t('warning.unmute') }}</div>
			</div>
		</popup>
	</div>
</template>

<script lang="ts">
	import { Chat, ChatMessage } from '@/model/chat'
	import { Farmer } from '@/model/farmer'
	import { LeekWars } from '@/model/leekwars'
	import { Warning } from '@/model/moderation'
	import { SocketMessage } from '@/model/socket'
	import { store } from '@/model/store'
	import { Component, Prop, Vue, Watch } from 'vue-property-decorator'
	import ChatInput from './chat-input.vue'

	@Component({
		components: { 'chat-input': ChatInput }
	})
	export default class ChatElement extends Vue {
		@Prop() channel!: string
		muteDialog: boolean = false
		unmuteDialog: boolean = false
		muteFarmer: Farmer | null = null
		reportDialog: boolean = false
		reportFarmer: Farmer | null = null
		reportContent: string = ''
		userScroll: boolean = false
		unread: boolean = false
		reasons = [
			Warning.INCORRECT_FARMER_NAME,
			Warning.INCORRECT_AVATAR,
			Warning.FLOOD_CHAT,
			Warning.RUDE_CHAT,
			Warning.PROMO_CHAT
		]
		get loading() {
			return !!this.channel && !store.state.chat[this.channel]
		}
		get chat() {
			return this.channel ? store.state.chat[this.channel] : null
		}
		get privateMessages() {
			return this.channel && this.channel.startsWith('pm-')
		}

		created() {
			this.$root.$on('chat', this.newMessage)
			this.$root.$on('resize', this.updateScroll)
			this.$root.$on('wsconnected', this.update)
		}

		beforeDestroy() {
			this.$root.$off('chat', this.newMessage)
			this.$root.$off('resize', this.updateScroll)
			this.$root.$off('wsconnected', this.update)
		}

		newMessage(e: any) {
			if (e[0] === this.channel) {
				this.updateScroll()
				if (!this.scrollBottom()) { this.unread = true }

				// On reçoit un message sur un chat de conversation privée, il est lu tout de suite
				this.read()
			}
		}

		scrollBottom() {
			const messages = this.$refs.messages as HTMLElement
			if (!messages) { return true }
			return messages && Math.abs((messages.scrollTop + messages.offsetHeight) - messages.scrollHeight) < 3
		}

		mounted() {
			this.updateScroll()
		}

		scroll() {
			if (this.scrollBottom()) {
				this.userScroll = false
				this.unread = false
			} else {
				this.userScroll = true
			}
		}

		updated() {
			this.updateScroll()
		}

		updateScroll(force: boolean = false) {
			if (!this.userScroll || force) {
				const messages = this.$refs.messages as HTMLElement
				setTimeout(() => {
					if (messages) {
						messages.scrollTop = messages.scrollHeight + 1000
						this.unread = false
					}
				}, 60)
			}
		}

		@Watch('channel', {immediate: true})
		update() {
			if (!this.channel) { return }
			if (this.channel === 'team') {
				if (!this.$store.state.chat.team) {
					this.$store.commit('init-team-chat')
					LeekWars.socket.send([SocketMessage.TEAM_CHAT_ENABLE_FAST])
				}
			} else if (this.channel.startsWith('pm-')) {
				const id = parseInt(this.channel.replace('pm-', ''), 10)
				if (!this.$store.state.chat['pm-' + id] || this.$store.state.chat['pm-' + id].messages.length < 50) {
					if (id === 0) { return }
					LeekWars.get('message/get-messages/' + id + '/' + 50 + '/' + 1).then(data => {
						this.$store.commit('clear-chat', 'pm-' + id)
						for (const message of data.messages.reverse()) {
							this.$store.commit('pm-receive', {message: [id, message.farmer_id, message.farmer_name, message.content, false, message.farmer_color, message.avatar_changed, message.date]})
						}
						for (const farmer of data.farmers) {
							this.$store.commit('add-conversation-participant', {id, farmer})
						}
					})
				}
				this.read()
			} else {
				if (this.channel in this.$store.state.chat && this.$store.state.chat[this.channel].invalidated) {
					this.$store.commit('clear-chat', this.channel)
				}
				LeekWars.socket.enableChannel(this.channel)
			}
		}

		sendMessage(message: any) {
			if (message.startsWith('/ping')) {
				this.$store.commit('last-ping', Date.now())
			}
			if (this.channel === 'team') {
				LeekWars.socket.send([SocketMessage.TEAM_CHAT_SEND, message])
			} else if (!this.channel || this.privateMessages) {
				this.$emit('send', message)
			} else {
				LeekWars.socket.send([SocketMessage.FORUM_CHAT_SEND, this.channel, message])
			}
		}

		report(message: ChatMessage) {
			this.reportDialog = true
			this.reportFarmer = message.author
			this.reportContent = message.texts.reduce((a, b) => a + b + "\n", "")
		}

		mute(farmer: Farmer) {
			this.muteDialog = true
			this.muteFarmer = farmer
		}

		muteConfirm() {
			if (!this.muteFarmer) { return }
			LeekWars.socket.send([SocketMessage.CHAT_REQUEST_MUTE, this.channel, this.muteFarmer.id])
			this.muteFarmer.muted = true
			this.muteDialog = false
		}

		unmute(farmer: Farmer) {
			this.unmuteDialog = true
			this.muteFarmer = farmer
		}

		unmuteConfirm() {
			if (!this.muteFarmer) { return }
			LeekWars.socket.send([SocketMessage.CHAT_REQUEST_UNMUTE, this.channel, this.muteFarmer.id])
			this.muteFarmer.muted = false
			this.unmuteDialog = false
		}

		read() {
			if (this.chat && this.chat.conversation) {
				LeekWars.socket.send([SocketMessage.MP_READ, this.chat.conversation.id])
				this.chat.conversation.unread = false
			}
		}
		// TODO
		// ChatController.prototype.mute_user = function(data) {
		// 	var moderator_name = data[2]
		// 	var muted = data[3]
		// 	this.msg_elem.find('.chat-message[author=' + muted + ']').find('.chat-message-messages').html('censuré par <b>' + moderator_name + '</b>')
		// }
	}
</script>

<style lang="scss" scoped>
	.chat {
		position: relative;
		display: flex;
		flex-direction: column;
	}
	.loader {
		height: calc(100% - 100px);
		display: flex;
		align-items: center;
		justify-content: center;
	}
	.messages {
		height: calc(100% - 40px);
		overflow-y: scroll;
	}
	.message {
		display: flex;
		align-items: flex-start;
		margin: 6px 8px;
		color: #aaa;
	}
	.avatar-wrapper {
		position: sticky;
		top: 8px;
	}
	.avatar {
		width: 42px;
		height: 42px;
		flex: 0 0 42px;
	}
	.bubble {
		margin-left: 8px;
		padding: 3px 7px;
		border-radius: 4px;
		background: white;
		position: relative;
		box-shadow: 0px 2px 1px -1px rgba(0,0,0,0.1), 0px 1px 1px 0px rgba(0,0,0,0.07), 0px 1px 3px 0px rgba(0,0,0,0.06);
		min-width: 0;
		.report, .mute, .unmute {
			display: none;
			cursor: pointer;
		}
	}
	.bubble:hover {
		.report, .mute, .unmute {
			display: inline;
		}
	}
	.author {
		font-weight: 500;
		display: block;
		padding-bottom: 2px;
		padding-right: 60px;
		color: #777;
	}
	.text {
		word-break: break-word;
		color: #333;
	}
	.text.large-emojis ::v-deep .emoji {
		font-size: 26px;
		line-height: 34px;
		width: 30px;
		height: 30px;
	}
	.right {
		font-size: 13px;
		position: absolute;
		top: 3px;
		right: 0;
		.time {
			padding-right: 4px;
		}
		.v-btn {
			margin: 0;
			margin-top: -3px;
			margin-left: -5px;
			height: 24px;
			width: 24px;
		}
		i.v-icon {
			font-size: 18px;
			color: #aaa;
		}
	}
	.message-actions .v-icon {
		margin-right: 4px;
	}
	.text ::v-deep a {
		color: #5fad1b;
	}
	.text ::v-deep .v-icon {
		color: #5fad1b;
		font-size: 18px;
		margin-right: 4px;
		vertical-align: baseline;
	}
	.br-notification {
		background: #5fad1b;
		color: white;
		display: inline-block;
		.author {
			color: white;
		}
	}
	.no-messages {
		height: calc(100% - 60px);
		display: flex;
		align-items: center;
		justify-content: center;
		color: #aaa;
	}
	.chat-new-messages {
		position: absolute;
		bottom: 39px;
		right: 0;
		left: 0;
		background: #5fad1b;
		color: white;
		text-align: center;
		line-height: 30px;
		cursor: pointer;
	}
	.chat-disconnected {
		height: 30px;
		line-height: 30px;
		background-color: #d3324d;
		color: white;
		text-align: center;
		transition: height ease 0.5s;
		position: absolute;
		top: 0;
		width: 100%;
	}
	.separator {
		text-align: center;
		color: #777;
		display: flex;
		align-items: center;
		margin: 12px 0;
		&:before, &:after {
			border-bottom: 1px dashed #aaa;
			width: 100%;
			content: " ";
			flex: 1;
		}
		&:before {
			margin-right: 10px;
		}
		&:after {
			margin-left: 10px;
		}
	}
</style>